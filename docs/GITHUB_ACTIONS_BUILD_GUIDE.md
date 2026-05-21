# GitHub Actions Windows 构建指南

本指南介绍如何在你的 GitHub 仓库中使用 Actions 自动构建 OpenHuman 的 Windows 安装包。

## 🚀 快速开始

### 方法一：手动触发构建（推荐）

1. 访问你的 GitHub 仓库页面
2. 点击顶部的 **Actions** 标签页
3. 选择 **Build Windows** 工作流
4. 点击右侧的 **Run workflow** 按钮
5. 选择 `main` 分支（或你想要构建的分支）
6. 点击绿色的 **Run workflow** 按钮

### 方法二：推送代码自动触发

向以下分支推送代码会自动触发构建：
- `main`
- `dev`
- `fix/windows`

### 方法三：创建 Pull Request 时自动构建

向 `main` 分支发起 PR 时会自动运行构建检查。

## 📦 构建产物

构建完成后，你可以在 Actions 运行页面下载以下产物：

| 产物名称 | 描述 |
|---------|------|
| `windows-msi` | Windows MSI 安装包 |
| `windows-nsis` | Windows NSIS (.exe) 安装包 |
| `windows-cli` | 独立的 CLI 二进制文件 |

### 下载步骤：

1. 在 Actions 中找到已完成的构建
2. 向下滚动到 **Artifacts** 部分
3. 点击需要的文件下载
4. 解压 zip 即可使用

## ⚙️ 工作流配置

### 主要特性

- ✅ 自动缓存 Rust 和 CEF 依赖（节省构建时间）
- ✅ 使用 Rust 1.93.0
- ✅ 使用 Node.js 24.x
- ✅ 构建 MSI 和 NSIS 两种格式
- ✅ 自动清理和取消并发运行

### 工作流程

```
Checkout → Setup Node.js → Setup Rust → Install Dependencies 
→ Build Tauri App → Upload Artifacts
```

## 🔧 自定义配置

### 修改触发分支

编辑 `.github/workflows/build-windows.yml`：

```yaml
on:
  workflow_dispatch:
  push:
    branches: [main, your-branch, another-branch]  # 添加你的分支
```

### 环境变量

你可以在仓库的 **Settings → Secrets and variables → Actions** 中配置以下变量：

| 变量名 | 描述 |
|-------|------|
| `BASE_URL` | 后端 API 地址 |
| `VITE_SENTRY_DSN` | Sentry DSN（错误监控） |
| `VITE_DEBUG` | 是否启用调试模式 |

## 📊 构建时间

- **首次构建**：约 30-45 分钟（需要下载所有依赖）
- **有缓存的构建**：约 10-20 分钟
- **代码变更后的增量构建**：约 5-15 分钟

## 🐛 常见问题

### Q: 构建失败怎么办？

A: 点击进入失败的 Actions 运行，查看详细的构建日志，定位问题。

### Q: 如何在本地模拟 Actions 构建？

A: 参考 [BUILD_WINDOWS_11_GUIDE.md](./BUILD_WINDOWS_11_GUIDE.md) 进行本地构建。

### Q: 我想构建其他平台？

A: 当前工作流只支持 Windows。如果需要多平台构建，参考 `build-desktop.yml` 工作流。

### Q: 构建产物可以在 GitHub Releases 发布吗？

A: 可以！参考项目中的 `release-staging.yml` 和 `release-production.yml` 工作流。

## 📚 相关文档

- [官方 Tauri 文档](https://tauri.app/)
- [GitHub Actions 文档](https://docs.github.com/actions)
- [本地 Windows 构建指南](./BUILD_WINDOWS_11_GUIDE.zh.md)
