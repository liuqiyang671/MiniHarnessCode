<div align="center">

# MiniHarnessCode

**轻量、本地、有记忆的终端 coding agent**

[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Version](https://img.shields.io/badge/Version-0.3.0-orange.svg)](pyproject.toml)

MiniHarnessCode 跑在本地仓库里，接上一个模型 provider，就能读代码、跑命令、改文件、
保留运行证据，并把有价值的上下文沉淀成本地记忆。

</div>

---

## 特性

| 能力 | 说明 |
| --- | --- |
| TUI / REPL / one-shot | 同一个 runtime，三种入口：Textual 终端界面、普通 REPL、一次性任务 |
| 多 provider 支持 | 兼容 OpenAI、Anthropic、DeepSeek 协议，配置灵活 |
| 工具执行 | 文件列表、读写、搜索、shell、patch、子 agent、todo |
| Plan mode | 先读代码和拆计划，再进入可写执行阶段 |
| 子 agent | 启动 bounded Explore / Worker 任务 |
| Skills | 复用 `/review`、`/test`、`/commit`、`/simplify` 等工作流 |
| 分层记忆 | working memory、daily logs、durable topics、auto-dream 后台整合 |
| 运行证据 | session JSON、event stream、run trace、task state、report |
| Sandbox | 对 `run_shell` 做可选隔离 |

## 快速开始

要求：Python 3.10+，以及至少一个可用的模型 provider key。

```bash
# 克隆并安装
git clone https://github.com/martin-los/pico.git
cd pico
pip install -e .

# 配置 provider
cp .pico.toml.example .pico.toml
# 编辑 .pico.toml，填入你的 api_key

# 启动
MiniHarnessCode
```

或使用一键安装脚本：

```bash
curl -fsSL https://raw.githubusercontent.com/martin-los/pico/main/install.sh | bash
```

开发模式下也可以直接跑：

```bash
uv run MiniHarnessCode
```

> 命名说明：用户可见启动命令和界面名称是 MiniHarnessCode；内部 Python 包名、源码目录、配置目录和历史数据路径仍保留 `pico` / `.pico`，以避免破坏现有导入和本地数据。

## 配置 provider

MiniHarnessCode 启动前先解析一个 **provider profile**。一个 profile 主要由四项组成：

| 字段 | 作用 |
| --- | --- |
| `protocol` | 请求协议，目前支持 `openai` 和 `anthropic`。 |
| `api_key` | 发给 provider 的 key。 |
| `base_url` | provider endpoint。 |
| `model` | 本次请求使用的模型名。 |

配置合并优先级是：

```text
CLI 参数 > 环境变量 > 项目 .pico.toml > 全局 ~/.config/pico/config.toml > 代码默认值
```

### 方式一：项目 `.pico.toml`（推荐）

```bash
cp .pico.toml.example .pico.toml
$EDITOR .pico.toml
```

`.pico.toml` 默认被 `.gitignore` 忽略，不要把真实 key 提交进 git。

最小可用示例：

```toml
provider = "deepseek"

[providers.deepseek]
protocol = "anthropic"
api_key = "sk-..."
base_url = "https://api.deepseek.com/anthropic"
model = "deepseek-v4-pro"

[providers.openai]
protocol = "openai"
api_key = "sk-..."
base_url = "https://api.openai.com/v1"
model = "gpt-4o"

[providers.anthropic]
protocol = "anthropic"
api_key = "sk-ant-..."
base_url = "https://api.anthropic.com"
model = "claude-sonnet-4-6"
```

注意：`provider = "deepseek"` 只是选择 profile 名字，真正决定请求格式的是
`protocol`。例如 DeepSeek 可以通过 Anthropic-compatible endpoint 使用，所以这里写
`protocol = "anthropic"`。

### 方式二：环境变量

```bash
export PICO_PROVIDER=deepseek
export DEEPSEEK_API_KEY=sk-...
export DEEPSEEK_BASE_URL=https://api.deepseek.com/anthropic
export DEEPSEEK_MODEL=deepseek-v4-pro

MiniHarnessCode
```

常用 provider 变量：

| Provider | 变量 |
| --- | --- |
| DeepSeek | `DEEPSEEK_API_KEY`, `DEEPSEEK_BASE_URL`, `DEEPSEEK_MODEL` |
| OpenAI-compatible | `OPENAI_API_KEY`, `OPENAI_BASE_URL`, `OPENAI_MODEL` |
| Anthropic-compatible | `ANTHROPIC_API_KEY`, `ANTHROPIC_BASE_URL`, `ANTHROPIC_MODEL` |

也可以用通用覆盖变量：

```bash
export PICO_API_KEY=sk-...
export PICO_BASE_URL=https://api.openai.com/v1
export PICO_MODEL=gpt-4o
```

### 方式三：命令行临时覆盖

```bash
MiniHarnessCode --provider openai --model gpt-4o --base-url https://api.openai.com/v1
MiniHarnessCode --provider deepseek --approval ask --max-steps 80
MiniHarnessCode --config /path/to/custom.toml --cwd /path/to/repo
```

完整配置说明见 [docs/configuration.md](docs/configuration.md)。

## 使用

### 启动方式

```bash
MiniHarnessCode                   # 默认 Textual TUI
MiniHarnessCode --repl            # 普通终端 REPL
MiniHarnessCode "找出测试失败的根因" # one-shot 任务
MiniHarnessCode --resume latest   # 续接最近 session
MiniHarnessCode --cwd /path/to/repo # 指定工作目录
```

### 常用参数

```bash
MiniHarnessCode --approval ask        # shell / 写文件前询问
MiniHarnessCode --approval auto       # 普通操作自动通过
MiniHarnessCode --approval never      # 非交互模式
MiniHarnessCode --sandbox best_effort # 尽量隔离 shell 命令
MiniHarnessCode --no-auto-dream       # 关闭后台 memory 整合
```

### Slash 命令

进入 TUI 或 REPL 后可以直接输入自然语言，也可以用 slash command：

```text
> /help
> /skills
> 找出测试失败的根因
> /plan 重构 provider 配置加载逻辑
> /review
> /test tests/test_config.py
> /remember 这个项目用 DeepSeek 的 Anthropic-compatible endpoint
> /dream
```

| 命令 | 作用 |
| --- | --- |
| `/help` | 查看内置命令 |
| `/skills` | 列出可用 skills |
| `/session` | 查看当前 session、events、run 路径 |
| `/history` | 列出历史 session |
| `/resume latest` | 续接最近 session |
| `/context` | 查看 prompt context 使用情况 |
| `/usage` | 查看 provider、model、token 元数据 |
| `/memory` | 查看 durable memory 索引 |
| `/working-memory` | 查看当前 session 工作记忆 |
| `/remember <text>` | 保存一条 durable note 到 daily log |
| `/dream` | 把 daily log 整合成 durable memory topics |
| `/plan <topic>` | 进入 plan mode |
| `/plan-exit` | 退出 plan mode |
| `/agents` | 查看子 agent 状态 |
| `/model <name>` | 当前 session 临时切模型 |
| `/compact` | 压缩较早的对话历史 |
| `/clear` | 开一个新的空 session |
| `/exit` | 退出 MiniHarnessCode |

## 架构概览

```text
pico/
├── cli.py                 # CLI 参数、启动模式、REPL 命令
├── config/                # provider profile、TOML、env 解析
├── core/                  # runtime、engine、session、workers、context
│   ├── runtime.py         #   MiniHarnessCode runtime 主类（类名仍为 Pico）
│   ├── engine.py          #   turn 控制循环
│   ├── context_manager.py #   prompt 构建
│   ├── tool_executor.py   #   工具调度
│   ├── worker_manager.py  #   子 agent 管理
│   └── plan_mode.py       #   plan mode 工作流
├── features/              # memory、skills、sandbox
│   ├── memory.py          #   分层记忆系统
│   ├── skills.py          #   技能插件系统
│   └── sandbox/           #   shell 沙箱隔离
├── providers/             # OpenAI / Anthropic compatible client
├── tools/                 # tool registry 和具体工具
├── tui/                   # Textual TUI
└── evaluation/            # run evidence、metrics、evaluation
```

## 本地数据

| 数据 | 路径 |
| --- | --- |
| 项目配置 | `.pico.toml` |
| 全局配置 | `~/.config/pico/config.toml` |
| 会话历史 | `.pico/sessions/<id>.json` |
| 事件流 | `.pico/sessions/<id>.events.jsonl` |
| 运行证据 | `.pico/runs/<run_id>/` |
| 记忆索引 | `.pico/memory/MEMORY.md` |
| Daily logs | `.pico/memory/logs/YYYY/MM/YYYY-MM-DD.md` |
| Durable topics | `.pico/memory/topics/*.md` |
| 用户 skills | `~/.pico/skills/<name>/SKILL.md` |
| 项目 skills | `skills/<name>/SKILL.md` 或 `.pico/skills/<name>/SKILL.md` |

## 测试

```bash
pip install -e ".[dev]"
pytest tests/ -q

# 真实 provider 烟测需要 key
PICO_LIVE_SMOKE=1 pytest tests/test_release_smoke.py -q
```

## 文档

| 入口 | 内容 |
| --- | --- |
| [配置](docs/configuration.md) | provider profile、`.pico.toml`、环境变量和 sandbox 配置 |
| [分层记忆 + auto-dream](docs/memory.md) | working memory、daily logs、durable topics 和后台整合 |
| [Skills](docs/skills.md) | `SKILL.md` 目录结构、内置技能和自定义 workflow |
| [Sandbox](docs/sandbox.md) | `run_shell` 隔离模式、backend 选择和文件系统边界 |

## License

MIT
