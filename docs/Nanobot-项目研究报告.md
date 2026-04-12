# Nanobot 项目研究报告

## 项目简介

nanobot 是一个超轻量级的个人 AI 助手框架，核心代码仅约 4000 行。它通过连接各种聊天平台（telegram, discord, whatsapp, feishu 等）并使用 LLM 提供商来提供智能代理功能。

---

## 一、API 能力实现架构

### 1. LLM 提供商系统 (`nanobot/providers/`)

**核心文件**:
- `base.py` - 抽象基类定义 (`LLMProvider`)
- `litellm_provider.py` - 主要实现，使用 LiteLLM 库
- `registry.py` - 提供商注册表

**工作原理**:
```
用户配置 → ProviderSpec → LiteLLM → 20+ LLM 提供商
```

支持的提供商包括:
- OpenRouter, Anthropic, OpenAI, DeepSeek, Groq, Gemini
- MiniMax, VolcEngine, DashScope, Moonshot, Zhipu
- vLLM (本地), 自定义 OpenAI 兼容端点

**添加新提供商的步骤** (仅需2步):
1. 在 `providers/registry.py` 添加 `ProviderSpec` 条目
2. 在 `config/schema.py` 添加配置字段

---

### 2. 工具系统 (`nanobot/agent/tools/`)

**核心架构**:
```python
# base.py - 抽象工具类
class Tool(ABC):
    name: str           # 工具名称
    description: str    # 工具描述
    parameters: dict    # JSON Schema 参数定义

    async def execute(self, **kwargs) -> str
    def to_schema(self) -> dict  # 转换为 OpenAI function 格式
```

**工具注册表** (`registry.py`):
- 动态注册/注销工具
- `get_definitions()` - 返回所有工具定义
- `execute()` - 验证参数并执行工具

**内置工具**:
| 工具 | 文件 | 功能 |
|------|------|------|
| Filesystem | `filesystem.py` | 读写文件、列出目录 |
| Shell | `shell.py` | 执行命令 (有安全防护) |
| Message | `message.py` | 发送消息到聊天频道 |
| Web | `web.py` | 网页搜索、抓取 |
| Cron | `cron.py` | 定时任务 |
| Spawn | `spawn.py` | 生成子代理 |

---

### 3. MCP 集成 (`nanobot/agent/tools/mcp.py`)

**功能**:
- 将 MCP 服务器工具包装为原生 nanobot 工具
- 支持 stdio 模式 (本地进程)
- 支持 HTTP 模式 (远程服务器)
- 工具名称前缀 `mcp_{server_name}_` 避免冲突

**配置格式** (在 `config.json`):
```json
{
  "tools": {
    "mcpServers": {
      "filesystem": {
        "command": "npx",
        "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path"]
      }
    }
  }
}
```

---

### 4. 代理核心循环 (`nanobot/agent/loop.py`)

`AgentLoop` 是核心处理引擎:

```
1. Build Context (ContextBuilder 组装提示词)
2. Call LLM (provider.chat() 发送消息+工具定义)
3. Check Response:
   - 如果有 tool_calls → 执行工具 → 循环
   - 如果有 content → 返回内容 → 结束
4. Error Handling (错误处理)
```

**关键方法**:
- `_run_agent_loop()` - 主循环
- `_process_message()` - 处理消息、命令、内存整合

---

### 5. 上下文构建 (`nanobot/agent/context.py`)

`ContextBuilder` 组装提示词:
1. **Identity** - 核心身份和 workspace 信息
2. **Bootstrap files** - AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md
3. **Memory** - 长期记忆 (MEMORY.md)
4. **Skills** - 已加载技能摘要
5. **Runtime context** - 当前时间、频道、聊天 ID

---

### 6. 内存系统 (`nanobot/agent/memory.py`)

两层记忆:
- **MEMORY.md** - 长期事实 (通过 LLM 更新)
- **HISTORY.md** - 可搜索的对话历史

当历史超过 `memory_window` 时自动整合

---

## 二、架构图

```
                    +------------------+
                    |   Message Bus    |
                    | (Inbound/Out)   |
                    +--------+---------+
                             |
                    +--------v---------+
                    |    AgentLoop     |
                    | (Main Controller)|
                    +--------+---------+
                             |
        +--------------------+--------------------+
        |                    |                    |
+-------v-------+  +---------v------+  +-----------v--------+
| ContextBuilder|  | ToolRegistry   |  |  LLMProvider      |
| (Prompts)     |  | (Tool Mgmt)    |  | (OpenAI/Anthropic)|
+---------------+  +--------+-------+  +---------+----------+
                           |                      |
            +--------------+---------------+      |
            |              |               |      |
      +-----v---+   +------v------+  +-----v--+  |
      | Built-in|   | MCP Tools   |  | LiteLLM|  |
      | Tools   |   | (External)  |  |  ...   |  |
      +---------+   +-------------+  +---------+  |
```

---

## 三、关键文件索引

| 文件路径 | 用途 |
|---------|------|
| `nanobot/providers/base.py` | 抽象 LLM 提供商接口 |
| `nanobot/providers/litellm_provider.py` | 主要多提供商实现 |
| `nanobot/providers/registry.py` | 提供商元数据和查找 |
| `nanobot/agent/tools/base.py` | 抽象 Tool 类 |
| `nanobot/agent/tools/registry.py` | 工具注册和执行 |
| `nanobot/agent/tools/mcp.py` | MCP 服务器集成 |
| `nanobot/agent/loop.py` | 核心代理循环逻辑 |
| `nanobot/agent/context.py` | 提示词/上下文组装 |
| `nanobot/agent/memory.py` | 记忆整合 |
| `nanobot/config/schema.py` | Pydantic 配置模型 |

---

## 四、学习建议

### 入门路径:
1. **先看 loop.py** - 理解代理如何与 LLM 交互
2. **再看 tools/** - 理解工具如何定义和执行
3. **然后 providers/** - 理解如何连接不同 LLM
4. **最后 channels/** - 理解如何连接聊天平台

### 核心概念:
- **Tool Calling** - LLM 返回工具调用请求
- **Prompt Caching** - Anthropic/OpenRouter 提示词缓存
- **Provider Registry** - 提供商发现和自动选择
- **Skills** - SKILL.md 格式的技能定义
- **Memory Consolidation** - 历史自动整合为长期记忆
