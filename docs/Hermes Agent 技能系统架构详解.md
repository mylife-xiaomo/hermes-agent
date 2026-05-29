# Hermes Agent 技能系统架构详解

> 基于 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) v0.11.0 源码分析
> 分析日期：2026-04-24

---

## 1. 概述

Hermes Agent 的技能系统采用**渐进式披露（Progressive Disclosure）**架构。核心思想：**不一次性把所有技能内容塞给 LLM，而是分层加载，按需展开，最小化 token 消耗**。

技能是存储在文件系统中的 Markdown 文件（`SKILL.md`），包含 YAML frontmatter 元数据 + Markdown 正文指令。代理可以在对话中自主加载、创建、改进技能，形成闭环学习能力。

---

## 2. 三层渐进式披露架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    渐进式披露三层架构                                 │
├──────────────┬──────────────────┬────────────────────────────────────┤
│   Tier 1     │    Tier 2        │         Tier 3                    │
│   索引层      │    全文层         │         子文件层                   │
├──────────────┼──────────────────┼────────────────────────────────────┤
│ 启动时注入    │ LLM 按需调用       │ LLM 按需调用                      │
│ 系统提示      │ skill_view(name)  │ skill_view(name, file_path)       │
├──────────────┼──────────────────┼────────────────────────────────────┤
│ 仅名称+描述   │ 完整 SKILL.md     │ 技能目录内特定文件内容              │
│ 极小 token    │ + 关联文件列表    │ 按需加载                           │
│              │ + 配置值+设置状态  │                                    │
└──────────────┴──────────────────┴────────────────────────────────────┘
```

### Tier 1 — 索引层（启动时）

触发时机：代理启动构建系统提示时。

输出格式：
```
<available_skills>
  mlops:
    - axolotl: Fine-tune models using Axolotl
    - vllm: Serve models with vLLM
  devops:
    - docker-deploy: Deploy containers to production
</available_skills>
```

### Tier 2 — 全文层（按需加载）

触发时机：LLM 看到索引后，判断某个技能相关，调用 `skill_view(name)`。

返回内容：完整 SKILL.md 正文 + frontmatter 元数据 + 关联文件列表（references/、templates/、scripts/、assets/）+ 配置值 + 设置状态。

### Tier 3 — 子文件层（按需读取）

触发时机：LLM 需要查看技能目录内的特定文件时，调用 `skill_view(name, file_path="references/api.md")`。

返回内容：技能目录内指定文件的完整内容。

---

## 3. 技能文件格式

每个技能是一个目录，包含一个 `SKILL.md` 文件和可选的子目录：

```
~/.hermes/skills/
├── mlops/
│   ├── DESCRIPTION.md          # 分类描述（可选）
│   ├── axolotl/
│   │   ├── SKILL.md            # 主技能文件（必需）
│   │   ├── references/         # 参考文档（可选）
│   │   │   └── api.md
│   │   ├── templates/          # 模板文件（可选）
│   │   │   └── config.yaml
│   │   ├── scripts/            # 脚本文件（可选）
│   │   │   └── train.sh
│   │   └── assets/             # 资源文件（可选）
│   └── vllm/
│       └── SKILL.md
└── devops/
    └── docker-deploy/
        └── SKILL.md
```

**SKILL.md 文件格式**：

```yaml
---
name: axolotl                    # 技能名称（必需，最长64字符）
description: Fine-tune models    # 技能描述（必需，最长1024字符）
platforms: [macos, linux]        # 平台限制（可选）
metadata:
  hermes:
    requires_toolsets:           # 依赖的工具集（可选，缺失则隐藏该技能）
      - web-browsing
    requires_tools:              # 依赖的工具（可选）
      - api-key
    config:                      # 技能配置声明（可选）
      - name: model_path
        description: Path to model
        type: string
        required: true
---

