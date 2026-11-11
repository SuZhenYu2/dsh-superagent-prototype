# 配置与提示词

系统将机器级部署配置、运行时设置和提示词版本分开：前者决定本地状态目录及资源上限，运行时设置决定来源、策略和执行方式，提示词决定模型收到的指令。面板入口见 [settings.tsx](../src/client/settings.tsx)，各字段的最终约束以 [deployment.ts](../src/deployment.ts)、[policy.ts](../src/policy.ts) 和对应 Source schema 为准。

## 机器级配置与启动

[deployment.ts](../src/deployment.ts) 的 `Config` 定义四项，实际范围集中在 [limits.ts](../src/limits.ts)：

| 字段 | 含义与默认值 |
| --- | --- |
| `stateDir` | 必填绝对路径；保存数据库、锁、worker 命令包装器、Git 配置，默认也存放 Ticket 工作目录。修改会重新挂载插件。 |
| `promptMaxChars` | 已保存提示词配置的最大字符数，默认 100000；动态生效。 |
| `admissionContextMaxPages` | 一次评估可读取的 Thread 上下文页数，默认 10；动态生效。 |
| `admissionContextTimeoutMs` | 一次评估上下文读取的超时毫秒数，默认 15000；动态生效。 |

`machineOf` 拒绝未知配置项，并对部分旧字段提示迁移位置；[index.ts](../src/index.ts) 创建运行时。GitHub 仓库与 CLI 放在 GitHub Source：启动时解析可执行文件、验证当前账号和仓库 push 权限。可选的飞书 CLI 必须是已就绪的 bot 身份，且 strict mode 为 `bot`；具体检查见 [deployment.ts](../src/deployment.ts)。失败以 [setup.ts](../src/setup.ts) 中的结构化 code、place、values 报告到对应设置页，修复后可重新检查。

## 运行时设置

[policy.ts](../src/policy.ts) 的 `SettingsSchema` 将设置分为 `paused`、`sources`、`policy`、`eval` 和 `executor`。各 Source 的具体字段与默认值由来源实现提供，参见 [github.ts](../src/sources/github.ts)、[feishu.ts](../src/sources/feishu.ts) 和 [registry.ts](../src/sources/registry.ts)。面板编辑入口分别在 [source.tsx](../src/client/source.tsx)、[eval.tsx](../src/client/eval.tsx)、[execution.tsx](../src/client/execution.tsx)。

| 设置块 | 字段和行为 |
| --- | --- |
| `source:<id>` | `enabled` 与来源专属 `config`；GitHub 配置仓库、账号和接收范围，飞书配置机器人与接收范围。账户相关字段变化会触发启动检查及重启。 |
| `policy` | `rules` 是含 `id`、`name`、`enabled`、`instruction` 的规则；`sources` 保存各来源的允许身份和 admission 表等。默认规则为 copy 和 visual。 |
| `eval` | `mode` 为 `off`、`observe`、`auto`，默认 `observe`；`timeoutSeconds` 默认 120，`model` 默认 `null`。角色模型包含 provider、model 和可选 reasoningEffort。 |
| `executor` | `model` 默认 `null`；`maxWorkers` 默认 3；`directory` 为空时使用 `<stateDir>/work`，否则必须是绝对路径；另有 `prepare` 和 `blockedCommands`。 |
| `pause` | 独立保存的 `{ paused }`，默认 false；暂停新的评估、worker 派发和渠道投递，但 Source 继续记录 Trigger。 |

允许名单与来源接收范围共同参与准入；已允许且直接指向 agent 的人类请求有明确授权路径。推断候选还受 admission、模式和启用规则约束；`observe` 可记录本会接受的候选但不自动接受，`auto` 才允许符合条件的推断工作，`off` 关闭推断处理。完整判定、例外与既有授权重验证见 [settingsPolicy](../src/policy.ts) 和 [admission.ts](../src/admission.ts)，不要仅凭模式推断所有请求的行为。

`policyVersion` 从规则、Eval 模式和各来源策略计算；模型、并发数、暂停和轮询周期不参与版本计算。保存导致策略版本变化时，[settings.ts](../src/app/settings.ts) 计算对未结束 Ticket 的影响并通知引擎；来源策略同步失败以 1 分钟至 1 小时的指数退避重试。

## 保存、并发与权限

设置按块携带 revision 保存，不同块可以独立并发；保存前及异步验证后都会检查 revision，过期则返回 409。服务端重新读取其他块并合并，未发生实际变化不产生新 revision。模型变更先检查模型可用性；必要时验证账户；配置变化可触发重启或请求立即执行一轮处理。实现见 [app/settings.ts](../src/app/settings.ts)，持久化见 [store.ts](../src/app/store.ts) 和 [kv.ts](../src/kv.ts)。

