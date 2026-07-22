# TECH_STACK.md — Plane 技术栈与依赖分析

## 分析快照

- 分支：`preview`
- HEAD：`7cef741c2`（`feat(api): add lite list endpoints for projects, members, cycles, and modules (#9410)`）
- 工作区状态：clean（任务开始前无未提交修改；本任务仅新增 `./docs-analysis/`）
- 子模块状态：**无**（仓库不存在 `.gitmodules`，`git submodule status` 为空）
- 分析范围：基于工作区实际源码、构建/运行配置、依赖清单、CI 配置的**静态只读**分析。未运行构建/测试/迁移。
- 未覆盖范围：私有 EE/Cloud 仓库（`plane-cloud`/`plane-ee`）内容；依赖锁文件逐行分析；动态运行行为；第三方依赖内部实现。

## 证据分类

- **Evidence**：版本号、配置项、import、初始化、调用点均给出 `路径:行号` 或配置键证据。
- **Inference**：由多处证据推断的用途或“是否进入运行时”的判断。
- **Unknown**：静态分析无法可靠确认（如生产是否实际启用某可选依赖）。

## 核心结论

[Evidence] Plane 是一个 **pnpm + Turborepo 单体仓库（monorepo）**，包含 6 个应用（`apps/{admin,api,live,proxy,space,web}`）与 15 个共享包（`packages/*`）。根 `package.json:3` 版本 `1.3.1`；`apps/api/pyproject.toml:3` 后端版本为 `0.24.0`（**前后端版本号不一致**）。
证据：`package.json:1-43`、`pnpm-workspace.yaml:1-6`、`apps/api/pyproject.toml:3`

[Evidence] 仓库整体处于**多次并行迁移中**：Next.js → React Router 7（三前端应用均含 `compat/next/` 兼容垫片）；UI 库 `@plane/ui` → `@plane/propel`；前端服务层本地 → 共享 `@plane/services`；MobX store 本地 → `@plane/shared-state`；后端 `/api/`（旧）与 `/api/v1/`（新）双 API 并存。
证据：`apps/web/vite.config.ts:28-31`、`packages/ui/package.json:37`、`apps/web/app/routes/extended.tsx`（空）、`plane/urls.py:17-24`

---

## 1. 编程语言与用途

| 语言 | 用途 | 证据 |
| -- | -- | -- |
| **TypeScript / JavaScript** | 前端（web/space/admin）、实时协同服务（live）、所有共享包 | `apps/web/package.json`、`apps/live/package.json`、`packages/*` |
| **Python** | 后端 REST API（Django）、Celery worker/beat | `apps/api/requirements/base.txt:4`（Django）、`apps/api/plane/celery.py:38` |
| **Shell / Caddyfile / YAML** | 部署、容器编排、入口脚本 | `docker-compose.yml`、`apps/proxy/Caddyfile.ce`、`deployments/**` |

[Inference] 后端语言版本：`apps/api/Dockerfile.api`（`python:3.12.10-alpine`，由 CI/部署 agent 确认）、`apps/api/Dockerfile.dev`（`python:3.12.5-alpine`）。

## 2. 前端框架与渲染模式

- **React 18.3.1**（catalog 固定版本 `pnpm-workspace.yaml:145`）。
- **React Router 7.15**（框架模式 `@react-router/dev/node/serve`，`pnpm-workspace.yaml:36-38`）。
- 渲染模式差异（关键）：
  - [Evidence] `apps/web`：`ssr: false`（纯客户端 SPA，nginx 托管静态 `build/client`）。`apps/web/react-router.config.ts:6`
  - [Evidence] `apps/admin`：`ssr: false`。`apps/admin/react-router.config.ts:13`
  - [Evidence] `apps/space`：`ssr: true`（`react-router-serve` Node 运行）。`apps/space/react-router.config.ts:11`
- **Vite 8**（`pnpm-workspace.yaml:182`）作为构建工具。
- **状态管理：MobX 6.12**（`mobx`/`mobx-react`/`mobx-utils`，`pnpm-workspace.yaml:134-136`），配合 **SWR 2.2.4**（`pnpm-workspace.yaml:170`）做数据获取缓存。
- **样式：Tailwind CSS 4.1.17**（CSS-first，`pnpm-workspace.yaml:172`），无 `tailwind.config.js`。
- **富文本编辑器：TipTap 2.22 + Yjs + Hocuspocus**（`pnpm-workspace.yaml:53-74`、`187-190`）。

## 3. 后端框架

