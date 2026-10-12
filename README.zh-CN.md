<div align="center">

# OpenBot

**真正由你的公司拥有的 AI 助手。** 它和 ChatGPT、Claude、Grok 形态相似，但有一个重要区别：它运行在你的基础设施中，且你可以修改它的任何部分。通过 AG-UI 接入任意代理技术栈。

每位数字同事都有独立的计算机：真实浏览器、独立登录态、独立文件，以及仅由你授予的工具。每次操作均先决策、后记录。

[**与工程师交流**](https://copilotkit.ai/talk-to-an-engineer?ref=openbot_readme) · [**快速开始**](#快速开始) · [**文档**](docs/README.md)

[![CI](https://github.com/CopilotKit/openbot/actions/workflows/ci.yml/badge.svg)](https://github.com/CopilotKit/openbot/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](./LICENSE)
![Alpha](https://img.shields.io/badge/status-alpha-orange.svg)

</div>

将任何以框架编写或手写的 AG-UI 代理接入后，它都会成为有独立频道的同事。你可观察其在独立屏幕上的工作；遇到不应独自处理的事情时接管，然后再交还。它能返回组件而不仅是文字，且整个系统可在你的机器上运行。

> **这是模板而非产品。** OpenBot 用于克隆和二次开发，并没有可注册的托管版本，也没有作为依赖发布的包。请用 `examples/` 下的示例租户包替换为自己的同事、频道和技能。

> **Alpha，仍在积极开发。** 预期会有粗糙边缘、缺陷和变化；欢迎提交 issue 与 PR。

> **可在你的机器运行。** 本文以笔记本为例。`.env.example` 中的 `OPENBOT_SINGLE_USER=true` 会将所有请求作为一个管理员处理；在他人可访问部署前，必须按[登录](#登录)配置身份验证。

## 它是什么

这是运行在你自有基础设施中的代理平台。Docker Compose 启动全部部件，数据保存在你的 PostgreSQL 中，模型由你选择。管理员提供的凭据静态加密且不会写入日志。

部署初始提供两个配置化的同事：处理日常工作的 **General Assistant**，以及回答公司问题的 **Knowledge**。配置端点后还会加入 **Risk Analyst**；`examples/fintech/catalog/` 中另有十余个按单一职责设计的示例。复制一个到 `examples/fintech/agents/`、在 `agents.yaml` 中编写，或通过 UI 的 `/bots` 创建，即可添加新同事。

任何 Bot 对计算机、文件、MCP 服务器或组件的操作都经过同一网关：网关决定、记录并执行（或拒绝）操作。这使代理不仅“能使用工具”，还“可以被安全地允许接近工具”。

## 基于 AG-UI

Bot 是任何讲 [AG-UI](https://github.com/ag-ui-protocol/ag-ui) 的端点；因此 OpenBot 不绑定某个框架。LangGraph、Mastra、CrewAI、Pydantic AI、Google ADK 或手写代理均以相同方式接入，治理能力附着于协议而非框架。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="assets/architecture-dark.svg">
  <img src="assets/architecture-light.svg" alt="用户与服务器交互，服务器经 AG-UI 将回合发送给 Bot。所有工具调用均回到网关；网关解析目标、依策略决策、写入审计记录，然后执行或说明拒绝规则。允许的浏览器与文件操作到达该 Bot 的独立计算机。决策写入 PostgreSQL，线程位于 CopilotKit Intelligence。">
</picture>

## 环境要求

- Docker：运行 PostgreSQL 与随附 Bot。
- [Bun](https://bun.sh) 1.3+：运行应用和 API 服务器。
- CopilotKit Intelligence 项目与许可证；可使用免费计划或自托管。
- 模型密钥：概念验证 Bot 使用 OpenAI；LangGraph Bot 支持 OpenAI、Anthropic 或 Google。

## 快速开始

> 使用 AI 助手进行配置时，可先将 [`prompt.txt`](prompt.txt) 粘贴给它。文件包含下列步骤及常见易错点。

1. 创建 `.env` 并安装依赖：

   ```sh
   cp .env.example .env
   bun install
   ```

2. 连接新的托管 Intelligence 项目：

   ```sh
   npx --yes copilotkit@latest login
   npx --yes copilotkit@latest project select
   bun scripts/setup-learning.ts
   ```

   该辅助程序会为所选项目创建或复用 `openbot` Learning 容器，并将运行时密钥与容器设置写入 `.env`。它会保留自定义容器分配，且不会覆盖已有 OpenBot 密钥。自托管时，将 `INTELLIGENCE_API_URL` 设为部署的 HTTPS API 根地址（本地可为 `http://localhost`），跳过两条 `npx` 命令运行该辅助程序即可。

3. 填入剩余必填值：

   - `OPENAI_API_KEY`

   除非自托管 Intelligence，请保留 `.env.example` 中的托管 URL。示例 `KEY_ENCRYPTION_KEY` 只适合本地使用；生产中请生成自己的值：

   ```sh
   openssl rand -base64 32
   ```

4. 启动：

   ```sh
   bash scripts/start.sh
   ```

5. 打开 <http://localhost:3010>。

`scripts/start.sh` 会启动 Docker 服务、执行迁移、在 3001 启动 API、启动例行任务 worker、在 3010 启动应用，并在打印后续步骤前检查健康接口。

`scripts/stop.sh` 会停止这些服务（包括不由 compose 管理的 Bot 计算机）；`--keep-computers` 会保留计算机。数据库、Bot 文件和浏览器配置文件均为卷，不会被删除。

### 在本机运行 Intelligence

若要在一台 Mac 上试用 Intelligence，请参阅[本地评估 Intelligence](https://docs.copilotkit.ai/intelligence/self-hosting-local)。免费 [Developer 计划](https://www.copilotkit.ai/pricing)可用；本地许可证有效期 30 天，可用 `npx copilotkit@latest local renew` 续期。该方案是面向 Docker Desktop/macOS 的预览版，不是生产部署。

`npx copilotkit@latest local connect` 会写入 `INTELLIGENCE_API_URL`、`INTELLIGENCE_GATEWAY_WS_URL` 和 `CPK_INTELLIGENCE_API_KEY`。OpenBot 从 `INTELLIGENCE_API_KEY` 读取项目密钥，因此在启动前复制 `cpk-...` 值：

```sh
INTELLIGENCE_API_KEY=cpk-...   # CPK_INTELLIGENCE_API_KEY 的值
```

## 部署

同一镜像包含应用、API、Bot 驱动的浏览器，以及可选 PostgreSQL；使用同一份 `.env`，无需 Kubernetes。

```sh
# 已发布镜像
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql \
  ghcr.io/copilotkit/openbot:latest

# 或构建当前源码树
docker build -t openbot .
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql openbot
```

此处应用也位于 3001，而源码克隆版使用 3010。`latest` 为最新发布版本；使用如 `:v0.0.9` 的标签可固定版本。若已有数据库，请关闭 `EMBEDDED_POSTGRES` 并设置 `DATABASE_URL`。详见[部署文档](docs/deployment.zh-CN.md)。

## 试用

- 打开 `/bot` 并询问：`Open news.ycombinator.com and tell me the top story.`
- 让 Bot 填写 <https://httpbin.org/forms/post>，然后检查 `/admin/audit`。
- 在 `/admin/boundaries` 添加拒绝规则或预设，再重试同一浏览器操作。
- 在 `/bots` 创建具有固定角色的 Bot，再与其创建频道。

## 主要界面

| 路由 | 用途 |
| --- | --- |
| `/` | 开始及浏览频道。 |
| `/channel/new` | 开始对话；在 To: 中选择两个以上 Bot 可发起群组。 |
| `/channel/:id` | 与同事对话、观看屏幕并查看执行内容。 |
| `/bot` | 直接与 Bot 对话；`?agent=<id>` 选择 Bot。 |
| `/bots`、`/bots/:id` | 管理可访问的 Bot 及其配置。 |
| `/skills` | 创建并启用个人技能。 |
| `/routines` | 查看或停止已设置的例行任务。 |
| `/settings` | 用户偏好、已连接账户、密码、通知、审批和记忆。 |
| `/admin/*` | 管理凭据、审批、计算机、边界、组件、插件、技能、学习、人员、身份提供商、企业设置与审计。 |

## 功能

- **每个 Bot 一台计算机**：每个 Bot 使用独立容器、`/workspace` 卷及浏览器配置；宿主支持时可设置 `COMPUTER_RUNTIME=runsc` 使用 gVisor。
- **Shell 而不只是浏览器**：Bot 可在工作区运行命令、安装依赖和处理文件；命令同样受网关和策略治理，并被审计。
- **网关是唯一入口**：它从服务器持有的快照解析目标、执行策略、写审计行，之后才调用计算机。
- **CEL 策略，默认拒绝**：规则可检查工具、意图、Bot/操作者、页面、元素、按键、命令、文件、MCP 与发起者字段。拒绝优先于允许；策略缺失或损坏均不放行。
- **可观察、可接管**：实时屏幕显示 Bot 正在看的内容，Activity 标签显示其运行、读取和保存的内容。遇到登录或 2FA 时可接管；人工控制期间 Bot 操作会被拒绝而非排队。
- **秘密不进入对话记录**：审计只记录请求了秘密及其长度，而不记录值。
- **自带代理与组件**：可注册任意 AG-UI 端点；React 组件可内置或在沙箱中创作并发布，按 Bot 授权数据函数。
- **受治理的 MCP**：目录内提供 Google Drive、Notion、Parallel Search 以及由 Composio 代理的应用；自定义服务器需通过 URL 校验，工具按读写影响分类。
- **技能是指令而非能力**：个人技能仅能附加到作者拥有的 Bot，部署技能由管理员拥有，均可在编辑器中以 `/` 调用。
- **登录、审计和加密凭据**：支持 Google、Microsoft、Okta、SAML 和 OIDC；审计页显示允许、拒绝和失败操作；凭据静态加密且 API 永不返回。
- **持久线程与例行任务**：线程和记忆由 CopilotKit Intelligence 持久化。例行任务以你的身份在指定频道运行，最短 15 分钟、最多 20 个启用任务，连续失败 10 次会自动停用。

## 接入自己的代理

任意 AG-UI 端点都可以成为 Bot。在 `/bots` 中创建时可配置名称、职务与角色描述、私有或公开可见性、可选 AG-UI 端点和只写授权头。

服务器在注册时及端点每次重定向时都用与浏览器导航相同的目标校验检查端点。没有自定义端点时，产品创建的同事使用已配置的 `MANAGED_AGENT_AG_UI_URL`；若其未配置则拒绝。

私有地址会被拒绝，除非列于 `AGENT_ENDPOINT_ALLOWED_HOSTS`：

```sh
AGENT_ENDPOINT_ALLOWED_HOSTS=agents.internal,10.0.0.42:9000
```

单独主机名允许该主机所有端口；带端口则固定端口。匹配完全精确，不支持通配符或后缀。该列表只影响代理端点，不能为代理额外授权，也不影响浏览行为。

租户包代理在 `TENANT_PACKAGE_DIR`（默认 `examples/fintech`）的 `agents.yaml` 或 `agents/` 目录中声明，类型为 `built-in`（系统提示词）或 `remote-ag-ui`（端点）。见[配置](docs/configuration.md)和[同事](docs/coworkers.md)。

## 配置

`.env.example` 是源模板。API 服务器拒绝在缺少下列任一项时启动：

- `DATABASE_URL`
- `KEY_ENCRYPTION_KEY`
- `INTELLIGENCE_API_URL`
- `INTELLIGENCE_GATEWAY_WS_URL`
- `INTELLIGENCE_API_KEY`

`COPILOTKIT_LICENSE_TOKEN` 可选：自托管且拥有许可证时会转发至运行时，托管 Intelligence 不需要它。

| 变量 | 用途 |
| --- | --- |
| `OPENBOT_SINGLE_USER` | 将所有请求作为一个管理员接入；未配置身份提供商时必需，示例默认开启。 |
| `OPENAI_BASE_URL` | 将 OpenAI 兼容调用指向网关或代理。 |
| `ANTHROPIC_BASE_URL`、`GOOGLE_GENERATIVE_AI_BASE_URL` | 对应两种 API 的等价设置。 |
| `COMPUTER_TOKEN`、`SUPERVISOR_TOKEN` | Bot 计算机与 supervisor 所需的秘密；`start.sh` 在开发环境提供固定回退值。 |
| `AGENT_TOOL_TOKEN` | Bot 回调已授权工具时提供的秘密；未设置时 Bot 无法调用工具。 |
| `COMPUTER_SUPERVISOR_URL` | 为每个 Bot 分配独立计算机而非共享计算机。 |
| `COMPUTER_RUNTIME` / `COMPUTER_SANDBOX` | 使用 gVisor 或 Chromium 沙箱。 |
| `EMBEDDED_POSTGRES` | 开启容器内数据库。 |
| `AGENT_COMPUTER_POLICY` | JSON 操作策略；格式错误会阻止服务器启动。 |
| `AGENT_COMPUTER_ALLOW_PRIVATE_HOSTS` | 允许 Bot 访问本机服务；仅本地可用，生产环境会拒绝。 |
| `AGENT_ENDPOINT_ALLOWED_HOSTS` | 可注册代理端点的私有地址白名单。 |
| `TENANT_PACKAGE_DIR` | 租户 YAML 所在目录，默认 `../examples/fintech`。 |
| `DEPLOYMENT_ID` | 多个部署共用 Intelligence 项目时的部署名称。 |

完整参考见[配置文档](docs/configuration.md)。

## 架构

| 服务 | 端口 | 职责 |
| --- | --- | --- |
| `app` | 3010 | React/Vite UI。 |
| `server` | 3001 | Hono API、CopilotKit 运行时、认证、策略、审计、插件、组件、同事和频道。 |
| `agent-computer` | 4100 | Chromium、`/workspace` 和浏览器配置。 |
| `agent-bot` | 4200 | 概念验证 AG-UI Bot。 |
| `agent-langgraph` | 4201 | LangGraph AG-UI Bot。 |
| `supervisor` | 主机 4500 / 容器 4300 | 创建和管理每个 Bot 的计算机。 |
| `worker` | 无 | 将到期例行任务交给服务器执行。 |
| PostgreSQL + pgvector | 5432 | 产品数据、策略、审计、凭据、授权、频道与组件元数据。 |
| CopilotKit Intelligence | 外部 | 持久线程与记忆。 |

服务器网关是 Bot 浏览器和文件工具调用的产品/API 路径。它解析目标、评估策略、写入审计，然后才调用 `agent-computer`。低层令牌保护接口必须保持私有，不能用于绕过网关。更多内容见[架构文档](docs/architecture.zh-CN.md)。

## 登录

`.env.example` 默认使用 `OPENBOT_SINGLE_USER=true`，即单管理员、无需登录。要让其他人访问，请删除该项并配置 Google、Microsoft 或 Okta 中至少一个；两者都没有时服务器会拒绝启动。多个提供商可同时配置。

```sh
BETTER_AUTH_URL=http://localhost:3001
BETTER_AUTH_SECRET=                          # openssl rand -base64 32
TRUSTED_ORIGINS=http://localhost:3010
INITIAL_ADMIN_EMAILS=you@example.com         # 逗号分隔
```

随后配置一个提供商，并在该提供商中注册相应回调 URI：

```sh
# Google — http://localhost:3001/api/auth/callback/google
GOOGLE_OAUTH_CLIENT_ID=
GOOGLE_OAUTH_CLIENT_SECRET=

# Microsoft — http://localhost:3001/api/auth/callback/microsoft
MICROSOFT_OAUTH_CLIENT_ID=
MICROSOFT_OAUTH_CLIENT_SECRET=
MICROSOFT_OAUTH_TENANT_ID=common

# Okta — http://localhost:3001/api/auth/callback/okta
OKTA_OAUTH_CLIENT_ID=
OKTA_OAUTH_CLIENT_SECRET=
OKTA_OAUTH_ISSUER=https://example.okta.com/oauth2/default
```

重启后，账户、会话和角色均保存在 PostgreSQL。公司自己的 SAML/OIDC 提供商可在运行时于“管理 → 身份提供商”注册，并按邮箱域名路由。OIDC 注册还需将发现文档中的每个主机加入 `TRUSTED_ORIGINS`。

- `INITIAL_ADMIN_EMAILS` 决定首位管理员；其中地址始终为管理员，不能在人员界面降级或移除，且每次登录都会重新读取。
- `MICROSOFT_OAUTH_TENANT_ID` 默认 `common`，会允许个人和工作账户；若 Entra 未返回 `email`，会依次回退到 `upn` 和 `preferred_username`。
- 半配置的提供商会在启动时被拒绝，例如只有 client id 没有 secret、过短的 `BETTER_AUTH_SECRET`，或没有凭据的 Okta issuer。
- 所有非 localhost 的部署均应置于 TLS 后方；普通 HTTP 页面不是安全上下文，登录 Cookie 也需要 `Secure`。

## 仅在本机使用时的注意事项

- `agent-computer` 控制保存真实登录态的浏览器；`docker-compose.yml` 将其绑定至回环地址，应保持如此。
- 通过 `/admin/credentials` 保存凭据；不要将凭据写进租户 YAML 或提交到仓库。
- `AGENT_COMPUTER_ALLOW_PRIVATE_HOSTS` 允许 Bot 访问本机服务，仅适合笔记本；生产环境设置它会被拒绝。
- 若部署需要访问内部网络代理，应改用 `AGENT_ENDPOINT_ALLOWED_HOSTS` 精确列出地址。

## 开发

```sh
bun run format:check
bun run lint
bun run typecheck
bun run test
bun run build
```

修改 Drizzle schema 后：

```sh
bun run --filter server db:generate
bun run --filter server db:migrate
```

完整本地栈使用 `bash scripts/start.sh`，停止用 `bash scripts/stop.sh`。仅需不带 Docker Bot 与计算机的应用/API 时使用 `bun run dev`。详情见[开发文档](docs/development.zh-CN.md)。

## 文档

- [产品主页](https://copilotkit.ai/openbot)
- [文档索引](docs/README.md)
- [架构（中文）](docs/architecture.zh-CN.md)
- [配置](docs/configuration.md)
- [开发（中文）](docs/development.zh-CN.md)
- [同事](docs/coworkers.md)
- [例行任务](docs/routines.md)
- [自动学习](docs/automatic-learning.md)
- [部署（中文）](docs/deployment.zh-CN.md)
- [Helm chart](charts/openbot/README.md)

## 贡献

- 开始较大改动前，请先创建 issue 或协调沟通。
- 保持改动聚焦；安装、配置、架构或用户行为变化时更新文档。
- 不要将秘密、服务账号 JSON、客户数据或本地转录内容放入仓库。
- 提交 PR 前运行[开发](#开发)中的检查。

## 许可证

[MIT](./LICENSE) © CopilotKit
