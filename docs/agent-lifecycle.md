# Hermes Agent 全生命周期源码分析

> 基于 `run_agent.py`（~12000 行）的完整生命周期解读。

## 概述

`run_agent.py` 是 Hermes Agent 的核心引擎。`AIAgent` 类管理从创建到销毁的完整生命周期，支持多模型提供商、工具调用循环、上下文压缩、provider fallback 等生产级特性。

一个 Agent 实例经历以下 6 个阶段：

```
实例化(__init__) → 系统提示构建 → 对话主循环 → 工具执行 → 响应完成 → 会话收尾
```

---

## 阶段 1: 实例化 (`__init__`, L698-1856)

```
AIAgent(base_url=..., model=..., api_key=...)
```

这一阶段完成 **所有运行时依赖的装配**，约 1150 行初始化逻辑。

### 1.1 环境与安全基础

```python
# L804: 包装 stdout/stderr，防止管道断裂崩溃
_install_safe_stdio()

# L48-63: 加载环境变量
load_hermes_dotenv(hermes_home=_hermes_home, project_env=_project_env)

# L837-873: 自动检测 api_mode
# 根据 provider 和 URL 自动路由到四种 API 模式：
#   - chat_completions      (标准 OpenAI 兼容)
#   - codex_responses       (OpenAI Responses API)
#   - anthropic_messages    (Anthropic 原生协议)
#   - bedrock_converse      (AWS Bedrock)
```

### 1.2 LLM 客户端构建

```python
# L1095-1275: 根据 api_mode 分四条路径构建客户端
if api_mode == "anthropic_messages":
    # Anthropic SDK（原生或 Bedrock）
    self._anthropic_client = build_anthropic_client(effective_key, base_url)
elif api_mode == "bedrock_converse":
    # boto3 直接调用
    self.client = None  # 不需要 OpenAI 客户端
else:
    # codex_responses 或 chat_completions
    # 使用 OpenAI SDK，支持 OpenRouter、Copilot、Qwen 等多种 provider
    self.client = self._create_openai_client(client_kwargs, reason="agent_init")
```

Provider fallback 链（L1281-1300）：主 provider 失败时自动切换备选 provider。

### 1.3 工具系统加载

```python
# L1303-1323: 获取所有可用工具 schema
self.tools = get_tool_definitions(
    enabled_toolsets=enabled_toolsets,
    disabled_toolsets=disabled_toolsets,
)

# L1310-1312: 构建校验集合
self.valid_tool_names = {tool["function"]["name"] for tool in self.tools}

# L1521-1535: 注入 memory provider 的工具 schema
for schema in self._memory_manager.get_all_tool_schemas():
    self.tools.append({"type": "function", "function": schema})
```

### 1.4 子系统初始化

| 子系统 | 位置 | 作用 |
|--------|------|------|
| `CheckpointManager` | L1375-1379 | 文件系统快照/回滚 |
| `TodoStore` | L1411-1412 | 任务列表管理 |
| `MemoryStore` | L1433-1448 | 持久记忆（MEMORY.md + USER.md） |
| 外部 Memory Provider | L1455-1513 | Honcho 等外部记忆后端 |
| `ContextCompressor` | L1563-1729 | 上下文自动压缩引擎 |
| `SubdirectoryHintTracker` | L1769-1771 | 工作目录追踪 |
| `IterationBudget` | L806-810 | 线程安全迭代计数器 |

### 1.5 校验与快照

```python
# L1732-1743: 拒绝 context window < 64K 的模型
if context_length and context_length < MINIMUM_CONTEXT_LENGTH:
    raise ValueError("Model context window too small")

# L1831-1856: 快照主运行时，用于 fallback 恢复
self._primary_runtime = {
    "model": self.model,
    "provider": self.provider,
    "base_url": self.base_url,
    "api_mode": self.api_mode,
    "client_kwargs": dict(self._client_kwargs),
    ...
}
```

---

## 阶段 2: 系统提示构建 (`_build_system_prompt`, L4057-4222)

每个 session 首次调用时构建，之后缓存到 `_cached_system_prompt`。

### 提示组成结构

```
┌─────────────────────────────────────────────┐
│ [Agent Identity]                            │  ← DEFAULT_AGENT_IDENTITY
│ [Platform Hints]                            │  ← CLI/Telegram/Discord 格式提示
│ [Memory Guidance]                           │  ← 如何使用 memory 工具
│ [Session Search Guidance]                   │  ← 如何搜索历史会话
│ [Skills Guidance]                           │  ← 技能系统使用说明
│ [Tool Use Enforcement]                      │  ← 某些模型强制工具调用
│ [Context Files]                             │  ← SOUL.md / AGENTS.md / .cursorrules
│ [Memory Context Block]                      │  ← 外部 memory provider 预取内容
│ [Tool List]                                 │  ← 可用工具名和描述摘要
└─────────────────────────────────────────────┘
```

