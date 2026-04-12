# Nanobot 渐进式学习路径

这是一个超详细的项目学习指南，帮助你从浅入深理解 nanobot 如何控制 AI。

---

## 阶段一：核心概念入门（1-2小时）

### 1.1 先看这个故事：AI Agent 是怎么工作的？

想象你在指挥一个助手：
1. **你说话** → 发送给 AI
2. **AI 思考** → 决定要不要调用工具
3. **AI 调用工具** → 如"帮我查天气"
4. **工具执行** → 返回结果
5. **AI 继续思考** → 基于结果继续回复或再次调用工具
6. **最终回答** → 返回给你

这就是 nanobot 的核心工作方式！

### 1.2 核心文件位置

```
nanobot/agent/loop.py     ← 最重要！AI 循环在这里
nanobot/agent/context.py   ← 提示词怎么构建
nanobot/providers/        ← 怎么连接各种 AI
nanobot/agent/tools/      ← AI 能调用什么工具
```

---

## 阶段二：理解 AI 循环（核心）

### 2.1 主循环流程 (`agent/loop.py`)

**核心方法：`_run_agent_loop()`**

```
┌─────────────────────────────────────────────────────────────┐
│                     run() → _dispatch()                     │
│                              ↓                              │
│              _process_message()                             │
│                              ↓                              │
│              build_messages() ← 组装提示词                   │
│                              ↓                              │
│                   _run_agent_loop()                         │
│                              ↓                              │
│    ┌──────────────────────────────────────────────┐        │
│    │  for 最多 40 轮:                              │        │
│    │    1. provider.chat() ← 发送消息给 AI         │        │
│    │    2. 检查返回:                                │        │
│    │       - 如果有 tool_calls → 执行工具 → 循环   │        │
│    │       - 如果有 content   → 返回答案           │        │
│    └──────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关键代码解析

**loop.py 第 177-254 行：主循环**

```python
async def _run_agent_loop(self, messages: list[dict]) -> LLMResponse:
    """核心循环：AI 思考 → 工具调用 → 重复"""
    max_iterations = 40

    for iteration in range(max_iterations):
        # 第 1 步：调用 AI
        response = await self.provider.chat(
            messages=messages,
            tools=self.tools.get_definitions(),  # 把工具交给 AI
        )

        # 第 2-1 步：AI 要求调用工具
        if response.tool_calls:
            # 执行每个工具
            for tool_call in response.tool_calls:
                result = await self.tools.execute(
                    tool_call.name,
                    tool_call.arguments
                )
                # 把工具结果添加回消息历史
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
            # 继续循环，AI 会看到工具结果继续思考

        # 第 2-2 步：AI 直接回答
        else:
            return response  # 完成！
```

### 2.3 小巧思 1：非流式调用 + 进度回调

**为什么不用流式？**
- 代码更简单
- 进度通过 callback 模拟推送

```python
# loop.py 第 191 行
response = await self.provider.chat(...)  # 不是流式！

# 收到 AI 的 thinking_blocks 时，通过 on_progress 推送
if response.thinking_blocks:
    for block in response.thinking_blocks:
        await self._on_progress(block["text"])
```

---

## 阶段三：理解工具系统

### 3.1 工具架构

```
Tool (抽象基类)
    ↓
┌─────────────────────────────────────────┐
│  ReadFileTool  │  WriteFileTool         │
│  ExecTool     │  WebSearchTool          │
│  MessageTool  │  CronTool               │
└─────────────────────────────────────────┘
    ↓
ToolRegistry (统一管理)
```

### 3.2 工具基类 (`tools/base.py`)

```python
class Tool(ABC):
    # 必须实现的属性
    @property
    def name(self) -> str: ...        # 工具名，如 "read_file"

    @property
    def description(self) -> str: ...  # 描述，AI 靠它理解用途

    @property
    def parameters(self) -> dict: ...  # JSON Schema 参数定义

    # 必须实现的方法
    async def execute(self, **kwargs) -> str: ...  # 执行逻辑
```

### 3.3 工具注册表 (`tools/registry.py`)

**注册工具：**
```python
registry.register(ReadFileTool())
registry.register(ExecTool())
```

**AI 调用工具时：**
```python
# 1. 转换为 OpenAI 格式
definitions = registry.get_definitions()
# [
#   {
#     "type": "function",
#     "function": {
#       "name": "read_file",
#       "description": "...",
#       "parameters": {...}
#     }
#   }
# ]

# 2. 执行工具
result = await registry.execute("read_file", {"path": "/home/user/test.txt"})
```

### 3.4 小巧思 2：工具安全防护

**shell.py 的 ExecTool 实现了多层防护：**

```python
# 1. 危险命令黑名单
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",     # 阻止 rm -rf
    r"\bdel\s+/[fq]\b",          # 阻止 del /f
    r"\bformat\b",               # 阻止 format
    r"\bshutdown\b",             # 阻止关机
    # ...
]

# 2. 允许列表（可选）
allow_patterns = [r"^git\s+", r"^ls\s+"]

