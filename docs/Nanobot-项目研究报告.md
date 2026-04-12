# Nanobot 项目研究报告

## 项目简介

nanobot 是一个超轻量级个人 AI 助手框架（约 4000 行核心代码），使用 Python 编写。它连接各种聊天平台并通过 LiteLLM 使用 LLM 提供商来提供智能代理功能。

**当前版本**: v0.1.5

---

## 一、核心架构

### 架构总览

```
                      ┌─────────────────────────────────────┐
                      │           ChannelManager             │
                      │   (发现、启动、管理所有 Channel)      │
                      └──────────┬──────────────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
   ┌─────v─────┐  ┌──────────────v──┐  ┌──────────────v──────┐
   │  Telegram │  │     Feishu      │  │     WebSocket       │
   │  Discord │  │     QQ          │  │     WhatsApp        │
   │  Dingtalk│  │     ...        │  │     ...             │
   └─────┬─────┘  └───────────────┬──┘  └──────────────────────┘
         │                       │                          │
         └───────────────────────┼──────────────────────────┘
                                 │ MessageBus
                    ┌────────────v────────────┐
                    │      AgentLoop          │
                    │  (核心代理循环引擎)       │
                    └────────────┬────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
  ┌──────v──────┐  ┌─────────────v─────┐  ┌──────────────v──────┐
  │AgentRunner  │  │  ToolRegistry    │  │   LLMProvider      │
  │(执行引擎)    │  │  (工具注册执行)    │  │   (LLM 调用)       │
  └──────┬──────┘  └─────────────┬─────┘  └──────────────┬──────┘
         │                       │                       │
  ┌──────v──────┐  ┌─────────────v─────┐  ┌──────────────v──────┐
  │MemoryStore  │  │  内置工具+MCP     │  │   OpenAI/Anthropic │
  │Dream       │  │  (16+ 工具)        │  │   30+ 提供商        │
  │Consolidator│  └───────────────────┘  └────────────────────┘
  └────────────┘
```

### 消息流向

```
用户消息 (Telegram/Discord/...)
    ↓
Channel._on_message()
    ↓ InboundMessage
MessageBus.inbound queue
    ↓
AgentLoop.run() [per-session lock]
    ↓ _dispatch() → _process_message()
    ↓ build_messages() [system + history + current]
    ↓ AgentRunner.run() [async iterator]
    ↓ LLM chat() + tool execution
    ↓ SessionManager.save()
    ↓ OutboundMessage
MessageBus.outbound queue
    ↓
ChannelManager._dispatch_outbound()
    ↓
Channel.send() → 用户
```

---

## 二、Agent 系统

### 核心组件

| 文件 | 类/模块 | 职责 |
|------|---------|------|
| `agent/loop.py` | `AgentLoop` | 主代理编排器，消息分派，会话锁，pending 队列 |
| `agent/runner.py` | `AgentRunner` | 底层执行引擎，async iterator 产生 RunnerEvent |
| `agent/runner.py` | `AgentRunSpec` | 运行参数数据类 |
| `agent/context.py` | `ContextBuilder` | 组装提示词（system + history + runtime context）|
| `agent/memory.py` | `MemoryStore` | Git 持久化的长期记忆 |
| `agent/memory.py` | `Dream` | 周期性记忆整合（Phase1 总结 + Phase2 更新文件）|
| `agent/memory.py` | `Consolidator` | Token 阈值触发整合 |
| `agent/autocompact.py` | `AutoCompact` | 空闲会话主动压缩 |
| `agent/hook.py` | `AgentHook` | 生命周期钩子接口 |
| `agent/hook.py` | `CompositeHook` | 多钩子组合（错误隔离）|
| `agent/skills.py` | `SkillsLoader` | 技能发现和加载 |

### AgentLoop 详解

`AgentLoop` 是核心编排器：

```python
class AgentLoop:
    def __init__(self, bus, provider, sessions, tools, cron, ...):
        self._running = True
        self._session_locks = {}      # per-session 互斥
        self._pending_queues = {}      # mid-turn 消息队列

    async def run(self):
        """主循环：消费 inbound → 处理 → 写入 outbound"""
        while self._running:
            msg = await self.bus.consume_inbound()
            await self._dispatch(msg)

    async def _dispatch(self, msg):
        """per-session 锁，支持 mid-turn injection"""
        key = self._effective_session_key(msg)
        async with self._session_locks[key]:
            if key in self._pending_queues:
                self._pending_queues[key].put_nowait(msg)
            else:
                await self._process_message(msg)

    async def _process_message(self, msg):
        """消息处理主路径"""
        # 1. slash command 检查
        # 2. auto_compact.prepare_session()
        # 3. consolidator.maybe_consolidate_by_tokens()
        # 4. context.build_messages()
        # 5. AgentRunner.run() → LLM + tools
        # 6. _save_turn() → SessionManager.save()
```

