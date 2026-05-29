# Hermes Agent 源码启动指南

## 一、安装与启动

### 1. 一键安装（推荐）

```bash
./setup-hermes.sh
```

脚本自动完成：安装 uv → 创建 Python 3.11+ 虚拟环境 → 安装依赖 → 创建 .env → 创建 `hermes` 命令软链接 → 同步技能文件。

支持桌面/服务器和 Android/Termux 两种环境，脚本会自动检测。

### 2. 手动安装

```bash
uv venv                          # 创建虚拟环境
uv pip install -e ".[all]"       # 安装全部依赖
cp .env.example .env             # 创建环境变量文件
```

### 3. 启动入口

项目有三个入口命令（定义在 `pyproject.toml` 的 `console_scripts` 中）：

| 命令 | 入口文件 | 用途 |
|------|---------|------|
| `hermes` | `hermes_cli/main.py:main` | 交互式 CLI 聊天 |
| `hermes-agent` | `run_agent.py:main` | 独立代理运行器 |
| `hermes-acp` | `acp_adapter/entry.py:main` | ACP 协议适配服务器 |

## 二、主要启动方式

```bash
# 交互式聊天（最常用）
hermes
hermes chat

# 恢复之前的会话
hermes --resume <session_id>
hermes --continue

# 选择模型
hermes model

# 运行设置向导（配置 API Key 等）
hermes setup

# 启动消息网关
hermes gateway

# 诊断问题
hermes doctor
```

也可以用 Python 直接运行：

```bash
source venv/bin/activate
python -m hermes_cli.main          # 等同于 hermes
python run_agent.py                 # 等同于 hermes-agent
```

## 三、启动后可用的功能

### 1. 交互式聊天

- 多模型对话（OpenAI、Anthropic、Gemini、Bedrock、OpenRouter 等）
- 会话管理与历史记录
- 工具调用（40+ 内置工具）
- 技能加载（`--skills` 参数）

### 2. 工具系统

工具定义在 `tools/` 目录，通过 `tools/registry.py` 自动注册，包含 40+ 内置工具：

- 文件操作、代码执行、Shell 命令
- Web 搜索、图像生成
- 更多工具通过插件机制扩展

### 3. 消息网关

网关定义在 `gateway/` 目录，支持 15 个平台的消息收发：

| 分类 | 平台 |
|------|------|
| 即时通讯 | Telegram、Discord、Slack、WhatsApp、Signal、Matrix |
| 中国平台 | 微信、飞书、钉钉 |
| 其他 | Email、SMS、Webhook、Home Assistant、Mattermost、BlueBubbles |

```bash
hermes gateway              # 启动网关
hermes gateway install      # 安装为系统服务（桌面/服务器）
```

### 4. 技能系统

技能以 `SKILL.md` 格式存储在 `~/.hermes/skills/`，遵循 agentskills.io 标准：

- 代理可自主创建和改进技能
- 支持渐进式披露架构
- 首次安装时自动同步 `skills/` 目录下的内置技能

### 5. 记忆系统

| 类型 | 实现 |
|------|------|
| 短期记忆 | OpenAI 格式对话历史 |
| 长期记忆 | SQLite + FTS5 全文搜索 + LLM 摘要 |
| 技能记忆 | YAML 格式过程记忆 |
| 用户画像 | 跨会话持久化用户建模 |

### 6. 定时任务

```bash
hermes cron list             # 查看定时任务
```

定时任务定义在 `cron/` 目录。

### 7. 其他命令

```bash
hermes config                # 配置管理
hermes sessions browse       # 浏览历史会话
hermes logs                  # 查看日志
hermes profile               # 用户画像管理
hermes dashboard             # Web UI 面板
hermes update                # 更新代理
hermes status                # 检查配置状态
```

### 8. Python API 调用

可直接在 Python 代码中使用 `AIAgent` 类：

```python
from run_agent import AIAgent

agent = AIAgent(
    base_url="http://localhost:30000/v1",
    model="claude-opus-4-20250514"
)
response = agent.run_conversation("你的问题")
```

主要参数：

| 参数 | 说明 |
|------|------|
| `query` | 自然语言查询 |
| `model` | 模型名称，如 `"anthropic/claude-opus-4-6"` |
| `api_key` | API 密钥 |
| `base_url` | 模型 API 地址 |
| `max_turns` | 最大迭代次数（默认 10） |
| `enabled_toolsets` | 启用的工具集 |
| `disabled_toolsets` | 禁用的工具集 |
| `save_trajectories` | 保存对话轨迹 |
| `verbose` | 详细日志 |

## 四、配置文件

| 文件 | 说明 |
|------|------|
| `~/.hermes/config.yaml` | 主配置文件 |
| `~/.hermes/.env` | API 密钥和敏感信息 |
| `cli-config.yaml` | CLI 配置模板（模型、提供商、Token 限制等） |

首次使用建议先运行 `hermes setup` 完成 API Key 配置。

## 五、代理工作流程

代理运行遵循以下循环：

1. 构建上下文（记忆 + 系统提示 + 上下文文件）
2. 调用 LLM（附带工具 schema）
3. 解析响应中的 tool_calls
4. 并行执行工具（通过安全检查后）
5. 收集结果，重复直到完成或达到 max_turns

## 六、测试

```bash
# 运行全部测试（推荐，与 CI 一致）
scripts/run_tests.sh

# 运行单个测试文件
scripts/run_tests.sh tests/tools/test_file_operations.py

# Lint 检查
ruff check .
```
