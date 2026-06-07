# Agent 上下文渲染的缓存优化：为什么 Task Context 不能每轮全量重写

在 Agent 产品里，Task Context 解决的是“Execution Agent 当前应该看到什么任务状态”的问题。

但还有一个容易被低估的问题：

> 当结构化 Task Context 被渲染成 LLM input 时，它会不会破坏 prompt cache？

这个问题听起来偏工程实现，但它直接影响 Agent 产品体验。多步任务通常需要反复调用模型，每一步都要带上系统规则、任务目标、工具历史、当前状态、权限约束和证据材料。如果上下文渲染方式不对，就会同时带来三个问题：

- 输入 token 反复膨胀；
- prefill latency 增加；
- prompt cache 命中率下降。

对于 Execution Agent，这不是一个边角优化。它会影响长任务能不能稳定、便宜、低延迟地运行。

## 问题：全量重写 Context 很直观，但缓存很差

很多上下文治理方案会自然走向一种实现：

```text
system
full regenerated task execution context
prior AgentLoop messages
```

也就是说，每一轮调用模型之前，Context Manager 都重新收集事实，生成一个完整的结构化执行上下文，然后放在消息列表前部。

这个方式有一个优点：每次调用都很“自包含”。模型总能在前部看到最新任务状态。

但它也有一个严重缺点：**cache-hostile**。

因为 full context 里通常包含大量高变化字段：

- `task_id`、`root_task_id`、`parent_task_id`；
- 当前状态、claim 状态、turn index；
- event id、observation id、trace id；
- 最近工具结果；
- 文件片段；
- snapshot 引用；
- 新增 workspace evidence。

Prompt caching 通常依赖 prefix matching。也就是说，下一次请求和上一次请求从开头开始越相同，可复用的缓存越多。

如果 volatile 字段出现在请求前部，稳定前缀会很快结束。即使后面的执行 transcript 大部分没有变化，缓存也无法充分命中。

这就出现了一个矛盾：

```text
上下文治理希望每轮都提供最新状态
prompt cache 希望请求前缀尽量稳定
```

如果不处理这个矛盾，Context Manager 反而会把原本 append-only 的 AgentLoop 变成一个每轮重写的大 prompt。

## 原来的 AgentLoop 为什么缓存友好

传统 AgentLoop 天然有一个缓存友好的形状：

```text
turn 1:
system
task

turn 2:
system
task
assistant_1
tool_1

turn 3:
system
task
assistant_1
tool_1
assistant_2
tool_2
```

每一轮都只是追加新消息。

这意味着，上一轮请求几乎是下一轮请求的前缀。多步任务里，prompt cache 可以复用大量前缀 token。

问题在于，原始 AgentLoop 缺少治理。它容易把历史消息、工具结果、用户约束和临时状态混在一起，缺少结构化任务视图。

所以目标不应该是退回 unmanaged prompt，而应该是：

> 让 Context Manager 治理 append-only transcript，而不是每轮用完整上下文替换它。

## 方案：Cache-aware Append-only Rendering

更合理的渲染形态是：

```text
stable start context
append-only execution transcript
minimal context deltas
periodic context checkpoints
```

这四部分对应不同职责。

## 1. Stable Start Context

Stable Start Context 是任务开始时建立的稳定前缀。

它应该包含：

- system prompt；
- Context Manager contract；
- 稳定项目规则；
- 稳定输出要求；
- 原始任务目标；
- durable constraints；
- 长期有效的权限边界。

这些内容在同一个 Task run 中应该尽量 byte-stable。只要事实没有变化，就不要在后续调用中重写它。

同时，Stable Start Context 里不应该放大量 volatile 信息：

- 原始 event id 列表；
- observation id；
- trace id；
- snapshot id；
- timestamp；
- 最近工具结果；
- 每轮变化的 selected snippets。

这些信息对审计和诊断有价值，但不一定应该进入稳定 prompt 前缀。

一个关键原则是：

> 模型需要的是行动所需信息，系统需要的是审计所需信息。两者不能混成一份 prompt。

## 2. Append-only Execution Transcript

Stable Start Context 之后，执行历史应该尽量保持 append-only。

包括：

- assistant messages；
- tool calls；
- tool observations；
- user confirmations；
- interruption / retry messages；
- 必要的 runtime observations。

Context Manager 不能为了“刷新上下文”而随意重排、改写或插入旧的 assistant/tool 消息。

这里不仅是缓存问题，也是协议问题。OpenAI、Anthropic、Gemini 等模型 API 对 tool call / tool result 的消息顺序都有约束。

至少要保证：

```text
assistant(tool_calls=[...])
  必须跟随对应 tool result

tool result
  必须能对应到前面的 tool_call_id

历史消息
  不应被随意重写成新的上下文摘要
```

如果 Context Manager 破坏了工具调用顺序，轻则 API 报错，重则 Agent 的执行状态被污染。

## 3. Minimal Context Delta

不是所有状态变化都需要重写完整 Task Context。

普通回合里，如果有新信息影响下一步决策，可以在消息尾部追加一个短 delta：

- 最新用户指令；
- pending approval；
- interruption request；
- 重要工具错误；
- 重要文件变更摘要；
- 新的 workspace evidence；
- 当前步骤变化。

例如：

```text
Context Delta:
- User approved modifying frontend upload limit.
- Latest grep found nginx/client_max_body_size in deploy/nginx.conf.
- Next step should inspect backend upload parser before editing config.
```

Delta 的价值是给模型补充“现在发生了什么”，而不是替换前面的稳定上下文和执行 transcript。

