# 部署

OpenBot 可作为一个镜像运行：其中包含应用、API、Bot 使用的 Chromium 浏览器以及可选 PostgreSQL。使用与本地相同的 `.env`，无需 Kubernetes；如需多副本、每 Bot 独立计算机和集群定时任务，则使用 Helm chart。

## 快速运行

```sh
# 运行发布镜像
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql \
  ghcr.io/copilotkit/openbot:latest

# 或从当前源码构建
docker build -t openbot .
docker run -p 3001:3001 --env-file .env \
  -e EMBEDDED_POSTGRES=on -v openbot-data:/var/lib/postgresql openbot
```

镜像中的应用和 API 都通过 3001 提供服务。`latest` 是最近发布版；使用如 `:v0.0.9` 的标签可固定版本。若使用已有 PostgreSQL，请不要设置 `EMBEDDED_POSTGRES`，并提供 `DATABASE_URL`。

此镜像适合单实例部署：每个实例共享镜像内的浏览器。要让每个 Bot 都有独立计算机、使用集群 CronJob 执行例行任务或扩展 API，请使用 [Helm chart](../charts/openbot/README.md)。

## 暂存附件清理

编辑器中拖入的文件会在消息发送前进入数据库；只有 `server/` 下的 `bun scripts/cull-staged-attachments.ts` 会回收从未发送的文件。Helm chart 每小时执行它；`docker-compose.yml` 和 `scripts/start.sh` 不会执行。

生产部署必须自行安排它，例如每小时运行一次：

```sh
docker run --rm --env-file .env --entrypoint sh openbot \
  -c "cd /app/server && bun scripts/cull-staged-attachments.ts"
```

它只需要 `DATABASE_URL`，可将保留小时数作为唯一可选参数，默认 24 小时。每个用户跨所有频道最多可持有 32 个未发送附件，每个最大 8 MiB（即最多 256 MiB）。第 33 个会被拒绝，且提示会承诺一天内清理未发送文件；若不运行清理任务，长期部署最终会让达到上限的用户无法再附加文件。

## 最低规格

在真实镜像、一个 Bot、arm64 上测得：

| | 实测 | 最低 | 建议 |
| --- | --- | --- | --- |
| 内存 | 空闲 409 MB；3 次页面加载后 498 MB；快照后 548 MB | **2 GB** | **4 GB** |
| vCPU | 空闲 3–6%，页面渲染时突发 | **1** | **2** |
| 磁盘 | 1.4 GB 镜像 | **4 GB** | 8 GB，并为 `/workspace` 预留空间 |

尽管单 Bot 实测约 550 MB，2 GB 仍是下限：每多一个并发页面约增加 100–200 MB，Playwright 建议每个并发浏览器约 1 GB。4 GB 能让少量并发 Bot 保持从容。

**不要配置共享内存。** Chromium 使用 `--disable-dev-shm-usage`，会写入 `/tmp` 而不是 `/dev/shm`，因此默认 64 MB 无关紧要。AWS Fargate 也不支持 `sharedMemorySize`；该标志避免 Chromium 在 Fargate 上崩溃。

## 必需配置

| 变量 | 说明 |
| --- | --- |
| `DATABASE_URL` | 带 `vector` 扩展的 PostgreSQL；使用 `EMBEDDED_POSTGRES=on` 时不需要。 |
| 身份提供商 | `GOOGLE_OAUTH_*`、`MICROSOFT_OAUTH_*` 或 `OKTA_OAUTH_*`，另需 `BETTER_AUTH_URL`、`BETTER_AUTH_SECRET` 与 `INITIAL_ADMIN_EMAILS`。 |
| `EMBEDDED_POSTGRES` | 设置为 `on` 时在容器内运行数据库；默认关闭。 |
| `KEY_ENCRYPTION_KEY` | base64 编码的 32 字节值，使用 `openssl rand -base64 32` 生成；生产环境会拒绝示例密钥。 |
| `INTELLIGENCE_API_URL`、`INTELLIGENCE_GATEWAY_WS_URL`、`INTELLIGENCE_API_KEY` | CopilotKit Intelligence；可使用免费计划或自托管。 |
| `COPILOTKIT_LICENSE_TOKEN` | 可选；托管 Intelligence 不提供它，仅在自托管 Intelligence 拥有许可证时设置。 |
| 模型密钥 | `OPENAI_API_KEY` 或所配置模型提供商对应的密钥。 |

未设置 `COMPUTER_TOKEN` 时会在启动时生成。所有需要它的进程都在容器内，因此不需要向外共享。

`MANAGED_AGENT_AG_UI_URL` 在此不是必需项。镜像不带 `agent-langgraph` 或 `agent-bot`；不设置它时，内置的 Risk Analyst 同事会被省略，而不会注册到不存在的主机。只有在容器确实能访问 Bot 时才设置该变量及 `MANAGED_AGENT_TOKEN`；如果 `.env` 仍有笔记本默认 `http://localhost:4201/ag-ui`，请清除它。

