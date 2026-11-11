# 管理面板、状态与 API

管理面板是嵌入 DeepSeek Harness Web 的 React 客户端，用于观察从 Source、Trigger、Eval 到 Ticket 的工作链路，并执行有限的管理操作。服务端将领域记录投影成面板视图；面板不是独立的领域数据源。入口和宿主注册分别见 [client/index.tsx](../src/client/index.tsx)、[app/http.ts](../src/app/http.ts)，类型契约见 [app/api.ts](../src/app/api.ts)。

## 界面结构

根组件 [app.tsx](../src/client/app.tsx) 提供全局搜索、术语表、打开主 Session 的入口以及六个顶层页签。搜索结果分为 Ticket、Thread 和 Trigger；被过滤且没有 Thread 的 Trigger 展示原因，不提供 Thread 链接。搜索请求以序号丢弃过期响应。

| 页签 | 主要内容与入口 |
| --- | --- |
| Overview | 需要人工处理的工作、系统问题、最近活动和执行负载；[overview.tsx](../src/client/overview.tsx)，服务端聚合见 [views/overview.ts](../src/app/views/overview.ts)。 |
| Source | 来源运行状态、设置问题、过滤和丢弃情况；[source.tsx](../src/client/source.tsx)、[views/sources.ts](../src/app/views/sources.ts)。 |
| Trigger | 按订阅、评估中、最近 24 小时、全部筛选 Thread，进入详情查看时间线；[trigger.tsx](../src/client/trigger.tsx)、[thread.tsx](../src/client/thread.tsx)。 |
| Eval | 查看 Decision、原因和规则，可按类别及 24 小时窗口筛选；[eval.tsx](../src/client/eval.tsx)。 |
| Execution | 查看执行负载、暂停处理、筛选 Ticket 和进入详情；[execution.tsx](../src/client/execution.tsx)、[ticket.tsx](../src/client/ticket.tsx)。 |
| Settings | Source、Eval、Execution、Prompts、Deployment 五组；[settings.tsx](../src/client/settings.tsx)，配置语义见 [配置与提示词](06-configuration-prompts.md)。 |

Thread 和 Ticket 的列表、详情和搜索由 [views/threads.ts](../src/app/views/threads.ts)、[views/tickets.ts](../src/app/views/tickets.ts) 组织。界面标识与领域阶段的映射见 [marks.ts](../src/client/marks.ts) 和 [core/stage.ts](../src/core/stage.ts)，不要将页面筛选值直接当作底层持久状态。

## 路由与客户端状态

[route.ts](../src/client/route.ts) 定义类型化路由、筛选值及设置分组；路由并非通过 URL 路径表达，而是将当前 route、返回栈和上次设置分组保存在 `sessionStorage` 的 `dsh-superagent/route`。解析时丢弃无效页签和筛选项；不可读时回到 Overview。[app.tsx](../src/client/app.tsx) 中普通跳转最多保留 20 项返回记录，筛选等可替换当前记录，切换顶层页签则清空返回栈。

[provider.tsx](../src/client/provider.tsx) 提供翻译、类型化 API 调用、导航、通知与全局 `state`。其 `useAction` 防止同一动作重入，执行后可刷新，失败时显示通知。全局状态包括 setup、运行中的 worker 数量和重启恢复计数；设置触发重启时每秒轮询，检测到服务先离线再恢复后增加恢复计数。重启检测设置 30 秒超时，但观察到离线时状态变化会重置计时器，所以总等待时间可能超过 30 秒。

[client/api.ts](../src/client/api.ts) 的 `usePoll` 默认每 5 秒请求一次，页面隐藏时延长至六倍，重新可见时立即请求。请求串行化，失败保留上次成功数据，响应相同则避免更新，卸载或依赖变更后丢弃旧响应。`ApiError.missing` 区分服务确认的 404 和服务不可达，避免把重启期间空响应误判为记录删除。

设置草稿由 [draft.tsx](../src/client/draft.tsx) 管理：`clean → editing → saving → awaitingRevision`，直到轮询读到提交后的 revision 才清除。草稿可存入 `sessionStorage`；服务端出现新版本时，以 base、本地、远端三方合并，对按 `id` 或 `chatId` 唯一标识的数组逐项合并，冲突保留本地值并报告路径。409 后重新加载再合并，校验错误显示字段路径。