（Markdown 正文：具体的技能指令、步骤、注意事项）
```

---

## 4. 完整加载和使用流程

### 4.1 流程总览图

```
                          ┌──────────────┐
                          │   代理启动    │
                          └──────┬───────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │  build_skills_system_prompt()  │
                 │  构建技能索引（Tier 1）          │
                 └───────────────┬───────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              ┌─────▼─────┐           ┌───────▼──────┐
              │ LRU 缓存？  │           │  磁盘快照？   │
              │  命中→返回  │           │  命中→解析   │
              └─────┬─────┘           └───────┬──────┘
                    │ 未命中                  │ 未命中
                    └────────────┬────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │  iter_skill_index_files()      │
                 │  全量扫描 ~/.hermes/skills/     │
                 │  找到所有 SKILL.md 文件          │
                 └───────────────┬───────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │  parse_frontmatter()           │
                 │  解析每个 SKILL.md 的元数据      │
                 └───────────────┬───────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │  过滤管道：                      │
                 │  ① 平台兼容性检查               │
                 │  ② 禁用技能过滤                 │
                 │  ③ 工具/工具集依赖检查            │
                 └───────────────┬───────────────┘
                                 │
                                 ▼
                 ┌───────────────────────────────┐
                 │  生成索引 + 写入缓存/快照        │
                 │  注入到系统提示的                 │
                 │  <available_skills> 区域       │
                 └───────────────┬───────────────┘
                                 │
                                 ▼
                    ┌────────────────────────┐
                    │     代理对话循环开始     │
                    └────────────┬───────────┘
                                 │
                                 ▼
              ┌──────────────────────────────────┐
              │  LLM 看到系统提示中的技能索引       │
              │  判断哪些技能与当前任务相关          │
              └──────────────┬───────────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              ┌─────▼──────┐   ┌─────▼──────────┐
              │ 不相关→跳过  │   │  相关→调用工具   │
              │            │   │ skill_view(name)│
              │            │   └─────┬──────────┘
              │            │         │
              │            │         ▼
              │            │  ┌──────────────────────┐
              │            │  │  技能查找（三级优先）  │
              │            │  │  ① 插件技能(冒号分隔)  │
              │            │  │  ② 本地目录直接匹配    │
              │            │  │  ③ 目录名遍历搜索      │
              │            │  └─────────┬────────────┘
              │            │            │
              │            │            ▼
              │            │  ┌──────────────────────┐
              │            │  │  安全检查管道          │
              │            │  │  ① 受信任目录验证      │
              │            │  │  ② Prompt 注入检测     │
              │            │  │  ③ 平台兼容性检查      │
              │            │  │  ④ 禁用状态检查        │
              │            │  └─────────┬────────────┘
              │            │            │
              │            │            ▼
              │            │  ┌──────────────────────┐
              │            │  │ _build_skill_message()│
              │            │  │ 组装完整技能消息        │
              │            │  │ ① 模板变量替换         │
              │            │  │ ② 内联 Shell 展开      │
              │            │  │ ③ 注入目录绝对路径      │
              │            │  │ ④ 注入配置值           │
              │            │  │ ⑤ 列出关联文件         │
              │            │  │ ⑥ 附加设置状态         │
              │            │  └─────────┬────────────┘
              │            │            │
              │            │            ▼
              │            │  ┌──────────────────────┐
              │            │  │  注入到对话上下文中    │
              │            │  │  LLM 按技能指令执行   │
              │            │  └─────────┬────────────┘
              │            │            │
              │            │   ┌────────┴────────┐
              │            │   │                 │
              │            │   ▼                 ▼
              │            │  需要子文件？    任务完成？
              │            │  skill_view(     保存/更新
              │            │   name,           技能
              │            │   file_path)     (skill_manage)
              │            │   ── Tier 3 ──
              │            └──────────────────────
              └───────────────────────────────────
