# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Hermes Agent 是一个自我改进的 AI 代理，由 Nous Research 开发。核心特性：从经验中创建技能、使用中自我改进、跨会话记忆、多平台消息网关、多模型支持。使用 Python 3.11+，MIT 许可证。

## 常用命令

### 安装与设置
```bash
# 使用 uv 创建虚拟环境并安装全部依赖
uv venv
uv pip install -e ".[all]"

# 开发依赖
uv pip install -e ".[dev]"

# 快速设置脚本
./setup-hermes.sh
```

### 运行测试
```bash
# 运行全部测试（推荐，保证与 CI 一致）
scripts/run_tests.sh

# 运行单个测试文件
scripts/run_tests.sh tests/tools/test_file_operations.py

# 运行单个测试用例
scripts/run_tests.sh tests/tools/test_file_operations.py::TestFoo::test_bar

# 传递 pytest 参数
scripts/run_tests.sh --tb=long -v

# 直接使用 pytest（不推荐，可能与 CI 行为不一致）
pytest -o "addopts=" -n 4 -m "not integration" --ignore=tests/integration --ignore=tests/e2e
```

测试脚本会自动：使用 4 个 xdist worker、设置 TZ=UTC/LANG=C.UTF-8/PYTHONHASHSEED=0、清除所有凭证环境变量。

### Lint
```bash
ruff check .
```

### 运行
```bash
hermes              # 交互式 CLI
hermes model        # 选择 LLM 模型
hermes gateway      # 启动消息网关（Telegram、Discord 等）
hermes setup        # 完整设置向导
```

## 架构

### 入口点
- `hermes` CLI → `hermes_cli/main.py:main`
- `hermes-agent` → `run_agent.py:main`（核心代理运行器）
- `hermes-acp` → `acp_adapter/entry:main`

### 核心模块
```
run_agent.py          # AIAgent 类：对话循环、工具调用、响应管理
model_tools.py        # 工具编排，并行执行与安全检查
toolsets.py           # 工具集组合/分发
cli.py                # 交互式 TUI（prompt_toolkit）
```

### 目录结构
```
agent/          # 核心代理：提示构建、记忆管理、模型元数据、错误分类
tools/          # 工具实现：registry.py 自注册，40+ 工具
hermes_cli/     # CLI 界面：TUI、命令补全、配置管理
gateway/        # 多平台消息：Telegram、Discord、Slack、WhatsApp、Signal
skills/         # 技能系统：SKILL.md 格式，agentskills.io 标准
environments/   # RL 训练环境、SWE 基准
plugins/        # 可插拔扩展：记忆后端、图像生成、UI
acp_adapter/    # Agent Client Protocol 适配器
cron/           # 定时任务调度
tui_gateway/    # TUI 与网关桥接
```

### 关键设计模式

**工具注册表（Registry）**：`tools/registry.py` 维护中心注册表。工具在导入时通过 `registry.register()` 自注册，线程安全的快照机制支持并发访问。

**环境后端**：`tools/environments/base.py` 定义抽象接口。六种实现：Local、Docker、SSH、Modal、Daytona、Singularity。统一 spawn-per-call 模型。

**模型适配器**：`agent/` 下有多模型提供商适配器（OpenAI、Anthropic、Gemini、Bedrock、OpenRouter 等），通过 `hermes model` 切换，无需改代码。

**技能系统**：渐进式披露架构。技能以目录 + SKILL.md 文件形式存储在 `~/.hermes/skills/`。代理可自主创建和改进技能。

**记忆系统**：
- 短期：OpenAI 格式对话历史
- 长期：SQLite + FTS5 全文搜索 + LLM 摘要
- 技能：YAML 格式过程记忆
- 用户画像：跨会话持久化用户建模

### 代理循环
1. 构建上下文（记忆 + 系统提示 + 上下文文件）
2. 调用 LLM（附带工具 schema）
3. 解析响应中的 tool_calls
4. 并行执行工具（通过安全检查后）
5. 收集结果，重复直到完成或达到 max_turns

### 配置
- `~/.hermes/config.yaml` — 主配置
- `~/.hermes/.env` — API 密钥和密钥
- `cli-config.yaml.example` — CLI 配置模板
- 环境变量优先级最高

### 测试结构
- `tests/` — 单元测试，按模块组织（`tests/tools/`、`tests/honcho_plugin/` 等）
- `tests/conftest.py` — 共享 fixture，自动清除凭证
- `tests/integration/` 和 `tests/e2e/` — 默认被排除
- 使用 pytest + xdist 并行 + pytest-split 分片

### CI
GitHub Actions 工作流：自动化测试、Docker 发布、Nix 打包、文档部署、供应链安全审计。
