# BACKEND_Development.md — 后端开发视角

## 分析快照

- 分支：`preview`；HEAD：`7cef741c2`；工作区：clean（仅新增 `./docs-analysis/`）；子模块：无。
- 分析范围：`apps/api`（Django）静态只读；未运行迁移/测试。
- 未覆盖范围：私有 EE 仓库；动态运行行为。

## 证据分类

- **Evidence**：`路径:行号` + 符号。
- **Inference**：由多处证据推导。
- **Unknown**：静态无法确认。

## 核心结论

[Evidence] 后端为 **Django 4.2 + DRF 3.15 + Celery 5.4**，WSGI 运行；自定义用户模型 `db.User`；Postgres 主存储 + Redis 缓存/会话 + RabbitMQ broker + S3/MinIO 对象存储。提供**两套并行 REST 表面**（`/api/` 旧、`/api/v1/` 新），均无 service/repository 层，业务逻辑散布于 view/serializer/model。

---

## 1. 入口与初始化

- `manage.py:10`、`plane/wsgi.py:16-18`（`get_wsgi_application()`）、`plane/asgi.py:13-18`、`plane/celery.py:20` 均 `DJANGO_SETTINGS_MODULE = plane.settings.production`。
- [Evidence] `plane/settings/common.py` 为真实配置基类；`production.py:17` 追加 `scout_apm.django`；`local.py:14` 追加 `debug_toolbar`；`test.py:14-16` 追加测试 app。
- 容器入口：`bin/docker-entrypoint-{api,worker,beat,migrator}.sh`（`docker-compose.yml:45,60,76,92`）。

## 2. 配置加载

- 数据库：`DATABASE_URL` 优先，否则 `POSTGRES_*`；可选读副本（`common.py:205-238`）。
- Redis：`REDIS_URL`（SSL 检测 `rediss`，`common.py:242-263`）。
- Celery broker：`AMQP_URL` 或 RabbitMQ 各字段（`common.py:318-330`）。
- 存储：`USE_MINIO` + S3 endpoint（`common.py:299-316`）。
- 运行时配置表：`InstanceConfiguration`（加密 key/value，`plane/license/models/instance.py`）→ `get_configuration_value()`。

## 3. 日志初始化

- `RequestLoggerMiddleware`（`plane/middleware/logger.py:26`）：请求耗时/状态 → logger `plane.api.request`。
- `APITokenLogMiddleware`（`logger.py:80`）：持久化 `/api/v1/` 请求/响应到 `APIActivityLog`（经 `process_logs.delay`）；**敏感头脱敏**（`x-api-key/authorization/cookie`），API key 仅存 HMAC-SHA256 `token_identifier`（`logger.py:117-146`）。
- Celery：JSON formatter（`plane/celery.py:99-112`）。

## 4. 服务/路由注册

[Evidence] 根路由 `plane/urls.py:17-24`：

| 前缀 | 模块 | 风格 | 鉴权 |
| -- | -- | -- | -- |
| `/api/` | `plane.app.urls`（19 组） | DRF `ModelViewSet`/`BaseAPIView` | Session（关 CSRF） |
| `/api/v1/` | `plane.api.urls`（12 组） | `GenericAPIView` 的 `…APIEndpoint` | API Key + 限流 |
| `/api/public/` | `plane.space.urls`（4 组） | 公开看板（部分 `AllowAny`，门控 `DeployBoard`） | Session/AllowAny |
| `/api/instances/` | `plane.license.urls` | 实例管理 | 管理员 |
| `/auth/` | `plane.authentication.urls` | credentials/magic/OAuth/密码 | — |
| `/` | `plane.web.urls` | SPA 兜底 | — |

- 完整路由清单见 `docs-analysis/evidence/backend-url-index.txt`（**395 条**）。
- v1 新增 **lite 端点**：`*-lite/`（`plane/api/views/project.py:342`、`cycle.py:360`、`module.py:281`、`member.py:233,280`）——HEAD commit 的内容。

## 5. API 接口结构（以 Issue 创建为例）

```
入口：POST /api/v1/workspaces/{slug}/projects/{project_id}/work-items/
  → IssueListCreateAPIEndpoint.post()                 plane/api/views/issue.py:449
参数：path (slug, project_id) + JSON body（IssueSerializer 字段）
参数验证：IssueSerializer.is_valid() + ProjectEntityPermission + ApiKeyRateThrottle(60/min)
返回值：201 IssueSerializer.data | 400 errors | 409 external_id 冲突
调用服务：无 service 层；直接 serializer.save()
数据访问：Issue ORM + transaction.atomic + pg_advisory_xact_lock 生成 sequence_id
事务边界：sequence_id 生成在事务+advisory lock 内（plane/db/models/issue.py:180-214）；
         活动记录/通知/webhook 为 fire-and-forget 异步（无统一事务）
错误类型：IntegrityError→400, ValidationError→400, ObjectDoesNotExist→404, KeyError→400, else→500
调用方：web/space 前端、外部 API key 客户端
Evidence：plane/api/views/issue.py:256-522
```