```

### 4.2 阶段详解

#### 阶段一：启动时索引构建

**入口函数**：`agent/prompt_builder.py` → `build_skills_system_prompt()`

```
1. 获取技能目录列表（本地 ~/.hermes/skills/ + 外部目录）
2. 构建缓存 key = (目录路径, 可用工具集, 平台, 禁用列表)
3. 尝试 LRU 内存缓存 → 命中则直接返回
4. 尝试磁盘快照（.skills_prompt_snapshot.json）→ 命中则解析快照
5. 冷启动：全量扫描 + 过滤 + 写入快照 + 存入 LRU
6. 组装系统提示片段，注入 <available_skills> 区域
```

**缓存策略**：
| 层级 | 存储位置 | 有效性验证 | 生命周期 |
|------|---------|-----------|---------|
| LRU 内存缓存 | 进程内 OrderedDict | cache key 精确匹配 | 进程退出即失效 |
| 磁盘快照 | .skills_prompt_snapshot.json | mtime/size 清单 | 文件变更后失效 |

**关键文件扫描函数**：`agent/skill_utils.py` → `iter_skill_index_files()`

```python
# 递归遍历技能目录，yield 所有 SKILL.md 的路径
# 排除 .git/.github/.hub 目录
# 按相对路径排序，确保输出稳定
def iter_skill_index_files(skills_dir: Path, filename: str):
    matches = []
    for root, dirs, files in os.walk(skills_dir, followlinks=True):
        dirs[:] = [d for d in dirs if d not in EXCLUDED_SKILL_DIRS]
        if filename in files:
            matches.append(Path(root) / filename)
    for path in sorted(matches, key=lambda p: str(p.relative_to(skills_dir))):
        yield path
```

**过滤管道**（`prompt_builder.py` → `_skill_should_show()`）：

```
SKILL.md 发现
    │
    ▼
平台兼容性检查 ──── 不兼容 → 跳过
    │ (platforms: [macos, linux])
    │ 兼容
    ▼
禁用技能检查 ────── 已禁用 → 跳过
    │ (config.yaml 中的 disabled_skills)
    │ 未禁用
    ▼
工具依赖检查 ────── 依赖缺失 → 跳过
    │ (requires_toolsets / requires_tools)
    │ 依赖满足
    ▼
加入索引 → 生成 "name: description" 条目
```

#### 阶段二：LLM 按需加载技能

**入口函数**：`tools/skills_tool.py` → `skill_view()`

当 LLM 判断某个技能与当前任务相关时，通过工具调用 `skill_view(name)` 加载完整内容。

**技能查找优先级**：
```
skill_view("axolotl")
    │
    ├─ 名称含冒号？ ──→ 走插件系统（如 "my-plugin:axolotl"）
    │                    parse_qualified_name() 分割命名空间
    │                    get_plugin_manager().find_plugin_skill()
    │
    ├─ 直接路径匹配 ──→ ~/.hermes/skills/mlops/axolotl/SKILL.md
    │
    ├─ 目录名匹配 ──→ 遍历所有 SKILL.md，匹配 parent.name
    │
    └─ 旧版 .md 文件 ──→ 直接匹配 name.md（向后兼容）
```

**安全检查管道**：
```
文件读取
    │
    ▼
受信任目录验证 ──── 不在信任目录 → 记录警告
    │
    ▼
Prompt 注入检测 ──── 检测到注入模式 → 记录警告
    │ (_INJECTION_PATTERNS)
    ▼
平台兼容性检查 ──── 不兼容 → 返回错误
    │
    ▼
禁用状态检查 ────── 已禁用 → 返回错误
    │
    ▼
文件路径请求？ ──── 有 file_path → 路径遍历防护 + 二次校验
    │                            → 读取子文件（Tier 3）
    ▼
无 file_path → 继续全文加载（Tier 2）
```

#### 阶段三：技能内容组装

**入口函数**：`agent/skill_commands.py` → `_build_skill_message()`

```
原始 SKILL.md 内容
    │
    ▼
模板变量替换
    │ ${HERMES_SKILL_DIR} → 技能目录绝对路径
    │ ${HERMES_SESSION_ID} → 当前会话 ID
    ▼
内联 Shell 展开（可选）
    │ !`command` → 执行命令并替换为 stdout
    ▼
注入技能目录绝对路径
    │ [Skill directory: /home/user/.hermes/skills/mlops/axolotl]
    ▼
注入配置值
    │ 从 config.yaml 读取技能声明的 config 变量
    ▼
附加设置状态
    │ 缺少环境变量？→ 提示需要配置
    │ 跳过安装？→ 提示功能受限
    ▼
