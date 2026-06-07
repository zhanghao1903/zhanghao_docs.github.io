# OpenHands 系统性架构解剖：从 Agent Core 到沙箱编排

上一篇文章讨论的是 OpenHands 的抽象模型：Action、Observation、EventStream 和 Runtime / Sandbox 为什么重要。

这篇文章继续往下一层看：

> 当一个 Agent 产品从 demo 走向可持续运行的系统时，OpenHands 是如何把“模型调用工具”组织成一个完整的软件工程平台的？

我的结论是，OpenHands 最值得研究的地方不只是它能写代码、跑测试、开 PR，而是它把一个 Coding Agent 拆成了几条清晰的系统边界：

- Agent Core 负责推理和工具决策；
- Agent Server 负责远程执行和工作区隔离；
- App Server 负责会话、用户、沙箱和持久化编排；
- Sandbox 负责真实执行命令、编辑文件、运行浏览器和插件；
- EventStream 负责把整个执行过程变成可追踪、可恢复、可审计的事实链。

这篇不是源码逐行讲解，而是从产品架构视角拆解 OpenHands：它为什么这样分层、每层解决什么问题、哪些设计值得复用，哪些复杂度在早期产品里可以先不做。

> 说明：本文基于我对 OpenHands 公开源码和官方文档的静态调研整理。OpenHands 仍在快速演进，具体模块名和部署细节可能变化；这里关注的是相对稳定的架构思想。

## 一句话架构

可以把 OpenHands 理解成：

> 一个面向软件工程任务的 Execution Agent 平台，用事件流记录 Agent 与环境的交互，用沙箱隔离真实执行，用服务端编排会话生命周期，用 LLM 和工具系统完成开放式任务。

它不是一个轻量 prompt wrapper，也不是一个单纯的 Agent framework。

更准确地说，它是一个端到端产品系统。

```text
Browser / CLI / API Client
  ↓
App Server
  manages users, conversations, settings, sandbox lifecycle
  ↓
Agent Server
  hosts remote conversation and workspace APIs
  ↓
Agent Core
  runs ReAct-style loop with LLM and tools
  ↓
Sandbox
  executes commands, file edits, browser actions, plugins
  ↓
EventStream
  records Action / Observation / state changes
```

这个分层回答了一个很实际的问题：

> Agent 可以自主执行任务，但产品系统必须能管理它、隔离它、恢复它、审计它。

## 为什么不能只做一个 Agent Loop

最小 Agent Loop 很简单：

```text
while not done:
  build messages
  call LLM
  parse tool calls
  execute tools
  append tool results
```

这个循环可以完成 demo。

但它很快会遇到产品级问题：

- 多个用户如何隔离工作区；
- 长任务中断后如何恢复；
- 工具执行失败后如何追踪原因；
- 用户如何看到 Agent 每一步做了什么；
- 任务历史如何存储；
- 沙箱如何启动、暂停、销毁；
- 命令执行权限如何控制；
- LLM 成本如何追踪；
- 上下文过长时如何压缩；
- 出问题时如何 replay。

OpenHands 的复杂度，基本都是为了解决这些问题。

所以看 OpenHands 时，不应该只问“Agent Loop 怎么写”，而应该问：

> Agent Loop 外围需要哪些产品化支撑，才能让它长期、多人、可恢复地运行？

## 分层架构：从大脑到执行环境

我会用四层来理解 OpenHands。

| 层级              | 职责                                                      | 产品意义                            |
| ----------------- | --------------------------------------------------------- | ----------------------------------- |
| Agent Core        | LLM 调用、工具选择、一步步推进任务                        | Agent 的“推理与行动大脑”            |
| Agent Server      | 将 Agent Core 包装成远程服务，管理 workspace API 和事件流 | 让 Agent 可以被远程调用、被隔离部署 |
| App Server        | 管理用户、会话、设置、沙箱生命周期、存储和 API            | 产品系统的编排中枢                  |
| Frontend / Client | 展示任务过程、接收用户输入、控制暂停和恢复                | 把 Agent 执行过程产品化             |

除此之外，还有横切层：

- Event 系统；
- Sandbox provider；
- LLM provider 抽象；
- MCP 和内置工具；
- 存储系统；
- Context condenser；
- 权限和密钥系统。

这些横切层是 OpenHands 从“单次执行脚本”变成“产品平台”的关键。

## App Server：真正的产品编排层

从产品视角看，App Server 是最值得关注的一层。

Agent Core 解决的是“怎么做任务”，App Server 解决的是“如何把任务作为产品流程托管起来”。

它需要处理：

