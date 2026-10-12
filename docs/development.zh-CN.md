# 开发

## 设置

安装 Docker、[Bun](https://bun.sh) 1.3+、`lsof`、`openssl` 和 `curl`。下方 Intelligence 配置还需要 `npx`（Node）；首次运行时 `scripts/start.sh` 会用 `openssl` 创建生成的秘密。

```sh
cp .env.example .env
bun install
```

`.env` 创建后配置 CopilotKit Intelligence：

```sh
npx --yes copilotkit@latest login
npx --yes copilotkit@latest project select
bun scripts/setup-learning.ts
```

对于新的托管配置，该辅助程序会为所选项目配置项目密钥，并创建或复用 `openbot` Learning 容器。它验证容器后才将 `INTELLIGENCE_API_KEY` 与 `CPK_INTELLIGENCE_LEARNING_CONTAINER_ID` 写入 `.env`，已有密钥与自定义目标不会被覆盖。

新的自托管配置中，请将 `INTELLIGENCE_API_URL` 设置为部署的 HTTPS API 根地址，仅运行 `bun scripts/setup-learning.ts`。先安装 Chrome 或 Edge：辅助程序会打开独立浏览器窗口，完成该部署的登录、选择可访问项目、创建或复用 `openbot`，再生成运行时密钥。本地开发可使用回环 HTTP 地址。已有仅密钥的配置可通过经认证的 Intelligence 管理界面创建容器，再在 OpenBot Admin 分配它。托管方案无需许可证步骤。最后添加 `OPENAI_API_KEY`。

启动完整栈：

```sh
bash scripts/start.sh
```

## 运行服务

使用 `bash scripts/start.sh` 启动完整本地栈：它启动 Docker 服务、执行迁移、启动 API 服务器、应用和例行任务 worker，并检查健康接口。

使用 `bash scripts/stop.sh` 停止应用、worker、API、Docker 服务以及每个 Bot 的计算机。后者由 supervisor 创建而非 compose，因此即使 `docker compose down` 仍可能存在。传入 `--keep-computers` 可保留这些已登录浏览器；两种方式都不会删除数据。

仅在需要应用和 API、而不启动 Docker Bot 与计算机时使用 `bun run dev`。

| 服务 | 端口 |
| --- | --- |
| `app` | 3010 |
| `server` | 3001 |
| `agent-computer` | 4100 |
| `agent-bot` | 4200 |
| `agent-langgraph` | 4201 |
| `supervisor` | 主机 4500 / 容器 4300 |
| PostgreSQL | 5432 |

每次运行，`start.sh` 都会将已选 Docker 服务交给 `docker compose up -d`：配置变更的服务会重建，未变化的服务保持不动。已正常响应为 OpenBot 的 API 或应用进程不会被动；若 API 拒绝 worker 的秘密或缺少 routines 路由，脚本会重启它；端口被其他进程占用时会报告。

**此处没有清理暂存附件。** 投入编辑器的文件会在发送前储存；只有从 `server/` 运行的 `bun scripts/cull-staged-attachments.ts` 可回收从未发送的文件。Helm chart 每小时运行它，而 `docker-compose.yml` 和 `start.sh` 均不会。它只需要 `DATABASE_URL`，可选参数为保留小时数，默认为 24。通常本地小数据量无需处理；但每人跨所有频道最多可保留 32 个未发送文件，下一次拒绝会承诺其一天内清理。若从不运行清理，长期本地部署也可能让用户无法附加任何文件。编辑器中移除文件会立即删除，因此清理只处理遗弃草稿。生产配置详见[部署文档](deployment.zh-CN.md)。

## 迁移

修改 Drizzle schema 后：

```sh
bun run --filter server db:generate
bun run --filter server db:migrate
```

分享前审查生成的迁移文件；`start.sh` 启动栈时会应用已有迁移。

**不要手工编辑生成的迁移。** 编辑后文件会与生成器输出不一致。若生成 SQL 不适用于已有数据（例如已有行的表执行 `ADD COLUMN ... NOT NULL`），请拆分步骤：先生成可空列，执行数据步骤，再生成约束。

**收紧已有列的约束应在后续发布进行**，而非添加列的同一发布。滚动部署会在迁移后让旧、新副本同时服务；旧副本可能写入没有新列的行，在 `NOT NULL` 下会失败。先发布可空列，等待整组副本完成替换，再收紧。`accounts` 的 `issuer` 是实例：该列可空，且没有迁移收紧它。

**数据步骤应是独立迁移**，使用为此提供的标志：

```sh
bun run --filter server db:generate -- --custom --name=backfill_something
```

生成器只比较 schema，因此“provider 为 Google 的行填入 Google issuer”这类规则无法从 schema 得出。`--custom` 会创建登记到 journal 的空文件；它是唯一应由开发者手写的迁移。

CI 对此执行两项检查：用 `drizzle-kit check` 检查迁移冲突和间隙，并运行“生成后若工作区有改动则失败”的探针，拒绝没有对应迁移的 schema 改动。

若 `drizzle-kit migrate` 挂起后以非零退出且没有错误，通常是 journal 引用了不存在的迁移文件。rebase 可能造成这种情况：`meta/_journal.json` 已提交，恢复它可能恢复被重命名迁移的条目。`drizzle-kit check` 在此状态仍可能报告正常，因为它比较 schema 而不检查 journal 和目录是否一致。请比较 `meta/_journal.json` 与 `ls server/drizzle/*.sql`。

## 质量检查

提交 PR 前运行：

```sh
bun run format:check
bun run lint
bun run typecheck
bun run test
bun run build
```

集成测试需要带 pgvector 的 PostgreSQL。运行前，将 `TEST_DATABASE_URL` 指向专用数据库，例如：

```sh
postgres://openbot:openbot@localhost:5432/openbot_test
```

测试拒绝使用应用的 `openbot` 数据库。`DATABASE_URL` 仍是应用设置，但数据库客户端连接后会将它从进程环境移除以保留 Windows Bun 连接修复；因此测试使用不可变的 `TEST_DATABASE_URL` 作为夹具地址。若在正在使用的部署上执行，测试 Bot 会进入该部署的审计轨迹和活动报告；请创建独立数据库并在运行集成测试前迁移它。

CI 使用 `bun run test:ci`，除普通测试外还校验预期测试数量。

`bun run test:smoke` 独立运行，需要已启动部署及其会话：

```sh
bash scripts/start.sh
export OPENBOT_SMOKE_COOKIE='better-auth.session_token=...'
bun run test:smoke
```

它通过 HTTP 驱动运行中栈的一条用户旅程，覆盖其他测试无法到达的连接：server 到 supervisor 到 computer、浏览器执行前的网关决策，以及审计行落库。用 `OPENBOT_API_URL` 可指向其他部署。没有部署时，它会被 `bun run test` 跳过；单独运行时会说明需要启动什么。

会话不是可选便利项。该旅程验证的每条路由都受 `requireUser` 保护；缺少 Cookie 时，对计算机执行操作的三个测试返回 401，运行会将正常部署误报为损坏。请从目标部署中已登录浏览器的 DevTools（Application → Cookies）取得 Cookie。它具有该用户的权限范围，只应置于本次运行环境，绝不可放入文件或 PR 评论；未设置时测试会在第一个测试前停止并提示变量名。

`bun run test:live-screen` 也独立，原因类似：不需要部署，只需要本目录依赖：

```sh
cd agent-computer && bun install
cd .. && bun run test:live-screen
```

它以真实 Chromium 驱动真实计算机进程的实时屏幕：包括浏览器尚未启动时 socket 关闭、第二个连接从第一个夺取屏幕、滚轮拒绝非拥有 socket 的输入，以及按请求或空闲清理关闭浏览器。这些测试需要在模块顶层导入 Playwright 的 `agent-computer/src/index.ts`；而 `playwright` 仅声明于 `agent-computer/package.json`，根目录 `bun install` 不会安装它。因此未设置 `OPENBOT_LIVE_SCREEN=1` 时，文件会在导入前跳过，从而让 `bun run test` 与 CI 在未安装该依赖的环境继续工作。

## 贡献清单

- 保持改动聚焦。
- 不要将凭据、服务账户 JSON、客户数据和转录内容纳入版本控制。
- 将敏感行为放在服务器，而不仅是浏览器。
- 行为变化时更新[配置](configuration.md)、[架构](architecture.zh-CN.md)或根目录 [README](../README.zh-CN.md)。
- 运行上述质量检查，并在 PR 中附上结果。
