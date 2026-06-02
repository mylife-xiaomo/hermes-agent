# Hermes Agent 项目说明文档

## 📋 目录

- [项目概述](#项目概述)
- [核心特性](#核心特性)
- [快速开始](#快速开始)
- [使用方法](#使用方法)
- [架构设计](#架构设计)
- [配置文件](#配置文件)
- [开发指南](#开发指南)
- [应用场景](#应用场景)

---

## 项目概述

**Hermes Agent** 是由 Nous Research 开发的自学习 AI 智能体系统。它是目前唯一内置学习循环的智能体，能够从经验中创建技能、在使用过程中优化技能、跨会话构建用户画像模型，并搜索自己的历史对话。

### 核心理念

- **自我改进**：自动从复杂任务中学习并创建可复用的技能
- **平台无关**：可在 $5 VPS、GPU 集群或无服务器基础设施上运行
- **多平台交互**：通过 Telegram、Discord、Slack、WhatsApp 等与代理对话
- **模型自由**：支持任何 LLM 提供商，无供应商锁定

---

## 核心特性

### 1. 🧠 闭环学习系统

- **自主技能创建**：完成复杂任务后自动生成技能
- **技能自我优化**：在使用过程中持续改进技能
- **持久化记忆**：跨会话的用户建模和知识保存
- **FTS5 全文搜索**：支持 LLM 摘要的历史对话检索
- **Honcho 方言用户建模**：深度理解用户偏好和工作模式

### 2. 💬 多平台消息网关

支持 15+ 通讯平台，所有平台共享同一个代理进程：

| 分类 | 平台 |
|------|------|
| 即时通讯 | Telegram、Discord、Slack、WhatsApp、Signal、Matrix |
| 中国平台 | 微信、飞书、钉钉、企业微信、QQ Bot |
| 其他 | Email、SMS、Webhook、Home Assistant、Mattermost、BlueBubbles |

**特色功能**：
- 语音备忘录转录
- 跨平台对话连续性
- 统一的命令集（斜杠命令在所有平台通用）

### 3. 🎯 灵活的模型支持

支持 200+ 模型，随时切换无需修改代码：

- **Nous Portal** - 一站式订阅（模型 + Web 搜索 + 图像生成 + TTS + 云浏览器）
- **OpenRouter** - 200+ 模型聚合
- **NovitaAI** - AI 原生云平台
- **NVIDIA NIM** - Nemotron 系列模型
- **小米 MiMo**、**智谱 GLM**、**Kimi/Moonshot**、**MiniMax**
- **Hugging Face**、**OpenAI**、**Anthropic**、**Google Gemini**
- **自定义端点** - 支持本地部署模型

切换命令：`hermes model`

### 4. 🛠️ 强大的工具系统

**40+ 内置工具**：
- 文件操作（读写、搜索、批量处理）
- 代码执行（Python、Shell、JavaScript）
- Web 搜索（Firecrawl、Tavily 等）
- 图像生成（DALL-E、Midjourney、Stable Diffusion）
- 文本转语音（TTS）
- 浏览器自动化
- Git 操作
- Docker 管理

**6 种终端后端**：
- **Local** - 本地执行
- **Docker** - 容器隔离
- **SSH** - 远程执行
- **Singularity** - HPC 环境
- **Modal** - 无服务器（空闲时休眠，按需唤醒）
- **Daytona** - 无服务器沙箱

### 5. ⏰ 自动化调度

- 内置 cron 调度器
- 自然语言配置定时任务
- 支持日报、备份、审计等无人值守任务
- 任务结果可投递到任意消息平台

示例：
```bash
hermes cron add "每天上午9点发送天气报告到 Telegram"
hermes cron list
```

### 6. 🚀 并行与委托

- **子代理派生**：为并行工作流创建隔离的子代理
- **RPC 工具调用**：编写 Python 脚本直接调用工具，将多步骤流程压缩为零上下文成本的单次调用
- **批量处理**：`batch_runner.py` 支持并行轨迹生成

### 7. 🎨 丰富的用户界面

#### CLI（命令行界面）
- Rich 库提供的精美 UI（横幅、面板、动画）
- KawaiiSpinner 动画表情
- prompt_toolkit 自动补全
- 皮肤引擎（数据驱动的主题定制）

#### TUI（终端用户界面）
- 基于 React Ink 的现代化终端 UI
- 多行编辑
- 斜杠命令自动补全
- 会话历史浏览
- 中断并重定向
- 流式工具输出显示

激活方式：`hermes --tui` 或 `HERMES_TUI=1`

#### Web Dashboard
- 嵌入式 TUI 体验
- xterm.js + WebGL 渲染
- WebSocket 实时通信
- PTY 支持的完整终端体验

---

## 快速开始

### 安装

#### Linux/macOS/WSL2/Termux（推荐）

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc    # 或 source ~/.zshrc
hermes              # 开始聊天！
```

#### Windows (PowerShell) - 早期测试版

```powershell
iex (irm https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.ps1)
```

> **注意**：Windows 原生支持处于早期测试阶段。如需最稳定的 Windows 体验，建议在 WSL2 中使用 Linux 安装命令。

#### 从源码安装（开发者）

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
./setup-hermes.sh     # 自动安装 uv、创建虚拟环境、安装依赖
./hermes              # 启动
```

手动安装：
```bash
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[all,dev]"
```

### 首次设置

```bash
# 运行完整设置向导
hermes setup

# 或使用 Nous Portal（无需收集多个 API Key）
hermes setup --portal
```

设置向导会引导你：
1. 选择 LLM 提供商和模型
2. 配置 API Key
3. 启用/禁用工具
4. 设置消息平台（可选）
5. 配置个性化选项

---

## 使用方法

### 基础命令

```bash
# 交互式聊天
hermes
hermes chat

# 恢复之前的会话
hermes --resume <session_id>
hermes --continue

# 选择模型
hermes model

# 配置工具
hermes tools

# 查看配置状态
hermes status

# 诊断问题
hermes doctor

# 查看日志
hermes logs [--follow] [--level INFO] [--session <id>]

# 更新到最新版本
hermes update
```

### 消息网关

```bash
# 启动网关
hermes gateway

# 网关设置向导
hermes gateway setup

# 安装为系统服务（桌面/服务器）
hermes gateway install

# 查看网关状态
hermes gateway status
```

### 定时任务

```bash
# 添加定时任务（自然语言）
hermes cron add "每天早上9点发送新闻摘要"

# 列出所有任务
hermes cron list

# 删除任务
hermes cron remove <task_id>

# 启动调度器
hermes cron start
```

### 会话管理

```bash
# 浏览历史会话
hermes sessions browse

# 搜索会话
hermes sessions search "关键词"

# 导出会话
hermes sessions export <session_id>

# 删除会话
hermes sessions delete <session_id>
```

### 技能管理

```bash
# 查看所有技能
hermes skills

# 安装技能
hermes skills install <skill_name>

# 创建技能
hermes skills create

# 启用/禁用技能
hermes skills enable <skill_name>
hermes skills disable <skill_name>
```

### 配置管理

```bash
# 设置配置项
hermes config set <key> <value>

# 查看配置
hermes config get <key>

# 编辑配置文件
hermes config edit
```

### 用户画像

```bash
# 查看当前画像
hermes profile show

# 切换画像
hermes profile switch <name>

# 创建新画像
hermes profile create <name>
```

### Web Dashboard

```bash
# 启动 Web 面板
hermes dashboard

# 访问 http://localhost:8080
```

---

## 架构设计

### 项目结构

```
hermes-agent/
├── run_agent.py          # AIAgent 类 - 核心对话循环 (~12k LOC)
├── model_tools.py        # 工具编排、discover_builtin_tools()、handle_function_call()
├── toolsets.py           # 工具集定义、_HERMES_CORE_TOOLS 列表
├── cli.py                # HermesCLI 类 - 交互式 CLI 编排器 (~11k LOC)
├── hermes_state.py       # SessionDB - SQLite 会话存储（FTS5 搜索）
├── hermes_constants.py   # get_hermes_home() - 配置文件感知路径
├── hermes_logging.py     # setup_logging() - agent.log / errors.log / gateway.log
│
├── agent/                # 代理内部实现
│   ├── transports/       # 传输层适配器
│   ├── memory_provider.py    # 记忆提供者
│   ├── context_engine.py     # 上下文引擎
│   ├── conversation_loop.py  # 对话循环
│   └── ...                 # 其他核心组件
│
├── hermes_cli/           # CLI 子命令、设置向导、插件加载器、皮肤引擎
│   ├── commands.py       # 斜杠命令注册表
│   ├── skin_engine.py    # 皮肤主题引擎
│   ├── plugins.py        # 插件管理器
│   └── ...
│
├── tools/                # 工具实现（自动发现）
│   ├── registry.py       # 工具注册中心
│   ├── environments/     # 终端后端（local、docker、ssh、modal、daytona、singularity）
│   └── *.py              # 各个工具文件
│
├── gateway/              # 消息网关
│   ├── run.py            # 网关主入口
│   ├── session.py        # 会话管理
│   └── platforms/        # 各平台适配器（telegram、discord、slack、whatsapp 等 15+）
│
├── ui-tui/               # Ink (React) 终端 UI
│   └── src/              # entry.tsx, app.tsx, gatewayClient.ts + components
│
├── tui_gateway/          # Python JSON-RPC TUI 后端
│
├── acp_adapter/          # ACP 服务器（VS Code / Zed / JetBrains 集成）
│
├── cron/                 # 调度器
│   ├── jobs.py           # 任务定义
│   └── scheduler.py      # 调度逻辑
│
├── plugins/              # 插件系统
│   ├── memory/           # 记忆提供者插件（honcho、mem0、supermemory）
│   ├── context_engine/   # 上下文引擎插件
│   ├── model-providers/  # 推理后端插件
│   ├── kanban/           # 多代理看板调度器
│   └── ...
│
├── skills/               # 内置技能
├── optional-skills/      # 可选技能（默认不激活）
├── tests/                # 测试套件（~17k 测试，~900 文件）
└── docs/                 # 文档
```

### 核心组件

#### 1. AIAgent 类 (`run_agent.py`)

核心对话循环，包含约 60 个参数：

```python
class AIAgent:
    def __init__(self,
        base_url: str = None,
        api_key: str = None,
        provider: str = None,
        model: str = "",
        max_iterations: int = 90,
        enabled_toolsets: list = None,
        disabled_toolsets: list = None,
        platform: str = None,
        session_id: str = None,
        credential_pool=None,
        # ... 更多参数
    ): ...

    def chat(self, message: str) -> str:
        """简单接口 - 返回最终响应字符串"""

    def run_conversation(self, user_message: str, system_message: str = None,
                         conversation_history: list = None) -> dict:
        """完整接口 - 返回包含 final_response + messages 的字典"""
```

**对话循环流程**：
```python
while (api_call_count < self.max_iterations and self.iteration_budget.remaining > 0):
    if self._interrupt_requested: break
    
    # 1. 调用 LLM
    response = client.chat.completions.create(
        model=model, 
        messages=messages, 
        tools=tool_schemas
    )
    
    if response.tool_calls:
        # 2. 执行工具调用
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args)
            messages.append(tool_result_message(result))
        api_call_count += 1
    else:
        # 3. 返回最终答案
        return response.content
```

#### 2. CLI 架构 (`cli.py`)

- **Rich** - 用于横幅/面板的精美 UI
- **prompt_toolkit** - 带自动补全的输入
- **KawaiiSpinner** - 动画表情反馈
- **皮肤引擎** - 数据驱动的 CLI 主题定制

**斜杠命令注册表** (`hermes_cli/commands.py`)：

所有斜杠命令在中央 `COMMAND_REGISTRY` 中定义，自动同步到：
- CLI 分发
- 网关分发
- 网关帮助文本
- Telegram BotCommand 菜单
- Slack 子命令映射
- 自动补全

添加新命令只需三步：
1. 在 `COMMAND_REGISTRY` 中添加 `CommandDef`
2. 在 `HermesCLI.process_command()` 中添加处理器
3. （可选）在 `gateway/run.py` 中添加网关处理器

#### 3. 工具系统

**工具注册** (`tools/registry.py`)：

```python
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

**自动发现**：任何 `tools/*.py` 文件中包含顶层 `registry.register()` 调用的文件都会被自动导入。

**工具集定义** (`toolsets.py`)：
- `_HERMES_CORE_TOOLS` - 所有平台继承的核心工具集
- 自定义工具集 - 可按需启用/禁用

#### 4. 消息网关 (`gateway/`)

**平台适配器** (`gateway/platforms/`)：

每个平台一个适配器文件，实现统一的接口：
- 消息接收
- 消息发送
- 身份验证
- 会话管理

**内置钩子** (`gateway/builtin_hooks/`)：
- 扩展点，用于始终注册的网关钩子

#### 5. 记忆系统

| 类型 | 实现 |
|------|------|
| 短期记忆 | OpenAI 格式对话历史 |
| 长期记忆 | SQLite + FTS5 全文搜索 + LLM 摘要 |
| 技能记忆 | YAML 格式过程记忆 |
| 用户画像 | 跨会话持久化用户建模 |
| Honcho 集成 | 方言用户建模 |

#### 6. 技能系统

- 遵循 [agentskills.io](https://agentskills.io) 开放标准
- 存储在 `~/.hermes/skills/`
- 格式：`SKILL.md`（Markdown + YAML 元数据）
- 代理可自主创建和改进技能
- 渐进式披露架构

---

## 配置文件

### 主要配置文件

| 文件 | 用途 |
|------|------|
| `~/.hermes/config.yaml` | 主配置文件（非敏感设置） |
| `~/.hermes/.env` | API 密钥和敏感信息 |
| `~/.hermes/skins/*.yaml` | 自定义皮肤主题 |
| `~/.hermes/skills/` | 自定义技能 |
| `~/.hermes/plugins/` | 自定义插件 |

### Config.yaml 顶级配置项

```yaml
model:                    # 模型配置
agent:                    # 代理行为配置
terminal:                 # 终端配置
compression:              # 上下文压缩配置
display:                  # 显示配置（包括皮肤）
stt:                      # 语音转文本配置
tts:                      # 文本转语音配置
memory:                   # 记忆系统配置
security:                 # 安全配置
delegation:               # 委托配置
smart_model_routing:      # 智能模型路由
checkpoints:              # 检查点配置
auxiliary:                # 辅助 LLM 任务配置
curator:                  # 后台技能维护配置
skills:                   # 技能系统配置
gateway:                  # 网关配置
logging:                  # 日志配置
cron:                     # 定时任务配置
profiles:                 # 用户画像配置
plugins:                  # 插件配置
honcho:                   # Honcho 集成配置
```

### 添加新配置项

1. **config.yaml 选项**：
   - 添加到 `DEFAULT_CONFIG`（`hermes_cli/config.py`）
   - 如果需要迁移现有用户配置，增加 `_config_version`

2. **.env 变量**（仅用于密钥）：
   - 添加到 `OPTIONAL_ENV_VARS`（`hermes_cli/config.py`）
   - 提供元数据（描述、提示、URL、类别等）

### 配置加载器

| 加载器 | 使用场景 | 位置 |
|--------|---------|------|
| `load_cli_config()` | CLI 模式 | `cli.py` |
| `load_config()` | 大多数 CLI 子命令 | `hermes_cli/config.py` |
| 直接 YAML 加载 | 网关运行时 | `gateway/run.py` + `gateway/config.py` |

---

## 开发指南

### 环境设置

```bash
# 克隆仓库
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 使用一键安装脚本（推荐）
./setup-hermes.sh

# 或手动设置
uv venv .venv --python 3.11
source .venv/bin/activate
uv pip install -e ".[all,dev]"
```

### 运行测试

```bash
# 运行全部测试（与 CI 一致）
scripts/run_tests.sh

# 运行单个测试文件
scripts/run_tests.sh tests/tools/test_file_operations.py

# Lint 检查
ruff check .
```

### 添加新工具

#### 方法 1：插件方式（推荐用于自定义工具）

创建 `~/.hermes/plugins/<name>/plugin.yaml` 和 `__init__.py`：

```python
# ~/.hermes/plugins/myplugin/__init__.py
def register(ctx):
    ctx.register_tool(
        name="my_custom_tool",
        schema={...},
        handler=my_handler,
    )
```

#### 方法 2：核心工具（需要贡献到上游）

1. 创建 `tools/your_tool.py`：

```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def example_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="example_tool",
    toolset="example",
    schema={"name": "example_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: example_tool(param=args.get("param"), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

2. 在 `toolsets.py` 中添加工具集

### 添加斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 中添加：

```python
CommandDef("mycommand", "Description of what it does", "Session",
           aliases=("mc",), args_hint="[arg]"),
```

2. 在 `HermesCLI.process_command()` 中添加处理器：

```python
elif canonical == "mycommand":
    self._handle_mycommand(cmd_original)
```

3. （可选）在 `gateway/run.py` 中添加网关处理器

### 添加自定义皮肤

创建 `~/.hermes/skins/<name>.yaml`：

```yaml
name: cyberpunk
description: Neon-soaked terminal theme

colors:
  banner_border: "#FF00FF"
  banner_title: "#00FFFF"
  banner_accent: "#FF1493"

spinner:
  thinking_verbs: ["jacking in", "decrypting", "uploading"]
  wings:
    - ["⟨⚡", "⚡⟩"]

branding:
  agent_name: "Cyber Agent"
  response_label: " ⚡ Cyber "

tool_prefix: "▏"
```

激活：`/skin cyberpunk` 或在 `config.yaml` 中设置 `display.skin: cyberpunk`

### 依赖固定策略

所有依赖必须有上限以限制供应链攻击面：

| 来源类型 | 处理方式 | 示例 |
|---------|---------|------|
| PyPI 包 | `>=floor,<next_major` | `"httpx>=0.28.1,<1"` |
| Git URL | Commit SHA | `git+https://...@<40-char-sha>` |
| GitHub Actions | Commit SHA + 注释 | `uses: actions/checkout@<sha>  # v4` |
| CI-only pip | `==exact` | `pyyaml==6.0.2` |

---

## 应用场景

### 1. 个人 AI 助手

**场景**：在 Telegram/Discord 上随时与 AI 对话

**优势**：
- 24/7 在线，运行在 VPS 上
- 记住你的偏好和历史
- 可以执行实际任务（文件操作、代码运行等）

**设置**：
```bash
hermes gateway setup    # 配置 Telegram/Discord
hermes gateway start    # 启动网关
```

### 2. 自动化运维

**场景**：定时备份、监控、报告生成

**示例任务**：
- 每天上午 9 点发送服务器状态报告
- 每周日凌晨 2 点备份数据库
- 每小时检查日志异常并告警

**设置**：
```bash
hermes cron add "每天早上9点发送服务器状态到 Telegram"
hermes cron add "每周日凌晨2点备份数据库"
hermes cron start
```

### 3. 代码助手

**场景**：文件操作、代码生成、测试运行

**能力**：
- 读取/写入文件
- 运行测试
- Git 操作
- 代码审查
- 重构建议

**使用**：
```bash
hermes
> 帮我重构这个函数，提高可读性
> 为这个模块编写单元测试
> 检查最近的 git commit 是否有问题
```

### 4. 研究工具

**场景**：Web 搜索、数据收集、批量处理

**能力**：
- Web 搜索（Firecrawl、Tavily）
- 批量数据处理
- 数据可视化
- 文献整理

**使用**：
```bash
hermes
> 搜索关于量子计算的最新论文
> 分析这些数据并生成图表
> 整理这些文献的引用格式
```

### 5. 多代理协作

**场景**：复杂任务分解和并行执行

**能力**：
- 派生子代理处理子任务
- 并行工作流
- 结果汇总

**使用**：
```bash
hermes
> 同时分析这三个数据集，每个用一个子代理
> 汇总结果并生成报告
```

### 6. 技能学习

**场景**：从使用中自动学习和优化工作流程

**能力**：
- 自动创建技能
- 技能自我优化
- 跨会话知识积累

**示例**：
- 第一次：手动执行数据分析流程
- 第二次：代理建议使用已保存的技能
- 第三次：技能已优化，执行更快

### 7. 智能家居控制

**场景**：通过 Home Assistant 集成控制智能家居

**能力**：
- 控制灯光、温度、安防
- 自动化场景
- 语音控制

**设置**：
```bash
hermes gateway setup    # 配置 Home Assistant
```

### 8. 团队协作

**场景**：在 Slack/Discord 中作为团队助手

**能力**：
- 回答团队成员问题
- 执行常用任务
- 项目管理集成
- 自动化工作流

---

## Python API 使用

### 基础用法

```python
from run_agent import AIAgent

# 创建代理实例
agent = AIAgent(
    base_url="http://localhost:30000/v1",
    model="claude-opus-4-20250514",
    api_key="your-api-key"
)

# 简单对话
response = agent.chat("你好，请介绍一下自己")
print(response)

# 完整对话（带历史）
result = agent.run_conversation(
    user_message="帮我写一个 Python 函数",
    conversation_history=[...]
)
print(result['final_response'])
```

### 高级用法

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url="https://openrouter.ai/api/v1",
    model="anthropic/claude-opus-4",
    max_iterations=90,
    enabled_toolsets=["core", "web", "files"],
    disabled_toolsets=["experimental"],
    save_trajectories=True,
    verbose=True
)

# 带工具调用的对话
result = agent.run_conversation(
    user_message="搜索最新的 AI 新闻并总结",
    system_message="你是一个专业的技术分析师"
)

# 访问完整消息历史
for msg in result['messages']:
    print(f"{msg['role']}: {msg['content']}")
```

### 批量处理

```python
from batch_runner import BatchRunner

runner = BatchRunner(
    input_file="tasks.jsonl",
    output_file="results.jsonl",
    parallel=5
)
runner.run()
```

---

## 常见问题

### Q: Hermes Agent 和 OpenClaw 有什么区别？

A: Hermes Agent 是 OpenClaw 的演进版本，增加了：
- 自我改进的学习循环
- 更丰富的平台支持
- 更强的工具系统
- 更好的性能优化

如果你从 OpenClaw 迁移，可以使用：
```bash
hermes claw migrate
```

### Q: 需要什么样的硬件配置？

A: 
- **最低**：$5 VPS（1GB RAM，单核 CPU）
- **推荐**：2GB+ RAM，双核 CPU
- **最佳**：GPU 加速（用于本地模型）

### Q: 可以在没有 GPU 的情况下运行吗？

A: 可以！Hermes Agent 本身不需要 GPU。它通过 API 调用云端模型。如果你想运行本地模型，才需要 GPU。

### Q: 数据安全吗？

A: 
- API 密钥存储在 `~/.hermes/.env` 中
- 对话历史存储在本地 SQLite 数据库
- 支持命令审批机制
- 可配置 DM 配对（防止陌生人访问）
- 支持容器隔离

### Q: 如何备份我的数据？

A: 备份 `~/.hermes/` 目录：
```bash
tar -czf hermes-backup.tar.gz ~/.hermes/
```

### Q: 如何更新到最新版本？

A: 
```bash
hermes update
```

或从源码：
```bash
git pull
./setup-hermes.sh
```

### Q: 遇到问题怎么办？

A: 
1. 运行诊断：`hermes doctor`
2. 查看日志：`hermes logs --follow`
3. 查看文档：https://hermes-agent.nousresearch.com/docs/
4. 提交 Issue：https://github.com/NousResearch/hermes-agent/issues
5. Discord 社区：https://discord.gg/NousResearch

---

## 资源链接

- **官方文档**：https://hermes-agent.nousresearch.com/docs/
- **GitHub 仓库**：https://github.com/NousResearch/hermes-agent
- **Discord 社区**：https://discord.gg/NousResearch
- **Skills Hub**：https://agentskills.io
- **Nous Research**：https://nousresearch.com
- **Issue 追踪**：https://github.com/NousResearch/hermes-agent/issues

---

## 许可证

MIT License - 详见 [LICENSE](LICENSE) 文件

---

## 致谢

由 [Nous Research](https://nousresearch.com) 构建和维护。

感谢所有贡献者和社区成员！