### AgentRunner 详解

`AgentRunner` 是底层执行引擎（async iterator 模式）：

```python
class AgentRunner:
    async def run(self, spec: AgentRunSpec) -> AsyncIterator[RunnerEvent]:
        """
        AgentRunSpec 包含：
        - messages: list[dict]
        - tools: list[dict]
        - provider: LLMProvider
        - hooks: list[AgentHook]
        - max_iterations: int
        - injection_cb: callback for mid-turn
        """
        for iteration in range(spec.max_iterations):
            # 1. before_iteration hooks
            # 2. LLM chat()
            # 3. on_stream hooks (token by token)
            # 4. before_execute_tools hooks
            # 5. tool execution
            # 6. after_iteration hooks
            # 7. yield RunnerEvent

@dataclass
class AgentRunSpec:
    messages: list[dict]
    tools: list[dict]
    provider: LLMProvider
    hooks: list[AgentHook]
    max_iterations: int = 40
    injection_cb: Callable | None = None

class RunnerEvent(NamedTuple):
    type: str  # "delta", "tool_call", "tool_result", "done", "error"
    data: Any
```

### Hook 系统

```python
class AgentHook(ABC):
    async def before_iteration(self, ctx) -> None: ...
    async def on_stream(self, ctx, delta: str) -> None: ...
    async def before_execute_tools(self, ctx) -> None: ...
    async def after_iteration(self, ctx, response) -> None: ...
    def finalize_content(self, ctx, content: str) -> str: ...  # pipeline

class CompositeHook(AgentHook):
    """fan-out 到多个 hook，错误隔离"""
```

---

## 三、工具系统

### 架构

```
Tool (ABC)
    │
    ├── Schema (参数验证 + 类型转换)
    │
    ├── ToolRegistry (动态注册/执行)
    │       │
    │       ├── 内置工具 (filesystem, shell, web, cron, message, spawn, ...)
    │       └── MCPToolWrapper (MCP 服务器工具包装)
    │
    └── 内置工具
            ├── ReadFile / WriteFile / EditFile / Glob / Grep / ListDir
            ├── ExecTool (有安全防护)
            ├── WebFetchTool / WebSearchTool
            ├── CronTool (add/list/remove 定时任务)
            ├── MessageTool (发送消息到频道)
            ├── SpawnTool (生成子代理)
            ├── NotebookTool (Jupyter notebook)
            ├── SandboxTool (沙箱执行)
            └── FileStateTool (文件锁定状态)
```

### 工具基类

```python
class Tool(ABC):
    @property
    def name(self) -> str: ...

    @property
    def description(self) -> str: ...

    @property
    def parameters(self) -> dict: ...  # JSON Schema

    async def execute(self, **kwargs) -> str: ...

class Schema:
    """参数验证 + 类型转换"""
    @staticmethod
    def validate(params: dict, schema: dict) -> list[str]: ...
    @staticmethod
    def coerce(params: dict, schema: dict) -> dict: ...
```

### 安全防护 (ExecTool)

```python
# 1. 危险命令黑名单
deny_patterns = [
    r"\brm\s+-[rf]{1,2}\b",   # rm -rf
    r"\bdel\s+/[fq]\b",        # del /f
    r"\bformat\b",             # format
    r"\bshutdown\b",            # shutdown
    ...
]

# 2. 可选允许列表
allow_patterns = [r"^git\s+", r"^ls\s+"]

# 3. 路径遍历防护
# 4. 工作区限制 (restrict_to_workspace)
# 5. 超时控制
```

### MCP 集成

```python
class MCPToolWrapper(Tool):
    """包装 MCP 服务器工具为原生 nanobot 工具"""
    # 工具名前缀: mcp_{server_name}_
    # 支持 stdio (本地进程) 和 HTTP 模式
```

---

## 四、内存系统 (新)

### 三层架构

```
┌─────────────────────────────────────────┐
│         session.messages                │
│    (短中期对话，上下文窗口内)             │
└────────────────┬────────────────────────┘
                 │ maybe_consolidate_by_tokens()
┌────────────────▼────────────────────────┐
│       memory/history.jsonl              │
│  (append-only, cursor-based, 压缩摘要)   │
└────────────────┬────────────────────────┘
                 │ Dream (cron)
┌────────────────▼────────────────────────┐
│   SOUL.md / USER.md / MEMORY.md         │
│   (GitStore 版本化长期记忆)              │
└─────────────────────────────────────────┘
```

### MemoryStore

