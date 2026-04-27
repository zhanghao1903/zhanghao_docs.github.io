
# OpenHands Agent SDK 源码解剖报告

> 研究对象：`OpenHands/software-agent-sdk` 当前工作树  
> 关注范围：`openhands-sdk`、`openhands-tools`，并覆盖部分 `openhands-agent-server` 桥接层  
> 目标：理解 SDK 的核心模块、主运行链路、能力边界、可改进点，并为“复刻简化版 Agent SDK”提供路线图。

---

## 目录

- [0. 总览](#0-总览)
- [1. 核心模块划分](#1-核心模块划分)
- [2. 核心模块深度分析](#2-核心模块深度分析)
  - [2.1 Agent](#21-agent)
  - [2.2 LocalConversation](#22-localconversation)
  - [2.3 LLM](#23-llm)
  - [2.4 Event ↔ Message 桥](#24-event--message-桥)
- [3. SDK 能力边界](#3-sdk-能力边界)
- [4. 可改进点与 MVP 切片](#4-可改进点与-mvp-切片)
- [5. 复刻路线图](#5-复刻路线图)
- [6. 综合点评](#6-综合点评)

---

## 0. 总览

这是 **OpenHands Agent SDK 仓库**，主要包含以下几个子包：

| 子包 | 定位 | 说明 |
|---|---|---|
| `openhands-sdk` | 核心 SDK | Agent 主循环、LLM 抽象、Conversation、Event、Tool、MCP、Plugin、Hook 等核心能力 |
| `openhands-tools` | 内置工具实现 | Terminal、FileEditor、Browser、Glob、Grep、Patch、Task Tracker 等 |
| `openhands-workspace` | 运行时沙箱 | Local / Remote workspace 抽象，承接文件、命令执行等运行环境能力 |
| `openhands-agent-server` | 远端服务 | 通过 HTTP / WebSocket 暴露 Agent 能力，用于容器化或远端运行 |

一个最小调用链可以理解为：

```text
LLM → Agent(llm, tools) → Conversation(agent, workspace) → send_message() → run()
```

从源码结构看，OpenHands SDK 并不是一个“薄 SDK”，而是一个 **半业务框架（half-framework）**：它不仅封装了 LLM 和 Tool，还内置了 prompt 拼装策略、event-sourcing 持久化、hook lifecycle、安全确认、stuck detection、subagent registry、plugin marketplace 等上层平台能力。

---

## 1. 核心模块划分

### 1.1 关键结论

OpenHands SDK 的复杂度明显高于“底层 Agent SDK”。一个真正底层的 Agent SDK 通常只需要：

```text
LLM + Tool + Loop
```

但 OpenHands SDK 内部已经包含了 Agent 平台级能力，因此更适合被理解为：

> 面向软件工程场景的垂直 Agent 框架。

---

### 1.2 模块树

以下是 `openhands-sdk/openhands/sdk/` 的核心目录结构：

```text
sdk/
├── llm/              ★ LLM 抽象层：基于 litellm，封装 completion / responses 双 API、retry、fallback、streaming、subscription auth
│   ├── llm.py                 - LLM 主类（约 1645 行 Pydantic 模型）
│   ├── llm_registry.py        - 多 LLM 实例按 usage_id 注册查找
│   ├── fallback_strategy.py   - 失败时按序切换备用模型
│   ├── llm_profile_store.py   - LLM profile 持久化
│   ├── message.py             - Message / TextContent / ImageContent / ThinkingBlock
│   ├── streaming.py           - StreamChunk 类型定义
│   └── router/ mixins/ options/ auth/ utils/
│
├── agent/            ★ Agent 主体：定义 step() 循环、tool dispatch、parallel 执行、critic 钩子
│   ├── base.py                - AgentBase（抽象，frozen Pydantic）
│   ├── agent.py               - Agent（具体实现，约 1054 行）
│   ├── acp_agent.py           - ACP 协议 Agent 变体（含义仍需进一步确认）
│   ├── response_dispatch.py   - 把 LLM 响应分类为 TOOL_CALLS / CONTENT / REASONING_ONLY / EMPTY
│   ├── critic_mixin.py        - critic 评分钩子
│   ├── parallel_executor.py   - 多工具并发 + 资源锁
│   └── utils.py               - prepare_llm_messages / make_llm_completion
│
├── conversation/     ★ 会话运行时：状态管理、event log、运行循环、本地 vs 远端实现
│   ├── conversation.py        - Conversation 工厂（__new__ 分派 Local / Remote）
│   ├── base.py                - BaseConversation 抽象基类
│   ├── state.py               - ConversationState（带 auto-save、FIFOLock）
│   ├── event_store.py         - 文件持久化 EventLog（~/events/event-{idx}-{id}.json）
│   ├── stuck_detector.py      - 死循环模式检测（4 种模式）
│   ├── fifo_lock.py           - 公平 FIFO 可重入锁
│   ├── resource_lock_manager.py - 工具粒度资源互斥
│   ├── secret_registry.py     - per-conversation 密钥注入
│   ├── impl/                  - LocalConversation / RemoteConversation
│   └── visualizer/            - 终端可视化
│
├── event/            ★ 事件类型库：event-sourcing 的领域模型
│   ├── base.py                - Event / LLMConvertibleEvent（events_to_messages 桥）
│   ├── llm_convertible/       - ActionEvent / ObservationEvent / MessageEvent
│   ├── condenser.py           - 上下文压缩事件
│   ├── streaming_delta.py     - 不持久化的 token 流增量
│   ├── conversation_state.py  - WebSocket 同步事件
│   ├── hook_execution.py
│   ├── llm_completion_log.py
│   └── token.py
│
├── tool/             ★ 工具协议：Pydantic Action / Observation + Registry
│   ├── tool.py                - ToolDefinition[ActionT, ObservationT] 泛型
│   ├── schema.py              - Action / Observation Schema + JSON-Schema 转换
│   ├── registry.py            - 锁保护的工厂注册表（3 种 factory 类型）
│   ├── spec.py
│   └── builtins/
│
├── mcp/              MCP 适配：把 fastmcp client 包成普通 Tool
│   ├── client.py
│   ├── definition.py
│   ├── tool.py
│   ├── utils.py
│   └── exceptions.py
│
├── skills/           知识单元：Markdown / SKILL.md + keyword/task 触发器 + `!cmd` 渲染期执行
├── plugin/           Plugin 包装层：聚合 skills + tools + hooks + agents + commands，可从 git 拉取
├── subagent/         Subagent 注册与定义文件加载
├── critic/           答案质量评分器，可触发重做
├── hooks/            生命周期 hooks：PreToolUse / PostToolUse / Stop / SessionStart 等
├── security/         confirmation_policy + LLM analyzer + ensemble + defense_in_depth
├── workspace/        命令执行 / 文件 IO 抽象：Local vs Remote 容器
├── io/               FileStore：事件 / 状态的存储抽象，支持 Local FS、in-memory、LRU 缓存
├── secret/           密钥源：Static / Lookup + 渲染时变量展开
├── settings/         运行时配置：带 schema 导出，给 UI 使用
├── context/          AgentContext：拼装 system prompt 后缀 + 加载 skills + condenser
├── observability/    Laminar / OTEL tracing
├── extensions/       Web fetch 等小扩展
├── marketplace/      Plugin / skill 市场类型
├── logger/           结构化日志 + rolling file
└── utils/            cipher、async_executor、redact、truncate、pydantic_diff 等工具函数
```

---

### 1.3 模块依赖关系

```text
                  ┌──────────────┐
   user code ───▶ │ Conversation │ ──▶ workspace/  io/  hooks/  security/
                  └──────┬───────┘
                         │
                         ▼
                  ┌──────────────┐         ┌────────┐
                  │   Agent      │ ─────▶ │  llm/  │ ── litellm
                  │  step loop   │         └────────┘
                  └──┬────────┬──┘              ▲
                     │        │                 │
                     │        └─▶ event/ ───────┘
                     │              events_to_messages
                     ▼
              ┌──────────┐  ┌──────┐  ┌────────┐
              │  tool/   │◀─│ mcp/ │  │ critic │
              └────┬─────┘  └──────┘  └────────┘
                   │
                   ▼
              openhands-tools/*
              Terminal / FileEditor / Browser / Glob / Grep / ...
```

`context/`、`skills/`、`plugin/`、`subagent/` 都是在 `Conversation._ensure_plugins_loaded()` 阶段被聚合进 Agent 的旁路组件，不在最核心的 `step()` 主路径上。

---

## 2. 核心模块深度分析

真正决定整个 Agent 行为的 4 个模块是：

1. `Agent.step()`
2. `Conversation.run()`
3. `LLM`
4. `Event ↔ Message` 桥

其他模块大多是围绕这条主链路的增强、装饰或产品化能力。

---

### 2.1 Agent

#### 关键结论

`Agent` 是一个 **无状态、frozen Pydantic 配置对象**，主要包含：

- `llm`
- `tools`
- `condenser`
- `critic`
- `system_prompt`

所有运行态都被推到 `ConversationState` 中。这是一个比较干净的 **配置 / 运行态分离** 设计。

---

#### `Agent.step()` 调用流程

源码位置：`agent/agent.py:475–603`

```text
step(conversation, on_event, on_token):
  1. 检查未完成的 ActionEvent
       └─ 确认模式下可能存在挂起 action
       └─ 如果有，直接走 _execute_actions()，跳过 LLM

  2. 检查最后一条 user message 是否被 UserPromptSubmit hook 拦截

  3. prepare_llm_messages(state.events, condenser, llm)
       ├─ events_to_messages()
       │   └─ 把 ActionEvent / ObservationEvent / MessageEvent
       │      按 llm_response_id 批量合并为 Message[]
       └─ 如果 token 超限，触发 CondensationRequest，走 condenser

  4. make_llm_completion(llm, messages, tools=tools_map.openai_schema, on_token)
       ├─ 走 LLM.completion() 或 LLM.responses()
       └─ 捕获 LLMContextWindowExceedError 后，通过 condenser 重试

  5. classify_response(message)
       └─ LLMResponseType ∈ {
            TOOL_CALLS,
            CONTENT,
            REASONING_ONLY,
            EMPTY
          }

  6. 分派响应：
     ├─ TOOL_CALLS
     │   └─ _handle_tool_calls()
     │       ├─ 解析 tool call
     │       ├─ 校验 Pydantic Action
     │       ├─ 判断 confirmation_policy
     │       └─ _execute_actions(action_events)
     │           └─ ParallelToolExecutor.execute_batch(...)
     │              每个 tool 调用返回 ObservationEvent 或 AgentErrorEvent
     │
     ├─ CONTENT
     │   └─ emit MessageEvent，必要时设置 execution_status = FINISHED
     │
     └─ REASONING_ONLY / EMPTY
         └─ 发 MessageEvent，仅包含 thinking
```

---

#### 架构点评

**优点：**

- “分类响应 → 分发执行”的流程非常清晰。
- `tools_map` + 严格的 Pydantic Action 校验，可以避免 LLM 幻觉参数直接打到副作用函数。
- Agent 本身不持有运行态，便于复用和持久化。

**问题：**

- `Agent` 主类继承 `CriticMixin`、`ResponseDispatchMixin`、`AgentBase`，阅读主链路时需要跨多个文件。
- `agent.py` 单文件约 1054 行，`step()`、`_get_action_event()`、`_execute_actions()`、确认逻辑都放在一起，复杂度较高。

**MVP 砍法：**

复刻时只保留：

```text
step() 主路径 + 简单 tool dispatch
```

可以先砍掉：

```text
critic / parallel / confirmation / blocked-actions / condenser fallback
```

一个 200 行左右的 `step()` 就足够跑通核心闭环。

---

### 2.2 LocalConversation

#### 关键结论

`Conversation` 本身是一个 `__new__` 工厂，根据 workspace 类型分派：

```text
Conversation
├── LocalConversation
└── RemoteConversation
```

真正的本地运行循环在 `LocalConversation` 中。它依赖 `ConversationState` 这个 Pydantic 模型做隐式自动持久化：

```text
ConversationState.__setattr__ → _save_base_state(fs)
```

---

#### `Conversation.run()` 调用流程

源码位置：`conversation/impl/local_conversation.py:721–864`

```text
run():
  _ensure_plugins_loaded()   # 第一次 run 时 lazy 加载 plugin
  _ensure_agent_ready()      # 双重检查锁

  while True:
    with self._state:        # FIFOLock，保证 send_message 能公平插队

       if status in {PAUSED, STUCK}:
           break

       if status == FINISHED:
           run Stop hooks
           if hook deny:
               reset 为 RUNNING
               注入反馈消息
           else:
               break

       if stuck_detector.is_stuck():
           status = STUCK
           continue

       agent.step(self, on_event, on_token)
       iteration += 1

       if status == WAITING_FOR_CONFIRMATION:
           break

       if iteration >= max_iterations:
           status = ERROR
           break
```

---

#### ConversationState 持久化模型

| 类型 | 字段 / 组件 | 说明 |
|---|---|---|
| 公有字段 | `id` | Conversation ID |
| 公有字段 | `agent` | 当前 Agent 配置 |
| 公有字段 | `workspace` | 当前工作空间 |
| 公有字段 | `execution_status` | 执行状态 |
| 公有字段 | `confirmation_policy` | 安全确认策略 |
| 公有字段 | `security_analyzer` | 安全分析器 |
| 公有字段 | `stats` | 统计信息 |
| 公有字段 | `secret_registry` | 密钥注册表 |
| 公有字段 | `agent_state` | Agent 状态 |
| 公有字段 | `hook_config` | Hook 配置 |
| 公有字段 | `tags` | 标签 |
| 私有字段 | `_fs` | FileStore |
| 私有字段 | `_events` | EventLog |
| 私有字段 | `_cipher` | 密钥加密 |
| 私有字段 | `_lock` | FIFOLock |

`EventLog` 单独以文件形式保存：

```text
events/event-{idx}-{event_id}.json
```

它还带有：

- 30 秒锁超时
- 跨进程 stale-index 重扫
- 断点恢复能力

断点恢复逻辑：

```text
ConversationState.create()
  ├─ 如果 BASE_STATE 存在 → 反序列化并 resume
  └─ 如果不存在 → 创建新的 ConversationState
```

---

#### 架构点评

**优点：**

- `event-sourcing + auto-save + FIFOLock` 组合，让交互式 Agent 场景更安全。
- 用户在 `run()` 期间 `send_message()` 的场景被考虑到了，这对 CLI / IDE Agent 很关键。

**问题：**

- `ConversationState` 同时承担了：
  - Pydantic 持久化模型
  - FIFO 锁
  - 事件日志载体
  - cipher 容器
  - 自动保存触发器
- 这违反单一职责原则（SRP）。
- 通过 `__setattr__` 触发自动保存，调试时不直观。
- `stuck_detector` 使用经验阈值，跨模型、跨任务未必稳定。

**MVP 砍法：**

```text
list[Event] + JSON 文件 dump
```

即可替代 95% 的早期功能。锁可以先用标准 `threading.RLock`，`stuck_detector` 直接砍掉。

---

### 2.3 LLM

#### 关键结论

`LLM` 是建立在 LiteLLM 之上的厚封装类。它是一个 Pydantic 模型，同时承载了：

- 40+ 配置字段
- 6 个 mixin
- Chat Completions API
- Responses API
- retry
- fallback
- telemetry
- prompt cache
- vision toggle
- reasoning effort
- subscription OAuth

这说明它已经不是一个简单的 LLM Client，而是一个平台级 LLM 适配层。

---

#### `LLM.completion()` 调用流程

源码位置：`llm/llm.py:710–880`

```text
completion(messages, tools, on_token, ...):
  1. format_messages_for_llm(messages)
       ├─ Message.to_chat_dict() 逐条转换
       └─ _apply_prompt_caching()
          └─ 给 Anthropic 加 cache_control

  2. retry_decorator(_transport_call)(...)
       └─ _transport_call(): litellm.completion(...)
            ├─ stream=True  → 逐 chunk 调 on_token(LLMStreamChunk)
            └─ stream=False → 一次性返回 ModelResponse

  3. 如果 transient error 重试耗尽
       └─ fallback_strategy.try_fallback()
          └─ 按 LLMProfileStore 顺序切换模型

  4. Message.from_llm_chat_message(resp.choices[0].message)
       ├─ 抽 reasoning_content
       ├─ 抽 thinking_blocks
       └─ 抽 tool_calls

  5. 返回 LLMResponse
       ├─ message
       ├─ MetricsSnapshot
       └─ raw_response
```

---

#### 架构点评

**优点：**

- 把 prompt cache、reasoning models、Anthropic thinking blocks、OpenAI Responses API、subscription OAuth 等差异化特性统一进一个 `Message` 模型中。
- 应用层不需要感知不同模型供应商的细节。

**问题：**

- `LLM` 类约 1645 行，复杂度较高。
- Telemetry / Retry / NonNativeToolCalling / Subscription 全混入到 `LLM`，定位行为时需要跨多个文件。
- `LLMRegistry + LLMProfileStore + FallbackStrategy` 三层设计，对普通使用者来说过重，更像为大型托管平台准备。

**MVP 砍法：**

最小核心路径：

```text
litellm.completion(model, messages, tools) + Message ↔ dict
```

可以保留：

```text
Retry: tenacity
```

先砍掉：

```text
fallback / subscription / profile_store / 非原生 tool calling / responses-api
```

预计 200 行左右即可完成简化版 LLM 层。

---

### 2.4 Event ↔ Message 桥

#### 关键结论

这是整个 SDK 最值得学习的设计。

OpenHands SDK 对外采用 event-sourcing 历史，对 LLM 则使用 message 列表，两者通过 `events_to_messages()` 做单向转换。

这正是 **event-sourcing 在 Agent 场景中能跑起来的关键**。

---

#### 转换流程

```text
state.events: list[Event]
   │
   ▼
LLMConvertibleEvent.events_to_messages()
   │
   ├─ ActionEvent(同 llm_response_id 的多条)
   │     └─ 合并为 assistant Message
   │        { role, thought, tool_calls=[...], reasoning, thinking_blocks }
   │
   ├─ ObservationEvent
   │     └─ 转换为 tool Message
   │        { role="tool", content, tool_call_id, name }
   │
   ├─ MessageEvent
   │     └─ 直接 yield 内嵌 llm_message
   │
   └─ CondensationSummaryEvent
         └─ 转换为 system Message
            { summary text, 替换被遗忘的 events }
   │
   ▼
list[Message]
   │
   ▼
format_messages_for_llm()
   │
   ▼
LiteLLM
```

其中，`_combine_action_events()` 专门处理“一次 LLM response 包含多个 parallel tool_calls”的情况。

如果不合并，而是把它们拆成多条 assistant 消息，OpenAI / Anthropic 的工具调用序列往往会报错。

---

#### 架构点评

**优点：**

- Event 是 append-only、可审计、可序列化。
- Message 是无状态、可重建。
- 主循环不需要维护复杂 prompt 状态，只需依赖 `to_llm_message()` / `events_to_messages()`。

**问题：**

- `StreamingDeltaEvent` 名义上属于 Event 体系，但语义上不持久化，命名规范略有不一致。

**MVP 必抄：**

即使只复刻 50% 功能，`Event ↔ Message` 桥也应该 100% 保留。

---

## 3. SDK 能力边界

### 3.1 关键定位

OpenHands SDK 的定位是：

> 半业务框架，而不是底层能力层。

它远超过“API client + tool spec”的最小集，但又没有到完整 IDE / Agent 平台的程度。

使用它就意味着接受它的整套世界观：

- `confirmation_policy`
- `hook lifecycle`
- `plugin` 协议
- `event-sourcing`
- `ConversationState`
- `skills / subagent / marketplace`

---

### 3.2 开箱即用能力

| 能力 | 模块 | 备注 |
|---|---|---|
| 多模型对话 | `llm/` | 底层基于 litellm，支持 Anthropic / OpenAI / OpenRouter / Bedrock / 本地 vLLM 等 |
| 多步 tool-using agent loop | `agent/` + `conversation/` | 支持并发执行、资源锁 |
| 内置工具 | `openhands-tools/` | bash、file edit、glob、grep、browser、patch、task tracker 等 |
| MCP 客户端 | `mcp/` | 基于 fastmcp 封装 |
| 事件日志 + 断点恢复 | `event_store.py` + `state.py` | JSON 文件，单机持久化 |
| 上下文压缩 | `context/condenser/` | 触发于上下文窗口超限 |
| 远端 Agent | `openhands-agent-server/` + `RemoteConversation` | 支持容器化沙箱 |
| 安全确认 + 风险评分 | `security/` | LLM analyzer + 规则 ensemble |
| Plugin marketplace | `plugin/` | 支持 `github:owner/repo` 缩写 |
| Skills | `skills/` | `SKILL.md` / 关键词触发 / `!cmd` 内联 |
| Subagent 注册与委派 | `subagent/` | 文件式定义 |
| Hook 系统 | `hooks/` | PreToolUse / Stop / SessionStart 等 |
| OTEL / Laminar 观测 | `observability/` | optional |
| Subscription 登录 | `llm/auth/` | OAuth 设备流 |
| Stuck detector | `conversation/stuck_detector.py` | 4 种硬编码模式 |

---

### 3.3 边界外能力

| 能力 | 当前状态 |
|---|---|
| UI | 不带 Web / Desktop GUI，只有终端 visualizer |
| 多 Agent orchestration / planner | 不是真正的多 Agent 编排框架，主要是单 Agent + 工具调用 + subagent 委派 |
| Eval 框架 | 不带完整任务评测框架，`scripts/event_sourcing_benchmarks/` 更像性能基准 |
| 向量库 / 长期 memory | 不带 RAG / Vector Store，主要依赖 condenser 总结 |
| Cost budgeting 强制中断 | 有 metrics，但不强制 enforce |
| RBAC / 多租户 | Server 更偏 localhost-friendly，没有完整鉴权层 |
| 跨 conversation 知识迁移 | 每个 conversation 独立持久化目录 |

---

### 3.4 依赖上层系统补齐的能力

| 能力 | 谁来补 |
|---|---|
| 命令式审批 UI | CLI / IDE / Web 前端 |
| Plugin / Skill 内容 | OpenHands extensions repo + 用户 |
| LLM 提供商凭证 | 环境变量或 CredentialStore |
| 容器 / K8s 沙箱 | `openhands-agent-server` 部署方 |
| 任务评测 / SWE-bench 集成 | GitHub Actions 或外部 eval 系统 |

---

### 3.5 定位总结

OpenHands SDK 是一个 **软件工程领域的垂直 Agent 框架**。

它默认假设你的 Agent 主要在处理：

- 代码
- 文件
- Shell
- Git diff
- 软件工程任务

如果你想做“通用 Agent 底座”，其中很多假设反而会成为负担，例如：

- bash 工具默认存在
- git diff 内置
- `SKILL.md` 来自 OpenHands marketplace
- plugin 可以注入多个子系统

---

## 4. 可改进点与 MVP 切片

### 4.1 关键结论

如果用 Python 在 1–2 周内复刻一个简化版，OpenHands SDK 的核心路径其实只有 4 块：

```text
LLM → Agent.step → Tool 执行 → Event / Message 转换
```

剩下大量内容都是成熟产品才需要的“腰带与吊带”。

---

### 4.2 主要设计问题

#### 问题 1：一等公民太多，认知负担重

目前并存的扩展机制包括：

```text
Tool / MCPTool / Skill / Plugin / Subagent / Hook / Critic / SecurityAnalyzer
```

每一种都有自己的注册表、加载器和生命周期。

从用户视角看，问题是：

> 我想加一个能力，到底应该写 Tool、Skill、Plugin、Hook，还是 Subagent？

**建议：**

- 一等公民只保留 `Tool`。
- `Skill` 退化成 system prompt 后缀。
- `Plugin` 退化为“一个目录里放一堆 Tool / Skill”。
- 先砍掉 `Critic` 和 `Subagent`。

---

#### 问题 2：LLM 类是“上帝对象”

`LLM` 同时承载：

- 40+ 字段
- 6 个 mixin
- completion / responses 双路径
- telemetry
- retry
- fallback
- subscription
- non-native tool calling

这是补丁式增长的典型表现。

**建议：**

拆成：

```text
LLMConfig（纯数据）
LLMClient（供应商适配器）
MessageAdapter（Message ↔ dict 转换）
```

让不同供应商细节归适配器处理。

---

#### 问题 3：ConversationState 角色过载

`ConversationState` 同时是：

1. Pydantic 持久化模型
2. FIFO 锁
3. 事件日志载体
4. cipher 容器
5. 自动保存触发器

**建议：**

拆成：

```text
ConversationState  # 纯数据
ConversationStore  # 持久化
ConversationLock   # 并发控制
```

---

#### 问题 4：隐式 `__setattr__` auto-save

字段赋值会触发磁盘写入，这会带来：

- 调试时看不到显式保存调用栈
- 意外字段赋值可能触发写盘
- 测试时更容易漏 mock

**建议：**

改成显式：

```text
state.save()
```

或者在 `Conversation.run()` 关键节点统一保存。

---

#### 问题 5：Stuck detector 是经验性硬编码

当前 stuck detector 使用 4 种模式和经验阈值，例如：

```text
action_observation = 4
monologue = 3
```

这类规则跨模型、跨任务未必稳定。长时间合理探索的会话也可能被误判为 STUCK。

**建议：**

- MVP 中直接砍掉。
- 线上版本可以考虑 LLM self-eval，让 critic LLM 根据历史给出“卡住置信度”。

---

#### 问题 6：Plugin 是“全家桶”

一个 Plugin 可以同时注入：

```text
skills + tools + hooks + agents + commands + mcp_config
```

这意味着一个 plugin 一拉下来，就可能改变 6 个子系统的状态，安全审查面很宽。

**建议：**

- plugin 只允许声明 `skills + tools`。
- `hooks / agents` 强制本地配置。

---

#### 优点：Event / Message 桥非常值得保留

`LLMConvertibleEvent.events_to_messages()` 是整个 SDK 中最值得学习的设计。

它把 append-only 的 Event 历史转换为 LLM 可消费的 Message 序列，让 Agent 主循环保持干净。

---

## 5. 复刻路线图

### 5.1 Week 1：必须实现的核心路径

目标代码量：约 800–1200 行。

| 复刻目标 | 对应原模块 | 简化方案 |
|---|---|---|
| `LLM(model, api_key).complete(messages, tools)` | `llm/llm.py` | 直接 `litellm.completion`，砍 fallback / profile / subscription / responses-api / non-native tools |
| `Message{role, content, tool_calls, tool_call_id}` | `llm/message.py` | 只支持 text + tool_use + tool_result，砍 image / thinking / reasoning_content |
| Tool 基类 + `register_tool` | `tool/tool.py` + `tool/registry.py` | Pydantic Action / Observation + 全局 dict registry |
| 3 个内置工具 | `openhands-tools/*` | `bash` / `file_read` / `file_write`，使用 `subprocess` + `open()` |
| Event 类型 | `event/llm_convertible/` | `ActionEvent` / `ObservationEvent` / `MessageEvent`，frozen Pydantic |
| `events_to_messages()` | `event/base.py` | 必须保留批量合并 tool_calls 的语义 |
| `Conversation.send_message()` / `run()` | `LocalConversation` | 单线程 while loop + max_iterations |
| 简易持久化 | `event_store.py` + `state.py` | 一个 `state.json` + 一个 `events.jsonl` |

---

### 5.2 Week 2：增强能力

| 增强能力 | 价值 | 何时加 |
|---|---|---|
| MCP 客户端 | 高 | 用户需要接外部服务时 |
| Confirmation policy | 中 | 上线给真人使用之前 |
| 简单 condenser | 中 | 第一次撞上下文窗口时 |
| Streaming | 中 | 加 UI 时 |
| Skills | 低 | 想做自定义角色时 |

---

### 5.3 直接砍掉的部分

早期复刻时可以先砍掉以下模块，节省约 70% 代码量：

```text
subagent/
critic/
plugin/ 全部
marketplace/
skills/ 全部（先用 system prompt 替代）
security/ensemble
security/defense_in_depth
security/grayswan
stuck_detector
parallel_executor
resource_lock_manager
fifo_lock（换 threading.RLock）
LLMRegistry
LLMProfileStore
FallbackStrategy
RemoteConversation
整个 agent_server
observability/laminar
extensions/
hooks/ 全部
auth/subscription 流
ACPAgent
responses-api 路径
non-native tool calling mock
高级 condenser（只保留极简截断）
```

---

## 6. 综合点评

### 6.1 最值得学的设计

1. **Event ↔ Message 桥**  
   `LLMConvertibleEvent + events_to_messages()` 的合并语义非常关键，尤其是多个 tool calls 的处理。

2. **Conversation 工厂分派**  
   `Conversation.__new__` 根据 workspace 类型分派 `LocalConversation / RemoteConversation`，适合本地与远端运行统一抽象。

3. **Agent 配置 / 运行态分离**  
   `Agent` 无状态，`ConversationState` 有状态，这个方向是对的。

---

### 6.2 最该警惕的点

不要过早实现这些“看起来很产品化”的子系统：

```text
condenser
critic
stuck_detector
subagent
marketplace
plugin
security analyzer
```

它们都是上线之后才知道是否真正需要的东西。过早实现会污染主路径，让 MVP 复杂度急剧上升。

---

### 6.3 复刻心法

先把 `Agent.step()` 写成 80 行左右的纯主循环：

```text
while not done:
    messages = events_to_messages(events)
    response = llm.complete(messages, tools)
    if response.tool_calls:
        action_events = parse_tool_calls(response.tool_calls)
        observations = execute_tools(action_events)
        events.extend(action_events + observations)
    else:
        events.append(MessageEvent(response.content))
        done = True
```

只要能跑通一个最小任务，例如：

```text
写一个 facts.txt 文件
```

就说明核心链路已经成立。之后再根据真实场景反向加复杂度。

---

## 7. 仍需进一步确认的点

以下内容在当前分析中仍带有“推测”性质，需要单独阅读对应源码进一步确认：

| 待确认项 | 建议阅读文件 |
|---|---|
| `ACPAgent` 的具体协议含义 | `acp_agent.py` |
| `critic` 在 step 循环中的精确触发点 | `critic_mixin.py` |
| `fallback_strategy` 的指标合并细节 | `fallback_strategy.py:83–100` |

其他结论基本都来自源码结构和当前分析锚点。