### 缓存策略

```python
# L8804-8843: 跨 turn 缓存
if self._cached_system_prompt is None:
    if stored_prompt:
        # 继续会话 — 复用上一 turn 的系统提示（Anthropic 前缀缓存命中）
        self._cached_system_prompt = stored_prompt
    else:
        # 新会话 — 从头构建
        self._cached_system_prompt = self._build_system_prompt(system_message)
```

只在 **上下文压缩** 时通过 `_invalidate_system_prompt()` 重建。

**关键设计**：系统提示跨 turn 不变 → Anthropic prompt caching 前缀完全一致 → 缓存命中率接近 100%，输入成本降低 ~75%。

---

## 阶段 3: 对话主循环 (`run_conversation`, L8630-11942)

这是 Agent 的核心运行时。

### 入口签名

```python
def run_conversation(
    self,
    user_message: str,
    system_message: str = None,
    conversation_history: List[Dict[str, Any]] = None,
    task_id: str = None,
    stream_callback: Optional[callable] = None,
    persist_user_message: Optional[str] = None,
) -> Dict[str, Any]:
```

### 3.1 预处理（L8658-8997）

```
L8670   恢复主 provider（上一 turn 用了 fallback 时）
L8675   输入净化（surrogate 字符、memory-context 标签）
L8852   预飞压缩 — 历史已超阈值时先压缩再开始
L8926   插件 pre_llm_call 钩子（注入额外上下文）
L8979   外部 memory provider 预取
```

### 3.2 核心循环结构（L8999-11749）

```python
while (api_call_count < max_iterations and iteration_budget.remaining > 0) \
      or budget_grace_call:

    # 1. 预算检查 & interrupt 检查
    if self._interrupt_requested:
        break
    if not self.iteration_budget.consume():
        break

    # 2. 构建 API 消息
    api_messages = [系统提示] + [对话历史] + [工具 schema]
    #    - 注入 memory 预取上下文到 user message
    #    - 应用 Anthropic 缓存标记
    #    - 净化 surrogate 字符

    # 3. 调用 LLM（内层重试循环）
    while retry_count < max_retries:
        # Nous rate limit 守卫
        # 流式优先调用（_interruptible_streaming_api_call）
        # 响应验证
        # 错误分类 & fallback 切换

    # 4. 响应归一化（通过 transport 层）
    normalized = transport.normalize_response(response)

    # 5. 分支处理
    if assistant_message.tool_calls:
        # → 阶段 4: 工具执行
    else:
        # → 阶段 5: 响应完成
```

### 3.3 错误恢复层级

主循环有三层错误恢复：

```
Layer 1: 单次 API 调用重试（max_retries=3，带抖动退避）
    ↓ 全部失败
Layer 2: Provider fallback（切换到备选 provider）
    ↓ 无可用 fallback
Layer 3: 上下文压缩（压缩历史消息后重试）
    ↓ 压缩后仍然失败
Layer 4: 返回错误，持久化会话状态
```

---

## 阶段 4: 工具执行

工具执行有两条路径，由 `_should_parallelize_tool_batch()` 决定。

### 4.1 并行执行 (`_execute_tool_calls_concurrent`, L7779-8080)

```python
# L241-253: 只读安全工具列表
_PARALLEL_SAFE_TOOLS = frozenset({
    "ha_get_state", "read_file", "search_files", "session_search",
    "skill_view", "skills_list", "vision_analyze", "web_extract", "web_search",
})

# L238: 永远不能并行的交互式工具
_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})

# L256: 按路径分区的文件工具（路径不冲突时可并行）
_PATH_SCOPED_TOOLS = frozenset({"read_file", "write_file", "patch"})
```

使用 `ThreadPoolExecutor`，最多 8 个 worker。

### 4.2 串行执行 (`_execute_tool_calls_sequential`, L8082-8463)

交互式工具或路径冲突时回退到串行。每个工具调用流程：

```
安全检查 → handle_function_call(function_name, function_args) → 结果追加到 messages
```

### 4.3 工具调用后处理

```python
# L11338-11343: execute_code 调用退还迭代预算
if tool_names == {"execute_code"}:
    self.iteration_budget.refund()

# L11359-11380: 自动上下文压缩
if compression_enabled and compressor.should_compress(real_tokens):
    messages, system_prompt = self._compress_context(messages, ...)

# L11382-11384: 增量保存 session log
self._save_session_log(messages)
```

