# Nanobot 本地运行指南

## 环境要求

- Python 3.11+
- Windows/macOS/Linux

## 当前环境状态

| 检查项 | 状态 |
|--------|------|
| Python 版本 | ✅ 3.11.0 (满足 >=3.11) |
| 核心依赖 | ✅ 已安装 (typer, pydantic, rich, httpx 等) |
| nanobot 模块 | ✅ 可导入 |

## 安装缺失依赖

nanobot 需要以下依赖，如果运行时提示缺少模块，执行：

```bash
pip install prompt-toolkit litellm loguru croniter json-repair oauth-cli-kit
```

完整依赖列表（从 pyproject.toml）：

```bash
# 核心依赖
pip install typer litellm pydantic pydantic-settings websockets websocket-client \
    httpx oauth-cli-kit loguru readability-lxml rich croniter dingtalk-stream \
    python-telegram-bot lark-oapi socksio python-socketio msgpack slack-sdk \
    slackify-markdown qq-botpy python-socks prompt-toolkit mcp json-repair

# 可选依赖
pip install -e ".[matrix]"  # Matrix 支持
pip install -e ".[dev]"    # 开发依赖 (pytest, ruff)
```

## Windows 编码问题

Windows 控制台默认编码是 GBK，无法显示 emoji。需要设置 UTF-8 编码。

### 方法 1：使用启动脚本（推荐）

在项目根目录创建 `run.bat`：

```bat
@echo off
chcp 65001 >nul
set PYTHONIOENCODING=utf-8
python -m nanobot %*
```

然后运行：
```bat
run.bat --help
run.bat agent -m "你好"
run.bat status
run.bat onboard
```

### 方法 2：直接设置环境变量

```cmd
set PYTHONIOENCODING=utf-8
python -m nanobot --help
```

### 方法 3：在 VS Code 终端中运行

在 VS Code 中打开终端（默认 UTF-8），然后直接运行：

```bash
python -m nanobot --help
```

## 常用命令

| 命令 | 说明 |
|------|------|
| `python -m nanobot onboard` | 初始化配置和工作区 |
| `python -m nanobot status` | 查看状态 |
| `python -m nanobot agent -m "你好"` | 单条消息聊天 |
| `python -m nanobot agent` | 交互式聊天 |
| `python -m nanobot gateway` | 启动网关 |

## 配置

1. 首次运行 `python -m nanobot onboard` 初始化
2. 编辑 `~/.nanobot/config.json` 添加 API Key
3. 推荐使用 OpenRouter（免费 API），示例配置：

```json
{
  "providers": {
    "openrouter": {
      "api_key": "your-api-key-here"
    }
  },
  "agents": {
    "defaults": {
      "model": "openai/gpt-4o-mini"
    }
  }
}
```

## 测试

```bash
# 测试 CLI 是否正常工作
python -m nanobot --help

# 查看状态
python -m nanobot status
```
