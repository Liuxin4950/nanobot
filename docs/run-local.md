# Nanobot 本地运行指南

## 环境要求

- Python 3.11+
- Windows/macOS/Linux

## 安装

### 1. 克隆并安装

```bash
git clone https://github.com/HKUDS/nanobot.git
cd nanobot
pip install -e .
```

### 2. 可选依赖

```bash
# Matrix 支持
pip install -e ".[matrix]"

# WeCom 支持
pip install -e ".[wecom]"

# Weixin 支持
pip install -e ".[weixin]"

# 开发依赖 (pytest, ruff)
pip install -e ".[dev]"
```

## 当前环境状态

| 检查项 | 状态 |
|--------|------|
| Python 版本 | ✅ 3.11+ |
| 核心依赖 | ✅ 已安装 |
| nanobot 模块 | ✅ 可导入 |

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
| `nanobot onboard` | 初始化配置和工作区 |
| `nanobot status` | 查看状态 |
| `nanobot agent -m "你好"` | 单条消息聊天 |
| `nanobot agent` | 交互式聊天 |
| `nanobot gateway` | 启动网关（连接所有启用的频道）|
| `nanobot channels login <channel>` | 频道登录（如 WhatsApp 扫码）|
| `nanobot channels status` | 显示频道状态 |
| `nanobot cron add/list/remove` | 管理定时任务 |

## 配置

1. 首次运行 `nanobot onboard` 初始化
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

### 使用上游仓库（HKUDS/nanobot）

如果你是从 fork 开发的，需要设置上游仓库：

```bash
git remote add upstream https://github.com/HKUDS/nanobot.git
git fetch upstream
git merge upstream/main
```

更新代码后重新安装：
```bash
pip install -e .
```

## 测试

```bash
# 测试 CLI 是否正常工作
nanobot --help

# 查看状态
nanobot status

# 运行测试
pytest
```

## Lint 和格式化

```bash
# Lint
ruff check .

# 格式化
ruff format .
```