- 创建 conversation；
- 绑定 user / settings / secrets；
- 启动或复用 sandbox；
- 在 sandbox 未就绪时暂存用户消息；
- 转发消息到 Agent Server；
- 接收 webhook 或 event stream；
- 持久化事件和元数据；
- 支持 pause、resume、archive；
- 对外提供 REST API；
- 给前端提供可查询的事件历史。

这类职责不是 LLM 能替代的。

它们是 Agent 产品的操作系统。

一个成熟 Agent 产品，一定需要类似 App Server 的中枢。区别只是实现轻重不同。

## Agent Server：把 Agent 从本地脚本变成远程能力

Agent Server 的价值在于把 Agent 执行包装成一个远程服务。

它通常承担三类职责：

- 远程 conversation API；
- workspace / sandbox 连接；
- 实时事件传输。

这样做的好处是：

- 前端、CLI、自动化平台都可以通过 API 使用 Agent；
- Agent 执行可以放在容器、远程机器或云端环境；
- 多用户系统可以集中管理工作区；
- 任务流可以通过 HTTP / WebSocket 传输；
- Agent Core 可以独立演进。

这也是 OpenHands 和很多本地 CLI coding assistant 的关键差异。

本地 CLI 更轻，启动更快，用户体验直接。

但如果目标是 SaaS、团队协作、云端工作区、远程执行或企业部署，Agent Server 几乎是必然形态。

## Sandbox：执行环境是一等产品边界

OpenHands V1 里更推荐使用 sandbox 这个词，而不是旧文档里的 runtime。

一个 sandbox 可以理解成：

> Agent 真正运行命令、修改文件、启动服务、访问浏览器和加载插件的隔离环境。

常见 provider 可以分成三类。

| Sandbox provider | 形态                         | 适用场景                   | 风险                       |
| ---------------- | ---------------------------- | -------------------------- | -------------------------- |
| Docker           | Agent Server 跑在容器里      | 本地部署、自托管、较强隔离 | 需要 Docker 权限和资源管理 |
| Process          | 直接在本机进程运行           | 开发调试、受控环境         | 隔离弱，不适合开放给用户   |
| Remote           | 调用远端工作区或云端运行环境 | 托管服务、K8s、企业部署    | 成本、延迟、资源编排更复杂 |

这个抽象非常重要。

因为 Agent 产品的商业化边界，往往不是模型 API，而是执行环境：

- 用户是否愿意把代码交给你的云端；
- 企业是否要求私有化部署；
- 是否支持本地开发者工作流；
- 是否能按任务隔离权限；
- 是否能限制 CPU、内存、网络和磁盘；
- 是否能在任务结束后销毁环境；
- 是否能支持远程 workspace 恢复。

这也是为什么我认为：

> Sandbox 不是技术实现细节，而是 Agent 产品的信任边界。

## EventStream：系统事实的来源

OpenHands 的另一个核心是事件流。

所有重要交互都应该被记录为事件：

```text
User Message
  ↓
Agent Action
  ↓
Environment Observation
  ↓
Agent State Change
  ↓
Next Action
```

事件流的核心不是“留日志”，而是建立系统事实。

一个 Event 通常应该回答：

- 谁触发了它；
- 什么时候发生；
- 事件类型是什么；
- 它依赖哪个上游事件；
- 携带了什么工具调用元数据；
- 产生了什么 Observation；
- 消耗了多少 token 和成本；
- 是否改变了 Agent 状态。

这让系统可以做很多事情：

- 前端逐步展示 Agent 执行过程；
- 后端恢复中断会话；
- 用户追踪哪一步出错；
- 审计系统检查权限和操作；
- 开发者 replay 历史任务；
- Context condenser 基于事件压缩上下文；
- 未来把事件作为训练或评估数据。

如果没有 EventStream，Agent 产品只能展示最终答案。

有了 EventStream，Agent 产品才有过程、状态、因果链和恢复能力。

## Action / Observation：工具调用不应该只是消息

上一篇文章已经讨论过 Action / Observation，这里放到系统架构里看。

Action 是 Agent 想让环境发生的事：

```text
CmdRunAction
FileReadAction
FileWriteAction
FileEditAction
BrowseURLAction
BrowseInteractiveAction
MCPAction
AgentFinishAction
```

Observation 是环境返回的事实：

```text
CmdOutputObservation
FileReadObservation
ErrorObservation
SuccessObservation
BrowserObservation
MCPObservation
UserRejectObservation
```

这个模型的价值在于，它没有把所有东西都塞进 `assistant` / `tool` message。

工具结果如果只是普通文本，系统很难知道：

- 这是 stdout 还是 stderr；
- 命令是否成功；
- 修改了哪个文件；
- 是否来自浏览器截图；
- 是否是 MCP 工具返回；
- 是否由用户拒绝产生；
- 是否可以重试；
- 是否应该进入长期上下文。

