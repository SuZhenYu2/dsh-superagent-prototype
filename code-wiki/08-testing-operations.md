# 测试、运维与排障

本文区分仓库中定义的测试和部署后的实际验收；没有运行或没有外部环境支持的检查，不应记录为已通过。部署前置条件见 [开发与部署](07-development-deployment.md)，原始依据见 [部署指南](../docs/006-deployment.md)。

## 测试入口

在仓库根目录安装依赖后执行：

```sh
pnpm typecheck
pnpm test
pnpm build
pnpm check
pnpm exec vitest run tests/service-script.spec.ts
```

前四条分别执行静态类型检查、全部测试、构建和三者的顺序组合；最后一条用于定位单个测试文件。脚本定义见 [package.json](../package.json)，Vitest 匹配 `tests/**/*.spec.{ts,tsx}`，并去重 React 依赖，见 [vitest.config.ts](../vitest.config.ts)。`pnpm check` 已包含前三项，无需为同一轮验收重复运行。命令结果依赖本机 Node、pnpm、依赖及运行环境，应保留退出码和失败输出。

| 测试主题 | 代表文件 | 关注点 |
| --- | --- | --- |
| 领域与调度 | [core-engine.spec.ts](../tests/core-engine.spec.ts)、[queue.spec.ts](../tests/queue.spec.ts)、[runtime-concurrency.spec.ts](../tests/runtime-concurrency.spec.ts) | 核心引擎、队列与并发行为。 |
| 配置与部署 | [deployment.spec.ts](../tests/deployment.spec.ts)、[setup 相关测试](../tests/environment.spec.ts)、[service-script.spec.ts](../tests/service-script.spec.ts) | 配置、执行环境和服务控制。服务脚本测试使用临时目录及模拟程序，不等于真实 DSH 部署。 |
| 输入与外部渠道 | [sources.spec.ts](../tests/sources.spec.ts)、[github-read.spec.ts](../tests/github-read.spec.ts)、[feishu-read.spec.ts](../tests/feishu-read.spec.ts) | Source 行为及外部消息读取逻辑。 |
| 持久化与恢复 | [store.spec.ts](../tests/store.spec.ts)、[migration.spec.ts](../tests/migration.spec.ts)、[retry-health.spec.ts](../tests/retry-health.spec.ts) | 状态存储、迁移及重试健康状态。 |
| 面板 | [client.spec.tsx](../tests/client.spec.tsx)、[panel-settings.spec.tsx](../tests/panel-settings.spec.tsx)、[panel-deployment.spec.tsx](../tests/panel-deployment.spec.tsx) | 客户端及设置、部署界面。 |

这些测试文件是入口示例，不代表覆盖所有功能。单元或模拟测试不能替代真实账号权限、模型调用、外部事件和完整交付链路验证。

## 本次核验记录

2026-11-10 在 macOS、Node.js `v26.12.0`、pnpm `11.7.0` 环境下，对提交 `8c32a9dc5d72` 对应的工作区进行了检查：

- 默认 `pnpm check`：类型检查通过；测试为 1195 通过、2 失败，因此组合命令未进入构建。两处失败均在 [environment.spec.ts](../tests/environment.spec.ts) 中，断言英文 Git 错误文本，而实际收到中文输出。
- `LC_ALL=C pnpm check`：类型检查通过；测试为 1196 通过、1 失败，剩余用例构造了仅包含指定环境变量的 Git 子进程环境，仍收到中文身份错误；因此不能假设只设置父进程 locale 就能通过所有测试。
- 单独执行 `pnpm build`：通过，生成服务端和客户端构建产物。
- 环境中未找到 `tmux`；未启动真实 DSH 服务，也未验证 GitHub、飞书、模型调用与最终交付。上述测试失败表明测试对错误文本语言存在依赖，不能据此断言实际凭据隔离功能失效，也不能称全量测试通过。

## 日常运行

```sh
scripts/dsh-superagent start
scripts/dsh-superagent status
scripts/dsh-superagent restart
scripts/dsh-superagent stop
```

默认 profile 为 `superagent`、健康地址为 `http://127.0.0.1:3082`；非默认环境可用 `--profile`、`--url` 或环境变量 `DSH_SUPERAGENT_PROFILE`、`DSH_SUPERAGENT_URL`。脚本在 tmux 中启动 DSH；停止和重启会等待执行器优雅退出，默认最多 300 秒，不强杀。具体行为和参数见 [服务脚本](../scripts/dsh-superagent)。

`status` 输出 JSON 中的 `pids`、`health`、`errors`、`setup`、`feishu` 和 `manually_stopped`。`start`、`restart` 和 `status` 的退出码：