> 注：`/api/v1/` 的 list 端点显式拒绝 `pql`/`filters` 参数（`plane/api/views/issue.py:317-329`），提示“此 Plane 版本不支持”，表明 PQL 属于 EE/Cloud 能力。

## 6. 应用服务 / 领域逻辑 / 数据访问

[Inference] **无独立应用服务层或仓储层**。典型分布：
- 校验与编排：视图（`get/post/patch/delete`）。
- 实体副作用：序列化器 `create/update`（M2M：assignees/labels，`plane/app/serializers/issue.py:199`）。
- 不变量：模型 `save()`（如 sequence_id、`created_by` 自动注入 `plane/db/models/base.py:23-44`）。
- 查询：视图 `get_queryset()` 内联 ORM（select_related/prefetch/annotate），无独立 repository。

## 7. 数据库 / 数据模型 / migration

- 模型基类链：`BaseModel(AuditModel)` ← `AuditModel(TimeAuditModel, UserAuditModel, SoftDeleteModel)`（`plane/db/mixins.py:85`）。UUID 主键、`created_at/updated_at`、`created_by/updated_by`（SET_NULL）、`deleted_at`。
- **软删除为默认**：`objects = SoftDeletionManager()` 过滤 `deleted_at__isnull=True`；`delete(soft=True)` 设 `deleted_at` 并异步 `soft_delete_related_objects.delay()` 级联（`plane/db/mixins.py:48-82`）。
- `ChangeTrackerMixin`（`mixins.py:92-221`）记录字段变更，供活动追踪使用。
- 核心模型：`User`、`Workspace`、`Project`/`ProjectMember`、`Issue`/`IssueSequence`/`IssueActivity`/`IssueAssignee`/`IssueLabel`、`Cycle`/`Module`/`Page`/`State`/`Label`/`Estimate`、`FileAsset`、`APIToken`、`Webhook`/`WebhookLog`、`DeployBoard`、`Instance`/`InstanceConfiguration`。集成模型：`integration/{github,slack}.py`（CE dormant）。
- migration：122 个（`plane/db/migrations/`）+ license app 自有 migrations。

## 8. 事务 / 并发

- sequence_id 生成：`transaction.atomic()` + `pg_advisory_xact_lock(project_id)` 原子化（`plane/db/models/issue.py:180-214`）。
- [Inference] 大多数写操作的“DB 变更”与“副作用（通知/webhook/活动）”跨进程异步，**无跨 DB+MQ 的分布式事务**；失败靠任务重试与人工。

## 9. 缓存 / 队列 / 后台任务

- 缓存：django-redis（`common.py:246-263`），默认 backend。
- 队列：RabbitMQ → Celery（`worker`、`beat`）。
- beat 调度（`plane/celery.py:44-95`）：邮件通知（5min）、实例遥测（6h）、硬删除（每日 00:00）、自动归档/关闭旧 issue（01:00）、各类日志/版本清理（02:00–03:45）。
- `~34` 个任务模块（`plane/bgtasks/`）：活动追踪、通知、webhook、邮件、版本同步、清理、导出、遥测、种子数据等。

## 10. 鉴权 / 权限

- Session：`SessionMiddleware` 按 path 选择 `session-id`/`admin-session-id`（`plane/authentication/middleware/session.py:16-66`）；`SESSION_ENGINE=plane.db.models.session`（自定义 Session，128 字符 key）。
- `BaseSessionAuthentication`（`plane/authentication/session.py:8`）**关闭 REST CSRF**。
- API Key：`APIKeyAuthentication`（`plane/api/middleware/api_authentication.py:17`），校验 `APIToken`（active/未过期/user active）。
- Credentials：email/password（`ENABLE_EMAIL_PASSWORD`）、magic_code。
- OAuth：Google/GitHub/GitLab/Gitea（`provider/oauth/*.py`），IDP sync 可选。
- Adapter：`complete_login_or_signup()` 统一处理 signup 门控、停用账号/bot 拒绝、SSRF 安全头像下载（`plane/authentication/adapter/base.py:309-408`）。
- 限流：anon 30/min、asset_id 5/min、api_key 60/min（`common.py:140-154`、`plane/api/rate_limit.py:12`）。
- 权限类：`ROLE` ADMIN=20/MEMBER=15/GUEST=5；`ProjectMemberPermission`/`WorkspaceMemberPermission` 等。**两套近乎相同**（`plane/app/permissions/` vs `plane/utils/permissions/`，`project.py` byte-identical）。

