# 依赖关系与宿主边界

本文区分源码 import、运行时注入和包声明：`A → B` 表示 A 使用 B，不表示 B 在启动时一定可用。`import type` 主要是编译期契约；Cordis 服务由宿主装配，不能仅凭类型 import 推断运行时已经安装。版本和完整清单以 [package.json](../package.json) 为准。

## 内部依赖方向

| 使用方 | 依赖及证据 | 边界 |
| --- | --- | --- |
| 插件入口 | [index.ts](../src/index.ts) → Runtime、部署配置；导出 [core/index.ts](../src/core/index.ts) | 宿主调用入口，入口创建 Runtime；它不是独立 HTTP 服务器。 |
| 应用装配 | [runtime.ts](../src/app/runtime.ts) → core、policy、sources、executor、infra 和 app 内组件 | 实际组合根；并非所有目录严格单向分层。 |
| 领域 | [engine.ts](../src/core/engine.ts) → 同目录领域模块和 [ports.ts](../src/core/ports.ts) | 核心经接口使用仓储、策略、来源范围、评估、执行和外发，不直接 import 它们的具体实现；但 [ids.ts](../src/core/ids.ts) 使用 Node `crypto`，不是完全无平台依赖。 |
| 来源 | [github.ts](../src/sources/github.ts)、[feishu.ts](../src/sources/feishu.ts) → 来源转换/读写模块、core 类型、部署与基础设施；[registry.ts](../src/sources/registry.ts) → core 端口、policy 和健康状态 | 各来源通过 [SourceHost](../src/sources/host.ts) 与宿主交互；Registry 实现 SourceScope 并选择 Channel。 |
| 执行 | [executor/dsh.ts](../src/executor/dsh.ts) → core、应用服务及 DSH 类型 | Executor 适配 worker 生命周期，通过 Engine 回报结果；其实现与 app 存在交叉引用，不能把 executor 视作完全独立的下层库。 |
| 控制面 | [http.ts](../src/app/http.ts) → 应用 API、视图和宿主 webServer；[client/api.ts](../src/client/api.ts) → API 类型、core 错误常量，同源 HTTP 请求 | 浏览器不直接调用 Runtime；API 的类型共享不等于将服务端应用完整打入浏览器。 |
| 公共能力 | [policy.ts](../src/policy.ts)、[deployment.ts](../src/deployment.ts)、[infra/commands.ts](../src/infra/commands.ts) | 策略和配置被多个部分使用；命令适配器提供 GitHub、飞书和 Git 进程调用。 |

例如，来源插件使用 SourceHost，Runtime 实现该服务，但 Runtime 也直接引用飞书设置和 GitHub PR 通知；这是实际存在的具体来源耦合，不应描述成完全插件无关。证据见 [runtime.ts](../src/app/runtime.ts) 和 [source-context.ts](../src/app/source-context.ts)。

## 运行时装配

1. [入口](../src/index.ts) 声明服务注入，转换部署配置并创建 Runtime；退出时停止 Runtime，账号变更可触发重启。
2. [Runtime](../src/app/runtime.ts) 锁定状态目录，创建 SQLite Store、Setup、Effects、SourceRegistry、Sessions、Settings、DshExecutor、MasterAssessor 和 Outbox；然后将 Store、Registry、策略函数及评估/执行/外发适配器传给 Engine。Engine 依赖端口，具体实现由组合根选择。
3. [bundle patch](../cordis.patch.yml) 安装主插件和 GitHub watcher，飞书 watcher 默认禁用；两个来源各自是独立插件，并通过宿主服务注册来源能力。具体注册和生命周期见 [github.ts](../src/sources/github.ts)、[feishu.ts](../src/sources/feishu.ts) 和 [host.ts](../src/sources/host.ts)。
4. [Setup](../src/app/setup.ts) 探测仓库及账号、准备 worker 环境；飞书探测失败单独记录并排除该来源。启动顺序、HTTP 注册及后台任务启动条件以 [runtime.ts](../src/app/runtime.ts) 为准。
5. Engine 通过端口排队工作，应用侧 [Effects](../src/app/effects.ts)、[Outbox](../src/app/outbox.ts)、[Sessions](../src/app/sessions.ts) 及执行器负责具体异步动作；消息经 Registry 分发到来源 Channel。

## DSH 与 Cordis 宿主

