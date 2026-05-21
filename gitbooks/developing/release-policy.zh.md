---
description: 发布周期、版本策略、OAuth 和安装包规则。讲解发布流程。
icon: ship
---

# 发布策略：最新桌面构建和 OAuth

本手册描述了如何在用户使用**过时的桌面安装包**完成 **OAuth**（包括 **Gmail**）时，引导用户使用**最新**版本。

## 分发渠道

- **[tinyhumansai/openhuman](https://github.com/tinyhumansai/openhuman/releases) 的 GitHub Releases** 是桌面构建的主要来源。
- **Tauri 更新器**端点（见 `scripts/prepareTauriConfig.js` 和发布工作流）应指向用户当前的发布产物。
- **淘汰旧版稳定构建**：当停止某个发布线时，移除或隐藏 **GitHub Releases** 上的过时安装包资产，更新 **网站/CDN** 下载链接指向 **releases/latest**（或当前版本），刷新**更新器清单**（如 Gist / `latest.json`），确保不指向已废弃的构建，并抽查确认旧的直接链接已**重定向、返回 404 或 410**。验证方式：尝试访问文档或书签中的旧版资产 URL，确认它们不再提供主要安装路径。

## OAuth 的最低应用版本要求

生产环境 Web 构建会在**构建时**嵌入**最低支持的应用程序语义化版本**，以防止用户在已弃用的二进制文件上完成 OAuth 深度链接。每个安装包都携带构建时的版本下限；如果用户从不升级，抬高下限需要他们安装**新**版本（或应用内更新）。可选的未来方案：通过**运行时 API** 实施动态下限，但仅将捆绑值作为后备。

| 变量 | 用途 |
| ---- | ---- |
| `VITE_MINIMUM_SUPPORTED_APP_VERSION` | 例如 `0.51.0` - 桌面应用必须**≥**此版本才能完成 `openhuman://oauth/success`。|
| `VITE_LATEST_APP_DOWNLOAD_URL` | 可选；默认为 `https://github.com/tinyhumansai/openhuman/releases/latest`。当版本门控阻止时打开此链接。|

将以上变量配置为 **GitHub Actions variables**。它们必须同时存在于**独立的 `pnpm build` 步骤**和 **`.github/workflows/build-desktop.yml`** 中的 **`tauri-apps/tauri-action`** 步骤环境变量中（该文件是被 `release-production.yml` / `release-staging.yml` 和 `build-windows.yml` 调用的工作流矩阵），以确保发布安装包中嵌入的 Vite 捆绑包包含版本门控。本地开发时**不要设置** `VITE_MINIMUM_SUPPORTED_APP_VERSION`（门控禁用）。

实现位置：`app/src/utils/oauthAppVersionGate.ts`、`app/src/utils/desktopDeepLinkListener.ts`。

## Gmail / Google Cloud OAuth

- Google Cloud Console 中的**重定向 URI** 必须与**当前**后端和隧道回调路径匹配。
- 桌面端 scheme（`openhuman://`）保持稳定；当设置了 `VITE_MINIMUM_SUPPORTED_APP_VERSION` 时，**已安装的二进制文件**必须满足最低版本要求。

## 发布检查清单（避免回归）

1. 根据现有版本工作流提升 `app/package.json`、`app/src-tauri/tauri.conf.json`（以及根目录 `Cargo.toml` / core）的版本号。
2. 当停止支持旧版安装时，在**发布前**或**与发布同时**将 **`VITE_MINIMUM_SUPPORTED_APP_VERSION`** 设置为新的版本下限（仓库 Actions variables + 上述两个工作流步骤）。
3. 从面向用户的界面移除、重定向或淘汰旧版稳定安装包和过时的**更新器**条目（GitHub Release 资产、网站、CDN、更新器源）。确认废弃的资产无法从默认安装/更新流程中访问。
4. 从 **releases/latest** 全新安装后，**烟雾测试 Gmail 连接**。
5. 完成[手动烟雾检查清单](../../docs/RELEASE-MANUAL-SMOKE.md)，然后在发布 PR 描述中逐字粘贴已完成的签收块（每个检查项保持勾选状态）。

## 工作流：预发布 vs 生产环境

两个一线 GitHub Actions 工作流，分别对应不同环境。根据意图选择，而非切换标志。

| 工作流 | 分支 | 版本提升 | 推送标签 | 并发组 | 适用场景 |
| ------ | ---- | -------- | -------- | ------ | -------- |
| [`release-staging.yml`](../../.github/workflows/release-staging.yml) | `main` | 仅 `patch` | `v<version>-staging` | `release-staging` | 切预发布构建供 QA 测试。频繁运行；小范围语义化版本更新。|
| [`release-production.yml`](../../.github/workflows/release-production.yml) | `main` | `patch` / `minor` / `major`（仅在 `main_head`） | `v<version>` | `release-production` | 提升已验证的预发布标签，或从 `main` HEAD 热修复。|

两个流程共用的矩阵构建/签名/Sentry-DIF/产物上传管道位于 [`.github/workflows/build-desktop.yml`](../../.github/workflows/build-desktop.yml) 中作为 `workflow_call` 可复用工作流。上述两个顶层工作流负责引用解析、版本提升、标签推送和发布/清理；构建本身是共享的。

### 切预发布构建

1. 从 `main` 通过 `workflow_dispatch` 运行 **Release (Staging)**。
2. 工作流在 `main` 上提升 `patch` 版本，提交 `chore(staging): vX.Y.Z`，推送分支，并在该提交处创建不可变的 `vX.Y.Z-staging` 标签。
3. 构建矩阵从**标签**（而非 main HEAD）运行，因此即使 `main` 继续演进，重新运行也会构建出字节完全相同的内容。
4. 失败时预发布标签自动删除；`main` 上的版本提升提交保留，以便下一次切预发布从 vX.Y.(Z+1) 继续。

没有独立的 `staging` 分支，预发布切分和生产提升都在 `main` 上进行。两者仅通过标签后缀（`-staging` vs 无）和触发的工作流来区分。

### 提升到生产环境（默认流程）

1. 通过 `workflow_dispatch` 运行 **Release Production**，设置 `release_source = staging_tag`（默认）。
2. 留空 `staging_tag` 提升最新的 `v*-staging`，或传入明确标签（如 `v1.2.4-staging`）以固定版本。
3. 工作流去除 `-staging` 后缀，在相同提交处创建 `v<version>`，并从该标签运行生产构建矩阵。**不再进行版本提升**，产物复用预发布已验证的内容。

### 从 `main` HEAD 热修复

1. 通过 `workflow_dispatch` 运行 **Release Production**，设置 `release_source = main_head` 和期望的 `release_type`（`patch` / `minor` / `major`）。
2. 工作流执行传统版本提升和标签路径：在 `main` 上提升版本，提交 `chore(release): vX.Y.Z`，推送，标签 `vX.Y.Z`，构建。
3. 仅在需要生产环境修复而不经过预发布时使用此方式。

### 标签策略和回滚

- **命名规范。** 预发布标签使用 SemVer 预发布后缀 `-staging`（`v1.2.4-staging`），这样在排序时会排在对应对应生产标签之前。提升到生产时原样去除后缀；捆绑在发布安装包中的版本在两个标签间保持一致。
- **冲突处理。** 如果目标标签在本地或 `origin` 已存在，两个工作流都会快速失败。通过删除过时标签（仅组织维护者）或跳过该版本号来解决。
- **回滚（生产环境）。** 构建矩阵失败会触发 `cleanup-failed-release`，删除草稿 GitHub Release 和 `v<version>` 标签。从中提升的预发布标签保持不变，修复后可重新提升。
- **回滚（预发布）。** 预发布构建失败会删除 `v<version>-staging` 标签。`main` 上的版本提升提交保留；下一次预发布切分会从新的 patch 版本号继续，而非重用原版本号（我们接受 patch 版本号出现小间隙，以避免并发合并时的竞争）。
- **谁能删除标签。** 与 `main` 相同的写入权限。工作流驱动的删除使用工作流的 token（通过 `actions/github-script`）；版本提升提交和标签推送仅使用 GitHub App token；手动删除（`git push --delete origin <tag>`）需要等效的维护者权限。
