# PRD.md — 产品视角（基于当前代码实际实现）

## 分析快照

- 分支：`preview`；HEAD：`7cef741c2`；工作区：clean（仅新增 `./docs-analysis/`）；子模块：无。
- 分析范围：基于源码验证的产品能力，对照 README/docs 声明。
- 未覆盖范围：私有 EE/Cloud 产品形态；Plane Cloud（SaaS）实际能力。

## 证据分类

- **Evidence**：源码直接证明的产品能力（`路径:行号`）。
- **Inference**：由多处证据推断。
- **Unknown**：仓库内无法确认（多为 EE/Cloud 能力）。

## 核心结论

[Evidence] 当前仓库实现的是 **Plane 社区版（CE）**——一个开源、自托管的**团队项目管理工具**，核心能力为 Work Items（issues）、Cycles（迭代）、Modules（模块）、Views（视图）、Pages（富文本/协同）、Intake（收件箱）、Analytics（分析）、Webhooks、API Key 访问。支持邮箱/密码、Magic Code、OAuth（Google/GitHub/GitLab/Gitea）登录；多工作区、项目成员与角色（Admin/Member/Guest）；实例管理（God mode）；公开项目看板（Spaces）。`Instance.edition` 默认 `PLANE_COMMUNITY`（`plane/license/models/instance.py`）。

[Evidence] README 列举的 5 大特性（Work Items / Cycles / Modules / Views / Pages / Analytics）**均有后端模型、REST 端点与前端 UI 支撑**，可验证为已实现。但部分高级能力（PQL 结构化查询、GitHub/Slack 深度同步、AI）在 CE 中为占位或受限。

---

## 1. 当前已实现产品

### 核心功能（已完整实现）

| 功能 | 源码证据 | 判断 |
| -- | -- | -- |
| **Work Items（Issues）** | `plane/db/models/issue.py`、`plane/api/views/issue.py`、`plane/app/views/issue/`、web `core/components/issues`（~293 文件） | ✅ 完整（CRUD、子 issue、关联、链接、附件、评论、reaction、订阅、归档、活动历史、版本） |
| **Cycles** | `plane/db/models/cycle.py`、`plane/api/views/cycle.py`、`plane/app/views/cycle/` | ✅ 完整（含 burn-down/进度、归档、issue 转移、lite 端点） |
| **Modules** | `plane/db/models/module.py`、`plane/api/views/module.py` | ✅ 完整（含归档、lite 端点） |
| **Views** | `plane/db/models/view.py`、`plane/app/views/...`、rich/work-item filters（`@plane/shared-state`、`@plane/utils`） | ✅ 实现（CE 仅客户端过滤，无 PQL） |
| **Pages** | `plane/db/models/page.py`、协同编辑（`@plane/editor` + `apps/live`） | ✅ 完整（实时协同、版本、标题同步） |
| **Analytics** | `plane/app/views/analytic/`、`ANALYTICS_BASE_API`（外部服务代理） | ⚠️ 部分（依赖外部 analytics 服务；`plane/analytics` app 为 stub） |
| **Intake** | `plane/db/models/intake.py`、`plane/api/views/intake.py` | ✅ 实现 |
| **States / Labels / Estimates** | `plane/db/models/{state,label,estimate}.py` | ✅ 实现 |
| **Notifications** | `plane/bgtasks/{notification,email_notification}_task.py` | ✅ 实现（异步、beat 5min 聚合邮件） |
| **Webhooks** | `plane/db/models/webhook.py`、`plane/bgtasks/webhook_task.py` | ✅ 完整（HMAC、SSRF 防护、重试熔断） |
| **API Key 访问** | `APIToken`、`APIKeyAuthentication`、`/api/v1/` | ✅ 实现（限流 60/min） |
| **导入/导出** | `plane/db/models/{importer,exporter}.py`、`plane/utils/porters/`、`export_task` | ✅ 实现（CSV/xlsx） |
| **用户/工作区/项目/成员/角色** | `User/Workspace/Project/ProjectMember`、`ROLE` 20/15/5 | ✅ 实现 |
| **鉴权** | credentials/magic/OAuth 4 家 | ✅ 实现 |
| **实例管理（God mode）** | `apps/admin`、`plane/license/` | ✅ 实现 |
| **公开看板（Spaces）** | `apps/space`、`plane/space/`、`DeployBoard` | ✅ 实现 |
| **国际化** | 19 语言 28 命名空间 | ✅ 实现（admin 未接入） |
| **实时协同（页面/富文本）** | `apps/live`（Hocuspocus/Yjs） | ✅ 实现 |

