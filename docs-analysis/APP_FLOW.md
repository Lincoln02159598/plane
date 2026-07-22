# APP_FLOW.md — 用户视角应用流程

## 分析快照

- 分支：`preview`；HEAD：`7cef741c2`；工作区：clean（仅新增 `./docs-analysis/`）；子模块：无。
- 分析范围：基于源码（前端路由 + 后端鉴权 + 部署）推导的真实用户流程；未做端到端运行验证。
- 未覆盖范围：私有 EE/Cloud 流程；Plane Cloud（SaaS）注册流。

## 证据分类

- **Evidence**：路由/端点/视图有源码证据（`路径:行号`）。
- **Inference**：由路由与组件推导的用户旅程。
- **Unknown**：仓库内无法确认（如外部 IdP 实际配置）。

## 核心结论

[Evidence] Plane 是一个**强登录导向**的多租户项目管理应用：用户必须登录后进入工作区 → 项目 → 执行任务；存在实例初始化（首次 setup）、多工作区/项目切换、三种客户端入口（web 主应用、space 公开看板、admin 实例管理）。鉴权为 cookie session（web/space）与独立 god-mode session（admin）。每个核心场景均可与源码调用链对应。

---

## 1. 安装 / 首次运行 / 实例初始化（自托管）

- 前置：部署（Docker Compose / AIO / Helm）后访问实例。
- 首次进入 `/god-mode/`（admin）：
  - `apps/admin/app/(all)/(home)/page.tsx:18` 检查 `instance.is_setup_done`。
  - 未 setup → `InstanceSetupForm`（配置实例）→ 完成后 `InstanceSignInForm`（首个管理员）。
  - 管理员登录：form POST `${API_BASE_URL}/api/instances/admins/sign-in/`（`apps/admin/app/(all)/(home)/sign-in-form.tsx:122`）。
- 系统响应：`Instance`/`InstanceConfiguration` 写入；`is_setup_done=True`。
- 异常：setup 失败 → `InstanceFailureView`。

## 2. 用户登录 / 注册 / OAuth

- 入口：`apps/web` `(home)` 登录页（`app/routes/core.ts`，`AuthBase` SIGN_IN）。
- CSRF：`AuthService.requestCSRFToken()` GET `/auth/get-csrf-token/`。
- 方式（`plane/authentication/urls.py`）：
  - 邮箱密码：POST `/auth/sign-in/`（`SignInAuthEndpoint`）。
  - Magic Code：`/auth/magic-generate/` → `/auth/magic-sign-in/` 或 `/magic-sign-up/`。
  - OAuth：`/auth/{google,github,gitlab,gitea}/`（initiate）→ `/auth/{provider}/callback/`。
- 后端：`Adapter.complete_login_or_signup()`（`plane/authentication/adapter/base.py:309-408`）——门控 `ENABLE_SIGNUP`、停用/bot 拒绝、密码强度、（OAuth）头像下载。
- 数据变化：`User` upsert + Session 写 Redis + Set-Cookie `session-id`。
- 前端：SWR `USER_INFORMATION` → MobX user → `AuthenticationWrapper`（`apps/web/core/lib/wrappers/authentication-wrapper.tsx:32`）重定向（onboarding 或最近工作区）。
- 异常：错误码 `EAuthenticationErrorCodes`（5000–5900，`apps/web/helpers/authentication.helper.tsx`）。

```
前置条件：实例已 setup，ENABLE_SIGNUP 开启（注册时）
→ 用户操作：输入凭据 / 点 OAuth
→ 系统响应：校验 → 建 Session → Set-Cookie
→ 数据变化：User.last_login_* 更新；Session 写入
→ 异常分支：停用账号/bot/弱密码/未邀请(signup 关) → 结构化错误码
→ 完成条件：session-id cookie 下发，前端跳转工作区
```

## 3. 无登录模式

[Evidence] **不存在完全无登录的工作区访问**。匿名仅可访问**已发布的公开看板**（`apps/space`，`issues/:anchor`），经 `plane/space/` 的 `AllowAny` 端点 + `DeployBoard`/`is_public` 门控（`plane/space/views/project.py:36` 等）。主应用 web 必须登录。