只有面板来源可以修改执行路径、prepare、被禁命令和来源账号相关字段；这是防止提示注入借工具直接修改这些设置的护栏，**不是操作系统安全边界**。暂停是独立块，因此不使并发中的 executor 编辑失效。策略同步和保存历史见 [settings-history.ts](../src/app/settings-history.ts)。客户端草稿恢复与冲突合并详见 [管理面板、状态与 API](05-panel-api.md)。

## 提示词来源与组装

[context.ts](../src/app/context.ts) 从文件加载默认文本，[prompts.ts](../src/app/prompts.ts) 定义可覆盖部分并组装角色提示词。默认文件使用英文，面板支持修改团队段落、覆盖内置部分以及添加团队 skills。

| 层次 | 文件和作用 |
| --- | --- |
| 共享与角色 | [shared.md](../prompts/shared.md)、[orchestrator.md](../prompts/orchestrator.md)、[worker.md](../prompts/worker.md)：共享约束和各自职责。 |
| 团队段落 | [communication.md](../prompts/team/communication.md)、[scope.md](../prompts/team/scope.md)、[delivery.md](../prompts/team/delivery.md)：沟通、Ticket 范围和交付要求。 |
| 工作流程 | [subscriptions.md](../prompts/subscriptions.md)、[replies.md](../prompts/replies.md)、[dispatch.md](../prompts/dispatch.md)、[work.md](../prompts/work.md)、[admission.md](../prompts/admission.md)：订阅、回复、派发、执行和准入。 |
| 交接 | [handoff.md](../prompts/handoff.md) 与 [handoff-orchestrator.md](../prompts/handoff-orchestrator.md)、[handoff-worker.md](../prompts/handoff-worker.md) 按角色组合。 |
| Source skills | [飞书 SKILL.md](../skills/dsh-superagent-feishu/SKILL.md)、[GitHub SKILL.md](../skills/dsh-superagent-github/SKILL.md)，并附加 [cli.md](../prompts/cli.md)；飞书 skill 仅在配置了 bot 时注册。 |

组装顺序为共享、角色、三个团队段落、订阅和回复、主 Session 的 dispatch 或 worker 的 work、仅主 Session 的 admission、角色交接、生成的执行环境、仅主 Session 的生成策略。空段落被跳过。策略段位于末尾，按当前设置生成版本、模式和启用规则；执行环境由安装配置或 Ticket 上下文生成。组装实现和预览返回的分段结果见 [app/prompts.ts](../src/app/prompts.ts)，环境变量见 [execution-environment.ts](../src/app/execution-environment.ts)。Worker 还继承宿主编码提示词和仓库 `AGENTS.md`；这些不等同于面板预览中的团队配置。

## 配置、覆盖与版本

`PromptConfig` 包含 `communication`、`scope`、`delivery`、`skills` 和可选 `overrides`。团队 skill 名须匹配 `^[a-z][a-z0-9-]*$`，不能占用内置 Source skill 名；注册时使用 `dsh-superagent-` 前缀。可覆盖的 key 及适用角色由 [builtInParts](../src/app/prompts.ts) 明确列举，包括 `runtime/shared`、角色、工作流、交接、环境和策略模板，以及 `skill/feishu`、`skill/github`。

生成模板必须保留占位符：`runtime/environment` 要求 `{variables}`；`runtime/policy` 要求 `{version}`、`{mode}`、`{rules}`；`runtime/policy/none` 要求 `{version}`、`{mode}`。覆盖文本不能为空，未知部分被拒绝；已保存但新版移除的部分在下次保存时丢弃。覆盖恢复成当前默认文本时会移除覆盖，以便日后自动跟随默认更新。完整配置的 JSON 字符串长度受 `promptMaxChars` 限制，不会静默截断。

覆盖记录所依据的内置文本 `base`。升级后若默认文本改变，`defaultChanged` 标记差异；面板可比较并选择保留自己的文本或使用新默认，未覆盖的部分自动跟随内置版本。界面实现见 [client/prompts.tsx](../src/client/prompts.tsx) 和 [prompt-config.ts](../src/client/prompt-config.ts)。

首次保存前为默认 revision 0；保存写入完整配置、新 revision、前驱、时间及来源。提交必须携带 `baseRevision`，版本不一致返回 409；历史分页可用于比较或恢复，恢复仍产生新版本。保存通过 [savePrompts](../src/app/prompts.ts) 通知主 Session，效果标记为 `next-request`：下次模型请求使用最新提示词，不意味着重启或取消正在运行的 Session。版本存储机制见 [versions.ts](../src/app/versions.ts)，Session 读取行为见 [sessions.ts](../src/app/sessions.ts)。

**安全边界：**提示词和 skill 只改变模型看到的指令，不改变代码实施的 gate、工具参数校验、身份与账户包装器。外部消息、评论和文档是输入数据，不应被当作权限来源；相关规则见 [shared.md](../prompts/shared.md)、[policy.ts](../src/policy.ts) 和 [tools/main.ts](../src/app/tools/main.ts)。