### 4.4 工具名校验与修复

```python
# L11102-11149: 三层校验
# 1. 工具名修复（_repair_tool_call）— 模糊匹配纠正拼写错误
# 2. 无效工具名重试（最多 3 次）— 返回错误让模型自纠
# 3. JSON 参数修复（_repair_tool_call_arguments）— 修复截断/尾逗号等
```

---

## 阶段 5: 响应完成（L11389-11749）

当模型返回无 `tool_calls` 的消息时，对话结束。

### 空响应多层 fallback

```
尝试 1: 流式部分恢复（已推送给用户的内容）
    ↓ 无已推送内容
尝试 2: 前一轮 housekeeping 工具旁的内容
    ↓ 非纯 housekeeping 工具
尝试 3: Post-tool 空 response nudge（追加提示让模型继续）
    ↓ 仍然空
尝试 4: Thinking prefill 重试（注入 <think\> 标签引导推理）
    ↓ 仍然空
最终: 返回 None 作为 final_response
```

### 迭代预算耗尽处理

```python
# L11747-11760: _handle_max_iterations()
# 向模型注入一条消息："你已用完所有迭代预算，请立即总结当前进度"
# 给模型最后一次 API 调用机会（grace call）
```

---

## 阶段 6: 会话收尾（L11762-11942）

```
┌─ 保存 trajectory ──────────────────────── L11767
│   将对话轨迹保存为 JSONL 文件（可选）
│
├─ 清理资源 ────────────────────────────── L11770
│   释放 VM（Docker/SSH）和 Browser 实例
│
├─ 持久化 session ──────────────────────── L11773
│   双写：JSON log + SQLite session store
│
├─ 诊断日志 ────────────────────────────── L11775-11817
│   记录退出原因、token 统计、工具调用次数
│   "just stops" 场景（agent 中途停工）特殊标记
│
├─ 插件钩子 ────────────────────────────── L11823-11840
│   post_llm_call: 持久化对话数据
│   on_session_end: 清理/刷新缓冲区
│
├─ 构建返回值 ──────────────────────────── L11846-11870
│   {
│     "final_response": str,
│     "messages": list,
│     "completed": bool,
│     "interrupted": bool,
│     "api_calls": int,
│     "input_tokens": int,
│     "output_tokens": int,
│     "estimated_cost_usd": float,
│     ...
│   }
│
├─ Memory provider 同步 ───────────────── L11900-11905
│   sync_all(): 同步本 turn 对话到外部记忆
│   queue_prefetch_all(): 预取下一 turn 上下文
│
├─ 后台记忆/技能审查 ───────────────────── L11909-11917
│   _spawn_background_review()
│   在响应交付后异步运行，不阻塞用户
│
└─ 清理中断状态 ────────────────────────── L11884
    clear_interrupt()
```

---

## 关键设计模式

### 1. 迭代预算 (`IterationBudget`, L192-233)

```python
class IterationBudget:
    """线程安全的迭代计数器"""

    def consume(self) -> bool:
        """消费一次迭代，返回是否允许"""
        with self._lock:
            if self._used >= self.max_total:
                return False
            self._used += 1
            return True

    def refund(self) -> None:
        """退还一次迭代（execute_code 等 RPC 调用不消耗预算）"""
```

- 父 agent 和子 agent 各自独立预算
- `execute_code` 调用自动退还（不消耗配额）
- 预算耗尽时给模型一次 grace call 总结进度

### 2. Provider Fallback (L6297-6546)

```
主 provider → 失败 → 切换 fallback provider → 本 turn 使用 fallback
    ↓ 下一 turn 开始
_restore_primary_runtime() → 恢复主 provider 重试
```

### 3. 自动上下文压缩 (L11359-11380)

```
每次工具执行后：
  估算当前 token 数
  → 超过阈值（默认 50% context window）
    → _compress_context()
      → 用辅助模型总结中间对话
      → 替换 messages 列表
      → 重建系统提示
      → 在 SQLite 中创建新 session（保留旧 session 的完整性）
```

### 4. Prompt Caching (L8804-8843, L9188-9193)

```
System Prompt: ────────────┐ cache breakpoint
Message 1: ────────────────┤ cache breakpoint
Message 2: ────────────────┤ cache breakpoint
Message 3: ────────────────┘ cache breakpoint
```

- `system_and_3` 策略：4 个断点（系统提示 + 最近 3 条消息）
- 系统提示跨 turn 不变 → 缓存前缀完全一致
- 只在上下文压缩时重建

### 5. 并行工具执行 (L7779-8080)

