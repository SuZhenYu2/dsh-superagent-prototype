# 执行、事务、存储与重试

## 执行器

[DshExecutor](../src/executor/dsh.ts)实现核心 Executor 接口：每个 Ticket 使用固定的 `worker-<ticketId>` Session，工作目录由配置的执行根目录、通过校验的名称和工单 ID 后缀构成。Assignment 可更换，但重试复用同一 Session、目录、分支与 PR；Session 创建后不会因执行器模型设置变化自动切换模型。来源 Trigger 和工单上下文经格式化后送入 worker，较长内容通过资源路径读取。

首次启动时，如果工作目录不存在，执行器可在执行根目录运行配置的 prepare 命令，超时为 10 分钟。失败记录输出并使工单失败；自动瞬时重试不涵盖 prepare，避免把失败后留下的目录当成准备完成。失败留下的非空目录可能被保留，而后续启动发现目录已存在就跳过 prepare，因此人工重试前应检查并安全处理残留目录，不能假设 prepare 会重跑。执行变量由[执行环境](../src/app/execution-environment.ts)组装，来源可追加各自变量；包装命令与目录只是约定，**不是安全隔离边界**，worker 以宿主操作系统用户权限执行。

### 并发、后台任务与交接

`WorkerSlots` 在异步启动之前同步预留名额，准备目录、启动、唤醒、运行中的 turn 和需等待的后台任务都会占槽。普通启动受 `maxWorkers` 限制；人的后续消息或状态请求可额外使用一个槽，避免新工作阻塞人工继续处理。启动与 Input 投递都通过持久化 effect 调度，暂停处理时不再启动或投递。

turn 结束后若仍有 shell job 或子智能体，执行器继续占槽并等待；通常最多等待 30 分钟，超时终止 shell job，子智能体可继续。向人交接时立即释放槽并停止子智能体；留给人使用的 shell job 可以持续到 Assignment 结束，不再占槽或因完成而唤醒 worker。Assignment 结束及宿主停止时的后台清理由 [background.ts](../src/executor/background.ts) 和 [DshExecutor](../src/executor/dsh.ts) 协同完成。

## SQLite 与事务

[Store](../src/app/store.ts)使用 Node 的 `DatabaseSync`，数据库开启 WAL 和 `synchronous=FULL`；创建状态目录时指定 `0700`，打开数据库时将数据库文件设置为 `0600`。已有状态目录不会因该创建调用自动修正权限，应单独检查。运行时持有状态目录锁后打开数据库；支持的旧 schema 会事务迁移，当前实现将版本 10 迁至 11，过旧或更新的版本拒绝打开。迁移细节见 [migration.ts](../src/app/migration.ts) 和 [runtime.ts](../src/app/runtime.ts)。

| 数据 | 存储及作用 |
| --- | --- |
| Trigger、Disposition、Thread、Ticket、Decision | 各自的表保存 JSON 聚合；`trigger_thread` 索引线程时间线。 |
| KV | 类型化键保存来源游标、配置与提示词版本等；键的所有权见 [kv.ts](../src/kv.ts)。 |
| Effect | 持久化待执行的副作用、顺序、代次、失败次数与到期时间。 |
| Outbox | 保存渠道消息及 `pending`、`sent`、`dropped` 回执，已结算 ID 用于去重。 |

活跃工单、活跃和已订阅线程，以及待发送消息使用 SQLite 生成列和部分索引缩小工作集；面板等读取查询在 [store-read.ts](../src/app/store-read.ts)。设置保存具有全局历史版本和按配置块的 revision，供并发修改检测使用。

`atomic` 使用可嵌套的 SQLite savepoint：内层失败回滚该层，外层提交之前内层成功也不会单独持久化。`afterCommit` 的回调只在最外层成功提交后执行；回滚的层丢弃其回调，回调异常仅报告，不能撤销已经提交的事务。它适合不要求崩溃恢复的内存操作；必须跨重启完成的副作用应在事务中写入 effect，而非只依靠回调。事务回调为同步函数，不应将跨 `await` 的远程调用当作 SQLite 事务的一部分。

## 持久化副作用与消息

[Effects](../src/app/effects.ts)以 `(kind, key)` 合并队列项，保留原顺序 `seq`，更新载荷时递增 `generation`；执行结束只删除或更新匹配原 `seq` 和 `generation` 的行，避免覆盖执行期间新排入的内容。注册处理器可设置并发、顺序和退避；`start` 前不执行，`drain` 等待运行中的任务，未完成项留在数据库供下次启动。种类包括 assessment、launch、delivery、relay、notice 和 reply。

队列默认失败后 2 秒重试，之后翻倍至 60 秒；`Later` 表示暂时没有条件执行，只推迟到初始间隔，不增加失败次数。默认单项连续失败三次后对应种类显示异常，delivery 的错误在工单上显示。顺序处理的种类由最早项阻塞后续项；具体处理器可覆盖并发和重试参数。持久化队列提供的是可恢复重试，而不是对外部系统的通用恰好一次保证：外部操作成功与本地记录完成之间仍可能崩溃，因此渠道和操作需要自己的幂等处理。

[Outbox](../src/app/outbox.ts)在同一事务中插入消息回执并排入 reply effect，重复消息 ID 不再入队。发送失败从 5 秒开始指数退避，最多 300 秒；同一工单发往同一会话的较早消息未完成时阻塞后续消息，但其连续失败三次后不再阻塞。来源禁用或缺失时等待；不再监听目标或目标永久不可达时记录 dropped，否则成功后记录 sent。发送中撤回与完成可能竞争，结算逻辑仍保留回执以防重复入队。

## 重试与重启恢复

- **执行瞬时失败**：[dsh.ts](../src/executor/dsh.ts)识别数值状态 408、429、至少 500，以及连接类错误；仅从错误文本匹配状态码时范围为 408、429、500–504。启动或 worker turn 有最多三次的自动瞬时失败处理，turn 初始退避为 15 秒；prepare 失败不自动重试。失败或 blocked 后，非人工输入不会自动唤醒，人工消息、后续有效报告或手动重试可解除等待。
- **人工重试**：工单处于投递失败、blocked 或 failed 等可重试状态时，以新的 Assignment 继续同一 Session 与工作目录，检查现有工作而不是假设从零开始。[工单阶段](../src/core/stage.ts)和[赋派逻辑](../src/core/assignments.ts)定义状态与转换。
- **宿主重启**：未处于稳定状态的运行中 Assignment 经恢复流程结束并重启，恢复说明要求先检查目录、git 和 GitHub，避免重复外部操作；如果最近报告为 `waiting`、`ready`、`blocked` 或 `failed` 且报告后没有已投递输入，则保留原 Assignment 和 Session，并在有待投递输入时继续投递。待处理输入、持久化 effect 和消息回执继续保留；输入已投递与已被 Session 读取是不同事实。实现参见 [assignments.ts](../src/core/assignments.ts)、[runtime.ts](../src/app/runtime.ts)、[sessions.ts](../src/app/sessions.ts) 和 [dsh.ts](../src/executor/dsh.ts)。
- **来源失败**：GitHub 的通知重试与飞书消费、历史补收各有独立策略，不应与 effect 重试混为一谈，详见[数据源](03-sources.md)。
- **模型预算状态**：[retry-health.ts](../src/app/retry-health.ts)根据 Session 结构化事件投影预算重试健康状态；它只观察并展示，不修改底层重试策略。