- **Django 4.2.30** + **Django REST Framework 3.15.2**（`apps/api/requirements/base.txt:4-5`）。
- **数据库驱动 psycopg 3.3.0**（PostgreSQL，`base.txt:8-11`）。
- **Celery 5.4.0** + `django_celery_beat 2.6.0` + `django-celery-results 2.5.1`（`base.txt:18-20`）。
- **ASGI/WSGI**：`uvicorn 0.29.0`（`base.txt:34`）+ `channels 4.1.0`（`base.txt:36`）。但 ASGI 实际仅 HTTP（无 websocket consumer，见 BACKEND/运行时文档）。
- **缓存/会话：redis 5.0.4 + django-redis 5.4.0**（`base.txt:13-14`）；broker 为 **RabbitMQ**（`plane/settings/common.py:318-330`）。

## 4. 实时协同服务（apps/live）

- **Node + Express 4.22**（`pnpm-workspace.yaml:109`）+ `express-ws`（WebSocket）。
- **Hocuspocus 2.15.2**（`@hocuspocus/server` 及 database/redis/logger 扩展，`pnpm-workspace.yaml:25-30`）协同服务器。
- **ioredis 5.7.0**（多实例 pub/sub，`pnpm-workspace.yaml:123`）。
- **Effect 3.20**（`@effect/platform`，仅用于 PDF 导出管线，`pnpm-workspace.yaml:16-17`）。
- **@react-pdf/renderer + sharp**（PDF 渲染 + 图片归一化）。
- 构建：**tsdown**（`pnpm-workspace.yaml:175`）→ `dist/start.mjs`。

## 5. 反向代理（apps/proxy）

- **Caddy 2.11.3**（`apps/proxy/Caddyfile.ce`、`Dockerfile.ce`，使用 `xcovy` + DNS/L4/otel 插件）。
- 负责 `/spaces/*`、`/god-mode/*`、`/live/*`、`/api/*`、`/auth/*`、`/<bucket>/*`、`/*` 的路由分发与 ACME 证书。

## 6. 桌面/移动/CLI

[Evidence] 当前仓库**未发现**桌面端（Electron/Tauri）、移动端（iOS/Android 原生）或独立 CLI 应用。`apps/` 下仅有 web/admin/space/api/live/proxy。`deployments/cli/` 是“Docker Compose 安装器”（`deployments/cli/community/install.sh`），不是产品 CLI。

## 7. 数据库 / ORM / 数据访问

- **PostgreSQL 15.7**（`docker-compose.yml:110`）。
- ORM 为 Django ORM；模型集中在 `apps/api/plane/db/models/`（~30 文件，约 4348 行，122 个 migration）。
- 自定义用户模型 `db.User`（`common.py:202`）；自定义 Session 模型（`common.py:376`）。
- 可选**读副本**（`ReadReplicaRouter`，`common.py:221-238`，需 `ENABLE_READ_REPLICA=1`）。
- 对象存储：**MinIO / S3**（`django-storages 1.14.2` + `boto3`，`common.py:299-316`）。

## 8. API 与通信协议

- **REST/JSON**（DRF，`JSONRenderer`）。`/api/` 与 `/api/v1/` 双表面（见架构文档）。
- **WebSocket**：仅 `apps/live` 提供协同 WebSocket（`/live/collaboration`），Django 侧无 WS。
- 认证：**Session Cookie**（前端默认，`withCredentials`）+ **CSRF**（`/auth/get-csrf-token/`）+ **API Key**（`X-API-Key`，仅 `/api/v1/`）。
- **OAuth**：Google / GitHub / GitLab / Gitea（`plane/authentication/provider/oauth/`）。
- 可选 **OpenAPI**（`drf-spectacular 0.28.0`，需 `ENABLE_DRF_SPECTACULAR=1`，`common.py:565`）。

## 9. 构建工具 / 包管理 / workspace

- **pnpm 11.3.0**（`package.json:42`）+ **pnpm catalog**（`pnpm-workspace.yaml:7-256` 统一版本）。
- **Turborepo 2.9.18**（`turbo.json`，`dependsOn: ^build` 任务图）。
- Node **22.18.0**（`.mise.toml`、`package.json:40`）。
- Python：`pip` + `requirements/{base,production,test,local}.txt`（`apps/api/requirements/`）。

## 10. 测试框架

