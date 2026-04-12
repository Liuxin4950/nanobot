# Nanobot 渐进式学习路径

这是一个超详细的项目学习指南，帮助你从浅入深理解 nanobot 的架构。

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
nanobot/agent/loop.py      ← 核心编排器（消息分派、会话锁）
nanobot/agent/runner.py    ← 底层执行引擎（LLM 调用、工具执行）
nanobot/agent/context.py    ← 提示词构建
nanobot/agent/memory.py     ← 记忆系统（MemoryStore, Dream, Consolidator）
nanobot/providers/          ← LLM 提供商
nanobot/agent/tools/        ← AI 能调用的工具
```

---

## 阶段二：理解 Agent 系统（核心）

### 2.1 双层架构：AgentLoop + AgentRunner

**为什么分成两层？**

- `AgentLoop` 负责编排：消息路由、会话锁、pending 队列、cron、heartbeat
- `AgentRunner` 负责执行：LLM 调用、工具执行、hooks、checkpoint

```
AgentLoop._process_message()
        ↓
AgentRunner.run() → AsyncIterator[RunnerEvent]
        ↓
LLM.chat() + ToolRegistry.execute()
```

### 2.2 主循环流程 (`agent/loop.py`)

```
┌─────────────────────────────────────────────────────────────┐
│                    run() → _dispatch()                       │
│                               ↓                              │
│            _process_message()                               │
│              ↓ 检查 slash command                           │
│              ↓ auto_compact.prepare_session()               │
│              ↓ consolidator.maybe_consolidate_by_tokens()  │
│              ↓ build_messages() ← 组装提示词                 │
│              ↓ AgentRunner.run()                            │
│                               ↓                              │
│    ┌──────────────────────────────────────────────┐        │
│    │  for 最多 40 轮:                              │        │
│    │    1. provider.chat() ← 发送消息给 AI         │        │
│    │    2. 检查返回:                                │        │
│    │       - 如果有 tool_calls → 执行工具 → 循环   │        │
│    │       - 如果有 content   → 返回答案           │        │
│    └──────────────────────────────────────────────┘        │
│              ↓ _save_turn() → SessionManager.save()        │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 AgentRunner 详解 (`agent/runner.py`)

```python
@dataclass
class AgentRunSpec:
    messages: list[dict]
    tools: list[dict]
    provider: LLMProvider
    hooks: list[AgentHook]
    max_iterations: int = 40
    injection_cb: Callable | None = None

async def run(self, spec: AgentRunSpec) -> AsyncIterator[RunnerEvent]:
    """Async iterator - 每个事件 yield 出来"""
    for iteration in range(spec.max_iterations):
        # 1. before_iteration hooks
        await self._call_hooks("before_iteration", ctx)

        # 2. LLM chat
        response = await spec.provider.chat(spec.messages, tools=spec.tools)

        # 3. on_stream hooks (token by token)
        if response.thinking_blocks:
            for block in response.thinking_blocks:
                yield RunnerEvent("delta", block["text"])
                await self._call_hooks("on_stream", ctx, delta)

        # 4. tool_calls 执行
        if response.tool_calls:
            for tc in response.tool_calls:
                result = await self.tools.execute(tc.name, tc.arguments)
                spec.messages.append({role: "tool", ...})
                yield RunnerEvent("tool_result", (tc.id, result))

        # 5. after_iteration hooks
        await self._call_hooks("after_iteration", ctx, response)

class RunnerEvent(NamedTuple):
    type: str   # "delta", "tool_call", "tool_result", "done", "error"
    data: Any
```

### 2.4 小巧思 1：per-session 锁 + mid-turn injection

```python
# AgentLoop._dispatch()
async def _dispatch(self, msg):
    key = self._effective_session_key(msg)
    async with self._session_locks[key]:  # 同一会话串行
        if key in self._pending_queues:
            # mid-turn: 消息在处理中，新消息排队
            self._pending_queues[key].put_nowait(msg)
        else:
            # 没有 pending，开始处理
            self._pending_queues[key] = asyncio.Queue()
            await self._process_message(msg)
            # 处理完，释放锁
            del self._session_locks[key]
```

---

## 阶段三：理解工具系统

### 3.1 工具架构

```
Tool (抽象基类)
    │
    ├── Schema (参数验证 + 类型转换)
    │
    └── ToolRegistry (统一管理)
            │
            ├── 内置工具 (filesystem, shell, web, cron, spawn, ...)
            └── MCPToolWrapper (包装 MCP 服务器工具)
```

### 3.2 工具基类 (`tools/base.py`)

