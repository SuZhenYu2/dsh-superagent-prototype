# 源码地图

按 `src` 实际目录列出全部文件；“关键导出”仅列定位入口，并非完整导出清单。链接相对于本文，类型、样式和无独立导出的入口也纳入。架构与数据流见 [系统架构](./01-architecture.md)。

## src 根目录

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [index.ts](../src/index.ts) | Cordis 插件入口，装配运行时与生命周期；`name`、`inject`、`apply`。 |
| [admission.ts](../src/admission.ts) | 准入表的数据类型、选项和默认值；`Admission`、`AdmissionRow`、`ADMISSION_DEFAULTS`。 |
| [deployment.ts](../src/deployment.ts) | 部署配置、机器信息、账号与运行环境探测；`Deployment`、`machineOf`、`resolveDeployment`、`workerProbe`。 |
| [kv.ts](../src/kv.ts) | 以 family/key 组织键值数据；`KV`、`Family`、`record`、`family`。 |
| [limits.ts](../src/limits.ts) | 集中维护输入及配置长度等限制；`LIMITS`。 |
| [navigation.ts](../src/navigation.ts) | 打开已知 Session 及导航错误；`openKnownSession`、`SessionNavigationError`。 |
| [policy.ts](../src/policy.ts) | 策略和设置 schema、准入结果及策略实现；`SettingsSchema`、`assessmentOutcome`、`policyVersion`、`settingsPolicy`。 |
| [setup.ts](../src/setup.ts) | 安装配置问题与错误类型；`SETUP_PROBLEMS`、`SetupProblem`、`SetupError`。 |

## core：领域核心

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [model.ts](../src/core/model.ts) | 定义 Trigger、Thread、Ticket、Assignment 等领域模型及状态查询；`liveTicket`、`ticketOutcome`、`runningAssignment`。 |
| [ports.ts](../src/core/ports.ts) | 定义存储、策略、评估、执行、外发和领域事件端口；`Repositories`、`Policy`、`Assessor`、`Executor`、`Outbox`。 |
| [engine.ts](../src/core/engine.ts) | 聚合领域操作并为不同调用方提供接口；`Engine`、`EngineForSources`、`EngineForMainSession` 等。 |
| [part.ts](../src/core/part.ts) | 领域组件的共享装配选项；`EngineOptions`。 |
| [intake.ts](../src/core/intake.ts) | 接收来源触发、范围检查和分发；`Intake`、`ReceiveResult`。 |
| [evals.ts](../src/core/evals.ts) | 协调待评估项及评估结果；`Evals`、`Outcome`。 |
| [tickets.ts](../src/core/tickets.ts) | Ticket 创建与生命周期操作；`Tickets`、`automatic`。 |
| [assignments.ts](../src/core/assignments.ts) | worker Assignment、输入和报告的领域操作；`Assignments`、`WorkerReport`。 |
| [replies.ts](../src/core/replies.ts) | 回复请求及外发状态的领域处理；`Replies`、`Reply`、`ReplyRequest`。 |
| [stage.ts](../src/core/stage.ts) | 计算 Ticket 展示阶段和重试资格；`ticketStage`、`canRetry`。 |
| [ids.ts](../src/core/ids.ts) | 构造领域标识；`key`。 |
| [errors.ts](../src/core/errors.ts) | 用户可见领域错误和错误码；`UserError`、`USER_ERRORS`。 |
| [reasons.ts](../src/core/reasons.ts) | 结构化原因代码及文本；`REASONS`、`Reason`、`reasonText`。 |
| [memory.ts](../src/core/memory.ts) | 内存版领域仓储实现；`MemoryRepositories`。 |
| [index.ts](../src/core/index.ts) | 领域公共入口，重导出模型、端口及相关接口。 |

