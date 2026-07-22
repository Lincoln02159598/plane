# FRONTEND_GUIDELINES.md — 前端开发与 UI 架构视角

## 分析快照

- 分支：`preview`；HEAD：`7cef741c2`；工作区：clean（仅新增 `./docs-analysis/`）；子模块：无。
- 分析范围：`apps/{web,space,admin}` + `packages/*` 静态只读；未运行 build/lint。
- 未覆盖范围：私有 EE 仓库填充的 `extendedRoutes`/`Base*Store`/`ee/` 内容；动态运行行为。

## 证据分类与“规范四层”说明

本文档严格区分：
1. **[Evidence-规范]**：源码中明确存在并实际执行的约定（给出 `路径:行号`）。
2. **[Inference-推断]**：可从重复代码模式推断的约定。
3. **[无证据]**：仓库中不存在证据的“一般最佳实践”（明确标注为缺失）。
4. **[Recommendation-建议]**：针对已确认问题提出的未来规范。

## 核心结论

[Evidence] 前端为 **3 个 React Router 7 应用 + 15 个共享包**。web/admin 纯 CSR（nginx SPA），space SSR（react-router-serve）。状态以 **MobX**（每应用独立 root store）为主、**SWR** 做数据缓存。UI 处于从旧库 `@plane/ui` 迁移到新设计系统 `@plane/propel` 的中间态。编辑器为 `@plane/editor`（TipTap + Yjs/Hocuspocus）。i18n 19 语言 28 命名空间。统一 design token 在 `@plane/tailwind-config/variables.css`。

---

## 1. 前端入口 / 页面 / 路由 / 布局

- **配置式路由**（`@react-router/dev/routes`，非文件约定）。
  - web：`apps/web/app/routes.ts:17`（`mergeRoutes(coreRoutes, extendedRoutes)` + catch-all `*`）；真实路由树 `app/routes/core.ts`（~405 行）：`(home)` 登录、`(all)` 下 `accounts/*、sign-up、onboarding、invitations`，`[workspaceSlug]` 分 `(projects)`/`(settings)`。`apps/web/app/routes/extended.tsx` 为**空数组**（EE 注入缝）。
  - space：`apps/space/app/routes.ts:10`（`index`、`:workspaceSlug/:projectId` 重定向、`issues/:anchor`、`*`）。
  - admin：`apps/admin/app/routes.ts:10`（`(all)/(home)` + `(all)/(dashboard)` 下 general/workspace/email/authentication/{github,gitlab,google,gitea}/ai/image）。
- 根：`app/root.tsx`（RR7 `Layout`/`meta`/`ErrorBoundary`/`HydrateFallback` + `ThemeProvider`）。
- [Evidence-规范] basename/base path 经 `react-router.config.ts` 的 `basename` + vite `base`，来自 `VITE_*_BASE_PATH`。

## 2. 组件层级与目录组织

- web：`@/*` → `./core/*`（`apps/web/tsconfig.json:5`）。
  - `app/`：**仅路由**（routes.ts/root.tsx/entry.client.tsx/(home) 页面与 layout/compat 兼容层）。
  - `core/`：**全部应用代码**——`components/`（~1170 `.tsx`，~58 特性目录，`issues` 占 293）、`hooks/`（store barrel + ~60 hooks）、`layouts/`、`lib/`（store-context、wrappers、app-rail、b-progress）、`services/`（本地 API 层）、`store/`（MobX）。
- space/admin：各自 `app/`（路由）+ `components/`、`store/`、`lib/`、`hooks/`、`helpers/`。
- [Inference-推断] 组件按“特性域”组织（issues/cycles/modules/views/pages…），非按“类型”（atoms/molecules）。

## 3. 状态管理（MobX）

