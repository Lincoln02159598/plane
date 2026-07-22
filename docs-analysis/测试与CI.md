# 测试与CI.md — 测试体系与 CI/CD 分析

## 分析快照

- 分支：`preview`；HEAD：`7cef741c2`；工作区：clean（仅新增 `./docs-analysis/`）；子模块：无。
- 分析范围：`.github/workflows/`、测试源码与配置、Docker 测试栈、部署发布流程的**静态只读**分析；未运行测试。
- 未覆盖范围：外部/EE 扫描流水线（如 Trivy 实际执行处）；私有仓库 CI。

## 证据分类

- 严格区分四层：**① 测试文件存在 ② 测试命令已配置 ③ 被 CI 执行 ④ 被跳过/禁用**。
- **Evidence**：`路径:行号` + 配置键；**Inference**：跨配置推导；**Unknown**：静态无法确认。

## 核心结论（最重要）

[Evidence] **CI 当前不执行任何测试套件**——既不跑后端 362 个 pytest 用例，也不跑前端/实时 vitest。`.github/workflows/` 中无任何 `pytest`/`vitest`/`turbo run test` 调用（grep 为空）。回归防护完全依赖人工 review + 格式/lint/类型/构建门禁 + CodeQL 安全扫描。`turbo.json:88-91` 定义了 `test` 任务，但**无 workflow 引用**。

---

## 1. CI/CD workflow 清单（`.github/workflows/`）

| 文件 | 触发 | 作用 | 产出 | Evidence |
| -- | -- | -- | -- | -- |
| `build-branch.yml` | push 到 `preview`/`canary` + dispatch | 构建/推送 7 个镜像（admin/web/space/live/backend/proxy/aio），Release 路径建 GH Release + AIO 镜像 | Docker Hub 镜像 + GH release | `build-branch.yml:33-36,271-410` |
| `pull-request-build-lint-api.yml` | PR 到 `preview`（`apps/api/**`）+ dispatch | **仅 `ruff check --fix apps/api`**（lint，无 pytest/build/test） | 无 | `pull-request-build-lint-api.yml:22,42` |
| `pull-request-build-lint-web-apps.yml` | PR 到 `preview` + dispatch | 4 job：`check:format`、`build`、`check:lint`、`check:types`（均 `--affected`） | 无（仅缓存） | `pull-request-build-lint-web-apps.yml:19,61,119,161` |
| `feature-deployment.yml` | dispatch only | 构建并部署 feature-preview 到 k8s（引用 `./aio/Dockerfile-app`，CE 无此目录） | feature 镜像 + 临时 namespace | `feature-deployment.yml:95` |
| `check-version.yml` | PR 到 `master` | 版本未变则失败（默认分支为 `preview`，可能从不触发） | 无 | `check-version.yml:4-6,32-40` |
| `codeql.yml` | push/PR 到 `preview`/`canary`/`master` + dispatch | CodeQL 分析（python+javascript 矩阵） | 安全告警 | `codeql.yml:5-8,22-25` |
| `copyright-check.yml` | PR 到 `preview` + dispatch | 校验版权头（`addlicense`） | 无 | `copyright-check.yml:3-13,33-45` |
| `i18n-sync-check.yml` | PR/push 到 `preview`（`packages/i18n/**`） | `sync-check.ts --ci` 校验语言键同步 | 无 | `i18n-sync-check.yml:51` |
| `react-doctor.yml` | PR + push 到 `main` | react-doctor 扫描 + PR 评论/状态 | 状态 + 评论 | `react-doctor.yml:9-18,46-63` |

- `CODEOWNERS:1-7`：映射各 `apps/*`、`deployments/`、lint 配置到维护者。
- `.husky/pre-commit` = `pnpm lint-staged`（`oxfmt` + `oxlint --fix --deny-warnings`）——**`--deny-warnings` 仅在本地强制，CI 未强制**。
- `.trivyignore:4-10`：抑制 `CVE-2026-30242`；**本仓库无 workflow 运行 Trivy**（消费方为外部/EE）。

### CI 关键缺口（证据）

