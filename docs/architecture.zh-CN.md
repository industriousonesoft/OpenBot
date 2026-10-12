# 架构

OpenBot 由 React 应用、Hono API 服务器、PostgreSQL、CopilotKit Intelligence、AG-UI Bot 端点以及受治理的浏览器计算机组成。

<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../assets/architecture-dark.svg">
  <img src="../assets/architecture-light.svg" alt="一次对话从应用到服务器，再经 AG-UI 到 Bot。Bot 的工具调用回到网关；网关解析目标、依配置策略决策、记录审计行，然后执行或拒绝。允许操作到达由 supervisor 创建、各自拥有 Chromium、登录态与工作区的 Bot 计算机。决策在 PostgreSQL，线程与记忆在 CopilotKit Intelligence。">
</picture>

修改图中内容后，以 `bun run diagram` 重新生成。

## 服务与端口

| 组件 | 端口 | 职责 |
| --- | --- | --- |
| `app` | 3010 | React/Vite 界面，提供频道、Bot 对话、实时屏幕、设置和管理页。 |
| `server` | 3001 | API、CopilotKit runtime、认证、角色、租户包、同事、频道、策略、审计、凭据、插件、组件和连接器。 |
| `agent-computer` | 4100 | Chromium、`/workspace`、浏览器配置、截图、快照和文件工具。 |
| `agent-bot` | 4200 | 概念验证 AG-UI Bot。 |
| `agent-langgraph` | 4201 | LangGraph AG-UI Bot。 |
| `agent-harness` | 4202 | 设置时选择的 Bot 框架 harness；位于 `harness` compose profile 后。 |
| `supervisor` | 主机 4500 / 容器 4300 | 创建、停止、重置和列出每个 Bot 的计算机容器。 |
| PostgreSQL + pgvector | 5432 | 产品数据、审计行、凭据、策略、授权、频道和组件。 |
| CopilotKit Intelligence | 外部 | 持久线程、记忆和实时网关。 |

`scripts/start.sh` 通过 Docker Compose 启动 PostgreSQL、`agent-computer`、`agent-langgraph`、`agent-bot`（`BOT_PROVIDER=anthropic` 时跳过）及 supervisor（除非 `OPENBOT_ONE_COMPUTER_EACH=false`），并执行 `migrate`。随后它在宿主机启动 `server`、例行任务 worker（`bun worker/src/index.ts`）和 `app`。compose 文件还定义了可选 SPIRE 服务，但脚本不会启动它们。

## 运行时流程

1. 应用打开频道或直连 Bot 会话。
2. 服务器解析已登录的操作者和选中的同事。
3. CopilotKit runtime 将本轮发送至配置的 AG-UI 端点。
4. 界面注册该 Bot 可用的前端工具：浏览器工具、MCP 工具和已授权组件。
5. 会产生副作用的浏览器、文件和 MCP 调用返回服务器进行授权与审计。
6. 服务器将结果流式返回应用和 Intelligence 线程。

## 浏览器操作治理

计算机本身不决定策略，服务器网关才是操作边界：

1. 从服务器保存的快照或请求对象解析目标；
2. 评估当前操作策略；
3. 写入决策的审计行；
4. 仅在决策允许转发时调用计算机；
5. 若已转发的操作失败，再写一条审计行。

策略规则可检查：

- `tool.name`、`intent`、`bot.id`、`actor.id`
- `page.url`、`page.host`
- `element.ref`、`element.role`、`element.name`、`element.type`
- `key`、`command`
- `file.path`、`file.name`、`file.extension`
- `mcp.server`、`mcp.tool`、`mcp.effect`
- `initiator.kind`、`initiator.id`：运行由什么触发，与它携带谁的权限不同。可为 `person`、`deployment`、`routine` 或 `handoff`；定时运行时 `actor.id` 是例行任务所有者，因此只有该字段可区分无人值守运行与人工输入。

规则使用 CEL 表达式，并提供不区分大小写的 `contains()` 和 `matches()`。拒绝规则先于允许规则计算，策略引擎默认拒绝：缺失或空策略不允许任何操作；损坏的 deny 规则会拒绝，损坏的 allow 规则不会授权。随附启动默认值明确为 `deny: []` 和 `allow: ["true"]`，除非被 `AGENT_COMPUTER_POLICY` 或管理员保存的策略替代。配置的策略 JSON 格式错误会使服务器无法启动。