## 11. 错误处理

- DRF 全局 handler：`plane/authentication/adapter/exception.py:17`。
- 各 base view `dispatch/handle_exception` 映射异常→HTTP（`plane/api/views/base.py:64-113`）。
- 认证错误码：`AUTHENTICATION_ERROR_CODES` + `AuthenticationException`（`plane/authentication/adapter/error.py`）。

## 12. 外部服务 / 文件系统 / 安全边界

- 外部：S3/MinIO（boto3）、SMTP（邮件）、（可选）OpenAI/PostHog/Scout/OTLP。
- 文件上传：`FileAsset` + S3Storage；MIME 白名单（`common.py:457-545`）；脚本类 MIME 强制 `Content-Disposition: attachment`（`SCRIPT_CAPABLE_MIME_TYPES`，`common.py:550-560`）。
- 安全：SSRF 集中防护（webhook/头像）、HMAC webhook 签名、API key HMAC 日志、`SECRET_KEY` 自检。

## 13. 扩展方式 / 测试方式 / 调试入口

- 扩展：新增 Django app + 注册 INSTALLED_APPS + urls 挂载；新增 Celery task（autodiscover 或 `CELERY_IMPORTS`）。
- 测试：`docker-compose-test.yml`（`api-tests`）+ pytest（`pytest.ini`）；详见 `测试与CI.md`。**CI 不执行**。
- 调试：`local.py` + `debug_toolbar`（DEBUG）；`--reuse-db --nomigrations`（`pytest.ini:13-16`）。

## 14. 已知限制 / 后端技术债务

- 双 API 表面 + 重复 permission/base view（详见架构文档）。
- 无 service 层，逻辑分散。
- GitHub/Slack 集成、`channels` WS、`plane/analytics` 在 CE dormant/名义。
- `run_tests.sh` 损坏（调用不存在的 `tests/run_tests.sh`）。
- issue/work-item 在 `/api/v1/` 下双重 URL（`old`/`new` patterns）。

---

## 已确认事实

- 两套 REST 表面（`/api/` ViewSets + `/api/v1/` APIViews），同一批模型；395 条路由。
- Session（关 CSRF）+ API Key 双鉴权；OAuth 4 家。
- 软删除默认 + 异步级联；advisory lock 生成 sequence_id；活动/webhook 异步管线。
- 122 migration；~34 Celery 任务；beat ~13 周期任务。

## 合理推断

- PQL/结构化 filters 与 GitHub/Slack 同步引擎属 EE/Cloud，CE 显式拒绝或无消费。
- 写操作的 DB 与副作用无分布式事务，依赖任务重试。

## Unknown 与待验证事项

- 登录限流实现细节（`plane/authentication/rate_limit*` 未细读）；生产是否启用 Scout/OTLP/读副本。

## 批判性评估

- 双 API + 双 permission 包是后端最大维护负担；近期安全 scoping commit 正好落在重复 permission 区，回归风险高。
- 异常→generic 消息的设计利于安全但牺牲可调试性；`handle_exception` 在 `BaseAPIView` 对部分异常未记日志（`BaseViewSet` 更完整），存在不一致。

## 建设性改善建议

- [Recommendation] 统一后端双 API 与 permission 包，引入薄 service 层供两套 API 共享。优先级：高；难度：高。
- [Recommendation] 修复 `run_tests.sh` 与测试文档路径漂移（`TESTING_GUIDE.md` 引用不存在的 fixture/路径）。优先级：中；难度：低。
- [Recommendation] 统一 `handle_exception` 日志策略，避免部分异常被静默吞掉。优先级：中；难度：低。

## 主要证据索引

- `plane/urls.py:17-24`、`plane/settings/common.py:97-392`
- `plane/api/views/base.py:49-266`、`plane/api/views/issue.py:256-522`
- `plane/db/mixins.py:48-221`、`plane/db/models/base.py:17-47`、`plane/db/models/issue.py:180-214`
- `plane/authentication/adapter/base.py:309-408`、`plane/authentication/session.py:8-11`、`plane/authentication/middleware/session.py:16-66`
- `plane/api/middleware/api_authentication.py:17-43`、`plane/api/rate_limit.py:12`
- `plane/bgtasks/issue_activities_task.py:1503-1587`、`plane/bgtasks/webhook_task.py:242-465`
- `plane/celery.py:38-118`、`plane/middleware/logger.py:26-146`
- `docs-analysis/evidence/backend-url-index.txt`