- 后端：**pytest 9.0.3 + pytest-django**（`apps/api/requirements/test.txt`），运行于 `docker-compose-test.yml`（`api-tests` 服务）。
- 前端/实时：**Vitest 4.1.8**（`apps/live`、`packages/codemods`，`pnpm-workspace.yaml:184`）。
- 设计系统：**Storybook 10.4.6**（`@plane/propel` 39 stories、`@plane/ui` 6 stories）。
- Lint/Format：**OxLint 1.51 + Oxfmt 0.35**（`.oxlintrc.json`、`.oxfmtrc.json`）；类型检查 TypeScript 5.8.3。
- 详见 `测试与CI.md`（关键结论：**CI 当前不执行任何测试套件**）。

## 11. CI/CD / 容器 / 部署

- CI：GitHub Actions，9 个 workflow（`.github/workflows/`）。
- 容器：Docker（每应用一个/多个 Dockerfile）；编排 `docker-compose.yml`（完整栈）、`docker-compose-local.yml`（开发）、`docker-compose-test.yml`（测试）。
- 部署形态：AIO 单容器（`deployments/aio/community/`）、CLI Compose 安装器（`deployments/cli/community/`）、Kubernetes（Helm chart，`deployments/kubernetes/community/README.md`）、Docker Swarm（`deployments/swarm/community/swarm.sh`）。

## 12. 日志 / 监控 / 可观测

- 后端日志：`python-json-logger`（Celery/`plane/celery.py:99-112`）；请求日志中间件 `plane/middleware/logger.py`（`RequestLoggerMiddleware`、`APITokenLogMiddleware`）。
- **APM：ScoutAPM**（`scout-apm`，仅 production settings，`plane/settings/production.py:17`）。
- **OpenTelemetry**（`opentelemetry-*`，`base.txt:68-73`）：实例遥测 metrics 推送（`plane/license/bgtasks/telemetry_metrics.py`，受 `Instance.is_telemetry_enabled` 控制）。
- 前端：**Sentry**（`turbo.json:5-33` 的 `SENTRY_*`/`VITE_SENTRY_*` env）+ **PostHog**（`common.py:365-366`）。
- 前端运行时 logger：`@plane/logger`（winston，`pnpm-workspace.yaml:185`，仅 `apps/live` 使用）。

## 13. 代码生成 / Feature flag

- i18n 类型生成：`packages/i18n/scripts/generate-types.ts`（`TTranslationKeys`）。
- React Router 类型：`react-router typegen`（各 app `check:types` 脚本）。
- OpenAPI schema：`drf-spectacular`（可选）。
- Feature flag：多为**环境变量开关**（`ENABLE_SIGNUP`、`ENABLE_EMAIL_PASSWORD`、`ENABLE_READ_REPLICA`、`ENABLE_DRF_SPECTACULAR`、`USE_MINIO`、各 `ENABLE_*_SYNC`），存于 `InstanceConfiguration`（`plane/license/`）。

## 14. 关键依赖“是否进入运行时”速查

下表区分“声明依赖 / 源码 import / 运行时初始化 / 仅测试 / 仅示例 / 疑似无效”。

| 依赖 | 声明 | import | 运行时初始化 | 进入关键路径 | 备注 | Evidence |
| -- | -- | -- | -- | -- | -- | -- |
| Django / DRF | ✅ | ✅ | ✅ WSGI app | 是 | 后端核心 | `base.txt:4-5`、`plane/wsgi.py:18` |
| Celery | ✅ | ✅ | ✅ worker/beat | 是（异步任务） | — | `plane/celery.py:38` |
| channels | ✅ | ✅ | ⚠️ ASGI 仅 HTTP | 否（无 WS consumer） | **名义存在，实时未用** | `plane/asgi.py:18` |
| openai | ✅ | ✅ | ⚠️ | 部分 | AI/GPT 端点（`external` URL 组 `GPTIntegrationEndpoint`） | `base.txt:38` |
| slack-sdk | ✅ | ✅(models) | ❌ | 否 | **CE 中无消费视图/任务**（dormant） | `plane/db/models/integration/slack.py` |
| scout-apm | ✅(prod) | ✅ | ⚠️ 仅 prod settings | 否（可选） | — | `plane/settings/production.py:17` |
| Hocuspocus / Yjs | ✅ | ✅ | ✅ live 服务 | 是（协同） | — | `apps/live/src/hocuspocus.ts:45` |
| ioredis | ✅ | ✅ | ⚠️ 可选 | 多实例 pub/sub | `REDIS_URL` 缺失则禁用 | `apps/live/src/redis.ts:62-66` |
| Storybook | ✅ | — | ❌ | 否（仅设计系统开发） | — | `pnpm-workspace.yaml:169` |
| effect / @effect/platform | ✅ | ✅ | ⚠️ 仅 PDF 导出 | 否 | 局部使用 | `apps/live/src/controllers/pdf-export.controller.ts` |
| posthog | ✅ | ✅ | ⚠️ | 客户端分析 | 受 env 控制 | `common.py:365-366` |