[服务端入口的 inject](../src/index.ts) 要求 `agents`、`sessionController`、`sessionPersistence`、`workspaceRegistry`、`permissionPresets`、`tools`、`systemPrompt`、`skills`、`webServer`、`llm`、`subprocess`。`settings` 通过条件注入使用，不在该必需列表中。服务名是上下文能力，不等同于 npm 包名；具体消费可见 [sessions.ts](../src/app/sessions.ts)、[http.ts](../src/app/http.ts) 和 [setup.ts](../src/app/setup.ts)。

浏览器入口的必需服务是 `slots`、`locale`、`sessions`、`uiWorkspace`；`configForms` 可选，缺失时机器设置只读，见 [client/index.tsx](../src/client/index.tsx)。[package.json](../package.json) 的 `dsh.client.inject` 声明客户端插件注入列表，与源码中的服务注入列表属于不同层级，不应逐项等同。面板使用 React，并经宿主模块加载器挂载，见 [tsdown.config.ts](../tsdown.config.ts)。

主 Session 和 worker 的 preset 由 [cordis.patch.yml](../cordis.patch.yml) 配置，包含 DSH agent-preset、shell/sandbox、文件、skill、压缩等宿主插件；worker 还配置子代理、搜索和待办等能力。这些插件名是运行时 bundle 配置，不代表都在本包 `peerDependencies` 中声明。preset 的选择见 [sessions.ts](../src/app/sessions.ts)。

## 外部依赖与运行环境

| 类别 | 依赖与用途 | 声明或证据 |
| --- | --- | --- |
| 直接生产依赖 | `proper-lockfile` 4.1.2 锁状态目录；`zod` 4.4.3 校验配置、输入和外部数据 | [package.json](../package.json)、[runtime.ts](../src/app/runtime.ts)、[policy.ts](../src/policy.ts) |
| 宿主 peer | `@deepseek-ai/cordis`、`schemastery`，以及 DSH agent、session/controller/persistence、workspace、scope、llm、tools、skill、attachment、subprocess、webserver、客户端 UI 等 | 完整名称与版本范围见 [package.json](../package.json)；例如 [deployment.ts](../src/deployment.ts) 运行时使用 schemastery，[delivery.ts](../src/app/delivery.ts) 使用 DSH LLM 值。大部分 DSH 类型 import 本身不是运行时值 import。 |
| Node 内置 | `node:sqlite` 提供同步 SQLite；另使用 fs、path、crypto、child_process 等 | [store.ts](../src/app/store.ts)、[commands.ts](../src/infra/commands.ts)、[ids.ts](../src/core/ids.ts)；Node 要求 `^22.19.0 || >=24.0.0`，见 [package.json](../package.json)。 |
| 外部命令与服务 | Git、已登录且对目标仓库有权限的 GitHub CLI；启用飞书时需要机器人身份的 `lark-cli` 和相应服务权限 | [deployment.ts](../src/deployment.ts)、[commands.ts](../src/infra/commands.ts)、[feishu.ts](../src/sources/feishu.ts)。这些是进程/网络依赖，不是 npm 直接依赖。 |
| 构建与测试 | TypeScript、tsdown、Vitest、jsdom、类型包；React 及 React DOM 在 devDependencies 中 | [package.json](../package.json)、[vitest.config.ts](../vitest.config.ts)。React/React DOM 虽只声明为开发依赖，客户端构建将它们外置，因此浏览器运行时仍需宿主提供，见 [tsdown.config.ts](../tsdown.config.ts)。 |

`@deepseek-ai/dsh-settings` 和 `@deepseek-ai/dsh-client-ui-plugin-manager` 在源码中用于类型，列于 devDependencies 而非 peerDependencies；不能据此宣称它们是本包声明的必需 peer。相反，peer 声明也不意味着每个包都被源码值 import，可能代表宿主兼容或装配要求。[package.json](../package.json) 同时声明 DSH 引擎 `^0.2.1-alpha.2` 和包管理器 `pnpm@11.7.0`。

## 构建产物边界

[tsdown.config.ts](../tsdown.config.ts) 将服务端入口及两个 watcher 编译为 Node ESM，`@deepseek-ai/*`、`zod`、`proper-lockfile` 不打包；客户端编译为供 `window.__ModuleLoader__` 加载的浏览器 CJS，React、`react/jsx-runtime` 和 React DOM 外置。包的导出、分发文件与 bundle patch 声明见 [package.json](../package.json)；部署时还依赖其中包含的 prompts、skills 和 locale 资源，相关读取见 [prompts.ts](../src/app/prompts.ts)、[skill-files.ts](../src/app/skill-files.ts) 和 [client/locales.ts](../src/client/locales.ts)。