- web：`apps/web/core/lib/store-context.tsx:12` 模块单例 `rootStore = new RootStore()` + `StoreContext`；`apps/web/core/store/root.store.ts:75` `CoreRootStore` 组装 ~30 子 store（workspaceRoot/projectRoot/cycle/module/issue/state/label/dashboard/analytics/projectPages/projectInbox/favorite/sticky/editorAsset/workItemFilters/powerK/timeline/commandPalette/router…）。
- 模式：每 store `makeObservable`，在构造器内实例化自己的 service（如 `store/workspace/index.ts:116` `this.workspaceService = new WorkspaceService()`），通过 `_rootStore` 跨 store 访问；组件经 store hooks（`core/hooks/store/use-*.ts`）+ `mobx-react` `observer` 消费。
- space：`store/root.store.ts:34` `RootStore`（11 store，含 `hydrate()` SSR）。admin：`store/root.store.ts:20` `RootStore`（4 store：theme/instance/user/workspace）。
- [Evidence-规范] `resetOnSignOut()`/`reset()` 重新实例化全部 store。
- [Inference-推断] `Base*Store` 命名（BaseWorkspaceRootStore…）是 **CE 扩展缝**，EE 覆写。`@plane/shared-state` 目前仅导出 `rich-filters` + `work-item-filters`；其内 `user.store.ts`/`workspace.store.ts` 为**未导出的空壳**（store 抽取迁移刚开始）。

## 4. 数据获取 / API 封装

- **两套并行服务层**：
  - 本地：`apps/web/core/services/api.service.ts:11` `APIService`（axios `{baseURL, withCredentials:true}` + **401 拦截器**重定向 `/?next_path=`，`:25-36`）；web 用 ~46 个本地 service 类。
  - 共享：`packages/services/src/api.service.ts:14` `APIService`（同配置但**无拦截器**）；space(20 import)/admin(8) 使用；web 仅用 6 处（`APITokenService` + file helper）。
- `API_BASE_URL = process.env.VITE_API_BASE_URL || ""`（`packages/constants/src/endpoints.ts:7`）。
- 鉴权：cookie + CSRF（`AuthService.requestCSRFToken` GET `/auth/get-csrf-token/`，`packages/services/src/auth/...`）；`signOut()` 构造 `<form>` POST `/auth/sign-out/`（`apps/web/core/services/auth.service.ts:62`）。
- **SWR 与 MobX 并用**：web 142 处；`WEB_SWR_CONFIG`（`packages/constants/src/swr.ts:16`，revalidateOnFocus/mount，errorRetryCount:3）在 `AppProvider` 包裹；典型 `useSWR("USER_INFORMATION", …)` 驱动 MobX action（`apps/web/core/lib/wrappers/authentication-wrapper.tsx:45`）。

## 5. 表单 / 校验 / 错误处理 / 加载 / 空状态

- 表单：`react-hook-form 7.51.5`（`pnpm-workspace.yaml:151`）。
- [Evidence-规范] 认证错误码集中在 `apps/web/helpers/authentication.helper.tsx`（`EPageTypes`、`EAuthenticationErrorCodes` 5000–5900），但**错误消息硬编码英文**（`:112` TODO 待国际化）。
- [Inference-推断] 加载/空状态组件由 `@plane/propel` 的 `empty-state` 与应用内 spinner 提供（无统一 HOC 证据）。
- [无证据] 未见统一的全局错误边界 HOC（除 RR7 `ErrorBoundary`）。

## 6. 样式系统 / Design Token / 主题

- **[Evidence-规范] 统一 token 系统存在且健康**：`packages/tailwind-config/variables.css`（1309 行）。`:root` 原始色阶（`--neutral-*`/`--brand-*`）+ 语义 token（`--bg-*`/`--txt-*`/`--border-*`/`--label-*`）+ 三个 `@theme`/`@theme inline` 块（行 692/727/979）映射到 Tailwind 命名空间（生成 `bg-canvas`/`text-primary`/`border-subtle`）。
- **Tailwind 4.1.17 CSS-first**，全仓**无 `tailwind.config.js`**（`@import "tailwindcss"` + `@source`）。三应用 `styles/globals.css` 均 `@import "@plane/tailwind-config/index.css"` 后叠加本地 CSS，**不重定义 token**。
- 暗色：`@custom-variant dark (&:where([data-theme*="dark"],…))`（`variables.css:1`），属性式，配合 `next-themes`（`ThemeProvider`）。
- 主题粒度分歧：web 5 主题（含高对比/自定义），space/admin 仅 light/dark → 高对比 token 仅 web 可达。
- [Evidence] 死配置：`packages/tailwind-config/package.json:10` `main` 指向不存在的 `tailwind.config.js`。