## app：应用装配与服务

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [runtime.ts](../src/app/runtime.ts) | 装配领域引擎、存储、来源和后台服务并管理生命周期；`Runtime`。 |
| [store.ts](../src/app/store.ts) | SQLite 状态、事务和持久化操作；`Store`、`Receipt`、`SettingsVersion`。 |
| [store-read.ts](../src/app/store-read.ts) | 提供存储读取查询集合；`storeReads`。 |
| [migration.ts](../src/app/migration.ts) | 数据库及设置的版本迁移；`SCHEMA_VERSION`、`migrateTo11`、`migrateSettings`。 |
| [passes.ts](../src/app/passes.ts) | 驱动接收和到期评估等处理轮次；`Passes`、`PassesHost`。 |
| [effects.ts](../src/app/effects.ts) | 持久化副作用队列、调度和重试；`Effects`、`EffectKind`、`EffectPayloads`。 |
| [outbox.ts](../src/app/outbox.ts) | 排队和发送外部消息并记录回执；`Outbox`、`OutboxHost`。 |
| [assessment.ts](../src/app/assessment.ts) | 主 Session 评估适配及评估输入校验；`MasterAssessor`、`AssessmentSchema`、`EvalReadSchema`。 |
| [sessions.ts](../src/app/sessions.ts) | 管理主 Session 和 worker Session；`Sessions`、`WORKER_PRESET`。 |
| [delivery.ts](../src/app/delivery.ts) | 向 Agent 投递消息；`AgentDelivery`、`messageId`。 |
| [execution-environment.ts](../src/app/execution-environment.ts) | 生成执行根目录和 Ticket 的环境变量；`executionRoot`、`ticketEnvironment`、`environmentLines`。 |
| [source-context.ts](../src/app/source-context.ts) | 为来源适配器组装运行时上下文；`sourceContextOf`、`SourceContextDeps`。 |
| [settings.ts](../src/app/settings.ts) | 设置读取、保存和冲突处理；`Settings`、`SettingsConflict`。 |
| [settings-history.ts](../src/app/settings-history.ts) | 设置变更与策略历史展示数据；`SettingsChange`、`policyHistory`、`HistoryView`。 |
| [setup.ts](../src/app/setup.ts) | 检查应用部署和来源配置状态；`Setup`、`setupChanged`。 |
| [prompts.ts](../src/app/prompts.ts) | 提示词配置、内置片段、组合与版本冲突；`Prompts`、`PromptConfigSchema`、`composePrompt`、`composeSkill`。 |
| [context.ts](../src/app/context.ts) | 主 Session、worker 和团队提示上下文；`masterPrompt`、`workerPrompt`、`guides`、`transcriptEntry`。 |
| [skill-files.ts](../src/app/skill-files.ts) | 获取内置 skill 与提示文件；`bundledSkills`、`promptFile`。 |
| [resources.ts](../src/app/resources.ts) | 面向工具读取的资源语法和分发；`RESOURCES`、`resourceGrammar`、`readResource`。 |
| [role-models.ts](../src/app/role-models.ts) | 解析与初始化角色模型；`resolveRoleModel`、`initializeRoleModel`。 |
| [versions.ts](../src/app/versions.ts) | 应用版本管理；`Versions`。 |
| [retry-health.ts](../src/app/retry-health.ts) | 汇总重试健康和预算状态；`RetryHealth`、`BudgetRetryStatus`。 |
| [cache.ts](../src/app/cache.ts) | 文本缓存；`TextCache`。 |
| [notices.ts](../src/app/notices.ts) | 构造设置、订阅、事件、交接与结果通知；`settingsNotice`、`eventNotice`、`resultRelay`。 |
| [links.ts](../src/app/links.ts) | 来源链接、回复关系及候选卡片；`sourceLink`、`repliedTo`、`candidateCard`。 |
| [cards.ts](../src/app/cards.ts) | 将 Trigger、Ticket、Thread 和 Decision 转为卡片或文本；`triggerCard`、`ticketCard`、`threadCard`、`decisionCard`。 |
| [api.ts](../src/app/api.ts) | 管理 API 路由和请求响应类型；`Api`、`ApiRequest`、`ApiResponse`。 |
| [http.ts](../src/app/http.ts) | 注册 HTTP 接口、同源检查与状态视图；`registerHttp`、`sameOrigin`、`stateView`。 |

## app/tools：Session 工具

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [main.ts](../src/app/tools/main.ts) | 构建主 Session 可调用工具；`mainTools`、`MainToolsHost`。 |
| [executor.ts](../src/app/tools/executor.ts) | 构建 worker 执行工具；`executorTools`、`ExecutorToolsHost`。 |
| [tool.ts](../src/app/tools/tool.ts) | 通用工具注册、读取和确认 schema；`tool`、`readTool`、`AcknowledgementSchema`。 |

