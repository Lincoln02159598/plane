# 仓库只读基线快照

> 本文件在 Phase 0 建立，用于区分「任务开始前已存在的修改」与「本任务新增的 `./docs-analysis/` 文件」。

## 分析时点

- 分析时间：2026-07-23（基于会话当前日期）
- 分析对应的 HEAD commit：`7cef741c29cf61d3bca18dc892e6af11a1e7becc`
- HEAD commit subject：`feat(api): add lite list endpoints for projects, members, cycles, and modules (#9410)`
- 当前分支：`preview`
- 主分支（默认用于 PR）：`preview`

## 工作区状态（任务开始前）

- 工作区修改状态：**clean（干净）**
  - `git status --short` 输出为空
  - `git status --porcelain | wc -l` = `0`
- 未跟踪文件数量：`0`（`git ls-files --others --exclude-standard | wc -l`）
- 结论：**任务开始前不存在任何用户原有的未提交修改。** 因此最终安全检查只需确认：除 `./docs-analysis/` 外不应出现新增/修改。

## 子模块状态

- `git submodule status --recursive` 输出为空。
- 仓库根目录**不存在 `.gitmodules` 文件**。
- 结论：**本仓库不使用 Git 子模块。** 所有「子模块相关」的文档章节将标注为「不适用 / 未发现」。

## 仓库类型

- 是否为 monorepo：**是**
- 工作区管理工具：pnpm workspaces（`pnpm-workspace.yaml`）+ Turborepo（`turbo.json`）
- 跟踪文件总数：`git ls-files | wc -l` = **5250**

## 顶层目录（按 git 跟踪文件数量）

| 目录 | 跟踪文件数 | 说明 |
| -- | -- | -- |
| apps/ | 3442 | 应用：admin, api, live, proxy, space, web |
| packages/ | 1736 | 共享包：codemods, constants, decorators, editor, hooks, i18n, logger, propel, services, shared-state, tailwind-config, types, typescript-config, ui, utils |
| deployments/ | 22 | 部署相关（Docker / Helm 等） |
| .github/ | 15 | CI/CD 工作流 |
| .claude/ | 5 | Claude Code 配置 |
| docs/ | 1 | 仓库自带文档 |
| .idx/ | 1 | Google IDX 配置 |
| .husky/ | 1 | Git hooks |

## 写入边界确认

- 本次任务的唯一写入目录：`./docs-analysis/`
- 允许写入路径：
  - `./docs-analysis/*.md`
  - `./docs-analysis/scripts/*`
  - `./docs-analysis/evidence/*`
- 除上述路径外，禁止修改任何源码、配置、测试、CI、migration、锁文件、数据库、子模块、构建产物、IDE/Git 配置。
- 仅使用只读命令（pwd/ls/find/rg/grep/git 只读子命令/sed/awk/head/tail/cat/wc/file/tree 等）。
- **不运行** install/build/test/lint/format/codegen/migration/dev server/watch/container build/publish/release/dependency update/database 命令。
