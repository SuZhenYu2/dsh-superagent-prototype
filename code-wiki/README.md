# dsh-superagent Code Wiki

这套文档面向接手、维护和扩展本仓库的开发者，以当前源码、配置和测试为依据，说明系统做什么、如何协作以及如何运行。分析基于 2026-11-10 工作区快照，Git 基准提交为 `8c32a9dc5d72`；后续代码变更应同步更新本 Wiki。原始设计文档和部署指南仍是重要上下文，具体实现与设计叙述不一致时，本文以源码为准并注明边界。

## 项目速览

[dsh-superagent](../README.md) 是面向单个 GitHub 仓库的主动式编码 Agent，作为 DeepSeek Harness（DSH）bundle 运行，支持 GitHub 和可选的飞书来源。它监听事件，在版本化策略与主 Session 评估约束下接收工作，为 Ticket 准备独立的 worker Session 和目录，再将结果发送回原渠道。React 管理面板提供观察、配置和人工干预能力。

**项目是研究原型，不是生产就绪软件，也不是官方 DeepSeek 产品。** worker 和主 Session 以运行账户的操作系统权限执行；命令包装器与策略不是安全沙箱，部署前应阅读[安全与隔离](07-development-deployment.md#安全与隔离)。

```text
GitHub / 飞书 → Source → Trigger / Thread → Policy gate + Eval
                                               ↓ 接受
                                      Decision + Mandate
                                               ↓
                                      Ticket / Assignment
                                               ↓
                                        DSH worker Session
                                               ↓
                         原渠道 ← Channel ← Outbox / Effects

管理面板 → 本机 HTTP API → 视图 / 设置 / Engine
```

来源事件也可能因订阅直接作为已有 Ticket 的后续上下文，不必每次都创建新工单。状态和待执行副作用保存在本机 SQLite 中；外部服务调用并不保证严格恰好一次。完整流程与异常分支见[系统架构](01-architecture.md)和[领域模型与 Engine](02-domain-engine.md)。

## 文档目录

| 文档 | 适合解决的问题 |
| --- | --- |
| [01 系统架构](01-architecture.md) | 系统如何分层、启动、装配并跨越事务与信任边界？ |
| [02 领域模型与 Engine](02-domain-engine.md) | Trigger、Thread、Eval、Mandate、Ticket 和 Assignment 如何关联？核心方法及状态转换是什么？ |
| [03 数据源](03-sources.md) | GitHub、飞书如何监听、去重、补收和回复？ |
| [04 执行与存储](04-execution-storage.md) | worker 如何运行？SQLite、持久化队列、重试和重启恢复如何工作？ |
| [05 管理面板与 API](05-panel-api.md) | 面板页面、客户端状态、HTTP 路由与访问边界是什么？ |
| [06 配置与提示词](06-configuration-prompts.md) | 机器配置、来源策略、执行设置、提示词和版本如何管理？ |
| [07 开发与部署](07-development-deployment.md) | 环境要求、构建、首次部署、安全、升级和回滚步骤是什么？ |
| [08 测试与运维](08-testing-operations.md) | 测试入口、服务控制、健康状态和常见故障如何排查？ |
| [09 源码地图](09-code-map.md) | `src/` 下各文件负责什么？从哪些导出和文件开始阅读？ |
| [10 依赖关系](10-dependencies.md) | 内部依赖方向、DSH 宿主服务、外部程序和构建产物有哪些？ |

## 推荐阅读路径

- **新成员**：先读[系统架构](01-architecture.md)和[领域模型](02-domain-engine.md)，再按需求查看[源码地图](09-code-map.md)。
- **接入或排障**：先读[数据源](03-sources.md)、[执行与存储](04-execution-storage.md)，再读[测试与运维](08-testing-operations.md)。
- **修改面板或配置**：先读[管理面板与 API](05-panel-api.md)，再读[配置与提示词](06-configuration-prompts.md)。
- **安装部署**：先读[开发与部署](07-development-deployment.md)，准备账号、独立 DSH profile 和模型服务后，再按[原部署指南](../docs/006-deployment.md)执行。
- **扩展来源**：先读[依赖关系](10-dependencies.md)和[数据源](03-sources.md)，再参考仓库的 [Source 扩展教程](../docs/008-cookbook.md)。

## 代码入口与目录

| 路径 | 定位 |
| --- | --- |
| [src/index.ts](../src/index.ts) | Cordis 服务端插件入口；创建 `Runtime`。 |
| [src/core/](../src/core) | 领域模型、端口、事务性 Engine 和状态规则。 |
| [src/app/](../src/app) | 组合根、应用服务、SQLite、配置、HTTP、Session 和持久化副作用。 |
| [src/sources/](../src/sources) | 来源契约、GitHub 与飞书接入。 |
| [src/executor/](../src/executor) | DSH worker 适配、后台任务和执行生命周期。 |
| [src/infra/](../src/infra) | 外部命令、执行环境、路径和健康状态。 |
| [src/client/](../src/client) | 嵌入 DSH 的 React 管理面板。 |
| [prompts/](../prompts)、[skills/](../skills)、[locale/](../locale) | 角色指令、来源技能及本地化资源。 |
| [tests/](../tests)、[scripts/](../scripts)、[examples/](../examples) | 测试、服务控制和 profile 示例。 |

## 开发速查

需要 Node.js `^22.19.0 || >=24.0.0`、pnpm `11.7.0`，并根据实际部署准备兼容 DSH、Git、GitHub CLI、Python 和 tmux。可选飞书能力还依赖机器人应用及 `lark-cli`。版本与部署前提见 [package.json](../package.json) 和[开发与部署](07-development-deployment.md)。

```sh
pnpm install
pnpm check
```

`pnpm check` 依次运行类型检查、Vitest 测试和 tsdown 构建；它不等同于真实 GitHub、飞书或模型端到端验收。本项目不是通过独立 `npm start` 启动的 Web 站点：实际运行需要先将 bundle 安装到 DSH profile，再通过 [scripts/dsh-superagent](../scripts/dsh-superagent) 管理服务。**创建全新 profile 的命令会覆盖已有同名配置**，执行前务必查看[首次部署](07-development-deployment.md#首次部署)。

## 范围与证据

- 本 Wiki 覆盖核心源码、配置、运行脚本、测试入口及原有设计文档；源码地图覆盖当前 `src/` 下的全部文件。各章节使用相对链接指向仓库证据，便于在本地或仓库浏览器中导航。
- 具体行为以实现和测试为准。原设计文档可能有更概括或已过时的表述，例如重启后稳定状态的 Assignment 可以保留，详见[重试与重启恢复](04-execution-storage.md#重试与重启恢复)。
- 文档中的命令和验收流程是操作指南，不隐含当前机器已完成账号配置、真实服务部署或端到端验证。依赖安装、模型服务与外部权限会影响实际结果。
- 仓库原始资料入口：[README](../README.md)、[SETUP](../SETUP.md)、[编号文档](../docs)、[包配置](../package.json) 和 [测试目录](../tests)。