## app/views：管理端读取投影

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [overview.ts](../src/app/views/overview.ts) | 概览和评估健康投影；`overview`、`evalsHealth`。 |
| [records.ts](../src/app/views/records.ts) | 公共记录、所有者、状态、摘要和决策投影；`snapshot`、`triggerView`、`ticketSummary`、`threadSummary`、`decisionView`。 |
| [settings.ts](../src/app/views/settings.ts) | 设置视图投影；`settingsView`、`SettingsView`。 |
| [sources.ts](../src/app/views/sources.ts) | 来源健康、配置状态及丢弃回复；`sourceHealth`、`sourcesStatus`、`droppedReplies`。 |
| [text.ts](../src/app/views/text.ts) | 面板视图的文本、Markdown 和属性处理；`plain`、`markdown`、`readable`、`preview`。 |
| [threads.ts](../src/app/views/threads.ts) | Thread、Decision 列表详情与搜索；`threadList`、`threadDetail`、`decisionList`、`search`。 |
| [tickets.ts](../src/app/views/tickets.ts) | Ticket 列表、详情、依据及执行负载；`ticketList`、`ticketDetail`、`basisFacts`、`executorLoad`。 |

## sources：外部来源

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [kind.ts](../src/sources/kind.ts) | 来源能力、通道、上下文和配置字段契约；`SourceKind`、`SourceChannel`、`ThreadContext`、`SourceInstance`。 |
| [host.ts](../src/sources/host.ts) | 来源宿主上下文与健康状态契约；`SourceHost`、`SourceContext`、`health`。 |
| [registry.ts](../src/sources/registry.ts) | 注册来源种类、实例并提供来源能力；`SourceRegistry`。 |
| [poll.ts](../src/sources/poll.ts) | 来源轮询的窗口、游标与暂停工具；`pollThreads`、`cursorAt`、`POLL_HORIZON_MS`。 |
| [admission-context.ts](../src/sources/admission-context.ts) | 读取准入判断所需的线程上下文窗口；`threadWindow`、`admissionContext`。 |
| [github.ts](../src/sources/github.ts) | GitHub 来源插件、轮询与健康处理；`githubKind`、`apply`、`pollRound`、`pollTicketPulls`。 |
| [github-trigger.ts](../src/sources/github-trigger.ts) | GitHub 事件、地址及评论转换为 Trigger；`githubAddress`、`commentTrigger`、`issueTrigger`、`ciTrigger`。 |
| [github-read.ts](../src/sources/github-read.ts) | 读取仓库、线程和 CI 检查；`githubThreadContext`、`readRepositoryPath`、`pullChecks`、`ciSummary`。 |
| [github-pull.ts](../src/sources/github-pull.ts) | 关联、读取和观察 PR，生成结束通知；`readPull`、`pullsOf`、`claimPull`、`observePull`。 |
| [github-channel.ts](../src/sources/github-channel.ts) | GitHub 消息发送与链接；`sendGithub`、`githubLink`。 |
| [feishu.ts](../src/sources/feishu.ts) | 飞书来源插件与配置；`feishuKind`、`feishuSettings`、`apply`。 |
| [feishu-trigger.ts](../src/sources/feishu-trigger.ts) | 飞书事件与历史消息转 Trigger；`feishuTrigger`、`feishuThreadId`、`historyEvent`。 |
| [feishu-read.ts](../src/sources/feishu-read.ts) | 读取飞书消息、链接和线程上下文；`readFeishu`、`feishuRead`、`feishuThreadContext`。 |
| [feishu-image.ts](../src/sources/feishu-image.ts) | 提取准入图片键并读取图片附件；`admissionImageKeys`、`readAdmissionImage`。 |
| [feishu-loops.ts](../src/sources/feishu-loops.ts) | 飞书消费循环、群名刷新、历史回填和线程轮询；`runFeishu`、`backfillChat`、`pollThreads`。 |
| [feishu-channel.ts](../src/sources/feishu-channel.ts) | 飞书消息发送与会话链接；`sendFeishu`、`feishuLink`、`feishuChatLink`。 |

## executor：worker 执行

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [dsh.ts](../src/executor/dsh.ts) | DSH worker Session 的启动、输入、并发槽与执行器适配；`DshExecutor`、`WorkerSlots`、`workerSessionId`。 |
| [background.ts](../src/executor/background.ts) | 等待、收集或停止后台任务和子代理；`untilQuiet`、`collectJob`、`stopSubagents`、`stopBackground`。 |