```python
class Tool(ABC):
    @property
    def name(self) -> str: ...        # 工具名，如 "read_file"

    @property
    def description(self) -> str: ... # 描述，AI 靠它理解用途

    @property
    def parameters(self) -> dict: ... # JSON Schema 参数定义

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
# 1. 转换为 OpenAI function calling 格式
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

### 3.4 小巧思 2：参数验证 Schema

```python
# tools/schema.py
class Schema:
    @staticmethod
    def validate(params: dict, schema: dict) -> list[str]:
        """验证参数，返回错误列表（空=有效）"""
        errors = []
        # 检查必填
        for required in schema.get("required", []):
            if required not in params:
                errors.append(f"{required} 是必填参数")
        # 检查类型
        for key, spec in schema["properties"].items():
            if key in params:
                expected = spec["type"]
                actual = type(params[key]).__name__
                if not isinstance(params[key], ...):  # type check
                    errors.append(f"{key} 应该是 {expected}")
        return errors

    @staticmethod
    def coerce(params: dict, schema: dict) -> dict:
        """类型转换"""
        # "20" (str) → 20 (int)
        # {"path": "/a/b"} → normalize path
        return params
```

### 3.5 小巧思 3：ExecTool 安全防护

```python
# tools/shell.py - ExecTool 实现了多层防护：

# 1. 危险命令黑名单
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",     # 阻止 rm -rf
    r"\bdel\s+/[fq]\b",          # 阻止 del /f
    r"\bformat\b",               # 阻止 format
    r"\bshutdown\b",             # 阻止 shutdown
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
│   6. cron_service.build_context() → 定时任务摘要            │
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

---

## 阶段五：理解 LLM 提供商

### 5.1 Provider 架构

```
┌─────────────────────────────────────────────┐
│           LLMProvider (抽象基类)            │
│   - chat() 发送聊天请求                      │
│   - chat_stream() 流式                      │
│   - _is_transient_error() 重试判断           │
└─────────────────────────────────────────────┘
                      ↓
    ┌─────────────────────────────────────┐
    │ OpenAICompatProvider (OpenAI 兼容)   │
    │ AnthropicProvider (原生 SDK)         │
    │ AzureOpenAIProvider                  │
    │ GitHubCopilotProvider / OpenAICodexProvider │
    └─────────────────────────────────────┘
```

**不是所有提供商都用 LiteLLM**：
- Anthropic 使用官方 `anthropic` SDK
- GitHub Copilot 和 OpenAI Codex 使用 OAuth
- Azure OpenAI 有自己的 provider
- 大多数使用 OpenAI 兼容接口

### 5.2 添加新提供商的"2 步法则"

**第 1 步：** 在 `providers/registry.py` 添加 ProviderSpec

```python
ProviderSpec(
    name="myprovider",              # 配置中用的名字
    keywords=("myprovider", "mymodel"),  # 模型名关键词
    env_key="MYPROVIDER_API_KEY",   # 环境变量
    backend="openai_compat",        # 大多数是 openai_compat
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

### 5.3 小巧思 4：自动检测 + 重试机制

```python
# providers/base.py - LLMProvider

# 重试判断
def _is_transient_error(self, e) -> bool:
    """是否是临时性错误（网络、限流等）"""
    # 429 Rate Limit → True
    # 500 Server Error → True
    # 401 Auth Error → False
    # ...

def _extract_retry_after(self, e) -> float | None:
    """从响应头解析 Retry-After"""
    # Retry-After: 30
    # Retry-After: Fri, 31 Dec 2026 23:59:59 GMT
```

---

## 阶段六：理解消息通道

### 6.1 Channel 架构

```
BaseChannel (抽象基类)
    │
    ├── telegram.py, discord.py, feishu.py, whatsapp.py
    ├── qq.py, weixin.py, dingtalk.py, wecom.py
    ├── matrix.py, email.py, slack.py
    ├── websocket.py (WebSocket 服务器)
    └── mochat.py
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

### 6.3 小巧思 5：统一的访问控制

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

## 阶段七：理解记忆系统（新）

### 7.1 三层架构

```
┌─────────────────────────────────────┐
│       session.messages              │
│    (短中期对话，上下文窗口内)         │
└───────────────┬─────────────────────┘
                │ maybe_consolidate_by_tokens()
┌───────────────▼─────────────────────┐
│      memory/history.jsonl           │
│  (append-only, cursor-based)        │
└───────────────┬─────────────────────┘
                │ Dream (cron, 默认 2h)
┌───────────────▼─────────────────────┐
│  SOUL.md / USER.md / MEMORY.md      │
│  (GitStore 版本化)                   │
└─────────────────────────────────────┘
```

### 7.2 Dream 两阶段

**Phase 1**: LLM 总结历史
```python
# memory.py - Dream
async def phase1_summarize(self, unconsolidated) -> str:
    """用 LLM 总结未整合的消息"""
    prompt = f"总结以下对话的关键信息：\n{unconsolidated}"
    return await self.provider.chat([{"role": "user", "content": prompt}])
```