# 3. 路径遍历防护
if ".." in command or command.startswith("/"):
    # 阻止访问上级目录或绝对路径

# 4. 工作区限制
if restrict_to_workspace:
    # 确保命令只在 workspace 目录内执行
```

### 3.5 小巧思 3：参数验证

```python
# base.py - validate_params() 方法
def validate_params(self, params: dict) -> list[str]:
    """验证参数，返回错误列表（空=有效）"""
    schema = self.parameters
    errors = []

    # 检查类型
    if schema["properties"]["path"]["type"] != type(params["path"]):
        errors.append(f"path 应该是 {schema['type']}")

    # 检查必填
    for required in schema.get("required", []):
        if required not in params:
            errors.append(f"{required} 是必填参数")

    return errors
```

---

## 阶段四：理解提示词构建

### 4.1 ContextBuilder (`context.py`)

提示词是这样组装的：

```
┌─────────────────────────────────────────────────────────────┐
│ build_system_prompt()                                       │
│   1. _get_identity()         → 运行时信息（OS、Python版本）  │
│   2. _load_bootstrap_files() → AGENTS.md, SOUL.md 等       │
│   3. memory.get_memory_context() → 长期记忆                 │
│   4. skills.get_always_skills() → 必加载技能               │
│   5. skills.build_skills_summary() → 所有技能列表           │
│                              ↓                              │
│   用 "\n\n---\n\n" 连接所有部分                                 │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 消息构建 (`build_messages`)

```python
def build_messages(self, history, current_message, ...):
    return [
        {"role": "system", "content": self.build_system_prompt()},
        ...history,                    # 对话历史
        {"role": "user", "content": runtime_context},  # 时间、频道
        {"role": "user", "content": user_content},     # 当前消息
    ]
```

### 4.3 小巧思 4：提示词缓存

```python
# litellm_provider.py
def _apply_cache_control(self, messages, tools):
    """给系统消息加缓存标记"""
    for msg in messages:
        if msg["role"] == "system":
            # 添加 cache_control: {"type": "ephemeral"}
            msg["content"] = [{
                "type": "text",
                "text": content,
                "cache_control": {"type": "ephemeral"}
            }]
    return messages, tools
```

**缓存效果：** Anthropic/OpenRouter 会缓存系统提示词，省钱又提速！

---

## 阶段五：理解 LLM 提供商

### 5.1 Provider 架构

```
┌─────────────────────────────────────────────┐
│           LLMProvider (抽象基类)            │
│   - chat() 发送聊天请求                      │
│   - get_default_model()                     │
└─────────────────────────────────────────────┘
                      ↓
    ┌─────────────────────────────────────┐
    │ LiteLLMProvider (主要实现)           │
    │ - 支持 20+ 提供商（OpenAI, Anthropic）│
    │ - 自动模型前缀处理                    │
    │ - 支持提示词缓存                       │
    └─────────────────────────────────────┘
```

### 5.2 添加新提供商的"2 步法则"

**第 1 步：** 在 `providers/registry.py` 添加 ProviderSpec

```python
ProviderSpec(
    name="myprovider",              # 配置中用的名字
    keywords=("myprovider", "mymodel"),  # 模型名关键词
    env_key="MYPROVIDER_API_KEY",   # 环境变量
    litellm_prefix="myprovider",   # 自动前缀
)
```

**第 2 步：** 在 `config/schema.py` 添加配置字段

```python
class ProvidersConfig(BaseModel):
    myprovider: ProviderConfig = ProviderConfig()  # 就这样！
```

**自动获得的能力：**
- ✅ 环境变量自动设置
- ✅ 模型名自动前缀
- ✅ `nanobot status` 显示
- ✅ API Key 前缀检测

### 5.3 小巧思 5：提供商自动检测

```python
# registry.py - find_gateway() 逻辑
def find_gateway(api_key: str = None, api_base: str = None):
    # 1. 通过 API key 前缀检测
    if api_key and api_key.startswith("sk-or-"):
        return OpenRouter  # OpenRouter 的 key 以 sk-or- 开头

    # 2. 通过 API URL 关键词检测
    if api_base and "openrouter" in api_base:
        return OpenRouter

    # 3. 通过模型名关键词检测
    if model.startswith("claude"):
        return Anthropic
```

---

## 阶段六：理解消息通道

### 6.1 Channel 架构

```
BaseChannel (抽象基类)
    ↓
┌──────────────────────────────────────────┐
│ TelegramChannel  DiscordChannel          │
│ FeishuChannel    WhatsAppChannel        │
│ SlackChannel     EmailChannel           │
│ ...                                     │
└──────────────────────────────────────────┘
```

### 6.2 消息流向

```
用户 (Telegram/Discord/...)
    ↓ 发送消息
Channel._on_message()
    ↓ 转换为 InboundMessage
MessageBus.publish_inbound()
    ↓
AgentLoop.run()
    ↓ 生成回复
MessageBus.publish_outbound()
    ↓
ChannelManager._dispatch_outbound()
    ↓ 找到对应 Channel
Channel.send()
    ↓ 发给用户
```