## 4. 工作区 / 项目创建与进入

- 创建工作区：登录后若无可进工作区 → `create-workspace` 路由（`apps/web/app/routes/core.ts`）。
- onboarding：首次登录 → onboarding 流（`onboarding` 路由）。
- 进入工作区：`[workspaceSlug]` → `(projects)` 工作区主页（active cycles/analytics/browse/notifications/stickies/workspace-views）。
- 项目：工作区内项目树（issues/cycles/modules/views/pages/intake + 归档）。
- 数据：`Workspace`/`Project`/`ProjectMember`；权限 `WorkspaceMemberPermission`/`ProjectMemberPermission`。
- 切换工作区/项目：`AuthenticationWrapper` 处理 last/fallback workspace；侧边栏（app-rail）导航。

## 5. 核心任务流程（Work Item 创建/更新）

- 前置：已进入某项目。
- 创建：web issue store → `IssueService` POST `/api/v1/.../work-items/`（或 `/api/.../issues/`）。
- 链路（见 `核心数据流.md`）：`IssueListCreateAPIEndpoint.post` → `IssueSerializer` → `Issue.save()`（advisory lock sequence_id）→ 异步 `issue_activity`/`model_activity`。
- 更新：PATCH issue（state/assignee/label…）→ `ChangeTrackerMixin` → `issue_activity`（活动 + 通知）→ `webhook_activity`（若订阅）。
- 数据变化：`Issue` + `IssueActivity`；附件→`FileAsset`+S3。
- 完成条件：列表实时更新（SWR/MobX）；协作者收到通知/webhook。

## 6. Pages 实时协同

- 前置：项目内打开 Page。
- 连接：`editor-body.tsx` 构造 WS URL → `HocuspocusProvider`（`packages/editor/src/core/hooks/use-yjs-setup.ts:62`）。
- 鉴权：live `onAuthenticate` 经 cookie 调 `/api/users/me/` 校验（`apps/live/src/lib/auth.ts`）。
- 编辑：Yjs 文档 → Redis 跨实例同步 → 10s debounce → PATCH `/api/.../pages/{id}/description/`。
- 异常：连接码 4000-4003 → force-close + 重连（≤3 次）；413 → 跨服 force-close。

## 7. 数据保存 / 同步 / 导入 / 导出

- 保存：所有写操作经 REST（同步 DB 写 + 异步副作用）。
- 同步：实时仅 Pages（live）；issues 等靠 SWR 重新验证（`revalidateOnFocus/mount`）。
- 导入：`importer` 模型 + 端点（CSV/外部源）。
- 导出：`exporter` + `export_task`（CSV/xlsx）；PDF 经 live `/live/pdf-export`。

## 8. 设置 / 多用户 / 多工作区切换 / 退出

- 设置：`(settings)` 路由（工作区设置 + 项目设置 + `settings/profile`）。
- 多工作区切换：侧边栏 + `AuthenticationWrapper` fallback。
- 退出：`signOut()` 构造 `<form>` POST `/auth/sign-out/`（`apps/web/core/services/auth.service.ts:62`）→ `resetOnSignOut()` 重置 MobX store（`root.store.ts:140`）→ 清 cookie。

## 9. 升级 / migration / 数据恢复

- migration：`migrator` 容器（`docker-compose.yml:84-98`，`docker-entrypoint-migrator.sh`，`restart: no`）一次性执行 `migrate`；local 用 `--settings=plane.settings.local`。
- 升级：`deployments/cli/community/install.sh` 拉取新镜像；CLI Compose 用 `APP_RELEASE`。
- 数据恢复：`deployments/cli/community/restore.sh`、`restore-airgapped.sh`、`migration-0.13-0.14.sh`。
- 软删除恢复：`HARD_DELETE_AFTER_DAYS`（默认 60，`common.py:424`）内可恢复；超期 `hard_delete`（每日 00:00 beat）。