它应该短、明确、靠近尾部。

## 4. Periodic Context Checkpoint

Delta 适合小变化，但长任务仍然需要周期性 current-state reminder。

可以每 N 步追加一个 checkpoint，例如每 5 个 execution steps 一次，或者在重要生命周期变化时触发：

- retry 开始；
- 用户 interruption 被处理；
- confirmation resolved；
- repeated tool errors；
- 文件变更超过阈值；
- context budget pressure 出现；
- 计划步骤进入新阶段。

Checkpoint 应该包含：

- 当前目标；
- 已完成事实；
- 关键观察；
- 待确认事项；
- 已修改文件；
- 推荐下一步。

Checkpoint 不应该包含：

- 大段文件内容；
- 完整 event history；
- 完整 tool result history；
- raw id 列表；
- 已在 stable prefix 中存在的重复目标文本。

一个好的 checkpoint 更像“当前执行状态摘要”，而不是“审计日志压缩包”。

## LLM Input 不是 Audit Log

缓存友好渲染背后还有一个更重要的边界：

> LLM input 不是 audit record。

LLM input 只应该包含模型下一步行动所需的信息。完整的 provenance、event ids、trace ids、candidate selection、excluded context、raw refs，应该进入 snapshot 和 trace。

两者职责不同：

| 载体              | 目标                     |
| ----------------- | ------------------------ |
| LLM input         | 让模型正确行动           |
| Context Snapshot  | 记录模型看到了什么       |
| Context Trace     | 解释哪些事实被选择或排除 |
| Event Log         | 保存系统事实账本         |
| Tool Result Store | 保存原始工具输出         |

如果把所有审计信息都塞进 prompt，就会出现两类问题：

- prompt 变贵、变慢、cache 命中变差；
- 模型可能把审计字段误当成任务上下文，反而干扰决策。

因此，Context Manager 的目标不是“让 prompt 包含所有事实”，而是“让 prompt 包含行动所需事实，并让系统能解释为什么这些事实被选中”。

## 和 Task Context 的关系

Task Context 是结构化任务状态。

Cache-aware rendering 是 Task Context 到 LLM input 的渲染策略。

它们不是同一层。

```text
Event Log / TaskBus / Workspace / Tool Results
        ↓
TaskExecutionContext
        ↓
Context Renderer
        ↓
Cache-aware LLM Input
```

如果没有 Task Context，append-only transcript 容易变成 unmanaged prompt。

如果没有 cache-aware rendering，Task Context 又容易变成每轮全量重写的大上下文块。

好的架构应该同时满足：

- 任务状态结构化；
- 稳定事实稳定渲染；
- 执行 transcript 保持 causal order；
- 新变化通过 delta 追加；
- 长任务通过 checkpoint 提醒；
- 审计信息进入 snapshot / trace；
- 大型工具结果通过 raw ref 按需读取。

## 产品价值

从产品角度看，缓存友好上下文渲染带来的价值很直接。

### 1. 降低长任务成本

多步 Agent 任务往往会重复发送大量前缀内容。只要 stable prefix 和 transcript 前段能命中缓存，cached token ratio 就会显著提升。

这会直接降低模型调用成本，尤其是在 Claude、Gemini、OpenAI 等支持 prompt caching 或类似前缀缓存能力的模型上。

### 2. 降低等待时间

Prefill latency 对 Agent 体验很敏感。

用户能接受模型思考几秒，但很难接受每一步都因为重复输入长上下文而变慢。缓存命中越好，长任务每一步的启动成本越低。

### 3. 保持执行因果顺序

Append-only transcript 不只是为了缓存，也让模型看到的执行历史保持因果顺序。

这对工具调用 Agent 很重要：它需要知道哪个 action 导致哪个 observation，哪个用户确认发生在哪个修改之前。

### 4. 减少上下文噪音

把 trace、snapshot、raw ids、完整工具输出从 prompt 里移出去，可以减少模型误读上下文的风险。

模型看到的是决策所需信息，系统保存的是审计所需信息。

## 测试标准

如果要把 cache-aware rendering 做成稳定能力，至少需要覆盖这些测试：

1. 稳定事实不变时，stable prefix 完全一致。
2. volatile ids 不出现在 stable prefix。
3. delta 和 checkpoint 追加在 transcript 后部，而不是插入前部。
4. assistant/tool-call/tool-result 顺序不被破坏。
5. 每次 rendered input 仍包含必要的 current-state facts。
6. snapshot / trace 能解释本次输入选择。
7. 大型 tool result 默认通过 summary + raw_ref 表达。
8. provider metadata 可记录 cached-token ratio 和 prefill latency。

这里最重要的是第一条和第四条。

如果 stable prefix 不稳定，缓存收益会大幅下降。  
如果 tool-call ordering 被破坏，Agent 执行链路会直接不可靠。

## 结论

Task Context 解决的是结构化上下文治理问题。

Cache-aware append-only rendering 解决的是结构化上下文如何进入模型的问题。

对 Agent 产品来说，这两者必须同时成立：

```text
没有 Task Context，Agent 缺少稳定任务状态。
没有缓存友好渲染，Task Context 会拖慢长任务执行。
```

真正可持续的 Agent 上下文系统，不应该每轮把完整状态重新塞到 prompt 前面。

它应该建立稳定前缀，保留 append-only 执行历史，用 delta 表达新变化，用 checkpoint 提醒当前状态，并把完整审计信息留在 snapshot、trace 和 event log 里。

这不是单纯的成本优化，而是 Agent 产品能否长期运行复杂任务的基础设施问题。
