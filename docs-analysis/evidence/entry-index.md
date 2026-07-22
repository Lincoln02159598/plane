# 入口与 URL 表面索引（只读证据）

> 由分析整理；用于交叉核对 SOURCE_ARCHITECTURE / BACKEND_Development / 运行时架构 的入口声明。

## 后端入口（apps/api，Django）

| 类型 | 文件 | 符号/键 | 说明 |
| -- | -- | -- | -- |
| CLI | apps/api/manage.py:10 | `DJANGO_SETTINGS_MODULE=plane.settings.production` | 管理命令入口 |
| WSGI | apps/api/plane/wsgi.py:18 | `application = get_wsgi_application()` | 主运行时（同步） |
| ASGI | apps/api/plane/asgi.py:18 | `ProtocolTypeRouter({"http": ...})` | 仅 HTTP，无 WS consumer |
| Celery | apps/api/plane/celery.py:38 | `app = Celery("plane")` | worker/beat；autodiscover（:116） |
| 设置 | apps/api/plane/settings/common.py | INSTALLED_APPS:97 / MIDDLEWARE:121 / REST_FRAMEWORK:138 | 基础配置 |

## 根 URL 表面（plane/urls.py:17-24）

| 前缀 | include | 模块风格 | 默认鉴权 |
| -- | -- | -- | -- |
| `/api/` | plane.app.urls | DRF ViewSets（19 组） | Session（关 CSRF） |
| `/api/v1/` | plane.api.urls | APIView（12 组，含 lite） | API Key + 限流 |
| `/api/public/` | plane.space.urls | 公开看板（4 组） | AllowAny + DeployBoard 门控 |
| `/api/instances/` | plane.license.urls | 实例管理 | 管理员 |
| `/auth/` | plane.authentication.urls | credentials/magic/OAuth/密码 | — |
| `/` | plane.web.urls | SPA 兜底 | — |

> 完整 395 条路由见 `backend-url-index.txt`。

## 前端入口

| 应用 | 路由配置 | 渲染 | base path | provider 链 |
| -- | -- | -- | -- | -- |
| web | apps/web/app/routes.ts:17（mergeRoutes） | CSR (ssr:false) | `/`（VITE_*） | AppProvider: provider.tsx:36 |
| space | apps/space/app/routes.ts:10 | SSR (ssr:true) | VITE_SPACE_BASE_PATH | AppProviders: providers.tsx:15 |
| admin | apps/admin/app/routes.ts:10 | CSR (ssr:false) | VITE_ADMIN_BASE_PATH | CoreProviders: providers/core.tsx:25 |

## 实时服务入口（apps/live）

| 文件 | 符号 | 说明 |
| -- | -- | -- |
| apps/live/src/start.ts:13 | `startServer` | 进程入口 + 信号处理 |
| apps/live/src/server.ts:27 | `Server` | Express+express-ws 装配 |
| apps/live/src/hocuspocus.ts:38 | `HocusPocusServerManager` | Hocuspocus 单例（debounce 10000） |
| apps/live/src/extensions/index.ts:13 | `getExtensions` | [Logger,Database,Redis,TitleSync,ForceClose] |
| apps/live/src/controllers/index.ts:12 | controllers | Collaboration/Document/Health/PdfExport |

## 部署进程（docker-compose.yml）

proxy(Caddy) → {web, admin, space, api, live}；api + worker + beat-worker + migrator（同镜像不同入口）；plane-db(PG15.7) / plane-redis(Valkey7.2.11) / plane-mq(RabbitMQ3.13.6) / plane-minio。