- **后端 pytest 从不在 CI 执行**；前端/实时 vitest 同样不在。
- `check:lint/types/format` 与 API `ruff` **仅 PR**（且 gated `draft==false && requested_reviewers != null`，`pull-request-build-lint-web-apps.yml:23-25` 等）→ **直接 push 到 `preview`（默认分支）绕过所有质量门禁**。
- `feature-deployment.yml:95` 引用 CE 不存在的 `./aio/Dockerfile-app`（该 workflow 在 CE 无法成功）。
- `check-version.yml`/`react-doctor.yml` 触发分支为 `master`/`main`，与默认分支 `preview` 不符，可能失效。

## 2. 后端测试（apps/api）

**布局**：真实测试在 `apps/api/plane/tests/`（Django app，`plane/settings/test.py:14-16`）；`apps/api/tests/` **仅含 `RUNNING_TESTS.md`**（无 `.py`）。

**四层状态**：① 存在 ✅（48 `test_*.py`，362 测试函数）② 命令配置 ✅（`pytest.ini`、`docker-compose-test.yml`、`run_tests.py`）③ CI 执行 ❌ ④ 跳过/禁用 ❌（无 skip/xfail，但对 CI 而言“dormant”）。

**分类**（`apps/api/plane/tests/`）：`smoke/`（1）、`unit/{models,serializers,settings,views,bg_tasks,middleware,utils}`（23）、`contract/api`（12，`/api/v1/`，`api_key_client`）、`contract/app`（12，`/api/`，`session_client`）。
标记使用：`@pytest.mark.django_db` ×184、`unit` ×53、`contract` ×37、`smoke` ×2、`parametrize` ×20；**无 skip/xfail**。

**配置/运行**：
- `pytest.ini:1-17`：`DJANGO_SETTINGS_MODULE=plane.settings.test`；markers `unit/contract/smoke/slow`；`addopts = --strict-markers --reuse-db --nomigrations -vs`（`slow` 声明但未用）。
- `docker-compose-test.yml`：`api-tests` 服务（`Dockerfile.dev`），运行期注入 `requirements/test.txt`，跑 `pytest`；依赖 tmpfs 的 test-db/redis/mq/minio（健康门控）。
- `run_tests.py`：支持 `-u/-c/-s`、xdist、`--cov=plane` + `coverage report --fail-under=90`（`run_tests.py:38-39,59-65`）——**90% 阈值仅本地，未接入 CI/compose**。
- **`run_tests.sh` 损坏**：调用 `tests/run_tests.sh`（不存在，`apps/api/tests/` 仅有 markdown）——**已 first-hand 确认**。

**fixtures**：`conftest.py` 提供 `api_client`/`create_user`/`api_token`/`api_key_client`/`session_client`/`create_bot_user`/`plane_server`(=live_server)/`workspace`；`factories.py`（`UserFactory`/`WorkspaceFactory`）。
**文档漂移**：`TESTING_GUIDE.md` 提及 `mock_redis`/`mock_elasticsearch`/`mock_celery` fixtures，但 `conftest.py` 中**不存在**；`RUNNING_TESTS.md`/`AGENTS.md` 指向错误的 `TESTING_GUIDE.md` 路径。

**测试依赖**（`requirements/test.txt`）：pytest 9.0.3、pytest-django 4.5.2、pytest-cov 4.1.0、pytest-xdist、pytest-mock、factory-boy、freezegun、coverage、httpx。**生产 `Dockerfile.api` 不安装这些**（仅 compose 运行期注入）。

## 3. 前端 / 实时 / 包测试（vitest）

**四层状态**：① 存在 ✅（仅 2 包）② 命令配置 ✅（`apps/live`、`packages/codemods`）③ CI 执行 ❌ ④ 跳过 ❌。

- `vitest.config.ts` 仅 2 个：`apps/live`、`packages/codemods`。
- 测试文件（穷尽）：`apps/live/tests/services/pdf-export/effect-utils.test.ts`、`apps/live/tests/lib/pdf/pdf-rendering.test.ts`、`packages/codemods/tests/function-declaration.spec.ts`、`packages/codemods/tests/remove-directives.spec.ts`。
- **web/space/admin 及其余包无测试文件**。
- `apps/live` 覆盖：`coverage.provider:"v8"`、`include src/**/*.ts`，**无 thresholds**；仅测 PDF 渲染 + effect-utils，`onAuthenticate`/Database/Redis/ForceClose/TitleSync/controllers **未测**。