### 6.3 小巧思 6：统一的访问控制

```python
# base.py - is_allowed()
def is_allowed(self, sender_id: str) -> bool:
    allow_list = self.config.allow_from or []
    if not allow_list:
        return True  # 空=允许所有人

    # 支持复合 ID：Telegram 用 "12345|username" 格式
    return any(part in allow_list for part in str(sender_id).split("|"))
```

---

## 阶段七：理解技能系统

### 7.1 技能格式 (SKILL.md)

```yaml
---
name: github
description: 使用 gh CLI 与 GitHub 交互
always: false  # 是否默认加载
metadata:
  nanobot:
    requires:
      bins: ["gh"]      # 需要 gh 命令
      env: ["GITHUB_TOKEN"]
---

# GitHub Skill

使用 `gh` 命令与 GitHub 交互...

## 可用命令
- `gh repo list`
- `gh issue list`
```

### 7.2 技能加载机制

```python
# skills.py
def list_skills(self):
    # 扫描两个目录
    # 1. workspace/skills/ （用户自定义）
    # 2. nanobot/skills/ （内置）
    return [skill for skill in dirs if skill/SKILL.md.exists()]

def build_skills_summary(self):
    # 生成 XML 格式的技能列表
    return """<skills>
  <skill>
    <name>github</name>
    <description>...</description>
    <location>nanobot/skills/github/SKILL.md</location>
  </skill>
</skills>"""
```

### 7.3 小巧思 7：按需加载

```python
# context.py - 构建提示词时
# 1. always=true 的技能 → 自动加载到提示词
always_skills = skills.get_always_skills()
if always_skills:
    parts.append(load_skills_for_context(always_skills))

# 2. 其他技能 → 列出摘要，AI 需要时自己读取
parts.append(build_skills_summary())
# AI 会自己用 read_file 工具读取需要的技能
```

---

## 阶段八：理解记忆系统

### 8.1 两层记忆

```
┌─────────────────────────────────────┐
│         MEMORY.md                   │
│    长期记忆（重要事实）               │
│    - 用户偏好                        │
│    - 关键信息                        │
│    - 由 LLM 定期更新                  │
└─────────────────────────────────────┘
              ↓ 自动整合
┌─────────────────────────────────────┐
│         HISTORY.md                  │
│    对话历史（可搜索）                 │
│    - 带时间戳的完整对话              │
│    - 可用 grep 搜索                   │
└─────────────────────────────────────┘
```

### 8.2 记忆整合触发

```python
# memory.py
async def maybe_consolidate(self, messages: list):
    if len(messages) > self.window_size:
        # 调用 LLM 来提取重要事实
        await self._consolidate_with_llm(messages)
```

### 8.3 小巧思 8：自动记忆更新

```python
# 整合时，LLM 收到指令：
"""
分析以下对话历史，提取需要长期记住的重要信息。
如果需要更新 MEMORY.md，调用 save_memory 工具。
"""
```

---

## 推荐学习顺序

```
第 1 天：
  1. 阅读 loop.py（理解 AI 循环）
  2. 阅读 tools/base.py + registry.py（理解工具系统）
  3. 试着加一个新工具

第 2 天：
  4. 阅读 context.py（理解提示词构建）
  5. 阅读 providers/registry.py（理解提供商系统）
  6. 试着加一个新提供商

第 3 天：
  7. 阅读 channels/base.py + manager.py（理解消息通道）
  8. 阅读 skills.py（理解技能系统）
  9. 试着加一个新通道

第 4 天：
  10. 阅读 memory.py（理解记忆系统）
  11. 阅读一个完整的 channel 实现（如 telegram.py）
  12. 深入理解整体架构
```

---

## 关键文件索引

| 文件 | 行数 | 核心功能 |
|------|------|----------|
| `agent/loop.py` | 500+ | AI 循环、工具调用 |
| `agent/context.py` | 200+ | 提示词构建 |
| `agent/tools/base.py` | 100+ | 工具基类 |
| `agent/tools/registry.py` | 60+ | 工具注册执行 |
| `providers/base.py` | 120+ | Provider 接口 |
| `providers/litellm_provider.py` | 300+ | 主 Provider 实现 |
| `providers/registry.py` | 450+ | 提供商注册表 |
| `channels/base.py` | 130+ | Channel 基类 |
| `channels/manager.py` | 250+ | 通道管理 |
| `agent/skills.py` | 220+ | 技能加载 |
| `agent/memory.py` | 180+ | 记忆系统 |

---

## 常见问题

**Q: 为什么不使用流式？**
A: 非流式代码更简单，进度通过 callback 模拟。思想链（thinking）会推送。

**Q: 如何添加新工具？**
A: 继承 Tool 类，实现 name/description/parameters/execute 即可。

**Q: 如何添加新通道？**
A: 继承 BaseChannel，实现 start/stop/send，导入到 manager.py。

**Q: 技能和工具的区别？**
A: 工具是 AI 可以调用的函数；技能是 AI 可以读取的文档（SKILL.md）。