```
判断是否可并行：
  ├─ 只读安全工具 → 并行
  ├─ 文件工具（路径不冲突）→ 并行
  ├─ 交互式工具（clarify）→ 串行
  └─ 路径冲突 → 串行

执行：ThreadPoolExecutor(max_workers=8)
```

### 6. /steer 机制 (L9060-9108, L3708-3758)

不中断当前循环，将用户追加指令注入 tool result：

```
用户发送 /steer "请也检查错误处理"
    ↓
_steer() 设置 _pending_steer
    ↓
下一轮 API 调用前：_drain_pending_steer()
    ↓
追加到最后一条 tool result 的 content 末尾
    ↓
模型在下一次迭代中看到引导文本
```

### 7. 中断机制 (L3607-3706)

```python
def interrupt(self, message=None):
    """外部触发中断"""
    self._interrupt_requested = True
    # 设置线程级中断信号（影响当前线程的工具执行）
    _set_interrupt(True, self._execution_thread_id)
    # 传播到所有活跃子 agent
    for child in self._active_children:
        child.interrupt(message)
```

循环每轮检查 interrupt flag，退避等待中也每 200ms 检查一次。

---

## 调用入口

### CLI 方式

```python
# run_agent.py:main() → L11959
agent = AIAgent(base_url=..., model=..., ...)
result = agent.run_conversation(user_query)
```

### 编程方式

```python
# 简单接口
agent = AIAgent(base_url="http://localhost:30000/v1", model="claude-opus-4.6")
response = agent.chat("Tell me about Python updates")

# 完整接口
result = agent.run_conversation(
    user_message="...",
    conversation_history=[...],
    stream_callback=my_callback,
)
```

### CLI TUI / Gateway 方式

```python
# hermes_cli/main.py → 创建 AIAgent 并注入 TUI 回调
agent = AIAgent(
    tool_progress_callback=...,
    thinking_callback=...,
    stream_delta_callback=...,
    clarify_callback=...,
    platform="cli",  # 或 "telegram", "discord" 等
)
```

---

## 生命周期流程图

```
                    ┌──────────────────────┐
                    │   AIAgent.__init__()  │
                    │  环境加载、客户端构建  │
                    │  工具加载、子系统初始化 │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  _build_system_prompt │
                    │  首次构建后缓存       │
                    └──────────┬───────────┘
                               │
              ┌────────────────▼────────────────┐
              │      run_conversation()          │
              │                                  │
              │  ┌─── 预处理 ─────────────────┐  │
              │  │ 净化输入、预飞压缩、         │  │
              │  │ 插件钩子、memory 预取       │  │
              │  └─────────┬──────────────────┘  │
              │            │                      │
              │  ┌─────────▼──────────────────┐  │
              │  │     主循环 (while)          │  │
              │  │                             │  │
              │  │  ┌─── 构建 API 消息 ──────┐ │  │
              │  │  │ 系统提示 + 历史 + 工具 │ │  │
              │  │  │ 缓存标记 + 净化        │ │  │
              │  │  └──────────┬─────────────┘ │  │
              │  │             │                │  │
              │  │  ┌──────────▼─────────────┐ │  │
              │  │  │  调用 LLM（带重试）     │ │  │
              │  │  │  流式优先、错误恢复     │ │  │
              │  │  └──────────┬─────────────┘ │  │
              │  │             │                │  │
              │  │     ┌───────▼────────┐       │  │
              │  │     │ 有 tool_calls? │       │  │
              │  │     └───┬───────┬────┘       │  │
              │  │         │       │            │  │
              │  │    Yes  │       │  No        │  │
              │  │  ┌──────▼──┐  ┌─▼─────────┐ │  │
              │  │  │ 工具执行 │  │ 响应完成   │ │  │
              │  │  │ 并行/串行│  │ 空 resp    │ │  │
              │  │  │ 压缩检查│  │ fallback   │ │  │
              │  │  └────┬────┘  └─────┬──────┘ │  │
              │  │       │             │        │  │
              │  │       │             │        │  │
              │  │  continue    break (exit)    │  │
              │  └─────────────────────────────┘  │
              │                                    │
              │  ┌─── 会话收尾 ─────────────────┐  │
              │  │ 保存 trajectory、清理资源     │  │
              │  │ 持久化 session、诊断日志      │  │
              │  │ Memory 同步、后台审查         │  │
              │  └──────────┬───────────────────┘  │
              │             │                      │
              └─────────────▼──────────────────────┘
                            │
                 ┌──────────▼───────────┐
                 │    返回 result dict    │
                 │ {final_response,       │
                 │  messages, completed,  │
                 │  token_stats, cost}    │
                 └──────────────────────┘
```