### 辅助功能

- 文件上传（MinIO/S3，MIME 白名单）、搜索、收藏、最近访问、stickies、issue types、drafts、时区、主题（含高对比）。

## 2. README/docs 声明 vs 源码验证（功能实现状态矩阵）

README（`README.md`）声称的能力逐项验证：

| 功能 | 文档声明 | 源码状态 | 运行时入口 | 测试证据 | 最终判断 |
| -- | -- | -- | -- | -- | -- |
| Work Items（含富文本/文件/子属性/关联） | ✅ | 模型+双 API+UI | `/api/v1/.../work-items/`、web | contract/app+contract/api 测试（CI 不跑） | **已完整实现** |
| Cycles（burn-down 等） | ✅ | 模型+API+UI | `/api/v1/.../cycles/` | contract 测试 | **已完整实现** |
| Modules | ✅ | 模型+API+UI | `/api/v1/.../modules/` | contract 测试 | **已完整实现** |
| Views（过滤/保存/分享） | ✅ | view 模型+filters DSL | 客户端过滤 | 无 | **已实现（CE 无 PQL）** |
| Pages（AI + 富文本 + 协同） | ✅（含 AI） | 协同编辑完整 | `/live/collaboration`、`/api/.../pages/` | 无（live 仅测 PDF） | **协同已实现；AI 见下** |
| Analytics（实时洞察） | ✅ | 代理外部 analytics 服务 | `/api/.../analytics/` | 无 | **部分实现（依赖外部服务）** |
| Docker 自托管 | ✅ | `docker-compose.yml` + 4 部署形态 | — | — | **已实现** |
| Kubernetes | ✅ | 外部 Helm chart（`deployments/kubernetes/community/README.md`） | — | — | **已实现（外部 chart）** |
| God mode 实例管理 | ✅ | `apps/admin` + `/api/instances/` | `/god-mode/` | 无 | **已实现** |

### 声明但源码未在 CE 验证 / 受限的能力

| 能力 | 声明/线索 | CE 源码实际 | 最终判断 |
| -- | -- | -- | -- |
| **AI（Pages/issue 描述）** | README Pages 提“AI capabilities”；`openai` 在依赖；`external` URL 组有 `GPTIntegrationEndpoint` | 后端有 GPT 端点，但具体 AI 体验多依赖外部/EE 配置 | **部分实现 / [Unknown] 生产启用情况** |
| **PQL（Plane Query Language）/ 结构化 filters** | `/api/v1/` 显式拒绝并提示“此 Plane edition 不支持” | CE 端点返回 400 要求客户端过滤 | **CE 不支持（EE/Cloud 能力）** |
| **GitHub 同步** | README 未明确，但存在 `integration/github.py` 模型 | **CE 无消费视图/任务/URL**（dormant） | **仅模型存在 / [Unknown] EE 是否实现** |
| **Slack 同步** | `slack-sdk` 依赖、`integration/slack.py` 模型 | **CE 无消费视图/任务**（dormant） | **仅模型存在 / [Unknown]** |
| **Plane Cloud（SaaS）** | README 引导注册 app.plane.so | 不在本仓库 | **[Unknown] 不在分析范围** |

## 3. 仅存在接口/配置/占位的功能

- [Evidence] `plane/analytics/`：仅 `apps.py`（stub）。
- [Evidence] `plane/asgi.py`：channels 配置但无 WS consumer（实时外置）。
- [Evidence] GitHub/Slack 集成模型：CE 无运行时消费。
- [Evidence] `@plane/services` 的 `live.service.ts`（12 行 stub，未 barrel 导出）、`IndexedDBService`、`IntakeService`、`ProjectViewService`、`ModuleService`（构造器缺失、0 消费者）。
- [Evidence] editor `src/ee/`（re-export stub）、web `extendedRoutes`（空）、`Base*Store`（CE 实现，EE 覆写占位）。

## 4. 产品边界 / 非目标 / 当前限制

- **边界**：CE 是自托管单实例/多工作区项目管理；**非**通用插件平台、**非**桌面/移动原生、**非**内置实时通信后端（实时由独立 live 服务）。
- **非目标（CE）**：PQL 结构化查询、GitHub/Slack 深度双向同步、云端协同（属 EE/Cloud）。
- **限制**：
  - CI 不执行测试（质量依赖 review）。
  - 双 API/双 UI 等迁移在途，存在行为漂移。
  - admin 完全未国际化、未用编辑器。
  - 实时协同 10s debounce 持久化（崩溃可能丢失少量编辑）。
  - 前端无自动化测试与 E2E。