```python
class MemoryStore:
    """Git 持久化的长期记忆"""
    def __init__(self, workspace: Path):
        self.git_store = GitStore(workspace / "memory")
        self.history_file = workspace / "memory" / "history.jsonl"

    async def get_memory_context(self) -> str: ...
    async def save(self, path: str, content: str, sha: str | None) -> str: ...
    def restore(self, sha: str) -> list[str]: ...
```

### Dream

Dream 是周期性记忆整合，分两阶段：

**Phase 1**: 用 LLM 总结未整合的历史
```
读取 history.jsonl 的新条目 + 当前 SOUL.md/USER.md/MEMORY.md
→ LLM 提取重要信息和更新
```

**Phase 2**: 用工具更新长期记忆文件
```
LLM 调用 save_memory 工具更新 SOUL.md/USER.md/MEMORY.md
(受 max_iterations 预算限制)
```

### AutoCompact

主动压缩空闲会话：

```python
class AutoCompact:
    """当会话空闲超过 session_ttl_minutes 时压缩"""
    def prepare_session(self, session: Session) -> bool:
        """返回 True 表示已压缩，需要注入摘要"""
```

---

## 五、Provider 系统

### 支持的提供商 (~30 个)

| 类型 | 提供商 |
|------|--------|
| OpenAI 兼容 | openrouter, deepseek, groq, gemini, moonshot, zhipu, minimax, dashscope, volcengine, stepfun, xiaomi_mimo, ... |
| 原生 SDK | anthropic (Anthropic SDK), azure_openai, openai_codex, github_copilot |
| 本地 | vllm, ollama, ovms |
| 自定义 | custom |

### ProviderSpec 注册表

```python
@dataclass
class ProviderSpec:
    name: str                    # 配置名
    keywords: tuple[str, ...]    # 模型名关键词
    env_key: str                # 环境变量
    backend: str                # openai_compat / anthropic / azure_openai / openai_codex / github_copilot
    is_gateway: bool = False
    is_local: bool = False
    supports_max_completion_tokens: bool = False
    supports_prompt_caching: bool = False
    detect_by_key_prefix: str | None = None
```

### 添加新提供商 (2 步)

1. 在 `providers/registry.py` 添加 `ProviderSpec`
2. 在 `config/schema.py` 的 `ProvidersConfig` 添加字段

### LLMProvider 基类

```python
class LLMProvider(ABC):
    @abstractmethod
    async def chat(self, messages, tools, **kwargs) -> LLMResponse: ...

    async def chat_stream(self, messages, tools, **kwargs) -> AsyncIterator[str]: ...

    def _is_transient_error(self, e) -> bool: ...
    def _extract_retry_after(self, e) -> float | None: ...
    def _enforce_role_alternation(self, messages) -> list: ...
    def _sanitize_empty_content(self, messages) -> list: ...
```

### 重试机制

- `standard` 模式: 3 次重试
- `persistent` 模式: 最多 10 次重试，延迟可达 60s
- `_RETRY_HEARTBEAT_CHUNK = 30`: 长重试期间可取消

---

## 六、Channel 系统

### 内置 Channel

| Channel | 文件 | 说明 |
|---------|------|------|
| Telegram | `telegram.py` | Bot API |
| Discord | `discord.py` | Bot API |
| Feishu | `feishu.py` | 飞书/Lark |
| WhatsApp | `whatsapp.py` | WhatsApp Business |
| QQ | `qq.py` | QQ 机器人 |
| WeChat | `weixin.py` | 企业微信 |
| DingTalk | `dingtalk.py` | 钉钉 |
| WeCom | `wecom.py` | 企业微信 |
| Matrix | `matrix.py` | Matrix 协议 |
| Email | `email.py` | SMTP/IMAP |
| WebSocket | `websocket.py` | WebSocket 服务器 |
| Slack | `slack.py` | Slack |
| MoChat | `mochat.py` | MoChat |

### BaseChannel

```python
class BaseChannel(ABC):
    name: str           # "telegram", "discord", ...
    display_name: str   # "Telegram", "Discord", ...

    def __init__(self, config, bus: MessageBus): ...

    async def start(self) -> None: ...     # 必须阻塞
    async def stop(self) -> None: ...      # 设置 _running = False
    async def send(self, msg: OutboundMessage) -> None: ...

    # 可选: 流式
    async def send_delta(self, chat_id, delta, metadata) -> None: ...

    # 基类提供
    async def _handle_message(self, sender_id, chat_id, content, media?, metadata?, session_key?): ...
    def is_allowed(self, sender_id) -> bool: ...
    async def login(self, force=False) -> bool: ...
    async def transcribe_audio(self, file_path) -> str: ...
```

### ChannelManager

