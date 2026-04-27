# OpenHands 系统性架构解剖报告

> 研究对象：OpenHands 主仓库（v1 架构，2026-04 最新）  
> 研究目标：彻底拆解其技术体系，支撑「复刻简化版」与「面试讲解」两个能力目标。

---

## 目录

- [0. 全景先摆在前面](#0-全景先摆在前面)
- [1. 架构层（Architecture）](#1-架构层architecture)
- [2. 功能边界（Capability & Scope）](#2-功能边界capability--scope)
- [3. 运行环境（Runtime Model）](#3-运行环境runtime-model)
- [4. 核心对象生命周期（Core Object Lifecycle）](#4-核心对象生命周期core-object-lifecycle)
- [5. 数据存储与数据流（Data Flow & Storage）](#5-数据存储与数据流data-flow--storage)
- [6. 面向复刻的实施指南](#6-面向复刻的实施指南)
- [7. 面试讲解 Cheat Sheet](#7-面试讲解-cheat-sheet)
- [8. 学习路径建议](#8-学习路径建议)
- [9. 确认 / 推测标注总览](#9-确认--推测标注总览)

---

## 0. 全景先摆在前面

OpenHands 实际上是一个**多产品矩阵**，而不是一个单体项目。阅读源码前必须先搞清楚这个分层，否则很容易在代码里迷路。

| 层级 | 产物 | 所在位置 | 职责 |
|---|---|---|---|
| L1 Agent Core (SDK) | `openhands-sdk` PyPI 包 | 另一仓库 `software-agent-sdk`（不在本仓库） | ReAct loop、`Agent.step()`、LLM 调用、工具调度 |
| L2 Agent Server | `openhands-agent-server` PyPI 包 | 另一仓库（不在本仓库） | 在沙箱容器内部运行的 HTTP 服务，驱动 Agent Core 执行动作、维护事件流 |
| L3 App Server（GUI 后端） | 本仓库 `openhands/app_server/` + `openhands/server/` | 本仓库 | 多会话编排、沙箱生命周期、存储抽象、REST API |
| L4 Frontend | `frontend/` React SPA | 本仓库 | 用户界面 |
| L5 Enterprise | `enterprise/` | 本仓库 | 多租户、SSO、集成（Slack / Jira / Linear）、K8s |

> **关键洞察**：复刻时，L1 是「大脑」（必做），L3 是「调度中枢」（简化必做），L2 是「沙箱代理」（可以省掉，直接把 Agent Core 内嵌运行），L4 / L5 都可以先不做。

```text
  浏览器 / CLI
       │ HTTP + WebSocket
       ▼
┌────────────────────────┐        ┌─────────────────────────────┐
│   App Server (L3)      │  HTTP  │  Agent Server (L2)          │
│  本仓库 openhands/     │───────▶│  容器内，外部包             │
│  - 多会话管理          │ /api/  │  - 驱动 Agent Core (L1)     │
│  - 沙箱编排            │◀──────│  - 产出 Event Stream        │
│  - 持久化              │Webhook│  - 执行 bash / edit / browse│
└────────────────────────┘        └─────────────────────────────┘
       │                                      │
       ▼                                      ▼
  FileStore / Postgres           Workspace / Tools / MCP
```

---

## 1. 架构层（Architecture）

### 1.1 Summary

OpenHands v1 是一个分层清晰的多租户 Agent 编排系统，采用了四个关键设计模式：

1. **Agent-Server 分离**：Agent 逻辑（L1 SDK）和会话托管（L3 App Server）彻底解耦，后者通过 HTTP 调用前者。
2. **Event-Sourcing**：所有交互都是 Event（Action / Observation），事件是系统的真相。
3. **ReAct loop（在 SDK 里）**：`plan → tool call → observation → plan → ...`，无显式 Planner-Executor 拆分，是一个「单 Agent + 多工具」的架构，而不是 AutoGPT 那种多 Agent 协作。
4. **Dependency Injection**：App Server 的 Injector 模式让 Sandbox / ConversationStore / EventService 等都是可插拔的抽象。

### 1.2 模块边界（本仓库 `openhands/` 内）

```text
openhands/
├── core/              # 配置、日志、schema（骨架）
├── events/            # Event 定义 + Action/Observation 类型 + EventStream + EventStore
├── app_server/        # ★ v1 当前主入口：REST API、会话服务、沙箱服务、DI
│   ├── app_conversation/    # 会话生命周期（最核心）
│   ├── sandbox/             # Docker / Process / Remote 三种沙箱服务
│   ├── event/               # EventService (filesystem / S3 / Firestore)
│   ├── event_callback/      # webhook 回调处理
│   ├── pending_messages/    # 沙箱未就绪时的消息队列
│   ├── settings/ secrets/   # 配置与密钥
│   ├── user/                # 用户上下文
│   └── services/            # 依赖注入框架
├── server/            # v0 旧入口（已 deprecated，2026-04 下线）
├── storage/           # FileStore 抽象 (local / S3 / GCS / memory)
│   └── conversation/  # ConversationStore
├── llm/               # LiteLLM 封装 + 重试 + 成本追踪 + 路由
├── mcp/               # MCP Client（消费外部 MCP server）
├── integrations/      # GitHub / GitLab / Bitbucket / AzDO / Forgejo
├── resolver/          # 基于 issue 的 Agent 触发器
└── utils/
```

### 1.3 模块调用关系：同步、异步、事件驱动

```text
HTTP 请求 ──► FastAPI Router ──► *Service (async)
                                    │
                                    ├─ 同步：对 Storage 的写入（async I/O）
                                    ├─ 异步：对 Agent Server 的 HTTP 调用 (httpx.AsyncClient)
                                    └─ 事件：Agent Server 通过 webhook 回推 Event
                                              └─► EventService.append() ──► FileStore
                                                       │
                                                       └─► 订阅者回调（可观测性、UI push）
```

- **同步链路**：用户首次 `POST` 消息 → `pending_messages` 表（SQL INSERT）。
- **异步调用链**：App Server 用 `httpx.AsyncClient` 非阻塞调用 Agent Server。
- **事件驱动链路**：Agent Server 通过 `OH_WEBHOOKS_0_BASE_URL` 把 Event 推回 App Server，App Server 再分发给订阅者（UI / EventCallback / Metrics）。

### 1.4 设计点评

| 优点 | 潜在问题 |
|---|---|
| Agent / Server 分离，Agent 可独立演进（SDK 独立仓库） | 两个 HTTP hop（UI → App Server → Agent Server）增加延迟 |
| 纯 Event-Sourcing，天然支持 replay、审计、回放调试 | `pickle` 用于 `agent_state.pkl`，不可跨版本读取，也有安全风险 |
| DI 设计让本地 Docker / 云 K8s / in-process 统一抽象 | v0 / v1 并存，`openhands/server/` 和 `openhands/app_server/` 迁移期代码混乱 |
| 多租户原生：`user_id` 从 API 层就贯穿到存储路径 | 状态分散在 FileStore + SQL + Agent Server 三处，一致性靠约定 |

---

## 2. 功能边界（Capability & Scope）

### 2.1 Summary

OpenHands 是一个**通用软件工程 Agent**，定位最贴近 Devin / Claude Code：给它一个代码仓库和 issue，它能独立地读写代码、跑测试、调浏览器、开 PR。它不是一个通用 Autonomous AGI 框架。

### 2.2 已确认支持的核心能力

| 能力 | 支持方式 | 关键代码 |
|---|---|---|
| Shell / 命令执行 | `CmdRunAction` → 沙箱内执行 → `CmdOutputObservation`（含 `exit_code`） | `events/action/commands.py` |
| 文件读写 / 编辑 | `FileReadAction` / `FileWriteAction` / `FileEditAction` | `events/action/files.py` |
| IPython 执行 | `IPythonRunCellAction`（Jupyter kernel gateway） | 依赖 `jupyter-kernel-gateway` |
| 网页浏览 | `BrowseURLAction` / `BrowseInteractiveAction` + Playwright（容器内） | `events/action/browse.py` |
| 原生 Function Calling | 自动识别模型能力（Claude / GPT-4o / o3 / Gemini 原生；其他走 XML prompt 模拟） | `llm/fn_call_converter.py` |
| MCP 工具接入 | 客户端支持 HTTP / SSE / stdio 三种 transport | `mcp/client.py` |
| 多模型 | LiteLLM：100+ provider；per-agent 配置；Condenser 独立 LLM | `llm/llm.py` + `core/config/llm_config.py` |
| 上下文压缩 | 10 种 Condenser 策略（NoOp / Recent / LLMSummarizing / ...） | `core/config/condenser_config.py` |
| Git 集成 | GitHub / GitLab / Bitbucket / Azure DevOps / Forgejo 原生 | `integrations/` |
| Issue Resolver | 触发 Agent 从 issue 直接产出 PR | `resolver/` |

### 2.3 弱支持 / 不支持的能力边界

| 能力 | 状态 | 备注 |
|---|---|---|
| 多 Agent 协作 | 仅有 `AgentDelegateAction` 原语，无 autogen / crewai 式编排 | 不是框架定位 |
| 自研长期记忆 | 无向量库；「记忆」= 事件流 + Condenser 摘要 | 没有 RAG / Memory Store |
| LLM 输出流式 | `llm.completion()` 主路径不支持 streaming；`streaming_llm.py` 是 v0 遗留 | 返回 `ModelResponse` 而非 chunk |
| Windows 原生 | MCP 层在 Windows 直接禁用 | `mcp/utils.py:72-76` |
| 离线纯 CPU 推理 | 支持 Ollama，但官方目标是云模型 | Kimi / Qwen / Ollama 都可 |

### 2.4 与同类产品的差异点

| 对比对象 | 差异 |
|---|---|
| AutoGPT | OpenHands 是会话式 + 人在回路（UI 可中断、可继续），AutoGPT 是一次性任务跑完 |
| Devin | 定位几乎相同，但 OpenHands 开源 + 自托管，沙箱通过 Docker 而非私有基础设施 |
| Claude Code | Claude Code 是 CLI 单会话在你本地直接跑；OpenHands 把 Agent 隔离到容器里，更适合多会话多租户 |
| LangChain / LlamaIndex Agents | 它们是组件库；OpenHands 是端到端产品（含 UI、持久化、部署） |

---

## 3. 运行环境（Runtime Model）

### 3.1 Summary

OpenHands 有 3 种沙箱实现 + 4 种部署形态，但都抽象在同一个 `SandboxService` 接口背后。

> 一个沙箱 = 一个独立 Agent Server 实例。

### 3.2 三种 Sandbox Service（可插拔）

```text
        SandboxService (ABC)
           │
  ┌────────┼────────────────────┐
  ▼        ▼                    ▼
Docker    Process             Remote
 ↓         ↓                    ↓
docker.from_env()    subprocess.Popen()    httpx → api.openhands.dev
容器                  本机进程               云端 Pod
```

| 模式 | 启动方式 | 适用场景 | 关键文件 |
|---|---|---|---|
| Docker | `docker.containers.run(image="ghcr.io/openhands/agent-server:1.15.0-python")` | 本地 GUI / 私有部署 | `app_server/sandbox/docker_sandbox_service.py` |
| Process | `subprocess.Popen` 启动 agent-server 进程 | 开发 / 测试 | `app_server/sandbox/process_sandbox_service.py` |
| Remote | `POST /sessions` 到远端 runtime 控制面 | 云 / K8s | `app_server/sandbox/remote_sandbox_service.py` |

### 3.3 一次沙箱启动的完整过程（Docker 模式）

```text
[App Server]
    │ 1. 生成 session_api_key = base62(os.urandom(32))
    │ 2. docker.containers.run(
    │      image="ghcr.io/openhands/agent-server:1.15.0-python",
    │      environment={
    │         OH_SESSION_API_KEYS_0=<key>,
    │         OH_WEBHOOKS_0_BASE_URL=http://host.docker.internal:3000/api/v1/webhooks,
    │         OH_CONVERSATIONS_PATH=/workspace/conversations,
    │         LLM_* (转发)
    │      },
    │      volumes={ workspace: /workspace/project },
    │      ports={ 8000, 60001 (vscode), 12000, 12001 })
    │ 3. 轮询 GET /alive 直到健康
    │ 4. POST /api/conversations { task, workspace, ... }
    ▼
[Agent Server 容器]
    - 启动 Agent Core (SDK)
    - 启动 OpenVSCode Server on :60001 (带 ?tkn=session_api_key)
    - 进入 ReAct loop
    - 每个 Event 通过 webhook 回推
```

### 3.4 隔离机制

| 隔离维度 | 实现 |
|---|---|
| 进程 / 命名空间 | Docker 容器（或 K8s Pod）自带 |
| 文件系统 | `/workspace/project` volume mount；每会话子目录（非 `NO_GROUPING` 时） |
| 认证 | `X-Session-API-Key` HTTP 头，每沙箱独立；无 JWT / OAuth |
| 网络 | host 或 bridge 模式；agent server 只允许 webhook 回 `host.docker.internal` |
| KVM 硬件虚拟化（可选） | `/dev/kvm` 设备直通，用于更强隔离 |

### 3.5 LLM 调用管理

- **封装层**：`openhands/llm/llm.py`（`LLM` 类），底层是 `litellm.completion()`。
- **配置方式**：`config.template.toml` 的 `[llm]` / `[llm.<name>]` 段 → `LLMConfig` Pydantic。
- **多模型策略**：`LLMRegistry` + `AgentConfig.llm_config`，每个 Agent 可以指定不同 LLM；Condenser 独立 LLM（常配更便宜的模型）。
- **重试机制**：`RetryMixin` + `tenacity`，对 `RateLimitError` / `Timeout` / `APIConnectionError` / `LLMNoResponseError` 指数退避；遇到空响应时动态把 `temperature` 从 0 抬到 1.0。
- **Prompt Caching**：自动为 Anthropic 模型加 `cache_control: ephemeral`。
- **成本追踪**：`Metrics` 类按调用累加；支持 `max_budget_per_task` 预算熔断。

### 3.6 设计点评

| 优点 | 潜在问题 |
|---|---|
| Docker / Process / Remote 统一接口，复刻时先做 Docker 一个就够 | Session key 是唯一鉴权，log / 监控中必须脱敏 |
| `max_num_conversations_per_sandbox` 支持多会话共享沙箱，节省资源 | 共享沙箱模式下，会话间文件系统隔离依赖目录约定，不够强 |
| LiteLLM 一键接入 100+ provider | Streaming 不支持是明显缺口 |

---

## 4. 核心对象生命周期（Core Object Lifecycle）

### 4.1 Conversation / Session

状态机来自 `data_models/conversation_status.py` + `app_conversation_service.py`。

```text
                      ┌── ERROR ◀──────┐
                      │                │
  [CREATE]            │   失败         │
   │                  │                │
   ▼                  │                │
 STARTING ──► WORKING ──► WAITING_FOR_SANDBOX ──► PREPARING_REPOSITORY
                │                                          │
                │                                          ▼
                │                              RUNNING_SETUP_SCRIPT
                │                                          │
                │                                          ▼
                │                              SETTING_UP_GIT_HOOKS
                │                                          │
                │                                          ▼
                │                              SETTING_UP_SKILLS
                │                                          │
                │                                          ▼
                └────────────────────────────► STARTING_CONVERSATION
                                                           │
                                                           ▼
                                                         READY / RUNNING
                                                           │
                                  ┌──── PAUSE ─────────────┤
                                  ▼                        │
                                STOPPED ──── resume ───────┤
                                                           │
                                                           ▼
                                                        ARCHIVED（终态）
```

关键状态位：

- `sandbox_status`：容器层视角（`MISSING` / `PROVISIONING` / `RUNNING` / `PAUSED` / `ERROR` / `STOPPED` / `ARCHIVED`）。
- `execution_status`：Agent 逻辑视角（来自 SDK）。
- 两者正交：容器可以是 `RUNNING`，但 Agent 可以是 `idle` / `working` / `finished`。

生命周期 API：

| 阶段 | 触发 | 效果 |
|---|---|---|
| 创建 | `POST /api/v1/app-conversations` | SQL 插入 metadata，异步触发 sandbox 启动 |
| 使用 | `POST /api/v1/conversations/{id}/pending-messages` | 若 `READY` 则转发给 Agent Server，否则入队 |
| 恢复 | `resume` → 重启沙箱 → 从 Event Store 重放历史 | 原生支持 resume |
| 持久化 | 每个 Event 落盘 `events/{id}.json`，metadata 每次变更落 `metadata.json` | 并行写 SQL |
| 销毁 | `DELETE` → `ARCHIVED`；默认不删文件（可审计） | 软删除 |

### 4.2 Event：Action / Observation

OpenHands 的核心二元对立模型：

```text
Action      = 「Agent 或 User 想让世界发生什么」
Observation = 「世界反馈回来的状态」
```

一次 Agent 迭代：

```text
[思考] → emit Action → 运行时执行 → emit Observation → 作为新输入喂回 LLM
```

常见 Action 类型包括：

- `CmdRunAction`
- `FileReadAction`
- `FileWriteAction`
- `FileEditAction`
- `BrowseURLAction`
- `BrowseInteractiveAction`
- `IPythonRunCellAction`
- `MessageAction`
- `SystemMessageAction`
- `AgentThinkAction`
- `AgentFinishAction`
- `AgentRejectAction`
- `AgentDelegateAction`
- `RecallAction`
- `MCPAction`
- `TaskTrackingAction`
- `LoopRecoveryAction`

常见 Observation 类型包括：

- 对应每个 Action 的 Observation
- `AgentStateChangedObservation`
- `ErrorObservation`
- `SuccessObservation`
- `UserRejectObservation`
- `LoopDetectionObservation`
- `MCPObservation`

Event 字段（`events/event.py`）：

```text
id (int, 自增)
timestamp (ISO)
source (USER / AGENT / ENVIRONMENT)
message (可选)
cause (上游 event id，形成因果链)
llm_metrics (成本/token)
tool_call_metadata (函数调用上下文)
```

生命周期：

```text
产生（Agent.step / 用户输入 / 环境反馈）
  → EventStream.add_event()（内存 queue + 订阅者广播）
  → EventStore.append() → events/{id}.json 落盘
  → （可选）SQL mirror + webhook fanout
  → 客户端 GET /events/search 拉取
  → 永不主动删除；随 conversation ARCHIVED 一起保留
```

### 4.3 Tool 调用

```text
LLM 生成 tool_calls
  ↓
[原生 FC 路径] 直接是 tool_calls 数组
[非原生路径] 用 <function=...> XML 解析 → 转成同样结构
  ↓
tool_name 路由：
  ├─ 内置工具（bash/edit/browse）→ Agent Server 执行 → Observation
  └─ MCP 工具 → MCPClient.call_tool() → MCPObservation
  ↓
新 Observation 作为 role="tool" Message 喂回 LLM
```

生命周期关键点：

- Tool 定义不持久化，每次启动沙箱时从内置 + MCP server 动态拉取。
- Tool 调用有 metadata 持久化（`ToolCallMetadata` 存在 Event 里）。
- MCP 连接按需建立，错误归类包括 timeout / protocol error / 窗口限制。

### 4.4 Memory

OpenHands 没有专门的 Memory Store。

> **记忆 = 事件流 + Condenser 摘要**

Condenser 策略（10 种）：

| 策略 | 说明 |
|---|---|
| NoOp | 不压缩 |
| Recent | 保留最近 N 个 |
| LLMSummarizing | 用独立 LLM 摘要老事件 |
| ObservationMasking | 保留 Event，但遮蔽旧 Observation 内容 |
| AmortizedForgetting | 智能遗忘 |
| LLMAttention | LLM 打分，保留 top-K |
| StructuredSummary | 结构化摘要 |
| CondenserPipeline | 串联多个 Condenser |
| ConversationWindow | 滑动窗口 |
| BrowserOutputCondenser | 专门遮蔽老浏览器输出（截图很占 token） |

触发方式：事件数超过 `max_size`（默认 100）时，由 SDK 内部调用；对用户透明。

---

## 5. 数据存储与数据流（Data Flow & Storage）

### 5.1 Summary

OpenHands 采用混合双写模型：

- **事件**：用 FileStore（本地 / S3 / GCS 三选一），一 Event 一 JSON 文件。
- **元数据 + 可查询数据**：用 Postgres（Enterprise 版）。

这是典型的「事件日志冷存储 + 元数据热索引」套路。

### 5.2 存储后端全景

| 数据类型 | 存储 | 格式 | 接口 |
|---|---|---|---|
| Events | FileStore (Local / S3 / GCS) | JSON（1 文件 1 event） | EventStore |
| Event 分页缓存 | FileStore | JSON（25 events / 页） | `EventStore._CachePage` |
| ConversationMetadata | FileStore + Postgres | JSON / SQLAlchemy ORM | ConversationStore |
| Agent 状态快照 | FileStore | Python pickle ⚠️ | 直接 FileStore |
| Secrets（API keys） | FileStore | JSON（`SecretStr` 脱敏） | SecretsStore |
| Settings | FileStore | JSON（Pydantic） | SettingsStore |
| Pending messages | Postgres | SQLAlchemy | PendingMessageService |
| Rate limiting / 会话锁 | Redis（仅 Enterprise） | - | - |

### 5.3 目录结构（Local 后端）

```text
~/.openhands/
└── sessions/{conversation_id}/       # 或 users/{user_id}/conversations/{id}/
    ├── events/
    │   ├── 0.json                    # User MessageAction
    │   ├── 1.json                    # Agent AgentThinkAction
    │   ├── 2.json                    # Agent CmdRunAction
    │   ├── 3.json                    # CmdOutputObservation
    │   └── ...
    ├── event_cache/
    │   └── 0-25.json                 # 分页缓存
    ├── metadata.json                 # ConversationMetadata
    ├── agent_state.pkl               # 可恢复的 Agent 内存态
    └── conversation_stats.pkl        # 累计成本/token
```

### 5.4 端到端数据流：一次用户消息

```text
┌─────────┐                 ┌───────────────┐              ┌──────────────┐               ┌───────────┐
│ Frontend│                 │  App Server   │              │ Agent Server │               │   LLM     │
│ (React) │                 │  (FastAPI)    │              │  (sandbox)   │               │ (LiteLLM) │
└────┬────┘                 └───────┬───────┘              └──────┬───────┘               └─────┬─────┘
     │ 1. POST /conversations/{id}/pending-messages                │                              │
     │    {content, role:"user"}                                   │                              │
     ├──────────────────────────────▶│                              │                              │
     │                               │ 2. 若 conv.status != READY,  │                              │
     │                               │    SQL INSERT pending_msg    │                              │
     │                               │ 3. 若 READY,                 │                              │
     │                               │    POST /api/conversations/  │                              │
     │                               │    {id}/events (httpx async) │                              │
     │                               ├─────────────────────────────▶│                              │
     │                               │                              │ 4. Agent.step()              │
     │                               │                              │    构造 messages             │
     │                               │                              │    (Condenser 裁剪)          │
     │                               │                              │    llm.completion(tools=...) │
     │                               │                              ├─────────────────────────────▶│
     │                               │                              │                              │
     │                               │                              │ 5. tool_calls 返回           │
     │                               │                              │◀─────────────────────────────┤
     │                               │                              │ 6. 本地执行 Action           │
     │                               │                              │    (bash/edit/browse/mcp)    │
     │                               │                              │    → Observation             │
     │                               │                              │ 7. EventStore.append()       │
     │                               │                              │    写 events/{id}.json       │
     │                               │                              │ 8. POST /api/v1/webhooks     │
     │                               │◀─────────────────────────────┤    (event fanout)            │
     │                               │ 9. 本地 EventService.save()   │                              │
     │                               │    + SQL metadata update     │                              │
     │                               │ 10. 若未 finish，回到步骤 4  │                              │
     │                               │                              │                              │
     │ 11. GET /events/search?start=N (轮询)                         │                              │
     ├──────────────────────────────▶│                              │                              │
     │   [events]                    │                              │                              │
     │◀──────────────────────────────┤                              │                              │
     │                               │                              │                              │
```

### 5.5 实时传输

- **Frontend ↔ App Server**：主要是 HTTP 轮询 `/events/search?start_id=N`；旧版 v0 走 Socket.IO（`openhands/server/listen_socket.py` 尚在）。
- **App Server ↔ Agent Server**：HTTP（`httpx.AsyncClient`）+ webhook 回推。
- **v1 app_server 无显式 WebSocket**：推测是刻意简化，为了兼容无状态 HTTP 层的多副本部署。

### 5.6 缓存层次

| 层 | 机制 | 说明 |
|---|---|---|
| LLM Prompt Cache | Anthropic / OpenAI provider 端 | App Server 侧只记录 hit / miss token 数 |
| Event 分页 cache | FileStore `event_cache/{start}-{end}.json` | 避免大量小文件 list |
| SQL 查询 cache | 无显式实现 | 依赖 PG 索引 |
| Redis | Enterprise 独有 | 做限流 |

### 5.7 设计点评

| 优点 | 潜在问题 |
|---|---|
| 一事件一文件：天然幂等、易于对象存储 | 海量小文件在 S3 上 `ListObjects` 很慢（靠 cache pages 缓解） |
| 文件 + SQL 双写：SQL 提供索引查询，文件提供完整回放 | 双写一致性全靠应用层保证，可能漂移 |
| `SecretStr` 脱敏做得好 | Secrets 不加密 at rest，依赖后端存储安全 |
| `agent_state.pkl` 让会话可恢复 | Pickle 跨版本 / 跨 Python 不兼容，是反序列化安全雷区 |

---

## 6. 面向复刻的实施指南

这是最值得重点看的章节。以下路线图按优先级排列。

### 6.1 MVP 必须优先实现：最小可用 Agent

目标：跑通一次「用户 → LLM → 工具 → 结果」闭环。

| # | 模块 | 必做内容 | 可对应文件 |
|---|---|---|---|
| 1 | Agent Loop | 一个 `while` 循环：`build messages → LLM.completion → parse tool_calls → 执行 → 追加 Observation` | 仿 SDK 的 `Agent.step()` |
| 2 | Event 模型 | `Action` + `Observation` 两个基类；至少 3 类 Action：`MessageAction`、`CmdRunAction`、`FinishAction` | 仿 `events/event.py` |
| 3 | LLM 封装 | 直接包一层 LiteLLM；重试用 `tenacity` | 仿 `llm/llm.py` 简化版 |
| 4 | 本地 bash 工具 | `subprocess` 跑命令 → `CmdOutputObservation` | 参考 openhands-tools |
| 5 | 本地文件工具 | read / write / edit（`str_replace` 即可） | - |
| 6 | EventStore（本地） | 一事件一 JSON 文件，自增 id | 仿 `events/event_store.py` |
| 7 | FastAPI 单会话 API | `POST /message`、`GET /events`，内存里跑 | 仿 `app_server/app_conversation/` 单 session 版 |

预估工作量：**2–3 周**可以复刻到能跑 SWE-bench 简单任务的程度。

### 6.2 第二阶段：达到产品雏形

| # | 模块 | 内容 | 何时做 |
|---|---|---|---|
| 8 | 多会话 + ConversationStore | `user_id` + `conversation_id` 维度 | 产品化之前 |
| 9 | Docker Sandbox | 把 bash / 文件工具关进容器 | 上线给外人用之前 |
| 10 | 原生 Function Calling | LiteLLM `tools=` 参数 + 自动识别模型 | 支持 Claude / GPT-4o 时 |
| 11 | Condenser（Recent 策略足够） | 简单保留最近 N 个事件 | 长任务出现时 |
| 12 | Frontend | 一个最小 React，轮询 `/events` 即可 | 给演示用 |

### 6.3 可延后：非关键能力

| 能力 | 为什么可延后 |
|---|---|
| MCP Client | 内置工具够用；MCP 是扩展性，不是核心能力 |
| 浏览器工具 | Playwright + VNC 复杂度极高；大多数软件任务不需要 |
| 多 LLM 路由 | 先用一个 Claude 跑通 |
| 独立 Condenser LLM | 用便宜的主模型摘要即可 |
| S3 / GCS Backend | Local FS 够跑一年 |
| Postgres 双写 | 纯 FileStore 能支撑 1k 会话量级 |
| Socket.IO / WebSocket | 轮询在本地体验够用 |
| Pending Message Queue | 同步架构里没这个问题 |
| Kubernetes / Remote Sandbox | 只有云产品需要 |
| Enterprise 整包 | 完全可以不做 |

### 6.4 可简化的 Trade-off

| OpenHands 的做法 | 简化版做法 |
|---|---|
| Agent Server 独立 HTTP 服务 | 直接把 Agent 内嵌在 App Server 进程里，省掉一个 hop |
| 10 种 Condenser 策略 | 只做 `Recent(N=50)` 一种 |
| `user_id` + 多租户从 API 层贯穿 | 单用户，path 里不带 `user_id` |
| FileStore 抽象（Local / S3 / GCS） | 只做 Local，接口留好即可 |
| Event cache pages | 先不做，直接顺序读 |
| 复杂 DI（Injector） | FastAPI 自带的 `Depends` 就够 |
| `agent_state.pkl` 恢复 | 只靠 event 重放即可恢复 |
| Native + non-native function calling 双路径 | 只支持 native FC，限定模型 |
| Pending message queue | 同步等沙箱起来再返回 |
| KVM 硬件隔离 | Docker 默认隔离就够 |

### 6.5 复刻版架构图（推荐终点）

```text
┌───────────────────────────────────────────┐
│          FastAPI Single Process           │
│                                           │
│  ┌──────────────┐   ┌──────────────────┐  │
│  │ REST Router  │──▶│ ConversationMgr  │  │
│  │ /message     │   └────────┬─────────┘  │
│  │ /events      │            │            │
│  └──────────────┘            ▼            │
│                     ┌─────────────────┐   │
│                     │   Agent Loop    │   │
│                     │  while not done:│   │
│                     │   build_msgs    │   │
│                     │   llm_call      │   │
│                     │   exec_tools    │   │
│                     └───────┬─────────┘   │
│                             │             │
│         ┌───────────────────┼──────────┐  │
│         ▼                   ▼          ▼  │
│   ┌──────────┐       ┌──────────┐  ┌────────┐
│   │ LiteLLM  │       │ Tools:   │  │ Event  │
│   │ client   │       │ - bash   │  │ Store  │
│   └──────────┘       │ - edit   │  │ (JSONL)│
│                      │ (docker) │  └────────┘
│                      └──────────┘           │
└─────────────────────────────────────────────┘
```

---

## 7. 面试讲解 Cheat Sheet

### 7.1 30 秒版

> OpenHands 是一个开源的软件工程 Agent 系统，架构上分 4 层：Agent 核心（SDK）、容器内的 Agent Server、编排层 App Server、前端 UI。核心抽象是 Event Sourcing——所有交互都是 Action / Observation 事件对。Agent 跑在 Docker 沙箱里，通过 HTTP + session key 跟编排层通信。LLM 层用 LiteLLM 封装，支持 100+ provider，独立 Condenser 做上下文压缩。

### 7.2 2 分钟版

在 30 秒版基础上补充：

- Event 模型是系统灵魂：12+ Action / 13+ Observation，全部 JSON 序列化，一事件一文件，天然支持 replay 和审计。
- 沙箱抽象可插拔：Docker / Process / Remote 三种实现，同一 `SandboxService` 接口。
- 存储是混合双写：事件走 FileStore（Local / S3 / GCS），元数据走 Postgres。
- Condenser 机制解决上下文爆炸：用独立的便宜模型做摘要，跟主模型解耦。
- 它跟 Devin 最像，跟 AutoGPT 不同：有人在回路 + 持久化会话，不是一次性 autonomous run。

### 7.3 5 分钟版

在 2 分钟版基础上补充：

- Agent loop 是经典 ReAct：`LLM → tool_calls → execute → observations → LLM`。
- Function calling 有 native / prompt-based 双路径，自动探测模型能力。
- Session 的状态机跨容器层（`sandbox_status`）和 Agent 层（`execution_status`）两个维度。
- 每沙箱一个 `session_api_key`（base62 32 字节随机），HTTP 头 `X-Session-API-Key` 是唯一鉴权方式。
- 无自建向量库：「记忆」= 事件流 + Condenser，是个务实选择。
- v0 `openhands/server/` 正在迁移到 v1 `openhands/app_server/`，读代码时认准 `app_server`。
- 最大的工程亮点是把 Agent 逻辑从编排逻辑拆出去，形成独立 SDK，让 Agent 能独立演进，并被其他产品线（CLI、Cloud）复用。

---

## 8. 学习路径建议

建议按这个顺序阅读源码：

1. 先读 `openhands/events/`：重点看 `event.py`、`action/`、`observation/`。掌握数据模型是理解其他模块的前提。
2. 再读 `openhands/app_server/app_conversation/live_status_app_conversation_service.py`：这是整个会话编排的中枢。
3. 对照读 `openhands/app_server/sandbox/docker_sandbox_service.py`：看沙箱怎么启动。
4. 最后读外部 SDK `software-agent-sdk`：真正的 Agent loop 在这里。
5. 配置侧读 `config.template.toml` + `openhands/core/config/`。
6. 不建议优先读 `openhands/server/`：这是 v0 legacy，容易浪费时间。

---

## 9. 确认 / 推测标注总览

| 结论 | 状态 |
|---|---|
| 4 层分层架构（SDK / AgentServer / AppServer / Frontend） | ✅ 已确认 |
| Event = Action + Observation | ✅ 已确认 |
| 3 种 Sandbox + 统一接口 | ✅ 已确认 |
| HTTP + session key 鉴权 | ✅ 已确认 |
| LiteLLM 作为 LLM 抽象 | ✅ 已确认 |
| ReAct loop 在外部 SDK | ✅ 已确认（本仓库 grep 不到 `agent.step`） |
| Condenser 10 种策略 | ✅ 已确认 |
| 一事件一 JSON 文件 | ✅ 已确认 |
| FileStore + Postgres 双写 | ✅ 已确认（Enterprise） |
| 浏览器工具跑在沙箱里 | 🟡 推测（从依赖和动作类型推断） |
| 真实 K8s 控制器在外部仓库 | 🟡 推测（本仓库只有 manifests） |
| Frontend 用 HTTP 轮询而非 WebSocket | 🟡 推测（v1 `app_server` 没看到 WS endpoint） |

---

## 后续可深入的方向

可以继续沿三个方向展开：

1. **复刻版 MVP 代码骨架**：直接搭一个最小 Agent 项目结构。
2. **源码级 walkthrough**：例如 Condenser 实现细节、Event Store pagination 算法、Sandbox 启动完整调用栈。
3. **面试表达材料**：把这份报告压缩成 1 页架构图 + 1 页讲解稿。