## 4. 类型检查 / Lint / Format

- oxlint（`.oxlintrc.json:1-53`）：插件 `react,typescript,jsx-a11y,import,promise,unicorn,oxc`；`correctness/suspicious/perf = warn`。
- oxfmt（`.oxfmtrc.json`）：`printWidth 120`、`tabWidth 2`、`trailingComma "es5"`、Tailwind class 排序。
- TypeScript 5.8.3（catalog）；app `check:types` = `react-router typegen && tsc --noEmit`。
- react-doctor `^0.4.2`（`package.json:28`），CI 扫描。

### `--max-warnings=N` lint 预算（当前告警数即失败阈值 → lint 在 N+1 前非阻塞）

| 包/应用 | `--max-warnings` |
| -- | -- |
| `apps/web` | **11957** |
| `packages/propel` | 3605 |
| `apps/admin` | 759 |
| `apps/space` | 676 |
| `packages/editor` | 416 |
| `apps/live` | 119 |
| `packages/utils` | 38 |
| `packages/ui` | 66 |
| 其余 | 小（i18n 9, services 6, hooks 4…） |
| `packages/logger`、`packages/shared-state` | **0（真正严格）** |

[Inference] `apps/web` 的 11957 意味着 lint 仅在告警超过该值才失败——**最大应用的 lint 实际为建议性**。

## 5. 覆盖率

- 后端：`pytest-cov` + `coverage`；`--fail-under=90` 仅在 `run_tests.py`（未接入 CI/compose/`pytest.ini`）→ **dead 阈值**。
- live vitest：v8 provider，无 thresholds。
- 无任何 workflow 上传覆盖率产物。

## 6. Docker / 容器 / 部署 / 发布

- `docker-compose.yml`（完整源构建栈）、`docker-compose-local.yml`（dev：仅 infra + api/worker/beat/migrator，`Dockerfile.dev` + bind mount）、`docker-compose-test.yml`（隔离 pytest）。
- Dockerfile（prod/dev 各一）：`Dockerfile.api`（python:3.12.10-alpine，**`chmod -R 777 /code`**，`:53`）；`Dockerfile.web`（3 阶段 turbo prune → nginx）；`Dockerfile.live`（3 阶段，运行期剥离 go/picomatch(CVE-2026-33671)/esbuild/tsgolint）；`Dockerfile.ce`（Caddy + xc dav 插件）。
- 部署：AIO 单容器（`deployments/aio/community/`，supervisord 编排 migrator/space/api/worker/beat/live/proxy）；CLI Compose 安装器（`deployments/cli/community/`，Swarm 风格 replicas）；Kubernetes（外部 Helm chart）；Swarm（`swarm.sh`）。
- `setup.sh`（开发引导，**会改状态**：生成 `.env`、写 `SECRET_KEY`、`pnpm install`）——非 CI 调用。

## 7. 版本 / 发布

- 根/JS `1.3.1`；API `0.24.0`（`apps/api/pyproject.toml:3`）——**版本分歧**（`.trivyignore` 按 0.24.0 匹配 CVE）。
- Release：`build-branch.yml` dispatch `build_type=Release` + SemVer → `publish_release`（GH release 附 setup.sh/restore.sh/compose 等）。
- `pnpm-workspace.yaml:241-256`：`allowBuilds`（sharp=false）、`minimumReleaseAgeExclude`（Storybook 10.4.6 豁免最小发布龄）。

## 测试覆盖矩阵（按模块）