## 5. 用户角色 / 输入 / 输出 / 外部依赖

- 角色：实例管理员（God mode）、工作区 Owner/Admin、项目 Member（Admin=20/Member=15/Guest=5）、匿名（公开看板 `AllowAny` + `DeployBoard` 门控）。
- 输入：Web UI（web/space/admin）、REST（cookie/API key）、WebSocket（协同）、Webhook（出站）。
- 输出：JSON 响应、邮件通知、出站 webhook、导出文件（CSV/xlsx）、PDF（live）、遥测（OTLP，可选）。
- 外部依赖：Postgres、Valkey/Redis、RabbitMQ、MinIO/S3、SMTP、（可选）OpenAI/PostHog/Scout/OTLP/Unsplash/GitHub OAuth。

## 6. 产品风险 / 数据与隐私边界

- 数据：用户内容（issues/pages/附件）存 Postgres + MinIO；会话存 Redis；日志（APIActivityLog/WebhookLog）有保留期（默认 7–14 天，`common.py:445-452`）。
- 隐私：遥测受 `Instance.is_telemetry_enabled` 控制（关则不上报，`telemetry_metrics.py:80`）；API key 仅 HMAC 日志；敏感头脱敏。
- 风险：CSRF 对 REST 关闭（依赖 cookie 属性）；公开看板 `AllowAny` 端点需确保均做发布门控；自托管默认 `trusted_proxies 0.0.0.0/0`。

## 7. README 愿景 vs 当前实现 vs 推断方向 vs 建议（分离）

1. **当前已实现产品**：见 §1（CE，16+ 核心功能完整）。
2. **README/docs 愿景**：强调“现代项目管理、无工具管理负担”、AI、Cloud——愿景层面，Cloud/AI 部分不在 CE 仓库。
3. **源码可推断的发展方向**：双 API 收敛（→ `/api/v1/`）、UI 库收敛（→ propel）、服务层收敛（→ `@plane/services`）、store 抽取（→ `@plane/shared-state`）、Next.js→RR 迁移收尾。
4. **改善建议**：见末尾（与各专题文档一致）。

---

## 已确认事实

- CE 实现 16+ 核心产品功能（work items/cycles/modules/views/pages/analytics/intake/webhooks/API key/导入导出/鉴权/实例管理/公开看板/i18n/实时协同）。
- README 5 大特性均有源码支撑；PQL、GitHub/Slack 深度同步、部分 AI 在 CE 不支持/占位。
- 版本：CE 社区版（`PLANE_COMMUNITY`）。

## 合理推断

- GitHub/Slack 同步引擎、PQL、完整 AI 体验位于私有 EE/Cloud 仓库。
- 多项内部迁移指示产品处于重构活跃期。

## Unknown 与待验证事项

- 生产是否启用 AI/Analytics 外部服务；EE/Cloud 的完整能力边界；公开看板所有 `AllowAny` 端点是否均做了发布门控（未逐一审计）。

## 批判性评估

- 产品功能广且核心完整，但“CE 占位（GitHub/Slack/PQL/部分 AI）+ EE 实现”的分裂使用户/开发者易误判能力边界——应在 UI/文档中显式区分。
- 质量保障（无 CI 测试）与产品成熟度不匹配，是发布风险点。

## 建设性改善建议

- [Recommendation] 在 UI/文档明确标注 CE 不支持的能力（PQL/GitHub-Slack 同步/部分 AI），避免误期望。优先级：中；难度：低。
- [Recommendation] 补 CI 测试执行（见 `测试与CI.md`）以匹配产品成熟度。优先级：高；难度：中。
- [Recommendation] 收敛双 API/UI（见架构文档）以降低产品行为漂移。优先级：中；难度：高。

## 主要证据索引

- README.md（声明）、`plane/db/models/*`、`plane/api/views/*`、`plane/app/views/*`
- `plane/api/views/issue.py:317-329`（PQL 拒绝）、`plane/db/models/integration/{github,slack}.py`（dormant）
- `plane/license/models/instance.py`（edition）、`plane/analytics/apps.py`（stub）、`plane/asgi.py:18`
- `apps/admin`、`apps/space`、`apps/web/core/components/issues`
- `docs-analysis/evidence/backend-url-index.txt`