## 计算机

`agent-computer` 要求 `COMPUTER_TOKEN`，只有 `/health` 不需该令牌；Docker Compose 将其绑定到 `127.0.0.1:4100`。

compose 将它和 PostgreSQL 放在不同网络中。Bot 有 shell，而 shell 能到达容器可到达的一切：因此数据库仅在 `data` 网络，与 `migrate` 同网；其他服务在 `default` 网络。若将 API 服务器运行在 Compose 内，它会加入两个网络，是两者唯一交汇处。

设置 `COMPUTER_SUPERVISOR_URL` 时，每个 Bot 获得独立的计算机容器、工作区卷与浏览器配置；不设置时，所有 Bot 共享 `AGENT_COMPUTER_URL`。

每台计算机还会过滤自身出站连接。浏览器和 shell 命令经由过滤代理，依 Bot 网络策略（`allow_all`、`defaults_plus_allowlist`、`allowlist_only` 或 `deny_all`）放行。其所有者的 Cloud 网络访问关闭时，Bot 使用 `deny_all`。服务器在执行前和计算机唤醒时推送策略；收到策略前不得出站。`EGRESS_POLICY_REQUIRED=0` 仅用于没有 API 服务器的独立计算机。所有模式均拒绝云元数据地址。

计算机镜像仅暴露 token 保护的服务 API。不要在公共网络暴露其端口，也不要直接调用它来绕过服务器网关。

## 身份、租户与授权

认证由 Better Auth 处理。`OPENBOT_SINGLE_USER=true` 为本地单管理员模式；生产部署应使用 Google、Microsoft、Okta 或运行时注册的 SAML/OIDC。服务器将人员、角色、会话和租户数据保存在 PostgreSQL。

租户包提供同事、提示词、技能和目录内容。所有者可创建私有 Bot；管理员可管理部署范围资源。组件、MCP 服务器、凭据和技能均显式按 Bot 授权；Bot 获得的是已授权能力，而非对部署的隐式完全访问。

## 审计、凭据与秘密

每个允许、拒绝及失败的受治理操作都会留下审计记录；拒绝记录会指出命中的规则。凭据通过管理界面写入，使用 `KEY_ENCRYPTION_KEY` 静态加密，API 不会返回明文。秘密值也会从审计和对话转录中脱敏。

当 Bot 请求人工接管时，系统记录 `computer.help_requested`、`computer.control_taken` 和 `computer.control_released`。人工驾驶期间 Bot 的计算机操作会被拒绝，而不会排队等待。

## 组件、MCP 与技能

内置 React 组件位于 `app/src/components/gallery/`；管理员可在 `/admin/playground` 编写沙箱组件并发布。每次组件调用前，服务器检查其是否存在、已发布且未对该 Bot 隐藏；数据函数逐组件授权。

MCP 连接器由服务器控制并按工具影响治理。目录中的工具可根据声明的读写类别处理；未知工具与自定义服务器工具默认作为写操作。自定义服务器须通过 URL 校验。技能是可授予 Bot 的指令文本而不是隐含能力：个人技能归作者所有，部署技能归管理员所有。

## 例行任务与持久化

CopilotKit Intelligence 保存线程和记忆；`DEPLOYMENT_ID` 用于让共享同一 Intelligence 项目的部署区分自己的线程。例行任务由独立 worker 触发，worker 将每次到期运行提交到服务器，以任务所有者身份在原频道执行。为避免意外支出，任务最低间隔为 15 分钟、最多 20 个启用任务，连续 10 次失败后自动停用。

## 安全边界摘要

- 网关是受治理动作的唯一产品入口，先审计再执行。
- 每 Bot 独立浏览器与工作区可限制横向影响。
- 默认拒绝策略、精确端点白名单和出站网络策略共同限制可达目标。
- 令牌保护的内部服务必须保留在回环或私有网络。
- 生产环境必须使用 TLS 与身份提供商；不要将单用户模式放在公开地址上。
