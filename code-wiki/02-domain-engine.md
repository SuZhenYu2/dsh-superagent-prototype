# 领域模型与 Engine

## 核心模型

模型定义见 [model.ts](../src/core/model.ts)，端口见 [ports.ts](../src/core/ports.ts)。

| 概念 | 含义与关键约束 |
| --- | --- |
| ActorRef / ThreadRef / ChannelRef | 分别标识来源中的参与者、讨论线程和可写地址；Thread 的全局键由 `source:id` 构成。 |
| Trigger | 来源产生的不可变事件，含参与者、人类标记、时间、文本和至少一个 Thread；首个 Thread 是评估主线程。`text`、`meta` 不可信。 |
| TriggerDisposition | 记录过滤原因、评估和决策、分发的 Ticket、仅作上下文的分发、退回及确认；它记录去向，不授予权限。 |
| Thread | 评估一致性边界，维护订阅、关联 Thread、待评估 Trigger 和正在进行的 Evaluation；同一 Thread 至多一个进行中的 Eval。 |
| Evaluation / Candidate | Eval 是线程上一批事件的评估状态，记录强度、策略版本、期限和投递时间；Candidate 是交给评估者的有界事实，包含相关 Ticket。 |
| Decision | 对评估的审计结果，记录原因、策略版本，可关联 Ticket 和 Mandate。当前决策结果为 `ignore`、`accept`、`stop`、`unavailable`；类型中的 `reopen` 供历史记录兼容，当前恢复不产生新的 `reopen` Decision。 |
| Mandate | accept 生成的不可变授权，包含 `explicit` 或 `inferred` 强度、策略版本、准入依据、发起者及 scope；恢复工作时可重验其依据。 |
| Ticket | 已授权工作聚合，记录来源、Mandate、渠道、输入、Assignment、关注的 Thread 和结束结果。未设置 `ended` 即为开放状态。 |
| Assignment / Input | Assignment 是 Ticket 与一次执行实例的绑定，状态为 `pending`、`running`、`ended`；Input 包含 Trigger、策略变化、恢复、文本或仅供上下文的 note，投递与读取分别记录。 |
| Follow / Subscription | Subscription 使 Ticket 收到线程事件；Follow 记录来源绑定的线程及关闭结果。仅为上下文订阅的线程不授予工作权限，也不因人类消息恢复已结束 Ticket。 |

Ticket 结束结果包括 `completed`、`merged`、`closed`、`dropped`、`stopped`；worker 报告状态为 `progress`、`waiting`、`ready`、`blocked`、`failed`、`dropped`。报告状态与 Ticket 结束结果不是同一状态机。[model.ts](../src/core/model.ts) 中的 `followupOwners`、`liveTicket` 和 `waitsForPerson` 是理解这些边界的重要辅助函数。

## Engine 结构和接口

[Engine](../src/core/engine.ts) 接收 [EngineOptions](../src/core/part.ts) 中的 repository、实时 Policy、SourceScope、Assessor、Executor、Outbox、时钟与事件回调，组合五个内部组件：Intake 处理接收，Evals 管理评估和决策，Tickets 管理工作生命周期，Assignments 管理执行回调，Replies 管理回复。大多数变更由 `repo.atomic` 包围；`evaluateDue` 对单个 Eval 或 Ticket 分别开启事务，而不是整个扫描共用一个事务。`candidate` 是查询。

Engine 通过 `Pick` 定义角色接口而不是向每个调用方暴露全部操作：`EngineForSources`、`EngineForMainSession`、`EngineForPeople`、`EngineForSettings`、`EngineForExecutor`。这些是 TypeScript 编译期接口，不是独立的运行时授权机制。

### 来源与时钟

| 方法 | 行为 |
| --- | --- |
| `receive(trigger)` | 接收并返回 `stored`、`duplicate` 或 `filtered`；无 Thread 会报错，过滤事件也会保存及记录过滤原因。 |
| `bindThread(ticketId, thread, own)` | 为开放 Ticket 订阅并绑定来源线程；`own` 标明由 Ticket 创建。 |
| `threadClosed(thread, outcome, at)` | 记录线程 `merged` 或 `closed`，在关注线程及未完成工作条件允许时结束 Ticket。 |
| `evaluateDue()` | 处理进行中 Eval 的暂停和超时；未暂停时唤醒有待投递输入的已结束 Ticket，并启动有待评估事件且没有当前 Eval 的线程。 |

### 主 Session 与人工操作

| 方法 | 行为 |
| --- | --- |
| `candidate(triggerId)` / `candidateDelivered(evalId)` | 查询持有 Trigger 的 Eval 候选；候选实际送达后开始按策略期限计时。 |
| `decide(triggerId, outcome)` | 对现有 Eval 或单个 Trigger 直接决策；拒绝过期 Eval ID、无权接受的推断候选等情况，已创建 Ticket 的 Trigger 不再创建第二个。 |
| `attach(ticketId, triggerId, note?)` | 将消息路由给 Ticket、处理原评估队列并尝试唤醒；不得将 worker 已退回的消息重新附给同一 Ticket。 |
| `requestStatus(ticketId)` | 仅请求运行中且处于 waiting 的 worker 报告状态，返回是否请求。 |
| `reply(request)` | 按用途验证并选择渠道，幂等排队确认、回答、注意事项或结果；发送由 Outbox 异步完成。 |
| `reevaluate(triggerId)` / `dismiss(decisionId)` | 重新排队评估，或隐藏已处理的 unavailable 问题；正在被 Eval 持有的 Trigger 不能重新排队。 |
| `unsubscribe(thread)` / `stopTicket(id, reason?)` / `retryTicket(id)` | 人工移除线程订阅、撤销 Ticket 授权并停止、或在可重试情况下结束停滞 Assignment 并新建一个。 |