## 10. 错误提示 / 异常恢复

- 前端：RR7 `ErrorBoundary`（`app/root.tsx`）；toast（`Toast`/`ToastProvider`）；认证错误码映射。
- 后端：generic 错误消息（隐藏细节）+ 日志；webhook 自动重试与熔断。
- live：`unhandledRejection`/`uncaughtException` 记录；崩溃由 Docker `restart: always` 恢复。

---

## 用户流程 ↔ 源码调用链对应

| 用户场景 | 前端入口 | 通信 | 后端入口 | 关键函数 | Evidence |
| -- | -- | -- | -- | -- | -- |
| 实例 setup | admin `(home)` | form POST | `/api/instances/admins/sign-in/` | `InstanceEndpoint` | `apps/admin/app/(all)/(home)/page.tsx:18` |
| 登录 | web `(home)` | POST `/auth/sign-in/` | `SignInAuthEndpoint` | `Adapter.complete_login_or_signup` | `plane/authentication/adapter/base.py:309` |
| 创建 issue | web project | POST `/api/v1/.../work-items/` | `IssueListCreateAPIEndpoint` | `Issue.save`（advisory lock） | `plane/api/views/issue.py:449` |
| 协同编辑 Page | web page editor | WS `/live/collaboration` | live `onAuthenticate` | `Database.storeDocument` | `apps/live/src/extensions/database.ts:72` |
| 退出 | web | POST `/auth/sign-out/` | `SignOutAuthEndpoint` | `resetOnSignOut` | `apps/web/core/services/auth.service.ts:62` |
| 公开看板（匿名） | space `issues/:anchor` | GET `/api/public/...` | space `AllowAny` view | `DeployBoard` 门控 | `plane/space/views/project.py:36` |

---

## 已确认事实

- 强登录导向；web/space/admin 三入口；cookie session（web/space）+ god-mode session（admin）。
- 完整 setup→登录→工作区→项目→任务→协同→退出旅程均有源码对应。
- 匿名仅限已发布看板；软删除 60 天可恢复。

## 合理推断

- 首次 setup 必须经 admin（god-mode）完成实例初始化与首位管理员。
- 多数数据为客户端拉取（SWR+MobX），实时协同仅限 Pages。

## Unknown 与待验证事项

- OAuth 各 IdP 的实际可用性取决于实例配置（`ENABLE_*_SYNC`、client secret）；公开看板所有匿名端点是否均做了发布门控（未逐一审计）。

## 批判性评估

- 登录旅程错误消息未国际化（admin 完全未 i18n），影响非英语用户首次体验。
- 匿名公开看板的 `AllowAny` 端点需确保每个都执行 `DeployBoard`/`is_public` 门控，否则存在越权读取风险（值得逐一审计）。

## 建设性改善建议

- [Recommendation] 审计 `plane/space/` 所有 `AllowAny` 端点，确保统一经 `DeployBoard`/发布门控后再返回数据。优先级：**高**；难度：低。
- [Recommendation] 登录/认证错误消息国际化；admin 接入 i18n。优先级：中；难度：低。
- [Recommendation] 为关键用户旅程（登录/创建 issue/协同）补端到端测试（当前无 E2E）。优先级：中；难度：中。

## 主要证据索引

- `apps/web/app/routes/core.ts`（路由树）、`apps/web/core/lib/wrappers/authentication-wrapper.tsx:32`
- `apps/admin/app/(all)/(home)/page.tsx:18`、`apps/admin/app/(all)/(home)/sign-in-form.tsx:122`
- `plane/authentication/urls.py:49-153`、`plane/authentication/adapter/base.py:309-408`
- `plane/api/views/issue.py:449`、`apps/live/src/lib/auth.ts:24-97`、`apps/live/src/extensions/database.ts:72-134`
- `plane/space/views/project.py:36`、`apps/web/core/services/auth.service.ts:62`
- `apps/space/app/routes.ts:10`、`docker-compose.yml:84-98`（migrator）、`deployments/cli/community/restore.sh`