## 7. 响应式 / 移动适配 / 可访问性

- [Evidence-规范] 拖拽用 `@atlaskit/pragmatic-drag-and-drop`（`pnpm-workspace.yaml:8-10`）；虚拟列表 `@tanstack/react-virtual`。
- [无证据] 未见统一的 a11y 策略文档；`jsx-a11y` 在 oxlint 插件中（`.oxlintrc.json:3`）为 warn 级。
- [无证据] 未见专门的移动端适配层（响应式依赖 Tailwind 断点，非独立移动实现）。

## 8. 国际化（i18n）

- `@plane/i18n`：i18next 25.10.9 + react-i18next + **i18next-icu 2.4.3**（ICU 复数）+ `i18next-resources-to-backend`（按 `(lang,ns)` 懒加载）。`src/core/instance.ts:16`（`fallbackLng:"en"`、`defaultNS:"common"`、28 命名空间）。
- **19 语言**（`packages/i18n/src/locales/`：en/fr/es/ja/zh-CN/zh-TW/ru/it/cs/sk/de/ua/pl/ko/pt-BR/id/ro/vi-VN/tr-TR）。
- 公共 API：`TranslationProvider`/`useTranslation`/`setLanguage`/`FALLBACK_LANGUAGE`/`SUPPORTED_LANGUAGES`。
- [Evidence] 消费：web 481 处、space 6 处、**admin 0（完全未国际化）**。
- [Evidence] `TTranslationKeys` 已由 `en` 生成，但 `t()` 仍 `(key: string)`（类型安全为“Phase 2”）；`coerceToString` 防 ICU `returnObjects` 崩溃（`hooks/use-translation.ts:25-37`）。
- 语言代码不一致：`xx`/`xx-UC`/`tr-TR` 混用；乌克兰语用 `ua` 而非 CLDR `uk`。

## 9. 图标 / 静态资源

- [Evidence-规范] 图标：`@plane/propel` 的 `ICON_REGISTRY` 点名映射（`src/icons/registry.ts:86-180`）；同时 `@fontsource/material-symbols-rounded`。lucide-react 0.469。
- 字体：`@fontsource*/inter`、`ibm-plex-mono`。

## 10. 前端测试 / 构建

- **[Evidence] 前端无自动化测试**：web/space/admin 无 `*.test.*`/`*.spec.*`，无 vitest 依赖。仅设计系统有 Storybook（propel 39、ui 6）。
- 构建：`turbo prune --scope=<app> --docker` + pnpm；web/admin → nginx SPA（`build/client`）；space → SSR（`react-router-serve`，存在 `nginx/nginx.conf` 但**生产未使用**）；live → tsdown Node。
- 类型检查：`react-router typegen && tsc --noEmit`；lint/format：OxLint/Oxfmt。详见 `测试与CI.md`。

## 11. 桌面/移动/浏览器差异

[Evidence] 无桌面/移动原生应用。三前端均为 Web。web/admin CSR（浏览器），space SSR（Node）。

## 12. 代码组织约定

- import：内部包 `workspace:*`，外部 `catalog:`（`AGENTS.md:16`）。
- 命名：camelCase 变量/函数，PascalCase 组件/类型（`AGENTS.md:20`）。
- [Evidence] `compat/next/` 兼容垫片（`next/link`/`next/navigation`/`next/script`）形状在三应用间不一致。

## 13. UI 技术债务 / 前端安全边界