## infra：基础设施

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [commands.ts](../src/infra/commands.ts) | 外部命令调用、错误和结果统计；`Commands`、`CliError`、`validItems`。 |
| [environment.ts](../src/infra/environment.ts) | worker 命令、Git 凭据和执行环境配置及校验；`writeEnvironment`、`applyEnvironment`、`runAsWorker`、`verifyEnvironment`。 |
| [health.ts](../src/infra/health.ts) | 健康状态条目和看板；`HealthBoard`、`HealthEntry`。 |
| [paths.ts](../src/infra/paths.ts) | 包根目录及包内路径解析；`packageRoot`、`packagePath`。 |
| [text.ts](../src/infra/text.ts) | 字数、消息文本校验与字符串工具；`wordCount`、`checkedImText`、`fill`、`sameName`。 |

## client：管理面板

| 文件 | 实际职责与关键导出 |
| --- | --- |
| [index.tsx](../src/client/index.tsx) | 客户端插件入口及注入；`inject`、`apply`。 |
| [app.tsx](../src/client/app.tsx) | 面板根组件；`App`、`AppProps`。 |
| [provider.tsx](../src/client/provider.tsx) | 面板状态、数据和操作上下文；`PanelProvider`、`usePanel`、`usePanelData`、`useAction`。 |
| [api.ts](../src/client/api.ts) | 客户端 API 调用、轮询、错误及视图类型；`caller`、`usePoll`、`ApiError`。 |
| [route.ts](../src/client/route.ts) | 标签、设置分组和客户端路由解析；`parseRoute`、`settingsRoute`、`initialLocation`。 |
| [overview.tsx](../src/client/overview.tsx) | 概览页面；`OverviewPage`。 |
| [ticket.tsx](../src/client/ticket.tsx) | Ticket 详情页面；`TicketPage`。 |
| [thread.tsx](../src/client/thread.tsx) | Thread 详情页面；`ThreadPage`。 |
| [rows.tsx](../src/client/rows.tsx) | Ticket、Thread、Decision 列表行和历史内容；`TicketLine`、`ThreadLine`、`DecisionLine`。 |
| [settings.tsx](../src/client/settings.tsx) | 设置页面入口；`SettingsPage`。 |
| [settings-kit.tsx](../src/client/settings-kit.tsx) | 设置上下文、草稿及字段组件；`SettingsProvider`、`useBlockDraft`、`ModelField`、`ConfigHistory`。 |
| [source.tsx](../src/client/source.tsx) | 来源设置和状态组件；`SourceSettings`、`SourceStatus`。 |
| [deployment.tsx](../src/client/deployment.tsx) | 部署配置表单与页面；`DeploymentForm`、`DeploymentPage`。 |
| [eval.tsx](../src/client/eval.tsx) | 评估状态及设置组件；`EvalStatus`、`EvalSettings`。 |
| [execution.tsx](../src/client/execution.tsx) | 执行状态及设置组件；`ExecutionStatus`、`ExecutionSettings`。 |
| [trigger.tsx](../src/client/trigger.tsx) | Trigger 状态组件；`TriggerStatus`。 |
| [prompts.tsx](../src/client/prompts.tsx) | 提示词配置页面；`PromptsPage`。 |
| [prompt-config.ts](../src/client/prompt-config.ts) | 提示词片段编辑、覆盖及差异处理；`withOverride`、`changedSections`、`lineDiff`。 |
| [draft.tsx](../src/client/draft.tsx) | 编辑草稿状态、合并和保存栏；`draftReducer`、`useDraft`、`merge3`、`SaveBar`。 |
| [controls.tsx](../src/client/controls.tsx) | 基础交互控件；`Button`、`Input`、`Switch`、`Modal`、`Icon`。 |
| [widgets.tsx](../src/client/widgets.tsx) | 可复用页面展示组件；`MarkdownLite`、`ConfirmButton`、`LoadState`、`PageHead`。 |
| [marks.ts](../src/client/marks.ts) | Ticket、Thread、来源和决策的展示标记；`ticketMark`、`threadMark`、`sourceMark`。 |
| [markdown.ts](../src/client/markdown.ts) | 轻量 Markdown 分块与行内解析；`markdownBlocks`、`inlines`、`linkify`。 |
| [text.ts](../src/client/text.ts) | 客户端本地化、错误、状态和时间文本；`errorText`、`localized`、`healthText`、`when`。 |
| [locales.ts](../src/client/locales.ts) | 中文词条和术语表；`zh`、`GLOSSARY`、`DshSuperagentKey`。 |
| [client.css](../src/client/client.css) | 管理面板样式表，无 TypeScript 导出。 |
| [css.d.ts](../src/client/css.d.ts) | CSS 模块的 TypeScript 声明，无运行时导出。 |