```python
class ChannelManager:
    def discover_all(self) -> list[BaseChannel]: ...
    async def start_all(self) -> None: ...
    async def stop_all(self) -> None: ...
    def dispatch_outbound(self, msg: OutboundMessage) -> None: ...
```

---

## 七、Cron 系统

### 架构

```
┌──────────────────────────────────────┐
│          CronService                 │
│  - 持久化到 workspace/cron/jobs.json │
│  - FileLock + action.jsonl 安全      │
│  - 三种调度: at / every / cron       │
└────────────────┬────────────────────┘
                 │ on_job callback
                 ▼
         AgentLoop.process_direct()
```

### CronSchedule 类型

```python
@dataclass
class CronSchedule:
    type: Literal["at", "every", "cron"]
    value: int | str  # timestamp / interval_ms / cron_expr
    timezone: str | None = None
```

### API

```python
service.add_job(id, schedule, payload, on_job, system?)
service.remove_job(id)
service.enable_job(id, bool)
service.list_jobs() -> list[CronJob]
service.run_job(id)
```

---

## 八、会话系统

### Session 数据类

```python
@dataclass
class Session:
    key: str                          # channel:chat_id
    messages: list[dict]              # 消息历史
    metadata: dict                    # 运行时信息
    last_consolidated: int = 0        # 已整合的 offset

    def get_history(max_messages) -> list[dict]: ...
    def retain_recent_legal_suffix(): ...
```

### 运行时检查点

AgentLoop 在处理过程中保存检查点，崩溃后可恢复未完成的 tool call（标记为 error）。

---

## 九、配置系统

### Schema

```python
class Config(BaseModel):
    agents: AgentsConfig
    channels: ChannelsConfig
    providers: ProvidersConfig
    api: ApiConfig
    gateway: GatewayConfig
    tools: ToolsConfig

class AgentsConfig(BaseModel):
    defaults: AgentDefaults
    unified_session: bool = False

class AgentDefaults(BaseModel):
    model: str
    provider: str | None
    timezone: str
    max_tokens: int | None
    context_window_tokens: int
    temperature: float
    disabled_skills: list[str]
    session_ttl_minutes: int
    dream: DreamConfig

class DreamConfig(BaseModel):
    interval_h: float = 2.0
    model_override: str | None
    max_batch_size: int = 20
    max_iterations: int = 10
```

### 环境变量

支持 `NANOBOT__` 前缀嵌套:
```bash
NANOBOT__PROVIDERS__OPENROUTER__API_KEY=sk-xxx
NANOBOT__AGENTS__DEFAULTS__MODEL=gpt-4o-mini
```

---

## 十、关键文件索引

| 文件 | 行数 | 核心功能 |
|------|------|----------|
| `agent/loop.py` | 640+ | AgentLoop 主循环 |
| `agent/runner.py` | 817 | AgentRunner 执行引擎 |
| `agent/memory.py` | 770 | MemoryStore, Dream, Consolidator |
| `agent/context.py` | 110+ | ContextBuilder 提示词构建 |
| `agent/tools/filesystem.py` | 633 | 文件系统工具 |
| `agent/tools/shell.py` | 224 | Shell 执行工具 |
| `agent/tools/search.py` | 555 | 搜索工具 |
| `providers/base.py` | 470 | LLMProvider 基类 |
| `providers/registry.py` | ~300 | ProviderSpec 注册表 |
| `channels/base.py` | 130+ | BaseChannel |
| `channels/manager.py` | 250+ | ChannelManager |
| `channels/websocket.py` | 457 | WebSocket 服务器 Channel |
| `cron/service.py` | 246+ | CronService |
| `config/schema.py` | ~400 | Pydantic 配置模型 |

---

## 十一、新增组件 (v0.1.5)

### AgentRunner (替代原 loop.py 的部分逻辑)

- `AgentRunSpec` 数据类封装所有运行参数
- Async iterator 模式 (`RunnerEvent`)
- 内置重试 (`_run_with_retry`)
- 支持并发工具执行

### MemoryStore + GitStore

- Git 版本化的记忆文件
- `git_store.save()` / `git_store.restore()`
- `history.jsonl` append-only 日志

### AutoCompact

- `session_ttl_minutes` 配置
- 空闲会话自动压缩到 Dream
- 恢复时注入摘要作为 runtime context

### WebSocket Channel

- 双向 WebSocket 通信
- Token 认证 + 短期 token 发放
- TLS/SSL 支持
- Delta 流式输出

### 新工具

- `NotebookTool` - Jupyter notebook 操作
- `SandboxTool` - 沙箱命令执行
- `FileStateTool` - 文件锁定状态
- `Schema` - 统一参数验证