OpenHands 的启发是：

> Agent 系统应该先结构化世界反馈，再把它渲染给模型。

这和传统聊天产品正好相反。

聊天产品先服务模型上下文，Agent 产品必须先服务执行事实。

## Conversation 生命周期：Agent 需要被托管

一个真实 Agent 任务不是“请求进来，响应出去”这么简单。

它有生命周期：

```text
CREATE
  ↓
STARTING
  ↓
PREPARING_SANDBOX
  ↓
READY
  ↓
RUNNING
  ↓
PAUSED / STOPPED / ERROR
  ↓
RESUME / ARCHIVE
```

这里至少有两个状态维度：

- sandbox 状态：环境是否存在、是否启动、是否暂停、是否异常；
- execution 状态：Agent 是否空闲、执行中、等待用户、完成或失败。

这两个状态不能混在一起。

容器可以是运行中的，但 Agent 可能已经完成任务。

Agent 可以等待用户确认，但 sandbox 仍然应该保留。

Sandbox 可以重启，但 conversation 的事件历史不能丢。

这个状态拆分对产品体验很关键。

用户看到的是“我的任务还在不在、能不能继续、能不能恢复”；系统内部需要知道的是“环境是否健康、Agent 是否还在执行、事件是否完整”。

## LLM Provider：模型抽象只是其中一层

OpenHands 通过 LiteLLM 一类封装接入多家模型 provider。

这很重要，但它不是架构核心。

LLM provider 层主要解决：

- 多模型配置；
- API 参数兼容；
- function calling 能力差异；
- 重试和超时；
- 成本追踪；
- prompt cache metadata；
- condenser 使用独立模型。

但 Agent 产品不能把 LLM provider 当成唯一抽象。

更合理的层次是：

```text
LLM Provider
  handles model calls

Agent Core
  decides next action

Tool / MCP Layer
  exposes capabilities

Sandbox
  executes actions

EventStream
  records facts

App Server
  manages lifecycle
```

模型可以替换，provider 可以扩展。

但如果 Event、Sandbox、Conversation、Store 的边界没设计好，换模型也救不了系统稳定性。

## Condenser：上下文治理的工程化入口

OpenHands 没有把 memory 做成一个神秘的长期记忆模块。

更务实的理解是：

> 当前可用记忆 = 完整事件历史 + 被渲染进模型窗口的压缩上下文。

这就需要 condenser。

Condenser 的职责不是“总结聊天记录”这么简单，而是决定哪些历史继续进入模型窗口，哪些被压缩、遮蔽、摘要或保留为可追溯引用。

常见策略可以包括：

- 保留最近 N 个事件；
- 摘要较早历史；
- 遮蔽旧 Observation 的大块输出；
- 对浏览器截图和长日志做特殊处理；
- 组合多个策略形成 pipeline；
- 用独立便宜模型做摘要。

这和我在 Task Context 文章里的观点一致：

> 上下文治理不是压缩算法，而是任务状态、事件历史、工具结果和模型输入之间的契约。

OpenHands 的启发是，Context 不应该直接等于 Message History。

Message History 是模型输入。

Event History 才是系统事实。

Condenser 是两者之间的渲染和治理层。

## 存储：事件冷存储 + 元数据热索引

OpenHands 的存储思路可以简化成两类。

| 数据                  | 更适合的存储    | 原因                         |
| --------------------- | --------------- | ---------------------------- |
| Event 原文            | 文件 / 对象存储 | 追加写、可回放、成本低       |
| Conversation metadata | 数据库          | 查询、筛选、排序、状态更新   |
| Pending messages      | 数据库          | 需要事务和状态流转           |
| Settings / secrets    | 配置存储        | 生命周期不同于事件           |
| Event cache page      | 文件 / 缓存     | 减少大量小文件读取           |
| Agent state snapshot  | 快照存储        | 用于恢复，但要注意兼容和安全 |

这种混合模型很实用。

所有事件都进数据库，查询方便，但成本和结构演进压力大。

所有东西都放文件，架构简单，但状态查询、分页、筛选和多租户管理会变难。

所以更合理的方式是：

```text
Event Store
  stores append-only full history

Metadata Store
  stores queryable lifecycle state

Cache / Index
  improves pagination and retrieval
```

这对个人复刻版也有启发：

早期可以只用本地 JSONL 或 SQLite。

但接口边界最好提前留出来，否则从 demo 迁到可运营系统时会很痛。

## 实时传输：轮询、Webhook 和 WebSocket 的取舍

Agent 产品需要实时反馈。

用户不可能等一个长任务跑完才看到结果。

常见方式有三种：