### Executor 与设置

| 方法 | 行为 |
| --- | --- |
| `assignmentStarted` / `assignmentInterrupted` | 记录启动并投递待处理输入；宿主中断后按报告及后续输入状态决定保留 Session 或恢复重启。 |
| `inputsDelivered` / `inputsRead` / `turnEnded` / `deliveryFailed` | 分别记录投递、Session 读取、轮次结束及投递失败；过期 Assignment 的部分回调会被忽略。 |
| `report(ticketId, report)` | 记录当前 Assignment 报告，维护 attention 和回答；`dropped` 结束 Ticket，`ready` 在关注线程满足结束条件后结束。 |
| `markUnrelated` / `followForContext` | worker 可移除允许移除的关联订阅，或订阅只读上下文；来源绑定及起始线程不能随意标记无关。 |
| `note(ticketId, text)` / `handback(ticketId, triggerId, reason)` | 前者添加不授予权限的上下文；后者将超出 Ticket 范围的消息退回，最后一个负责它的 worker 退回后才可重新评估。 |
| `policyChanged(from, to, effects)` | 通知受影响的开放 Ticket，并让进行中的 Eval 在新版本下重新开始。 |

精确参数、返回值和错误条件见 [engine.ts](../src/core/engine.ts)；执行细节见 [assignments.ts](../src/core/assignments.ts)、[tickets.ts](../src/core/tickets.ts) 和 [replies.ts](../src/core/replies.ts)。

## 关键流程

### 接收、分发与跟进归属

1. [Intake.receive](../src/core/intake.ts) 规范化时间、检查 Thread、按 Trigger ID 去重，随后由 SourceScope 判断是否监听。过滤后仍保留事件及 disposition。
2. 对所有所属 Thread 建立关联，向订阅 Ticket 添加 Input；同一 Ticket 经多个 Thread 到达时只分发一次，若至少一条路径不是上下文路径则不标记为仅上下文。
3. 人类在非上下文线程的消息可唤醒非人工停止的已结束 Ticket；人工停止撤销 Mandate，不再接收。机器人事件不会因此唤醒。
4. `followupOwners` 判断人类消息是否由已收到它的 worker 决定；若没有负责者，则排入主 Thread 的 `pending`。worker 可通过 `handback` 退回超范围消息，避免重复由多个评估者处理。

### 批量评估与授权

1. [Evals.start](../src/core/evals.ts) 先移除应由 worker 处理的跟进以及已创建过 Ticket 的事件，再以 Policy gate 对待处理 Trigger 判定准入。无可准入事件时记录 ignore。
2. 同批事件中 `explicit` 强于 `inferred`；同强度选时间较晚的事件，其参与者和准入依据成为评估依据。当前批次形成 Evaluation，新事件等待下一批。
3. Assessor 异步接收 Candidate。`candidateDelivered` 后才允许超时；超时最多重新排队两次，再记录 `unavailable`。暂停期间记录暂停时间，恢复后顺延 deadline。
4. `decide` 校验 Eval 和当前策略；推断请求只有在策略允许且匹配启用规则时才能接受。接受产生 Mandate 和 Ticket，再排队启动 Assignment；较旧策略的评估结果会被记录为不可用。可见 [evals.ts](../src/core/evals.ts) 和 [policy.ts](../src/policy.ts)。

### 执行、结束与恢复

1. Executor 异步启动 Assignment，成功后 `assignmentStarted` 触发待投递输入的派发；`inputsDelivered` 不等于 worker 已读取，后者由 `inputsRead` 单独记录。
2. worker 报告 `blocked` 或 `failed` 后，自动输入受到抑制，直到人类消息等条件使其继续；人工也可对符合条件的 Ticket 执行 retry。
3. `ready` 不一定立即结束：无来源绑定的关注线程时可完成；有关注线程时需满足 `followsDone`。若有 `own` 线程，要求自有线程均关闭；否则要求全部关注线程关闭。线程关闭后仍需考虑报告之后是否有尚未完成的工作。详见 [tickets.ts](../src/core/tickets.ts)。
4. 非人工结束的 Ticket 收到合适的人类跟进时，可在原 Mandate 下恢复；暂停时先保留待投递输入，恢复处理后再唤醒，且需要重验证授权。人工停止的 Ticket 不按此路径恢复。

## 测试线索

[core-engine.spec.ts](../tests/core-engine.spec.ts) 覆盖领域引擎行为；[followups.spec.ts](../tests/followups.spec.ts)、[stage.spec.ts](../tests/stage.spec.ts) 可用于追踪跟进与阶段规则；[runtime.spec.ts](../tests/runtime.spec.ts)、[effects.spec.ts](../tests/effects.spec.ts)、[store.spec.ts](../tests/store.spec.ts) 分别覆盖装配、副作用和存储。架构总览见 [系统架构](./01-architecture.md)。