## HTTP 访问边界

所有路由相对于 `/orchestrator/api/`，浏览器请求携带 `x-orchestrator-request: 1`。服务端要求 Web server 绑定 `127.0.0.1`，Host 必须是对应端口的 `127.0.0.1` 或 `localhost`，且 Origin 缺省或等于该 HTTP Host；不满足返回 403。这是本机管理接口的同源约束，不应将其描述为通用远程认证机制。实现见 [http.ts](../src/app/http.ts)，客户端编码路径参数并序列化 query 的实现见 [api.ts](../src/client/api.ts)。

请求体默认上限 128 KiB；提示词保存与预览使用 `4 × promptMaxChars + 4096` 字节上限，配置自身仍有字符限制。JSON 响应设置 `no-store` 和 `nosniff`。服务端等待运行时就绪后分派请求；正常响应为 200，格式、配置和用户错误通常为 400，缺失记录为 404，revision 冲突为 409，其他异常为 500。校验错误包含 `issues`，可定位到字段；结构化 setup 或用户错误由客户端本地化。

## 路由清单

以下均省略公共前缀；具体请求与响应类型以 [app/api.ts](../src/app/api.ts) 为准，参数校验和运行行为以 [http.ts](../src/app/http.ts) 为准。

| 方法与路径 | 用途与关键参数 |
| --- | --- |
| `GET state` | setup、账号摘要、masterId、paused、health 和执行负载；负载包含 `maxWorkers`。 |
| `GET overview`、`GET sources`、`GET settings` | 分别获取概览、来源状态、设置块及其 revision；完整视图由 [views/settings.ts](../src/app/views/settings.ts) 构建。 |
| `GET search?q=...` | 搜索相关 Trigger、Thread 和 Ticket；链接匹配依赖可用的来源链接缓存。 |
| `POST sources/:id/labels` | 重新读取来源显示名称，返回新增名称。 |
| `GET threads` | `filter=subscribed\|evaluating\|recent\|all`，支持 `offset`、`limit`，默认每页 20，最多 1000。 |
| `GET threads/:key` | 详情时间线，支持 `before`、`around`、`limit`，默认 50，最多 100。 |
| `POST threads/:key/unsubscribe` | 取消订阅并返回 `removed` 与上下文投递状态。 |
| `POST triggers/:id/requeue` | 将符合条件的 Trigger 重新送往评估，返回 `queued`。 |
| `GET decisions` | 支持逗号分隔的 `kind`、`window=24h`、`offset` 和 `limit`，默认 30，最多 200。 |
| `POST decisions/:id/dismiss` | 忽略指定 Decision。 |
| `GET tickets` | `state=open\|person\|working\|idle\|ended`，支持分页，默认 20，最多 1000。 |
| `GET tickets/:id` | Ticket 详情与执行环境；未完成部署配置时环境为空。 |
| `POST tickets/:id/retry`、`POST tickets/:id/stop` | 重试或停止 Ticket，是否允许由领域引擎决定。 |
| `POST executor/pause` | 请求体 `{ "paused": true }` 或 `false`；仅暂停处理而不是停止来源记录。 |
| `PUT settings/:block` | 请求体 `{ "value": ..., "revision": 版本号 }`；block 为 `policy`、`eval`、`executor` 或 `source:<id>`。 |
| `GET policy/history`、`POST policy-sync/:id` | 查看策略历史，或从来源同步其策略部分。 |
| `POST setup/check`、`GET models` | 再次检查部署并触发重启；读取模型目录。 |
| `GET prompts` | 可用 `revision` 查看指定版本，用 `offset` 翻页读取历史。 |
| `PUT prompts`、`POST prompts/preview` | 前者提交 `{ config, baseRevision }`；后者直接提交 config，返回组装后的角色提示词及 skills。 |

写入成功不一定表示主 Session 已收到通知：部分响应中的 `context_delivery` 区分 `injected` 和 `pending`。设置变更、提示词版本和通知机制分别见 [settings.ts](../src/app/settings.ts)、[prompts.ts](../src/app/prompts.ts)、[sessions.ts](../src/app/sessions.ts)。
