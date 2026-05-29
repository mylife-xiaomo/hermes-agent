# Hermes Agent 详细架构设计文档

> 版本：基于 main 分支 (commit 6fdbf2f2)
> 日期：2026-04-25

---

## 目录

1. [系统概述](#1-系统概述)
2. [总体架构](#2-总体架构)
3. [入口点与启动流程](#3-入口点与启动流程)
4. [核心代理循环（AIAgent）](#4-核心代理循环aiagent)
5. [工具系统](#5-工具系统)
6. [记忆系统](#6-记忆系统)
7. [技能系统](#7-技能系统)
8. [消息网关系统](#8-消息网关系统)
9. [插件系统](#9-插件系统)
10. [环境执行后端](#10-环境执行后端)
11. [CLI 与 TUI 系统](#11-cli-与-tui-系统)
12. [ACP 适配器](#12-acp-适配器)
13. [定时任务系统](#13-定时任务系统)
14. [上下文压缩与提示缓存](#14-上下文压缩与提示缓存)
15. [多模型适配](#15-多模型适配)
16. [关键方法调用流程](#16-关键方法调用流程)

---

## 1. 系统概述

Hermes Agent 是一个**自我改进的 AI 代理**，由 Nous Research 开发。其核心特性包括：

- **从经验中创建技能**：代理可自主创建、改进技能文档
- **使用中自我改进**：通过记忆系统从交互中学习
- **跨会话记忆**：持久化的长期记忆与用户画像
- **多平台消息网关**：支持 Telegram、Discord、Slack、WhatsApp、Signal、飞书、企业微信等 15+ 平台
- **多模型支持**：OpenAI、Anthropic、Gemini、Bedrock、OpenRouter 等数十种模型提供商
- **可插拔扩展**：插件系统 + MCP 工具协议

技术栈：Python 3.11+，MIT 许可证。

---

## 2. 总体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          用户交互层                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────────────┐   │
│  │  CLI/TUI  │  │  Gateway  │  │   ACP    │  │  API Server       │   │
│  │ (hermes)  │  │ (15+平台) │  │ (IDE集成) │  │ (OpenAI兼容)      │   │
│  └─────┬─────┘  └─────┬────┘  └─────┬────┘  └────────┬──────────┘   │
│        │              │              │                 │              │
│        └──────────────┴──────────────┴─────────────────┘              │
│                              │                                        │
│                    ┌─────────▼──────────┐                             │
│                    │    AIAgent 核心     │  ← run_agent.py (12000+行)  │
│                    │  (代理循环引擎)      │                             │
│                    └──┬──────┬──────┬───┘                             │
│                       │      │      │                                  │
│        ┌──────────────┤      │      ├──────────────┐                  │
│        ▼              ▼      │      ▼              ▼                  │
│  ┌──────────┐  ┌──────────┐  │  ┌──────────┐  ┌──────────┐          │
│  │ 提示构建  │  │ 记忆管理  │  │  │ 技能系统  │  │ 上下文压缩│          │
│  │ prompt_  │  │ memory_   │  │  │ skill_*  │  │ context_  │          │
│  │ builder  │  │ manager   │  │  │          │  │ compressor│          │
│  └──────────┘  └──────────┘  │  └──────────┘  └──────────┘          │
│                               │                                       │
│                    ┌──────────▼──────────┐                            │
│                    │    工具编排层         │                            │
│                    │   model_tools.py     │                            │
│                    └──────────┬──────────┘                            │
│                               │                                       │
│          ┌────────────────────┼────────────────────┐                  │
│          ▼                    ▼                     ▼                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐            │
│  │ 工具注册表    │  │ 工具集管理    │  │ 插件/MCP工具发现   │            │
│  │ registry.py  │  │ toolsets.py  │  │ plugins/mcp_tool │            │
│  └──────┬───────┘  └──────────────┘  └──────────────────┘            │
│         │                                                              │
│         ▼                                                              │
│  ┌─────────────────────────────────────────────────────┐              │
│  │              工具实现层 (tools/)                       │              │
│  │  40+ 自注册工具：web、terminal、file、browser、vision、│              │
│  │  skills、memory、cron、delegate_task、execute_code...  │              │
│  └─────────────────────────────────────────────────────┘              │
│                                                                        │
│  ┌─────────────────────────────────────────────────────┐              │
│  │              环境执行后端 (environments/)               │              │
│  │  Local | Docker | SSH | Modal | Daytona | Singularity │             │
│  └─────────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. 入口点与启动流程

### 3.1 入口点总览

| 命令 | 入口文件 | 用途 |
|------|---------|------|
| `hermes` | `hermes_cli/main.py:main` | 交互式 CLI |
| `hermes-agent` | `run_agent.py:main` | 核心代理运行器 |
| `hermes-acp` | `acp_adapter/entry.py:main` | Agent Client Protocol 适配器 |

### 3.2 CLI 启动流程

```
hermes_cli/main.py:main()
  ├── 解析命令行参数 (argparse)
  ├── 加载配置 (~/.hermes/config.yaml)
  ├── 初始化 AIAgent
  │     ├── 设置 model, provider, api_key, base_url
  │     ├── 调用 discover_builtin_tools() 触发工具自注册
  │     ├── 调用 discover_mcp_tools() 发现 MCP 工具
  │     ├── 调用 discover_plugins() 发现插件
  │     ├── 调用 get_tool_definitions() 获取启用的工具 Schema
  │     ├── 初始化 MemoryManager (记忆管理器)
  │     └── 初始化 TodoStore, MemoryStore, SessionDB
  ├── cmd_chat() / cmd_model() / cmd_gateway() ...
  │     ├── TUI 模式：_launch_tui() → Node.js 子进程
  │     └── 传统 REPL：prompt_toolkit 循环
  └── 用户消息 → agent.run_conversation()
```

### 3.3 ACP 启动流程

```
acp_adapter/entry.py:main()
  ├── 加载 ~/.hermes/.env
  ├── 创建 HermesACPAgent (acp_adapter/server.py)
  │     ├── 使用 ThreadPoolExecutor 包装同步 AIAgent
  │     ├── 实现 ACP JSON-RPC 协议
  │     └── SessionManager 管理会话
  └── 启动 JSON-RPC 监听 (stdin/stdout)
```

---

## 4. 核心代理循环（AIAgent）

**核心文件**: `run_agent.py` — `AIAgent` 类 (L680-11956)

### 4.1 AIAgent 初始化

`__init__()` 方法 (L698-1856) 接收 50+ 参数，核心参数包括：

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | str | 模型名称，默认 `anthropic/claude-opus-4.6` |
| `max_iterations` | int | 最大工具调用迭代次数，默认 90 |
| `enabled_toolsets` | List[str] | 启用的工具集 |
| `platform` | str | 平台标识 (cli/telegram/discord...) |
| `session_id` | str | 会话 ID |
| `stream_delta_callback` | callable | 流式 token 回调 |
| `status_callback` | callable | 状态通知回调 |
| `iteration_budget` | IterationBudget | 迭代预算管理 |

初始化流程关键步骤：
1. 创建 OpenAI 兼容客户端 (`_create_openai_client`)
2. 获取工具 Schema (`get_tool_definitions`)
3. 构建系统提示 (`_build_system_prompt`)
4. 初始化记忆管理器 (`MemoryManager`)
5. 初始化会话数据库 (`SessionDB`)

### 4.2 run_conversation 主循环

**核心方法**: `run_conversation()` (L8630-11942) — 代理的完整对话循环

```
run_conversation(user_message, system_message, conversation_history, task_id, ...)
│
├── [初始化阶段]
│   ├── 安装安全 I/O 包装器 (_install_safe_stdio)
│   ├── 设置会话日志上下文
│   ├── 恢复主运行时 (_restore_primary_runtime)
│   ├── 清理用户输入 (surrogate/内存标签)
│   ├── 生成 task_id (UUID)
│   ├── 重置重试计数器
│   ├── 连接健康检查 (_cleanup_dead_connections)
│   └── 初始化迭代预算 (IterationBudget)
│
├── [上下文构建阶段]
│   ├── 从历史恢复 Todo 状态 (_hydrate_todo_store)
│   ├── 追加用户消息到 messages
│   ├── 构建系统提示 (_build_system_prompt) — 首次构建后缓存
│   ├── 记忆提供者：on_turn_start() 通知
│   ├── 记忆提供者：prefetch_all() 预取上下文
│   └── 插件钩子：on_session_start / pre_llm_call
│
├── [代理循环] while api_call_count < max_iterations:
│   │
│   ├── 检查中断请求 (_interrupt_requested)
│   ├── 消费迭代预算 (iteration_budget.consume)
│   ├── 排空待处理的 /steer 文本 (_drain_pending_steer)
│   │
│   ├── [消息准备]
│   │   ├── 复制 messages → api_messages
│   │   ├── 注入外部记忆上下文到用户消息
│   │   ├── 注入插件上下文
│   │   ├── 拼接系统提示
│   │   ├── 应用 Anthropic 提示缓存策略
│   │   ├── 安全检查：孤立 tool_result、surrogate、非 ASCII
│   │   └── 估算 token 数
│   │
│   ├── [API 调用]
│   │   ├── 构建 API 参数 (_build_api_kwargs)
│   │   ├── 插件钩子：pre_api_request
│   │   ├── 流式路径：_interruptible_streaming_api_call()
│   │   │     ├── OpenAI Chat Completions 流
│   │   │     ├── Anthropic Messages 流
│   │   │     ├── Codex Responses 流
│   │   │   └── Bedrock 流
│   │   └── 非流式路径：_interruptible_api_call()
│   │
│   ├── [响应处理]
│   │   ├── 构建标准化 assistant 消息 (_build_assistant_message)
│   │   ├── 提取推理内容 (_extract_reasoning)
│   │   ├── 处理 finish_reason (stop / tool_calls / length)
│   │   └── 追加 assistant 消息到 messages
│   │
│   ├── [工具执行]
│   │   ├── finish_reason == "tool_calls" → 进入工具执行
│   │   ├── 判断是否可并行 (_should_parallelize_tool_batch)
│   │   ├── 并行路径：_execute_tool_calls_concurrent() (ThreadPoolExecutor)
│   │   ├── 串行路径：_execute_tool_calls_sequential()
│   │   └── 追加 tool result 消息到 messages
│   │
│   ├── [工具结果后处理]
│   │   ├── 应用 /steer 到工具结果
│   │   ├── 重试逻辑 (truncated / invalid JSON / empty content)
│   │   └── 上下文压缩检查 (_compress_context)
│   │
│   └── finish_reason == "stop" → 退出循环
│
├── [结束阶段]
│   ├── 处理最大迭代 (_handle_max_iterations)
│   ├── 持久化会话 (_persist_session)
│   ├── 记忆刷新 (flush_memories)
│   ├── 记忆提供者：sync_all()
│   ├── 插件钩子：on_session_end
│   └── 返回 {final_response, messages, api_calls, completed}
│
└── chat() — 简化包装，返回纯文本
```

### 4.3 迭代预算管理

`IterationBudget` 类 (L192-233) 提供线程安全的迭代计数：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self._max = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:   # 原子递增，返回是否还有余量
    def refund(self) -> None:    # 归还一次迭代（用于 execute_code）
```

### 4.4 工具执行策略

**并行执行判断** (`_should_parallelize_tool_batch`, L290-331)：
- 单个工具调用 → 串行
- 多个独立工具（无路径重叠）→ 并行
- 写操作 / delegate_task → 串行
- 路径重叠检测：通过 `_extract_parallel_scope_path` + `_paths_overlap` 避免并发写入冲突

**并行执行** (`_execute_tool_calls_concurrent`, L7779-8080)：
```python
# 使用 ThreadPoolExecutor 并行执行工具
with concurrent.futures.ThreadPoolExecutor(max_workers=len(tool_calls)) as pool:
    futures = {
        pool.submit(_run_tool, index, tool_call, ...): index
        for index, tool_call in enumerate(tool_calls)
    }
    for future in concurrent.futures.as_completed(futures):
        result = future.result()
        # 按原始顺序追加结果
```

---

## 5. 工具系统

### 5.1 工具注册表（ToolRegistry）

**文件**: `tools/registry.py`

`ToolRegistry` 是一个单例注册表，采用 **导入时自注册** 模式：

```python
class ToolEntry:  # L76-97
    """每个已注册工具的元数据"""
    name: str              # 工具名称
    toolset: str           # 所属工具集
    schema: dict           # OpenAI 格式的 JSON Schema
    handler: Callable      # 工具处理函数
    check_fn: Callable     # 可用性检查函数（如 API key 是否配置）
    requires_env: list     # 所需环境变量
    is_async: bool         # 是否异步
    max_result_size_chars: int  # 结果大小限制

class ToolRegistry:  # L100-433
    """线程安全的工具注册中心"""
    _tools: Dict[str, ToolEntry]
    _toolset_checks: Dict[str, Callable]
    _lock: threading.RLock  # 读写锁

    def register(self, name, toolset, schema, handler, check_fn=None, ...):
        """注册工具 —— 由各工具文件在 import 时调用"""

    def dispatch(self, name, args, **kwargs) -> str:
        """分发工具调用到对应 handler"""

    def get_definitions(self, tool_names, quiet=False) -> List[dict]:
        """获取工具 Schema（经 check_fn 过滤）"""
```

**线程安全机制**：通过 `_snapshot_state()` 返回注册表的一致快照，避免读写竞争：

```python
def _snapshot_state(self) -> tuple[List[ToolEntry], Dict[str, Callable]]:
    with self._lock:
        return list(self._tools.values()), dict(self._toolset_checks)
```

### 5.2 工具发现流程

```
model_tools.py 模块加载时:
  │
  ├── discover_builtin_tools()         # 扫描 tools/ 目录
  │     ├── AST 分析每个 .py 文件是否包含 registry.register() 调用
  │     ├── 按需 import 含有注册调用的模块
  │     └── 每个模块的顶层代码调用 registry.register()
  │
  ├── discover_mcp_tools()             # 发现 MCP 外部工具
  │     └── 读取 config.yaml 中的 mcp_servers 配置
  │
  └── discover_plugins()               # 发现插件工具
        ├── ~/.hermes/plugins/<name>/
        ├── .hermes/plugins/<name>/
        └── pip entry_points: hermes_agent.plugins
```

### 5.3 工具集管理（Toolsets）

**文件**: `toolsets.py`

工具集采用 **组合模式**，支持嵌套引用和递归解析：

```python
TOOLSETS = {
    "debugging": {
        "description": "Debugging toolkit",
        "tools": ["terminal", "process"],       # 直接包含的工具
        "includes": ["web", "file"]              # 递归包含的工具集
    },
    "hermes-gateway": {
        "description": "所有消息平台工具的并集",
        "tools": [],
        "includes": ["hermes-telegram", "hermes-discord", ...]  # 18个平台
    }
}

def resolve_toolset(name: str, visited: Set[str] = None) -> List[str]:
    """递归解析工具集，返回所有工具名列表"""
    # 支持 "all" / "*" 特殊别名
    # 环形依赖检测（visited 集合）
    # 钻石依赖去重
```

**核心工具列表** (`_HERMES_CORE_TOOLS`, L31-63) 定义了所有平台共享的工具集：

| 类别 | 工具 |
|------|------|
| Web | `web_search`, `web_extract` |
| 终端 | `terminal`, `process` |
| 文件 | `read_file`, `write_file`, `patch`, `search_files` |
| 视觉 | `vision_analyze`, `image_generate` |
| 技能 | `skills_list`, `skill_view`, `skill_manage` |
| 浏览器 | `browser_navigate`, `browser_snapshot`, `browser_click` 等 11 个 |
| 规划 | `todo`, `memory`, `session_search` |
| 代码执行 | `execute_code`, `delegate_task` |
| 消息 | `send_message` |
| 定时 | `cronjob` |
| 智能家居 | `ha_list_entities`, `ha_get_state`, `ha_list_services`, `ha_call_service` |

### 5.4 工具调度流程

```
handle_function_call(function_name, function_args, ...)
│
├── coerce_tool_args()           # 类型强制转换 ("42" → 42)
│
├── 检查是否代理循环工具 (_AGENT_LOOP_TOOLS)
│   └── todo / memory / session_search / delegate_task → 返回桩错误
│
├── 插件钩子：pre_tool_call
│   └── 可能返回 block_message → 拒绝执行
│
├── registry.dispatch(name, args)
│   ├── 获取 ToolEntry
│   ├── 异步工具 → _run_async(handler(**args))
│   └── 同步工具 → handler(**args)
│
├── 插件钩子：post_tool_call
│
├── 插件钩子：transform_tool_result（可修改工具输出）
│
└── 返回 JSON 字符串结果
```

### 5.5 Async 桥接机制

`_run_async()` (model_tools.py L81-131) 是所有异步工具的统一桥接入口：

```python
def _run_async(coro):
    """从同步上下文运行异步协程"""
    # 1. 已在异步上下文（gateway/RL）→ 新线程运行
    if loop and loop.is_running():
        return ThreadPoolExecutor.submit(asyncio.run, coro)

    # 2. 工作线程（并行工具执行）→ 线程本地持久化 event loop
    if not main_thread:
        return worker_loop.run_until_complete(coro)

    # 3. 主线程 → 共享持久化 event loop
    return tool_loop.run_until_complete(coro)
```

---

## 6. 记忆系统

### 6.1 三层记忆架构

```
┌─────────────────────────────────────────────────────┐
│                    记忆层次                           │
│                                                       │
│  ┌───────────────┐  ┌───────────────┐  ┌──────────┐ │
│  │   短期记忆     │  │   长期记忆     │  │ 技能记忆  │ │
│  │ (对话历史)     │  │ (MEMORY.md    │  │ (YAML    │ │
│  │  OpenAI 格式   │  │  + USER.md)  │  │  过程记忆) │ │
│  │  内存中        │  │  文件持久化    │  │ 跨会话    │ │
│  └───────────────┘  └───────────────┘  └──────────┘ │
│         │                  │                  │        │
│         └──────────┬───────┘                  │        │
│                    ▼                          │        │
│         ┌─────────────────────┐              │        │
│         │   MemoryManager     │              │        │
│         │  (多提供者编排)      │              │        │
│         └──────────┬──────────┘              │        │
│                    │                          │        │
│         ┌──────────▼──────────┐              │        │
│         │  MemoryProvider     │              │        │
│         │  (抽象基类)          │              │        │
│         └──────────┬──────────┘              │        │
│                    │                          │        │
│     ┌──────────────┼──────────────┐          │        │
│     ▼              ▼              ▼          │        │
│  内置提供者     Honcho         Mem0 /        │        │
│  (文件)       (外部API)      Hindsight      │        │
└─────────────────────────────────────────────────────┘
```

### 6.2 MemoryManager

**文件**: `agent/memory_manager.py`

```python
class MemoryManager:
    """多记忆提供者的编排器"""

    def add_provider(self, provider: MemoryProvider):
        """注册记忆提供者"""

    def prefetch_all(self, query: str) -> str:
        """在每轮开始前从所有提供者预取上下文"""

    def sync_all(self, messages: list):
        """在每轮结束后同步记忆到所有提供者"""

    def on_turn_start(self, turn_count: int, user_message: str):
        """通知提供者新轮次开始"""

    def build_system_prompt(self) -> str:
        """构建注入系统提示的记忆内容"""
```

### 6.3 MemoryStore（内置记忆）

**文件**: `tools/memory_tool.py`

```python
class MemoryStore:
    """文件支持的内存存储"""

    def add(self, key: str, content: str):
        """添加记忆条目"""

    def replace(self, key: str, content: str):
        """替换记忆条目"""

    def remove(self, key: str):
        """删除记忆条目"""

    def format_for_system_prompt(self) -> str:
        """返回冻结快照用于系统提示"""
```

持久化路径：
- `~/.hermes/memories/MEMORY.md` — 代理的长期记忆
- `~/.hermes/memories/USER.md` — 用户画像

### 6.4 会话数据库

使用 **SQLite + FTS5 全文搜索** 存储会话历史：

```
~/.hermes/sessions/<session_id>.db
  ├── messages 表：消息内容、角色、时间戳
  ├── FTS5 索引：全文搜索支持
  └── metadata：会话元数据
```

---

## 7. 技能系统

### 7.1 渐进式披露架构

```
┌──────────────────────────────────────────────┐
│  Tier 1：索引层                               │
│  系统提示中的技能名和描述列表                    │
│  → skills_list 工具                           │
├──────────────────────────────────────────────┤
│  Tier 2：内容层                               │
│  按需加载完整技能内容                           │
│  → skill_view(name) 工具                      │
├──────────────────────────────────────────────┤
│  Tier 3：子文件层                              │
│  访问技能附带的资源文件                          │
│  → skill_view(name, file_path) 工具            │
└──────────────────────────────────────────────┘
```

### 7.2 SKILL.md 格式

技能以 **目录 + SKILL.md** 文件形式存储在 `~/.hermes/skills/`：

```yaml
---
name: skill-name
description: "技能描述"
platforms: [macos, linux]          # 平台兼容性
metadata:
  hermes:
    tags: [tag1, tag2]
    config:
      - key: wiki.path
        description: "Wiki 路径"
        default: "~/wiki"
---

# 技能内容（Markdown）

支持模板变量：
- ${HERMES_SKILL_DIR}  — 技能目录路径
- ${HERMES_SESSION_ID} — 当前会话 ID

支持内联命令：
- `!command` — 执行 shell 命令并嵌入输出
```

### 7.3 技能加载流程

**文件**: `agent/skill_commands.py`, `agent/skill_utils.py`

```
skill_commands.py:
  ├── scan_skill_commands()      # 构建命令→技能映射
  ├── _load_skill_payload()      # 加载技能内容
  └── _build_skill_message()     # 组装技能注入消息

skill_utils.py:
  ├── discover_skills()          # 扫描技能目录
  ├── parse_skill_metadata()     # 解析 SKILL.md 元数据
  └── check_platform_compat()    # 平台兼容性检查
```

### 7.4 技能缓存机制

`prompt_builder.py` 中的 `_build_skills_manifest()` / `_load_skills_snapshot()` 实现磁盘缓存：

```
~/.hermes/skills/.prompt_snapshot.json
  ├── manifest: {文件路径: [mtime, size]}  # 用于增量检测
  ├── skill_entries: [...]                 # 技能元数据
  └── category_descriptions: {...}         # 分类描述
```

---

## 8. 消息网关系统

### 8.1 网关架构

```
┌────────────────────────────────────────────────────┐
│                   Gateway Daemon                    │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │          Base Platform Adapter                  │ │
│  │          (gateway/platforms/base.py)            │ │
│  │  ┌──────────────┐  ┌─────────────────────────┐ │ │
│  │  │ 消息接收      │  │ 消息格式化/发送          │ │ │
│  │  │ on_message()  │  │ send_message()          │ │ │
│  │  └──────┬───────┘  └──────────┬──────────────┘ │ │
│  └─────────┼─────────────────────┼────────────────┘ │
│            │                     │                    │
│            ▼                     │                    │
│  ┌─────────────────────┐        │                    │
│  │  AIAgent 实例        │        │                    │
│  │  (每会话独立)         │        │                    │
│  │  run_conversation()  │───────►│                    │
│  └─────────────────────┘        │                    │
│                                  │                    │
│  ┌───────────────────────────────┘                    │
│  │                                                     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │  │ Telegram  │ │ Discord  │ │  Slack   │ ...       │
│  │  └──────────┘ └──────────┘ └──────────┘           │
│  │                                                     │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐           │
│  │  │ WhatsApp  │ │ Signal   │ │  飞书    │ ...       │
│  │  └──────────┘ └──────────┘ └──────────┘           │
│  └────────────────────────────────────────────────────┘
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │  SessionContext (线程安全的会话变量)                │ │
│  │  set_session_vars() / get_session_env()         │ │
│  └─────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

### 8.2 消息处理流程

```
平台消息到达
  │
  ├── 平台适配器接收 (on_message)
  ├── SessionContext 设置会话变量
  ├── 创建/复用 AIAgent 实例
  │     ├── gateway 模式：每消息创建新 AIAgent
  │     └── 从会话数据库恢复历史
  │
  ├── agent.run_conversation(message)
  │     ├── 流式回调 → 实时发送部分响应
  │     └── 完成后发送完整响应
  │
  ├── 平台适配器格式化响应
  │     ├── 长消息分段 (Telegram UTF-16 长度限制)
  │     ├── Markdown 渲染
  │     └── 媒体附件处理
  │
  └── 通过平台 API 发送响应
```

### 8.3 支持的平台

15+ 消息平台：Telegram、Discord、Slack、WhatsApp、Signal、BlueBubbles (iMessage)、Home Assistant、Email、SMS (Twilio)、Mattermost、Matrix、钉钉、飞书、企业微信、微信公众号、QQ Bot、Webhook。

---

## 9. 插件系统

### 9.1 插件发现

**文件**: `hermes_cli/plugins.py`

插件来源（按优先级）：
1. **项目插件**：`./.hermes/plugins/<name>/`
2. **用户插件**：`~/.hermes/plugins/<name>/`
3. **捆绑插件**：`<repo>/plugins/<name>/`
4. **Pip 插件**：`hermes_agent.plugins` 入口点

### 9.2 插件钩子系统

```python
# 可用的生命周期钩子
hooks = {
    "on_session_start",     # 新会话创建
    "on_session_end",       # 会话结束
    "pre_llm_call",         # LLM 调用前（可注入上下文）
    "post_llm_call",        # LLM 调用后
    "pre_api_request",      # API 请求发送前
    "pre_tool_call",        # 工具执行前（可阻止执行）
    "post_tool_call",       # 工具执行后
    "transform_tool_result",# 工具结果变换
}
```

### 9.3 插件能力

```python
class PluginContext:
    def register_tool(self, name, schema, handler, toolset, ...):
        """注册自定义工具到注册表"""

    def register_toolset_alias(self, alias, toolset):
        """注册工具集别名"""

    def register_hook(self, hook_name, callback):
        """注册生命周期钩子"""
```

---

## 10. 环境执行后端

**文件**: `tools/environments/base.py`

### 10.1 抽象接口

```python
class BaseEnvironment(ABC):
    """环境执行后端的抽象接口"""

    @abstractmethod
    def execute(self, command: str, cwd: str = None) -> Tuple[int, str, str]:
        """执行命令，返回 (exit_code, stdout, stderr)"""

    @abstractmethod
    def spawn(self) -> "BaseEnvironment":
        """创建新的环境实例"""

    def cleanup(self):
        """清理资源"""
```

### 10.2 六种实现

| 实现 | 文件 | 说明 |
|------|------|------|
| Local | `environments/local.py` | 本地执行（默认） |
| Docker | `environments/docker.py` | Docker 容器隔离 |
| SSH | `environments/ssh.py` | 远程 SSH 执行 |
| Modal | `environments/modal.py` | Modal 云执行 |
| Daytona | `environments/daytona.py` | Daytona 开发环境 |
| Singularity | `environments/singularity.py` | Singularity 容器 |

**统一模型**：所有实现都采用 **spawn-per-call** 模型——每次命令执行都是独立的 `bash -c` 进程，通过 in-band 标记或临时文件持久化工作目录。

---

## 11. CLI 与 TUI 系统

### 11.1 CLI 架构

**文件**: `hermes_cli/main.py` (6600+ 行)

```
hermes_cli/main.py:main()
  ├── argparse 子命令系统
  ├── cmd_chat()        # 交互式对话
  ├── cmd_model()       # 模型选择
  ├── cmd_profile()     # 配置管理
  ├── cmd_gateway()     # 启动消息网关
  ├── cmd_acp()         # ACP 服务器模式
  ├── cmd_sessions()    # 会话管理
  └── cmd_config()      # 配置查看/编辑
```

### 11.2 TUI 模式

两种 UI 模式：
1. **传统 REPL**：基于 `prompt_toolkit` 的命令行界面
2. **TUI 模式**：Node.js 子进程（`ui-tui/`），通过 JSON-RPC 与 Python 后端通信

```
TUI 架构:
  Node.js 前端 ←→ JSON-RPC (stdin/stdout) ←→ Python 后端
                    │
                    ├── tui_gateway/entry.py (桥接层)
                    └── AIAgent (核心引擎)
```

---

## 12. ACP 适配器

**文件**: `acp_adapter/entry.py`, `acp_adapter/server.py`

Agent Client Protocol (ACP) 是一种 JSON-RPC 协议，用于 IDE 集成（VS Code、Zed、JetBrains）：

```python
class HermesACPAgent:
    """ACP 服务器实现"""
    - 使用 ThreadPoolExecutor 运行同步 AIAgent
    - 实现 ACP JSON-RPC 端点：
      - initialize / authenticate
      - list_sessions / create_session
      - send_message / cancel
    - stdout 仅用于 JSON-RPC（日志走 stderr）
```

---

## 13. 定时任务系统

**文件**: `cron/`

```python
# 核心功能
create_job(expression, prompt, ...)   # 创建定时任务
list_jobs()                           # 列出所有任务
remove_job(job_id)                    # 删除任务
tick()                                # 调度器心跳（每 60 秒调用一次）
```

任务存储：文件持久化（`JOBS_FILE`）
调度执行：由 gateway daemon 每 60 秒调用 `tick()` 触发
特性：支持 cron 表达式、间隔、一次性任务、自调度能力、隔离会话执行。

---

## 14. 上下文压缩与提示缓存

### 14.1 上下文压缩

当对话历史接近 token 限制时，`_compress_context()` (L7531-7631) 执行压缩：

```
_compress_context(messages, system_message)
│
├── 估算 token 数
├── 调用 LLM 生成对话摘要
├── 在 SQLite 中分割会话
│     ├── 旧消息 → 归档
│     └── 摘要 → 新会话起始
├── 重建系统提示
└── 返回压缩后的 messages
```

### 14.2 Anthropic 提示缓存

`_anthropic_prompt_cache_policy()` (L2384-2456) 和 `apply_anthropic_cache_control()` 实现自动缓存：

```
策略：
  ├── 系统 + 前 3 条消息 → cache_control breakpoint
  ├── 降低输入 token 成本约 75%
  ├── 自动检测：Anthropic / OpenRouter / 兼容端点
  └── 支持 cache_ttl 配置
```

### 14.3 系统提示构建与缓存

`_build_system_prompt()` (L4057-4222) 和 `build_context_files_prompt()` (prompt_builder.py)：

```
_build_system_prompt()
│
├── 基础系统提示（角色定义、行为规范）
├── 加载 SOUL.md (~/.hermes/SOUL.md) — 个性化
├── 加载 .hermes.md / HERMES.md — 项目级
├── 加载 AGENTS.md — 代理指令
├── 加载 CLAUDE.md — Claude 兼容
├── 加载 .cursorrules — Cursor 兼容
├── 加载上下文文件 (context files)
├── 构建技能清单 (skills manifest)
├── 注入环境提示 (build_environment_hints)
├── 注入 Nous 订阅能力
├── 注入记忆内容
└── 返回完整系统提示字符串
```

**缓存策略**：系统提示在首次调用后缓存到 `_cached_system_prompt`，仅在上下文压缩后失效重建。

---

## 15. 多模型适配

### 15.1 支持的提供商

| 提供商 | API 模式 | 说明 |
|--------|---------|------|
| OpenAI | `chat_completions` | GPT-4o, o1, o3 等 |
| Anthropic | `anthropic_messages` | Claude Opus, Sonnet, Haiku |
| Codex | `codex_responses` | OpenAI Responses API |
| Bedrock | `chat_completions` | AWS Bedrock 上的模型 |
| OpenRouter | `chat_completions` | 多模型路由 |
| Ollama | `chat_completions` | 本地模型 |
| Gemini | `chat_completions` | Google Gemini |
| Qwen Portal | `chat_completions` | 通义千问 |
| Nous Portal | `codex_responses` | Nous Research |

### 15.2 流式 API 调用

`_interruptible_streaming_api_call()` (L5480-6293) 是统一的流式调用入口：

```
_interruptible_streaming_api_call(api_kwargs)
│
├── 检测 provider 和 api_mode
│
├── OpenAI Chat Completions 流
│   └── _call_chat_completions() (L5585-5821)
│       ├── client.chat.completions.create(stream=True)
│       ├── 逐 chunk 处理 delta
│       ├── 触发 _fire_stream_delta() 回调
│       └── 收集完整响应
│
├── Anthropic Messages 流
│   └── _call_anthropic() (L5823-5878)
│       ├── anthropic_client.messages.stream()
│       ├── 处理 text / tool_use / thinking blocks
│       └── 收集完整响应
│
├── Codex Responses 流
│   └── _run_codex_stream() (L4782-4903)
│       ├── client.responses.create(stream=True)
│       └── 处理 Responses API 事件
│
└── Bedrock 流
    └── 通过 OpenAI 兼容层处理
```

### 15.3 Provider 回退机制

`_try_activate_fallback()` (L6297-6470) 和 `_restore_primary_runtime()` (L6474-6546)：

```
API 调用失败
  │
  ├── 429 Rate Limit → 尝试回退
  ├── 5xx Server Error → 尝试回退
  ├── Auth Error → 尝试凭据刷新
  │   ├── _try_refresh_codex_client_credentials()
  │   ├── _try_refresh_nous_client_credentials()
  │   └── _try_refresh_anthropic_client_credentials()
  │
  ├── _recover_with_credential_pool() → 从凭据池切换
  │
  └── _try_activate_fallback() → 切换到回退模型
        ├── 使用 fallback_model 配置
        ├── 替换 model / provider / api_key / base_url
        └── 标记 _fallback_activated = True
```

---

## 16. 关键方法调用流程

### 16.1 完整的用户消息处理流程

```
用户输入 "帮我搜索最新的 AI 新闻"
  │
  ▼
hermes_cli/main.py: cmd_chat()
  │
  ├── 解析斜杠命令（如果是 /skill 形式）
  ├── 应用技能内容（如果匹配）
  │
  ▼
run_agent.py: AIAgent.run_conversation(user_message)
  │
  ├── [1] 输入清理 (surrogate, memory标签)
  ├── [2] 迭代预算初始化
  ├── [3] 系统提示构建/缓存复用
  ├── [4] 记忆预取 (prefetch_all)
  │
  ├── [5] 代理循环 ─────────────────────────────────────────
  │       │
  │       ├── [5.1] 消息准备
  │       │     ├── 复制 messages → api_messages
  │       │     ├── 注入记忆上下文到用户消息
  │       │     ├── 拼接系统提示
  │       │     ├── Anthropic 缓存控制
  │       │     └── surrogate/ASCII 清理
  │       │
  │       ├── [5.2] API 调用
  │       │     ├── _build_api_kwargs(api_messages)
  │       │     │     ├── 选择 api_mode
  │       │     │     ├── 构建 messages / tools / parameters
  │       │     │     └── 注入 reasoning_config
  │       │     │
  │       │     └── _interruptible_streaming_api_call(api_kwargs)
  │       │           ├── 流式输出 → stream_delta_callback
  │       │           ├── 推理输出 → reasoning_callback
  │       │           └── 工具生成 → tool_gen_callback
  │       │
  │       ├── [5.3] 响应解析
  │       │     ├── _build_assistant_message()
  │       │     │     ├── 提取 text content
  │       │     │     ├── 提取 tool_calls
  │       │     │     ├── 提取 reasoning_content
  │       │     │     └── 判断 finish_reason
  │       │     │
  │       │     └── 追加到 messages
  │       │
  │       ├── [5.4] 工具执行 (if finish_reason == "tool_calls")
  │       │     │
  │       │     ├── 遍历 tool_calls
  │       │     │
  │       │     ├── 对每个 tool_call:
  │       │     │     ├── _repair_tool_call() — 修复名称/参数
  │       │     │     ├── _invoke_tool(name, args)
  │       │     │     │     ├── pre_tool_call 插件钩子
  │       │     │     │     ├── model_tools.handle_function_call()
  │       │     │     │     │     ├── coerce_tool_args()
  │       │     │     │     │     ├── registry.dispatch()
  │       │     │     │     │     │     ├── 查找 ToolEntry
  │       │     │     │     │     │     ├── check_fn 检查
  │       │     │     │     │     │     ├── 执行 handler (sync/async)
  │       │     │     │     │     │     └── 截断结果到 max_result_size_chars
  │       │     │     │     │     ├── post_tool_call 插件钩子
  │       │     │     │     │     └── transform_tool_result 插件钩子
  │       │     │     │     └── 返回 JSON 结果
  │       │     │     │
  │       │     │     └── 构造 tool role 消息追加到 messages
  │       │     │
  │       │     └── 继续循环 → [5.1]
  │       │
  │       └── [5.5] 最终响应 (finish_reason == "stop")
  │             ├── 提取 final_response 文本
  │             └── 退出循环
  │
  ├── [6] 结束处理
  │     ├── 记忆刷新 (flush_memories)
  │     ├── 记忆同步 (sync_all)
  │     ├── 会话持久化 (_persist_session → SQLite + JSON log)
  │     ├── 插件钩子: on_session_end
  │     └── 返回 {final_response, messages, api_calls, completed}
  │
  ▼
hermes_cli: 显示最终响应给用户
```

### 16.2 工具注册流程

```
# 每个工具文件的模式 (如 tools/web_search_tool.py)

from tools.registry import registry

# 模块顶层代码 — import 时自动执行
registry.register(
    name="web_search",
    toolset="web",
    schema={                          # OpenAI function calling schema
        "name": "web_search",
        "description": "Search the web...",
        "parameters": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "..."}
            },
            "required": ["query"]
        }
    },
    handler=handle_web_search,        # 实际执行函数
    check_fn=lambda: bool(os.getenv("BRAVE_API_KEY")),  # 可用性检查
    emoji="🔍",
)

def handle_web_search(query: str, **kwargs) -> str:
    """实际执行 web 搜索"""
    # ... 实现 ...
    return json.dumps({"results": [...]})
```

### 16.3 上下文文件发现流程

```
build_context_files_prompt(cwd)
│
├── _load_hermes_md(cwd)    # .hermes.md / HERMES.md — 递归查找到 git root
├── _load_agents_md(cwd)    # AGENTS.md — 仅顶层
├── _load_claude_md(cwd)    # CLAUDE.md — 仅当前目录
├── _load_cursorrules(cwd)  # .cursorrules + .cursor/rules/*.mdc
├── load_soul_md()           # ~/.hermes/SOUL.md — 个性化
│
├── 扫描上下文目录中的文件
│   ├── ~/.hermes/context/   # 全局上下文
│   └── .hermes/context/     # 项目上下文
│
├── _scan_context_content()  # 安全扫描注入内容
│
└── 拼接所有上下文内容
```

---

## 附录 A：目录结构总览

```
hermes-agent/
├── run_agent.py          # AIAgent 核心类 (12000+ 行)
├── model_tools.py        # 工具编排层
├── toolsets.py           # 工具集定义与解析
├── cli.py                # 传统 CLI 入口
│
├── agent/                # 核心代理模块
│   ├── prompt_builder.py     # 系统提示构建 (1100 行)
│   ├── memory_manager.py     # 记忆管理器
│   ├── memory_provider.py    # 记忆提供者抽象
│   ├── skill_commands.py     # 技能命令处理
│   ├── skill_utils.py        # 技能工具函数
│   └── ...                   # 模型元数据、错误分类等
│
├── tools/                # 工具实现 (40+ 工具)
│   ├── registry.py           # 工具注册表
│   ├── budget_config.py      # 结果大小限制
│   ├── mcp_tool.py           # MCP 外部工具适配
│   ├── environments/         # 执行环境后端
│   │   ├── base.py               # 抽象接口
│   │   ├── local.py              # 本地执行
│   │   ├── docker.py             # Docker 容器
│   │   └── ...                   # SSH, Modal, Daytona, Singularity
│   └── ...                   # 各工具实现文件
│
├── hermes_cli/           # CLI 界面
│   ├── main.py               # CLI 主入口 (6600+ 行)
│   ├── plugins.py            # 插件系统
│   ├── config.py             # 配置管理
│   └── completion.py         # Shell 补全
│
├── gateway/               # 消息网关
│   ├── platforms/             # 平台适配器
│   │   ├── base.py               # 基类
│   │   ├── telegram.py           # Telegram
│   │   ├── discord.py            # Discord
│   │   └── ...                   # Slack, WhatsApp, Signal 等
│   └── session_context.py    # 会话上下文管理
│
├── skills/                # 技能系统
│   └── ...                   # 内置技能
│
├── plugins/               # 可插拔扩展
│   └── ...                   # 内置插件
│
├── acp_adapter/           # Agent Client Protocol 适配器
│   ├── entry.py              # ACP 入口
│   └── server.py             # ACP 服务器
│
├── cron/                  # 定时任务
├── tui_gateway/           # TUI 网关桥接
├── environments/          # RL 训练环境
└── tests/                 # 测试套件
    ├── tools/                 # 工具测试
    ├── conftest.py            # 共享 fixture
    ├── integration/           # 集成测试
    └── e2e/                   # 端到端测试
```

---

## 附录 B：配置体系

```
配置优先级（从高到低）：
  1. 环境变量
  2. ~/.hermes/.env          # API 密钥
  3. ~/.hermes/config.yaml   # 主配置
  4. 项目级 .hermes/         # 项目特定配置
  5. 默认值                   # 代码中的默认值

关键配置项：
  - model / provider          # 模型选择
  - enabled_toolsets          # 启用的工具集
  - terminal.dangerous        # 危险命令审批
  - memory.nudge_interval     # 记忆提醒间隔
  - mcp_servers               # MCP 服务器配置
  - hooks                     # 生命周期钩子
  - providers_order           # OpenRouter 提供商顺序
```