列出关联文件
    │ [This skill has supporting files:]
    │ - references/api.md  ->  /home/user/.../references/api.md
    │ - templates/config.yaml  ->  /home/user/.../templates/config.yaml
    ▼
附加用户指令和运行时备注
    │
    ▼
最终组装的消息（注入到对话上下文）
```

#### 阶段四：斜杠命令注册

**入口函数**：`agent/skill_commands.py` → `scan_skill_commands()`

技能还会被注册为斜杠命令，在 CLI 和消息平台中可用：

```
~/.hermes/skills/mlops/axolotl/SKILL.md
    │
    ▼
读取 frontmatter: name = "axolotl"
    │
    ▼
名称规范化: "axolotl" → /axolotl
    │ （空格/下划线 → 连字符，去除特殊字符）
    ▼
注册到 _skill_commands 映射
    │ {
    │   "/axolotl": {
    │     "name": "axolotl",
    │     "description": "Fine-tune models",
    │     "skill_md_path": "/home/user/.hermes/skills/mlops/axolotl/SKILL.md",
    │     "skill_dir": "/home/user/.hermes/skills/mlops/axolotl"
    │   }
    │ }
    ▼
用户输入 /axolotl → 直接激活技能
```

---

## 5. 涉及的核心源码文件

| 文件 | 关键函数 | 职责 |
|------|---------|------|
| `agent/skill_utils.py` | `parse_frontmatter()`, `iter_skill_index_files()`, `skill_matches_platform()` | 技能元数据解析、文件扫描、平台匹配 |
| `agent/prompt_builder.py` | `build_skills_system_prompt()` | Tier 1 索引构建、缓存管理、系统提示注入 |
| `tools/skills_tool.py` | `skills_list()`, `skill_view()` | Tier 1/2/3 工具实现、安全检查 |
| `agent/skill_commands.py` | `_build_skill_message()`, `scan_skill_commands()` | 技能消息组装、斜杠命令注册 |

---

## 6. 缓存与性能优化

```
请求技能索引
    │
    ├─ 命中内存 LRU（max 8 条）────→ 直接返回，零 IO
    │
    ├─ 命中磁盘快照 ──────────────→ 跳过文件扫描，解析 JSON
    │   (.skills_prompt_snapshot.json)
    │   通过 mtime/size 清单验证
    │
    └─ 冷启动 ───────────────────→ 全量扫描 → 写入快照 → 存入 LRU
```

磁盘快照有效性验证：
- 快照中存储了每个 SKILL.md 文件的 `{mtime, size}` 清单
- 加载快照时，比对当前文件的 mtime/size 与清单
- 任何文件变更都会导致快照失效，触发全量重新扫描

---

## 7. 安全机制

| 安全层 | 位置 | 说明 |
|--------|------|------|
| 受信任目录验证 | `skill_view()` | 验证技能文件来自 `~/.hermes/skills/` 或配置的外部目录 |
| Prompt 注入检测 | `skill_view()` | 扫描内容中的常见注入模式（`_INJECTION_PATTERNS`），记录警告 |
| 路径遍历防护 | `skill_view()` | 阻止 `..` 路径，二次校验解析后路径仍在技能目录内 |
| 平台隔离 | `skill_matches_platform()` | 不同平台只显示兼容的技能 |
| 禁用控制 | `_is_skill_disabled()` | 用户可在 config.yaml 中禁用特定技能 |
| 工具依赖检查 | `_skill_should_show()` | 缺少依赖工具的技能不显示在索引中 |

---

## 8. 外部技能与插件系统

```
技能来源（按优先级排序）
    │
    ├─ 本地技能 ~/.hermes/skills/（最高优先级，可读写）
    │
    ├─ 外部目录（config.yaml 中的 skills.external_dirs，只读）
    │
    └─ 插件技能（qualified name，如 "my-plugin:skill"）
        通过 get_plugin_manager() 发现和加载
        支持插件包概念（一个插件可提供多个技能）
```

本地技能在名称冲突时总是优先。外部目录扫描时不使用磁盘快照（因为通常很小且可能被外部修改）。
