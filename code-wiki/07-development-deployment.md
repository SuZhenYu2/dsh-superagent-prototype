# 开发与部署

本文依据仓库配置与现有部署指南整理；命令默认在仓库根目录执行。项目是研究原型，并非生产就绪软件，使用前应审查并适配环境，参见 [README](../README.md)。完整交互式安装流程见 [SETUP.md](../SETUP.md)，手动安装的权威细节见 [部署指南](../docs/006-deployment.md)。

## 环境与依赖

| 项目 | 要求与用途 |
| --- | --- |
| Node.js | `^22.19.0 \|\| >=24.0.0`；不是任意 22.x 或 23.x。 |
| pnpm | [package.json](../package.json) 声明 `pnpm@11.7.0`；[pnpm-lock.yaml](../pnpm-lock.yaml) 固定依赖解析。 |
| DeepSeek Harness | `engines.dsh` 为 `^0.2.1-alpha.2`，并需配置可用的模型提供方。 |
| Git、tmux、Python | Git 2.32+、tmux、Python 3.8+；服务管理脚本使用 Python 和 tmux。 |
| GitHub CLI | `gh` 2.48+，需独立的代理账号及仓库写权限；目前目标是 github.com，不支持 GitHub Enterprise Server。 |
| 可选工具 | 使用 Feishu 需 `lark-cli` 和机器人应用；目标仓库使用 Git LFS 时需安装 `git-lfs`。 |

运行时直接依赖 `zod`、`proper-lockfile`；DSH/Cordis 包作为 peer dependencies 接入宿主。开发侧包含 TypeScript、tsdown、Vitest、React 和 jsdom，具体版本以 [package.json](../package.json) 为准。

## 本地开发与构建

```sh
pnpm install
pnpm typecheck
pnpm test
pnpm build
```

也可执行 `pnpm check`，它按类型检查、测试、构建顺序串行执行，前一步失败则停止。类型检查包含源码、测试及构建配置，选项见 [tsconfig.json](../tsconfig.json)；测试入口及覆盖范围见 [测试与运维](08-testing-operations.md)。

[tsdown.config.ts](../tsdown.config.ts) 将服务端入口 [index.ts](../src/index.ts)、[github.ts](../src/sources/github.ts)、[feishu.ts](../src/sources/feishu.ts) 构建为 `lib/` 下的 Node ESM，并生成类型声明；[客户端入口](../src/client/index.tsx) 单独输出浏览器侧 `lib/client.js`。DSH 相关包、React 等按配置作为外部依赖；`?raw` 文件由构建插件读取。包的导出和发布文件列表见 [package.json](../package.json)，bundle 插件及角色预设见 [cordis.patch.yml](../cordis.patch.yml)。修改源码后需重新构建，再重启实际运行的宿主以加载产物。

## 首次部署

1. 准备专用 GitHub 用户、仓库写权限和独立的 `gh` 配置；在该 wrapper 中登录并用 `gh-agent api user --jq .login` 确认身份。Feishu 为可选项，需创建并发布机器人应用、为其配置仅允许 bot 身份的独立 CLI。具体 wrapper 和授权步骤见 [账号配置](../docs/006-deployment.md#accounts)；不要将密钥放入聊天、仓库或命令历史。
2. 在已有源码目录执行 `pnpm install && pnpm build`。依照 [安装说明](../docs/006-deployment.md#install) 安装兼容的 DSH；文档示例使用 `@deepseek-ai/dsh@0.2.1-rc.3`。
3. **仅全新 profile** 执行以下命令：

   ```sh
   dsh superagent --from-default-profile web --dump-config > /dev/null
   dsh plugin --profile superagent add "link:$PWD"
   ```

   第一条会替换已有同名 profile 并删除其中的 `cordis.patch.yml`；升级或已有配置时不要执行。
4. 参考 [profile 示例](../examples/profile.cordis.patch.yml) 配置 `~/.dsh/profiles/superagent/cordis.patch.yml`，替换全部占位符：设置绝对路径 `stateDir`，并按需启用 Feishu。示例监听 `127.0.0.1:3082`；设置 `DSH_HOME` 可改变 DSH 根目录。已有配置先备份，再按实际情况合并，避免直接覆盖。
5. 执行 `scripts/dsh-superagent start` 和 `scripts/dsh-superagent status`，从日志中的地址进入面板。先配置 GitHub Source 的 `owner/name` 与代理 CLI，再配置 Policy 允许名单、Execution 目录及 prepare 命令；需要 Feishu 时配置 Source 和会话。初始化及默认值参见 [SETUP.md](../SETUP.md) 与 [Settings](../docs/006-deployment.md#settings)。
6. 以允许名单内的账号在目标 Issue 或新评论中提及代理，观察 Overview、Ticket、执行过程及原会话中的交付；默认轮询约一分钟。这是部署后的人工验收步骤，不代表本文已经执行。

Execution 可采用 [部署指南中的 worktree prepare 示例](../docs/006-deployment.md#settings)，必须先把示例克隆路径替换为本机路径。将个人 CLI 列入 blocked commands，但不要把它当作安全沙箱。

## 安全与隔离

Worker 和 router 预设以操作系统用户身份运行，具有 `danger-full-access`；凭据 wrapper、PATH shim 和 blocked commands 不能隔离文件系统或阻止按绝对路径执行程序。生产敏感环境应使用独立 OS 账号，且只放代理所需凭据。setup 会生成 `worker-bin` 与 `worker-gitconfig`，去除部分敏感环境变量并将 GitHub 凭据指向代理；具体限制见 [Setup checks](../docs/006-deployment.md#setup-checks) 和 [环境实现](../src/app/execution-environment.ts)。

管理 API 限本机 Host，并要求 `x-orchestrator-request: 1`；拥有面板级配置权限，不应暴露给不可信用户。请求示例、revision 并发控制和接口列表见 [Management API](../docs/006-deployment.md#management-api) 与 [api.ts](../src/app/api.ts)。

## 升级与回滚

等待执行器不在 turn 中，备份 `<stateDir>/orchestrator.sqlite`，再按 [升级步骤](../docs/006-deployment.md#upgrade) 更新代码、安装依赖、构建并重启，检查状态及标记为 Default updated 的提示词。数据库使用 WAL：只复制正在运行中的主数据库文件不一定构成一致备份；应先优雅停止并确认进程退出后再复制，或使用 SQLite 一致性备份机制，并验证备份可恢复。新版本迁移后的数据库不能由旧版本打开；回滚代码时需要升级前数据库备份。DSH 自身升级后也应重启实例，避免面板产物不匹配。