| 模块 | 文件存在? | 命令配置? | CI 执行? | 主要缺口 |
| -- | -- | -- | -- | -- |
| 后端 `apps/api` | ✅ 48 文件/362 fn（smoke/unit/contract） | ✅ pytest.ini+compose+run_tests.py | **❌** | 90% 阈值 dead；`run_tests.sh` 损坏；文档漂移 |
| `apps/web` | ❌ | 仅 lint/types/format | 仅 PR | 无测试；lint `--max-warnings=11957` |
| `apps/space` | ❌ | 仅 lint/types/format | 仅 PR | 无测试 |
| `apps/admin` | ❌ | 仅 lint/types/format | 仅 PR | 无测试 |
| `apps/live` | ✅ 2 文件 | ✅ `test: vitest run` | **❌** | 仅 PDF；auth/DB/redis/controllers 未测；无 thresholds |
| `packages/codemods` | ✅ 2 文件 | ✅ vitest | ❌ | 最小 |
| 其他包（ui/propel/editor/utils/i18n/hooks/services/...） | ❌（仅 Storybook） | lint/types（PR，`--affected`） | 仅 PR | 无单测；Storybook 非测试 |
| E2E/浏览器 | ❌ | ❌ | ❌ | 无 Playwright/Cypress |

---

## 已确认事实

- CI 不执行任何测试套件（后端 362 + 实时 4 用例均不跑）。
- 后端测试体系完善（48 文件、contract/unit/smoke、fixtures、dockerized），但仅手动可运行。
- 前端三应用零自动化测试；lint 预算巨大（web 11957）使其近乎建议性。
- 仅 `logger`/`shared-state` 真正严格（0 warnings）。
- 直接 push `preview` 绕过所有质量门禁。

## 合理推断

- 回归风险完全由人工 review + 静态门禁承担；在双 API/双 UI 等高重复代码下风险被放大。
- `check-version.yml`/`react-doctor.yml`/`feature-deployment.yml` 因分支或路径在 CE 可能失效。

## Unknown 与待验证事项

- `master`/`main` 分支是否存在（影响 check-version/react-doctor）；`./aio/Dockerfile-app` 是否由 EE 仓库提供；Trivy 实际执行处；CodeQL JS autobuild 覆盖度。

## 批判性评估

- “测试存在但 CI 不执行”是本仓库测试体系最关键的缺陷：362 个后端测试的价值被 CI 缺位大幅削弱。
- `chmod -R 777 /code`（Dockerfile.api:53）与 `trusted_proxies 0.0.0.0/0`（Caddyfile.ce:41）是安全 posture 上的宽松点。
- lint 预算制（`--max-warnings`）使技术债可视化但非强制。

## 建设性改善建议

- [Recommendation] **将后端 pytest（`docker-compose-test.yml`）接入 PR CI**（至少跑 unit+contract，再跑 smoke）。优先级：**高**；难度：中。
- [Recommendation] **将 `apps/live` vitest 接入 CI**，并补 onAuthenticate/Database/Redis 关键路径测试。优先级：高；难度：中。
- [Recommendation] 修复 `run_tests.sh` 与测试文档漂移；将 `--fail-under=90` 接入 CI 或移除。优先级：中；难度：低。
- [Recommendation] 在 push 到 `preview` 也运行 lint/types/format（不只 PR）；收紧 `--max-warnings` 预算或转 `--deny-warnings`。优先级：中；难度：中。
- [Recommendation] 修正失效 workflow 的触发分支；修正 `feature-deployment.yml` 对 `./aio/Dockerfile-app` 的引用。优先级：低；难度：低。
- [Recommendation] 收紧容器安全：移除 `chmod -R 777`、收紧 `trusted_proxies`。优先级：中；难度：低。

## 主要证据索引

- `.github/workflows/*.yml`（9 文件）、`CODEOWNERS`、`.trivyignore`、`.husky/pre-commit`
- `apps/api/pytest.ini:1-17`、`apps/api/run_tests.py:38-65`、`apps/api/run_tests.sh`（损坏）、`apps/api/plane/tests/conftest.py`
- `docker-compose-test.yml`、`docker-compose.yml`、`docker-compose-local.yml`
- `apps/api/Dockerfile.api:53`、`apps/live/Dockerfile.live:58-70`、`apps/proxy/Caddyfile.ce:41`
- `turbo.json:48-91`、`.oxlintrc.json`、`.oxfmtrc.json`、各 app `package.json`（`--max-warnings`）
- `apps/live/vitest.config.ts`、`packages/codemods/vitest.config.ts`
- `deployments/aio/community/{Dockerfile,supervisor.conf}`、`deployments/cli/community/install.sh`