| 退出码 | 含义 | 处理 |
| --- | --- | --- |
| 0 | 正常状态下服务可用；Source 启动中或 Feishu setup 未通过也可能返回 0。使用 `--respect-stop` 遇人工停止标记时，start/restart 会跳过启动并返回 0。 | 结合输出、`manually_stopped` 和 `feishu` 判断；不等同端到端验收。 |
| 1 | 未运行、重复运行、无响应或 runtime 错误。 | 检查进程和最新日志，必要时重启。 |
| 2 | 非 runtime 组件报错。 | 查看 `errors` 中的组件及详情；仅重启通常无法修复。 |
| 3 | 等待必需的 setup。 | 查看 `setup` 和面板提示，补齐配置。 |

状态判断时 runtime 错误优先返回 1；runtime 可用后必需的 setup 问题优先于非 runtime 组件错误，返回 3。参数无效时命令行解析器也可能返回 2，应结合错误输出区分，而非一律视为组件故障。

启动日志在仓库 `.artifacts/dsh-superagent-<profile>-<日期>-<时间>-<pid>.log`，脚本保留每个 profile 最近 20 个；控制文件在 `.artifacts/control/profiles/<profile>/`。检查日志时避免泄露账号、会话内容和其他敏感信息。

## 监控与状态检查

管理 API 可读取健康与配置；下面端口需与实际配置一致：

```sh
curl -sS -H 'x-orchestrator-request: 1' http://127.0.0.1:3082/orchestrator/api/state
curl -sS -H 'x-orchestrator-request: 1' http://127.0.0.1:3082/orchestrator/api/sources
```

`state` 包含 setup、各组件 health、执行负载与暂停状态；`sources` 提供渠道状态、统计和丢弃原因。`runtime` 是应用本身；`github`、`feishu`、`history` 与 `sync:<source id>` 对应渠道、历史恢复和名单同步；`assessment`、`launch`、`delivery` 等反映处理链路。响应及状态定义参见 [Management API](../docs/006-deployment.md#management-api) 和 [api.ts](../src/app/api.ts)。

定时监控应运行在独立于被监控服务的环境：退出码 2、3 通知并处理具体问题，不盲目重启；退出码 1 且非人工停止时，可使用 `scripts/dsh-superagent --profile superagent --respect-stop restart`。`stop` 写入人工停止标记，监控应尊重它；完整小时检查提示词见 [Hourly check](../docs/006-deployment.md#hourly-check)。

## 验收与故障定位

真实环境验收前确认模型、账号、网络、Source、Policy、Execution 已配置；在获授权的测试仓库，由允许名单内用户发起提及，观察触发、Ticket、独立目录、执行与原渠道交付。记录账号权限、时间、Ticket 和最终结果；不要把观察到 Ticket 等同于完整交付成功。

| 现象 | 排查方向 |
| --- | --- |
| 面板无法连接或退出码 1 | 查最新日志、配置合法性、DSH 版本、端口和重复进程；服务可能正在重启。 |
| 退出码 2 或 3 | 分别检查 `errors` 或 `setup`；修正外部认证后在面板选择 Check again。 |
| 找不到 GitHub CLI 或不能读取账号 | 检查 Source 中 wrapper 的绝对路径及 PATH，以代理身份执行 `auth login` 和 `api user --jq .login`。 |
| 仓库不可读或不可推送 | 检查仓库 `owner/name`、代理账号邀请和写权限；该项目不支持 GitHub Enterprise Server。 |
| prepare 失败或 Git 凭据冲突 | 查看 Ticket 错误，检查克隆路径、worktree 命令、Git 版本和仓库本地 credential/extraheader；人工重试前检查残留工作目录，因为目录已存在时 prepare 不会重跑。 |
| Feishu 不工作 | 检查 CLI 是否仅允许 bot 身份、应用凭据、发布状态和机器人是否在群内；Feishu setup 失败不一定使整体 status 非零。 |
| 提及后无动作 | 检查 Source 启用、作者允许名单、执行是否暂停；GitHub 在已有 Issue 正文编辑中新增提及不会触发，应发新评论。 |
| 设置保存失败 | 检查 API 错误：400 可能为值或账号检查失败，409 表示 revision 过期；重新读取配置后再修改。 |
| 会话恢复提示 `agent-preset/not-found` | 检查 profile 中 worker/router 预设配置是否被移除，详见 [Agent presets](../docs/006-deployment.md#agent-presets)。 |

更完整的错误文字和处理办法见 [Troubleshooting](../docs/006-deployment.md#troubleshooting)。升级和回滚时先备份数据库；恢复应在安全环境验证，避免覆盖仍需保留的运行数据。