**必须认证。** 未配置身份提供商时，部署会拒绝启动，避免公开地址上的每个访客默认为管理员。配置 Google、Microsoft 或 Okta；或显式设置 `OPENBOT_SINGLE_USER=true` 表示你确实需要开放的单用户部署。

该标志不能用于公网地址。若 `OPENBOT_SINGLE_USER=true` 同时设置了公网可路由的 `OPENBOT_PUBLIC_URL`、`OPENBOT_APP_URL` 或 `TRUSTED_ORIGINS`，启动会拒绝并指出地址。家庭 LAN、Tailscale、VPN 或 `.local` 名称等私有地址可用，但启动时会警告该网络上的每个人都是该管理员。公网部署应在前方配置登录提供商。

**务必放在 TLS 后。** 这不仅是 Cookie 的要求：`http://<地址>` 不是安全上下文，会缺少 localhost 上存在的部分浏览器 API。应用当前不依赖这些 API，但登录 Cookie 仍需要 `Secure`。

## 迁移

开启 `EMBEDDED_POSTGRES=on` 时，迁移在启动时执行，无需额外操作。

使用外部数据库时，迁移是发布步骤而非启动步骤：两个副本同时启动会竞争，且迁移失败应中止发布，而非让半迁移数据库继续提供流量。

```sh
docker run --rm --env-file .env --entrypoint sh openbot \
  -c "cd /app/server && bun scripts/migrate.ts"
```

`--entrypoint sh` 至关重要。镜像入口是 s6 的 `/init`；若不覆盖入口，镜像名后的命令只是 CMD，s6 会在启动所有服务（包括连接尚未迁移数据库的 API 和 Chromium）后才执行迁移。覆盖入口可确保只运行迁移。

使用 `scripts/migrate.ts`，而不是 `drizzle-kit migrate`。CLI 是开发依赖，生产镜像通过 `bun install --production` 构建，且 CLI 还需要 esbuild 读取 TypeScript 配置。该脚本使用运行时依赖 `drizzle-orm` 中的 migrator，并维护相同 journal；镜像启动路径和 Helm 迁移 Job 均使用它。

## 副本

Bot 用于解析元素引用的页面快照保存在 Postgres，因此第二个副本可以处理第一个副本创建快照后的点击。平台需要时可运行多个副本。但 supervisor 不在该镜像内，所以每个副本仍共享镜像内的一个浏览器。

## 平台说明

**Google Cloud Run。** 内存至少设置 2 GB。多实例可行；每个实例有独立浏览器，因此 Bot 登录态保留在处理它的实例上。Cloud Run 在 gVisor 下运行容器，Chromium 对此敏感，正式使用前应测试导航。若需一次性数据库，`gcloud run compose up` 也可部署整个 compose 文件。

**AWS。** ECS Express Mode 可从 ECR 镜像配置集群、负载均衡、HTTPS 与自动扩缩容；这是 AWS 在 App Runner 不再接纳新客户后推荐的路径。若需要任务定义和细粒度 IAM，使用 ALB 后的普通 ECS/Fargate。无需也无法配置共享内存。

**Kubernetes。** 本文此前描述的是手动运行一个容器。集群是另一种形态，也是为每个 Bot 分配独立计算机、无需容器外部调度即可运行例行任务、并将 API 扩展到单副本以上的唯一方式。见 [Helm chart](../charts/openbot/README.md)，同一模板覆盖 EKS、GKE、AKS 和自托管集群。

**Azure Container Apps。** 提供托管 ingress、TLS 与自定义域。注意其 **240 秒请求超时**：实时屏幕使用长连接，预期会重连。基础层每实例并发 WebSocket 上限为 350。

**Railway、Render、Fly.io。** 三者均可直接运行此镜像，且可一键提供 PostgreSQL，是从零开始部署的最短路径。

## 已知成本与 CSP

浏览器镜像只包含 Chromium 浏览器族。全量 Dockerfile 与已发布 `agent-computer` Dockerfile 均基于 Ubuntu，并运行 Playwright 固定版本的 `install --with-deps chromium`；最终镜像不包含 Firefox 或 WebKit。Playwright 版本须与 `agent-computer/package.json` 一致，否则浏览器协议与可执行文件版本可能不匹配。镜像保留官方 Node 24.18.1 提供的基础命令行工具 `node`、`npm` 和 `npx`。

严格的 Content-Security-Policy 必须允许主题初始化的内联脚本（使用 hash 或 nonce）。仓库本身未发送 CSP，因此默认可用；若在代理层添加 CSP，必须明确允许此脚本，否则 `script-src` 会阻止它并导致应用启动前以错误主题渲染。`'sha256-'` 形式的脚本体 hash 可跨重建使用，无需每请求 nonce。
