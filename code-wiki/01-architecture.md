# 系统架构

## 定位与边界

dsh-superagent 是运行在 DSH 中的主动式代理：接收 GitHub 及可选飞书事件，在策略约束下判断是否创建工作项，并为工作项启动独立的 worker Session。主 Session 负责判断、路由和协调；worker 在 Ticket 的授权范围内执行。来源事件和模型判断均不等同于授权。项目依赖和运行要求见 [package.json](../package.json)。

## 分层与依赖

```text
GitHub / 飞书                 管理面板 / 主 Session
       │                            │
       ▼                            ▼
  sources 适配器             HTTP、工具、视图
       └──────────┬─────────────────┘
                  ▼
       Runtime：装配、生命周期、调度
                  ▼
  Engine → Intake / Evals / Tickets / Assignments / Replies
                  │
        端口：Policy、Repository、Assessor、Executor、Outbox
                  ▼
   SQLite Store + Effects 队列 + DSH Session + 来源通道
```

| 部分 | 职责与入口 |
| --- | --- |
| 插件入口 | [src/index.ts](../src/index.ts) 声明 Cordis 依赖，创建 Runtime 并绑定关闭和重启生命周期。 |
| 领域核心 | [src/core/engine.ts](../src/core/engine.ts) 是领域变更入口；[model.ts](../src/core/model.ts) 定义模型，[ports.ts](../src/core/ports.ts) 定义外部能力的契约。核心不直接依赖 GitHub、飞书或 DSH 实现。 |
| 应用装配 | [runtime.ts](../src/app/runtime.ts) 连接 Store、策略、来源、主 Session、执行器和副作用；[passes.ts](../src/app/passes.ts) 推进接收及到期评估。 |
| 来源 | [registry.ts](../src/sources/registry.ts) 管理来源能力；[github.ts](../src/sources/github.ts)、[feishu.ts](../src/sources/feishu.ts) 接入具体来源，原生数据转换和读写分别位于同目录的 trigger、read、channel 等模块。 |
| 执行 | [dsh.ts](../src/executor/dsh.ts) 适配 DSH worker 生命周期；[sessions.ts](../src/app/sessions.ts) 管理 Session；主 Session 和 worker 的工具分别由 [main.ts](../src/app/tools/main.ts)、[executor.ts](../src/app/tools/executor.ts) 提供。 |
| 控制面 | [http.ts](../src/app/http.ts)、[api.ts](../src/app/api.ts) 和 [views](../src/app/views) 提供管理接口与读取视图；[client](../src/client) 是前端界面。 |
| 基础设施 | [store.ts](../src/app/store.ts) 持久化聚合、队列和设置；[effects.ts](../src/app/effects.ts) 调度副作用；[infra](../src/infra) 提供命令、路径和健康状态等能力。 |

## 装配与生命周期

1. [apply](../src/index.ts) 将部署配置转成机器配置，创建 Runtime；部署账号变更可触发插件重启。
2. Runtime 先锁定状态目录，再打开 `orchestrator.sqlite`，防止并发进程重复写入或迁移。它实例化 Store、Settings、SourceRegistry、DshExecutor、MasterAssessor 和 Outbox，并将它们作为端口注入 Engine。
3. HTTP 管理路由先注册，便于配置不完整时修复；setup 成功后安装 Session 和执行器、确保主 Session、同步来源设置，恢复执行器，启动 Passes 和 Effects。配置未完成时不启动后台处理。
4. 暂停或等待配置时，来源仍可记录输入，但评估、派发和外发受到暂停控制；关闭时生命周期信号用于取消运行中的工作。具体时序以 [runtime.ts](../src/app/runtime.ts) 为准。

## 事务与异步边界

Engine 通过同步的 `Repositories.atomic` 封装领域命令；[Store.atomic](../src/app/store.ts) 使用 SQLite savepoint，支持嵌套和回滚，最外层提交后才执行提交回调。状态目录锁与 SQLite WAL 是不同层面的保护：前者限制并发运行实例，后者是数据库日志模式。

外部工作不在事务中直接等待网络或模型。Assessor、Executor 和 Outbox 的端口在命令执行时排队，由 [Effects](../src/app/effects.ts) 持久化和异步派发；任务按种类配置并发、顺序及退避重试。相同 kind/key 的待处理任务可合并，generation 防止正在运行的旧任务删除更新后的任务。Outbox 另存消息回执和发送状态，见 [outbox.ts](../src/app/outbox.ts)。这提供可恢复的队列处理，不应理解为对外部系统的严格一次执行保证。

## 主要数据路径

1. 来源将原生事件转换为 Trigger，经来源范围过滤、去重后存储，并按订阅分发；未由 worker 负责的事件进入主 Thread 的待评估队列。见 [intake.ts](../src/core/intake.ts)。
2. Pass 驱动 Engine 批量评估，策略 gate 判断准入和强度，Assessor 将 Candidate 交给主 Session；主 Session 通过工具返回 Decision。见 [evals.ts](../src/core/evals.ts) 和 [assessment.ts](../src/app/assessment.ts)。
3. 接受后生成 Mandate 和 Ticket，Executor 异步启动 Assignment；输入投递、读取、报告和失败经 Engine 回调更新状态。见 [tickets.ts](../src/core/tickets.ts)、[assignments.ts](../src/core/assignments.ts)。
4. 回复进入 Outbox，经来源 Channel 发送；管理界面通过 HTTP 读取投影视图并发起人工操作。见 [replies.ts](../src/core/replies.ts)、[source-context.ts](../src/app/source-context.ts)。

## 授权与信任边界

来源范围决定是否监听，不等同于策略 gate 的工作授权。Trigger 的 `text` 和 `meta` 是不可信外部数据；Candidate 只提供有限事实，不授予权限。Policy 为评估提供版本、gate、推断接受条件及 Mandate 重验证；Ticket 创建后的 Mandate 保持不变。来源、策略和语义评估之间的契约见 [ports.ts](../src/core/ports.ts)，策略实现见 [policy.ts](../src/policy.ts)。

领域对象、方法与异常流程详见 [领域模型与 Engine](./02-domain-engine.md)。
