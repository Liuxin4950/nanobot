# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在本项目中工作提供指导。

## 项目简介

nanobot 是一个**超轻量级个人 AI 助手**（约 4000 行核心代码），使用 Python 编写。它连接各种聊天平台并通过 LiteLLM 使用 LLM 提供商来提供智能代理功能。

## 常用命令

```bash
# 以可编辑模式安装
pip install -e .

# 安装开发依赖（linting、测试）
pip install -e ".[dev]"

# 安装 matrix 支持
pip install -e ".[matrix]"

# 运行测试
pytest

# Lint 代码
ruff check .

# 格式化代码
ruff format .
```

## CLI 命令

| 命令 | 描述 |
|------|------|
| `nanobot onboard` | 初始化配置和工作区 |
| `nanobot agent -m "..."` | 与代理聊天 |
| `nanobot agent` | 交互式聊天模式 |
| `nanobot gateway` | 启动网关（连接已启用的聊天频道） |
| `nanobot status` | 显示状态 |
| `nanobot provider login <provider>` | 提供商 OAuth 登录（openai-codex、github-copilot） |
| `nanobot channels login` | 链接 WhatsApp（扫描二维码） |
| `nanobot channels status` | 显示频道状态 |
| `nanobot cron add/list/remove` | 管理定时任务 |

## 架构

### 核心组件

- **agent/** — 核心代理逻辑
  - `loop.py` — 主代理循环（LLM ↔ 工具执行）
  - `context.py` — 支持缓存的提示词构建器
  - `memory.py` — 带整合功能的持久化记忆
  - `skills.py` — 技能加载器（加载 SKILL.md 文件）
  - `tools/` — 内置工具（shell、filesystem、message、cron、MCP、web）

- **channels/** — 聊天平台集成
  - `telegram.py`, `discord.py`, `whatsapp.py`, `feishu.py`, `dingtalk.py`, `slack.py`, `email.py`, `qq.py`, `matrix.py`, `mochat.py`
  - 都继承自 `base.py` Channel 基类
  - 使用 `manager.py` 从配置加载/启用频道

- **providers/** — LLM 提供商
  - 使用 **LiteLLM** 提供统一 API
  - `registry.py` — 提供商注册表模式：添加新提供商只需 2 步（在注册表添加 ProviderSpec，在配置 schema 添加字段）
  - 支持：openrouter、anthropic、openai、deepseek、groq、gemini、minimax、volcengine、dashscope、moonshot、zhipu、vllm、custom、openai_codex、github_copilot

- **skills/** — 内置技能（每个目录包含 SKILL.md）
  - github、weather、summarize、tmux、clawhub、skill-creator、cron、memory

- **config/** — 配置管理
  - `schema.py` — Pydantic 配置模型验证
  - `loader.py` — 从 ~/.nanobot/config.json 加载配置

- **session/** — 对话会话管理
- **bus/** — 消息路由（事件、队列）
- **cron/** — 定时任务
- **heartbeat/** — 周期性唤醒任务

### 配置

- 配置文件：`~/.nanobot/config.json`
- 工作区：`~/.nanobot/workspace/`

### 添加新提供商

1. 在 `nanobot/providers/registry.py` 的 `PROVIDERS` 中添加 `ProviderSpec` 条目
2. 在 `nanobot/config/schema.py` 的 `ProvidersConfig` 中添加字段

环境变量、模型前缀和配置匹配都会自动工作。

## 测试

测试位于 `tests/` 目录，使用 pytest 的 asyncio 模式。运行单个测试文件：
```bash
pytest tests/test_filename.py
```

## Docker

```bash
docker build -t nanobot .
docker compose up -d nanobot-gateway
```