[Inference] `channels`、`slack-sdk`、GitHub 集成模型属于“声明 + 部分 import 但 CE 无运行时消费”的疑似 dormant/EE 预留依赖。

## 15. Git 子模块 / vendored source

[Evidence] **无 Git 子模块**（无 `.gitmodules`）。vendored 静态数据：`packages/utils/src/tlds.ts`（~16KB TLD 列表）。前端依赖均通过 pnpm catalog 管理，无内嵌第三方源码目录。

## 16. 平台专用依赖

[Inference] 无明显平台专用（桌面/移动）依赖。`@plane/typescript-config` 下 `nextjs.json` **无消费者**（Next.js 迁移残留，`react-router.json` 被三应用使用但未列入 `package.json files`）。

## 17. 依赖版本来源

[Evidence] 前端版本统一来自 `pnpm-workspace.yaml` 的 `catalog`（各包用 `"catalog:"` 引用）与 `overrides`；后端版本来自 `apps/api/requirements/*.txt`。Node 版本固定 `.mise.toml:2`（`node=22.18.0`）。

---

## 已确认事实

- 单体仓库 6 应用 + 15 包；前端 React 18 + RR7 + MobX + Tailwind 4；后端 Django 4.2 + DRF + Celery + Postgres + Redis + RabbitMQ；实时为独立 Node/Hocuspocus 服务；代理为 Caddy。
- 前后端版本号不一致（JS 1.3.1 / Python 0.24.0）。
- `ssr` 模式三应用不一致：web/admin 纯 CSR，space SSR。
- 多处并行迁移在途（Next.js→RR、ui→propel、本地服务→共享服务、双后端 API）。

## 合理推断

- `channels`、`slack-sdk`、GitHub 集成模型在 CE 中 dormant，真正的同步引擎很可能位于私有 EE/Cloud 仓库（与本仓库的 `ee/` 占位 stub 一致）。
- APM(Scout)、OpenTelemetry、PostHog、Sentry 均为可选可观测组件，是否启用取决于部署环境变量与实例配置。

## Unknown 与待验证事项

- 生产环境是否实际启用读副本、Scout APM、OTLP 收集器、PostHog（依赖运行时环境变量，静态无法确认）。
- 私有 EE 仓库（`plane-cloud`/`plane-ee`）填充了哪些 `ee/`、`extendedRoutes`、`Base*Store` 占位（不在本仓库）。

## 批判性评估

- **多套并行技术栈导致重复与漂移风险**：两套前端服务基类、两套后端 base view、两套 permission 包（byte-identical）、两套 UI 库、Next.js 兼容残留。维护成本与“只修了一处”的回归风险都高。
- `channels` 被声明且 import 但未注册 WS consumer，是“名义能力”，易误导读者认为 Django 提供实时能力。

## 建设性改善建议

- [Recommendation] 收敛后端双 API：将 `/api/` 逐步迁移至 `/api/v1/`，统一 base view 与 permission 包（消除 `plane/app/permissions` 与 `plane/utils/permissions` 的重复）。优先级：中；难度：高（涉及大量调用点）。
- [Recommendation] 完成前端服务层迁移：将 `apps/web/core/services/*` 的 46 个本地服务迁入 `@plane/services`，并在共享基类补回 401 拦截器。优先级：中；难度：中。
- [Recommendation] 对齐前后端版本号或建立映射文档，避免 `.trivyignore` 这类按版本匹配的扫描产生误判。优先级：低；难度：低。

## 主要证据索引

- `package.json:1-43`、`pnpm-workspace.yaml:1-256`、`turbo.json:1-93`
- `apps/api/requirements/base.txt:1-77`、`apps/api/requirements/test.txt`
- `apps/api/plane/settings/common.py:97-392`
- `apps/web/react-router.config.ts:6`、`apps/space/react-router.config.ts:11`、`apps/admin/react-router.config.ts:13`
- `apps/live/package.json`、`apps/live/src/hocuspocus.ts:45`
- `docker-compose.yml:1-178`
- `docs-analysis/evidence/backend-url-index.txt`（395 条路由清单）