- **UI 库迁移半途**：`@plane/ui`（HeadlessUI/Blueprint/Popper，字符串式 button 变体，仅 6 stories，内部 `dropdown`/`dropdowns` 重复）正被 `@plane/propel`（Base UI + cva + recharts，39 stories，token 驱动）取代；`ui` 反向依赖 `propel`。两套 `Button`/`Tooltip`/Icon 并存，均 `displayName="plane-ui-button"`。消费已反转：propel 751 文件 vs ui 521。
- **服务层迁移半途**：web 仍用 46 本地 service；共享基类缺 401 拦截器。`@plane/services` 的 `module/module.service.ts` 等缺构造器（`new ModuleService()` → `baseURL:undefined`），且 0 消费者。
- **Next.js 残留**：`compat/next/`、死代码 `apps/web/app/layout.tsx`、web `useRouter().push` 用 setTimeout + 强制尾斜杠。
- **store 抽取迁移刚开始**：`@plane/shared-state` 的 user/workspace store 为未导出空壳。
- 安全边界：鉴权靠组件包装（`AuthenticationWrapper`）而非 RR loader；CRF/cookie 由后端主导；前端不存敏感凭据（`VITE_*` 仅公开变量）。

---

## 已确认事实

- 3 前端应用：web/admin CSR、space SSR；配置式路由 + EE 注入缝（`extendedRoutes` 空）。
- MobX（每应用独立 root）+ SWR；两套服务层；统一 token 在 `variables.css`；Tailwind 4 CSS-first 无 config.js。
- 19 语言；admin 完全未国际化；UI 库 ui→propel 迁移中；前端无测试。

## 合理推断

- `Base*Store`/`extendedRoutes`/`ee/` 由私有 EE 仓库填充。
- 组件按特性域组织；加载/空状态无统一 HOC（靠 propel + 应用内组件）。

## Unknown 与待验证事项

- EE 扩展的实际内容；移动端无专门层是否为有意决策。

## 批判性评估

- 多项迁移半途（服务层、UI 库、store 抽取、Next.js→RR）叠加，重复与漂移显著；前端零自动化测试放大回归风险。
- 认证错误消息未国际化、admin 完全脱出 i18n/editor，是用户体验一致性的缺口。

## 建设性改善建议

- [Recommendation] 完成服务层迁移并补 401 拦截器；移除 0 消费者且构造器缺失的 stub service。优先级：中；难度：中。
- [Recommendation] 收敛 UI 库：弃 `@plane/ui`，全量 `@plane/propel`，统一 Button/Tooltip/Icon。优先级：中；难度：中。
- [Recommendation] 补前端关键路径测试（至少 web 的 issue/cycle/module 流程 + 鉴权包装），并接入 CI。优先级：高；难度：中。
- [Recommendation] 将认证错误消息迁入 i18n，admin 接入 i18n。优先级：低；难度：低。
- [Recommendation] 清理 Next.js 残留（`compat/next/` 统一、删死代码 layout）。优先级：低；难度：低。

## 主要证据索引

- `apps/web/react-router.config.ts:6`、`apps/web/app/routes.ts:17`、`apps/web/app/routes/extended.tsx`、`apps/web/app/root.tsx:79`、`apps/web/app/provider.tsx:36`
- `apps/web/core/store/root.store.ts:75`、`apps/web/core/lib/store-context.tsx:12`、`apps/web/core/services/api.service.ts:11-36`
- `apps/space/react-router.config.ts:11`、`apps/space/store/root.store.ts:34`
- `apps/admin/react-router.config.ts:13`、`apps/admin/store/root.store.ts:20`、`apps/admin/providers/core.tsx:25`
- `packages/services/src/api.service.ts:14`、`packages/constants/src/endpoints.ts:7`、`packages/constants/src/swr.ts:16`
- `packages/tailwind-config/variables.css`（token 系统）、`packages/tailwind-config/package.json:10`
- `packages/i18n/src/core/instance.ts:16`、`packages/i18n/src/locales/`（19 语言）
- `packages/propel/src/button/helper.tsx:10` vs `packages/ui/src/button/helper.tsx:7`