| 方式      | 优点                     | 缺点                       |
| --------- | ------------------------ | -------------------------- |
| HTTP 轮询 | 简单，容易部署           | 延迟和请求开销更高         |
| Webhook   | 服务间解耦，适合事件回推 | 不直接服务前端实时展示     |
| WebSocket | 体验好，适合流式事件     | 部署、鉴权、断线重连更复杂 |

OpenHands 的架构里可以看到这三类思路的组合。

这说明一个原则：

> Agent 过程展示不是 UI 小功能，而是系统事件传输能力的外显。

前端每看到一条“Agent 正在运行测试”，背后都需要一个可查询、可排序、可恢复的事件机制。

## 复刻时不必照搬全部复杂度

OpenHands 很强，但不适合原样作为早期产品模板。

如果我要复刻一个最小可用的软件工程 Agent，我会这样切：

| 阶段       | 必做                                                                                           | 暂缓                                    |
| ---------- | ---------------------------------------------------------------------------------------------- | --------------------------------------- |
| MVP        | 单 Agent Loop、Action / Observation、Local EventStore、bash/file tools、单模型、FastAPI 单会话 | 多租户、远程沙箱、复杂权限、MCP、浏览器 |
| 产品雏形   | 多会话、Docker sandbox、Recent condenser、基础前端、任务暂停和恢复                             | K8s、S3/GCS、企业集成、复杂 condenser   |
| 可运营系统 | 用户体系、资源限制、事件索引、成本追踪、权限确认、错误恢复、审计视图                           | 多 Agent 编排、插件市场、复杂工作流     |

最小架构可以是：

```text
FastAPI
  /message
  /events
  ↓
ConversationManager
  ↓
AgentLoop
  ↓
LLM Client + Tool Executor
  ↓
EventStore(JSONL)
```

第一版不要急着上：

- 10 种 condenser；
- 完整 MCP；
- 浏览器自动化；
- 多 provider 路由；
- K8s remote sandbox；
- Postgres + object storage 双写；
- 复杂 DI 框架；
- 企业 SSO。

这些都是放大系统能力的东西，不是验证核心闭环的前提。

但有三件事不要省：

- Action / Observation 结构化；
- append-only EventStore；
- 执行环境和 Agent Core 解耦。

这三件事一旦早期没有做，后面补会很难。

## 对 Agent 产品经理的启发

从产品经理视角，OpenHands 的架构给我五个启发。

第一，Agent 产品要定义执行事实，而不是只定义聊天体验。

聊天记录不能承担审计、恢复、错误分析和权限追踪。Agent 产品需要自己的事件模型。

第二，执行环境是信任产品的一部分。

是否本地、是否容器、是否远程、是否可销毁、是否能限制资源，都会影响用户是否敢把真实代码交给 Agent。

第三，Agent 不是单模型能力，而是生命周期管理能力。

创建、运行、暂停、恢复、归档、失败处理、成本控制、用户确认，这些都应该进入产品设计。

第四，上下文治理要服务任务，而不是服务消息窗口。

Event History、Task State、Tool Result、Summary、Memory、Prompt Cache 应该分层管理。

第五，早期产品要克制，但底层边界要正确。

可以先不做企业级部署，可以先不用远程沙箱，可以先不支持所有工具。

但 Action、Observation、EventStore、Sandbox 这些基础抽象应该尽早稳定。

## 结论

OpenHands 的系统架构可以被压缩成一句话：

> 用 EventStream 记录 Agent 与世界的交互，用 Sandbox 隔离真实执行，用 App Server 托管生命周期，用 Agent Core 持续做决策。

它的价值不只是“开源 Devin”，而是展示了一个 Execution Agent 产品要从哪些层面系统化：

- 抽象层：Action / Observation；
- 执行层：Sandbox；
- 编排层：App Server；
- 服务层：Agent Server；
- 状态层：Conversation lifecycle；
- 事实层：EventStream；
- 上下文层：Condenser；
- 存储层：EventStore + metadata；
- 产品层：可见、可控、可恢复、可审计。

如果目标只是做一个个人 CLI，很多设计都可以简化。

但如果目标是构建可持续迭代的 Agent 产品，OpenHands 最值得学习的不是代码量，而是边界感：

> Agent 可以自主行动，但系统必须始终知道它做了什么、为什么能做、在哪里做、做完后如何恢复和解释。

## 参考资料

- [OpenHands Sandbox Overview](https://docs.openhands.dev/openhands/usage/sandboxes/overview)
- [OpenHands Agent Server Package](https://docs.openhands.dev/sdk/arch/agent-server)
- [OpenHands Runtime Architecture](https://docs.openhands.dev/openhands/usage/architecture/runtime)