**Phase 2**: LLM 更新记忆文件
```python
# Phase 2: LLM 用工具更新 SOUL.md / USER.md / MEMORY.md
# 工具: save_memory(path, content)
# 受 max_iterations 预算限制
```

### 7.3 GitStore 版本化

```python
class GitStore:
    """记忆文件的 Git 版本化"""
    def save(self, path, content, sha=None) -> str:
        """保存并 commit"""
        # 写入文件 → git add → git commit
        # 返回新的 commit SHA

    def restore(self, sha) -> list[str]:
        """恢复到指定版本"""
        # git show SHA:path → 返回文件内容列表
```

### 7.4 小巧思 6：AutoCompact 主动压缩

```python
# autocompact.py
class AutoCompact:
    """会话空闲超过 session_ttl_minutes 时压缩"""
    def prepare_session(self, session) -> bool:
        """返回 True 表示已压缩，下次需要注入摘要"""
        if session.idle_time > self.session_ttl_minutes:
            # 1. 调用 Dream 压缩历史到 history.jsonl
            # 2. 保留最近几条消息
            # 3. 返回 True 表示需要注入摘要
            return True
```

---

## 阶段八：理解 Cron 系统

### 8.1 三种调度类型

```python
@dataclass
class CronSchedule:
    type: Literal["at", "every", "cron"]
    # at: 一次性 timestamp (ms)
    # every: 固定间隔 (ms)
    # cron: cron 表达式 + 可选时区
```

### 8.2 CronService

```python
class CronService:
    def __init__(self, store_path, on_job):
        self.store = CronStore(store_path)  # 持久化
        self._arm_timer()  # 设置下一个闹钟

    async def _on_timer(self):
        """timer 触发，运行所有到期任务"""
        due = self.store.get_due_jobs()
        await asyncio.gather(*[self._execute_job(j) for j in due])
        self._arm_timer()

    def _execute_job(self, job):
        """执行 job，调用 on_job callback"""
        # on_job → AgentLoop.process_direct()
        # 结果通过 bus.outbound → Channel → 用户
```

---

## 推荐学习顺序

```
第 1 天：
  1. 阅读 agent/loop.py（理解 AgentLoop 编排器）
  2. 阅读 agent/runner.py（理解 AgentRunner 执行引擎）
  3. 阅读 tools/base.py + tools/registry.py（理解工具系统）

第 2 天：
  4. 阅读 agent/context.py（理解提示词构建）
  5. 阅读 providers/base.py + providers/registry.py（理解提供商系统）
  6. 试着加一个新提供商

第 3 天：
  7. 阅读 channels/base.py + channels/manager.py（理解消息通道）
  8. 阅读 agent/skills.py（理解技能系统）
  9. 试着加一个新通道

第 4 天：
  10. 阅读 agent/memory.py（理解 Dream + MemoryStore）
  11. 阅读 cron/service.py（理解定时任务）
  12. 深入理解整体架构
```

---

## 关键文件索引

| 文件 | 行数 | 核心功能 |
|------|------|----------|
| `agent/loop.py` | 640+ | AgentLoop 编排器 |
| `agent/runner.py` | 817 | AgentRunner 执行引擎 |
| `agent/context.py` | 110+ | 提示词构建 |
| `agent/memory.py` | 770 | MemoryStore, Dream, Consolidator |
| `agent/tools/base.py` | 100+ | 工具基类 |
| `agent/tools/registry.py` | 60+ | 工具注册执行 |
| `agent/tools/filesystem.py` | 633 | 文件系统工具 |
| `providers/base.py` | 470 | LLMProvider 接口 |
| `providers/registry.py` | ~300 | 提供商注册表 |
| `channels/base.py` | 130+ | Channel 基类 |
| `channels/manager.py` | 250+ | 通道管理 |
| `cron/service.py` | 246+ | CronService |
| `config/schema.py` | ~400 | Pydantic 配置模型 |

---

## 常见问题

**Q: AgentLoop 和 AgentRunner 有什么区别？**
A: AgentLoop 是编排器（路由、会话锁、pending 队列）；AgentRunner 是执行引擎（LLM 调用、工具执行、hooks）。

**Q: 如何添加新工具？**
A: 继承 Tool 类，实现 name/description/parameters/execute 即可。

**Q: 如何添加新通道？**
A: 继承 BaseChannel，实现 start/stop/send，注册到 manager。

**Q: Dream 和 Consolidator 有什么区别？**
A: Consolidator 在 token 超过阈值时触发（Phase1 总结到 history.jsonl）；Dream 是 cron 周期性运行（Phase2 从 history.jsonl 更新到长期记忆文件）。

**Q: 技能和工具的区别？**
A: 工具是 AI 可以调用的函数；技能是 AI 可以读取的文档（SKILL.md），提供领域知识。
