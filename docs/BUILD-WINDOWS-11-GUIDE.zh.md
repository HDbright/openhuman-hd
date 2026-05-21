# Windows 11 构建 OpenHuman 安装包完整指南

本文档提供在 Windows 11 系统上从源码构建 OpenHuman Windows 安装包的详细步骤。

---

## 📋 目录

- [方式一：使用官方安装脚本（推荐用于安装）](#方式一使用官方安装脚本推荐用于安装)
- [方式二：从源码构建开发版](#方式二从源码构建开发版)
- [方式三：从源码构建发布版](#方式三从源码构建发布版)
- [方式四：使用 GitHub Actions 自动构建](#方式四使用-github-actions-自动构建)
- [前置依赖详解](#前置依赖详解)
- [常见问题排查](#常见问题排查)

---

## 方式一：使用官方安装脚本（推荐用于安装）

如果你只是想**安装**最新稳定版本的 OpenHuman，而不是从源码构建，使用官方安装脚本：

### 快速安装

以管理员身份打开 **PowerShell**，运行：

```powershell
irm https://raw.githubusercontent.com/tinyhumansai/openhuman/main/scripts/install.ps1 | iex
```

### 安装器功能

官方安装脚本会：
- ✅ 自动从 GitHub Releases 获取最新稳定版本
- ✅ 验证文件 SHA256 完整性（如果提供）
- ✅ 支持 MSI 和 NSIS (.exe) 两种格式
- ✅ 提供管理员权限申请（UAC）进行系统级安装
- ✅ 自动查找安装路径

### 预览安装（不执行）

```powershell
irm https://raw.githubusercontent.com/tinyhumansai/openhuman/main/scripts/install.ps1 | iex -DryRun
```

### 手动下载安装包

如果你更喜欢手动下载：
1. 访问 [GitHub Releases Latest](https://github.com/tinyhumansai/openhuman/releases/latest)
2. 找到 Windows x64 的 MSI 或 NSIS 安装包
3. 下载并运行安装程序

---

## 方式二：从源码构建开发版

如果你想**修改代码并测试**，按照以下步骤构建开发版。

### 步骤 1：安装前置依赖

#### 1.1 安装 Git for Windows

下载并安装：https://git-scm.com/download/win

安装选项：
- ✅ 勾选 "Add Git to PATH"
- ✅ 选择 "Use Git from Git Bash only" 或 "Use Git from the command line and also from 3rd-party software"
- ✅ 勾选 "Checkout as-is, commit Unix-style line endings"

#### 1.2 安装 Node.js 24.x

1. 访问 https://nodejs.org/
2. 下载并安装 Node.js 24.x LTS 版本
3. 验证安装：
```powershell
node --version  # 应显示 v24.x.x
npm --version
```

#### 1.3 安装 pnpm 10.x

```powershell
npm install -g pnpm@10
```

验证安装：
```powershell
pnpm --version  # 应显示 10.x.x
```

#### 1.4 安装 Rust 1.93.0

1. 访问 https://rustup.rs/
2. 运行安装程序
3. 验证安装：
```powershell
rustc --version  # 应显示 1.93.0
cargo --version
```

#### 1.5 安装 Visual Studio 2022 Build Tools

**必须安装**（构建 C++ 依赖项所需）：

1. 下载 Visual Studio 2022 Build Tools：https://visualstudio.microsoft.com/downloads/
2. 安装时勾选：
   - ✅ **"使用 C++ 的桌面开发"** (Desktop development with C++)
   - ✅ Windows 11 SDK（最新版本）

#### 1.6 安装 CMake

下载并安装：https://cmake.org/download/ （选择 Windows x64 Installer）

安装时选择：✅ "Add CMake to system PATH for all users"

验证：
```powershell
cmake --version
```

#### 1.7 安装 Ninja（可选但推荐）

```powershell
winget install Ninja-build.Ninja
```

或手动下载：https://github.com/ninja-build/ninja/releases

### 步骤 2：克隆并配置仓库

#### 2.1 克隆仓库

打开 **Git Bash**（或 PowerShell）：

```bash
# 克隆仓库
git clone https://github.com/tinyhumansai/openhuman.git
cd openhuman
```

#### 2.2 初始化 Git 子模块

```bash
git submodule update --init --recursive
```

这会下载 `app/src-tauri/vendor/` 下的 vendored Tauri/CEF 源码。

#### 2.3 安装 Node.js 依赖

```bash
pnpm install
```

### 步骤 3：使用开发脚本构建

项目提供了 `run-dev-win.sh` 脚本，自动处理复杂的环境配置：

#### 3.1 运行开发构建脚本

```bash
# 从项目根目录运行
bash ./scripts/run-dev-win.sh
```

**这个脚本会自动完成**：
- ⚙️ 配置 MSVC 编译环境（调用 `vcvars64.bat`）
- ⚙️ 设置 Windows SDK 路径
- ⚙️ 配置 Ninja 构建系统
- ⚙️ 查找并安装 pnpm/cargo/Node.js
- ⚙️ 安装 vendored tauri-cli（CEF-aware）
- ⚙️ 准备 CEF 运行时环境
- ⚙️ 启动 Tauri 开发构建

#### 3.2 首次运行说明

首次运行会下载约 **270MB** 的 CEF 运行时环境，耐心等待。

### 步骤 4：运行开发版应用

构建成功后，OpenHuman 会自动启动。如果没有启动，手动运行：

```powershell
# 开发版可执行文件路径
app\src-tauri\target\x86_64-pc-windows-msvc\debug\OpenHuman.exe
```

---

## 方式三：从源码构建发布版

如果你想构建**生产级别的安装包**（MSI/NSIS），执行以下步骤。

### 前置条件

确保已完成方式二的所有前置依赖安装。

### 构建步骤

#### 1. 配置环境变量（可选）

创建 `app/.env.local` 文件：

```env
VITE_BACKEND_URL=https://staging-api.tinyhumans.ai
VITE_SENTRY_DSN=your_sentry_dsn_here
VITE_DEBUG=false
```

#### 2. 构建前端资源

```powershell
cd app
pnpm run build:app
```

这会执行 TypeScript 编译和 Vite 构建，输出到 `app/dist/` 目录。

#### 3. 构建 Tauri 应用

```powershell
# 确保 vendored tauri-cli 已安装
pnpm tauri:ensure

# 构建 Windows 安装包（NSIS 和 MSI）
cargo tauri build --target x86_64-pc-windows-msvc
```

#### 4. 仅构建特定格式

```powershell
# 仅 NSIS (.exe)
cargo tauri build --target x86_64-pc-windows-msvc --bundles nsis

# 仅 MSI
cargo tauri build --target x86_64-pc-windows-msvc --bundles msi
```

### 输出位置

构建产物会生成在：

```
app/src-tauri/target/x86_64-pc-windows-msvc/release/bundle/
├── msi/
│   └── OpenHuman_x.x.x_x64.msi
├── nsis/
│   └── OpenHuman_x.x.x_x64-setup.exe
└── exe/
    └── OpenHuman.exe  # 可执行文件
```

---

## 方式四：使用 GitHub Actions 自动构建

如果你没有完整的 Windows 构建环境，可以使用 GitHub Actions。

### 方法 A：从 fix/windows 分支自动构建

推送代码到 `fix/windows` 分支会自动触发 Windows 构建：

```bash
git checkout -b fix/windows
# 进行代码修改
git add .
git commit -m "Your changes"
git push origin fix/windows
```

构建完成后：
1. 访问 Actions 运行页面
2. 下载构建产物：
   - `windows-msi` - MSI 安装包
   - `windows-nsis` - NSIS 安装包
   - `windows-cli` - 独立 CLI 二进制文件

### 方法 B：手动触发构建

1. 访问仓库的 GitHub Actions 页面
2. 选择 "Build Windows" workflow
3. 点击 "Run workflow"
4. 选择分支并运行

### 方法 C：创建发布构建

对于正式发布，使用发布工作流：

1. 访问 GitHub Actions
2. 选择 "Release (Staging)" 或 "Release Production"
3. 配置参数并触发

这会生成正式发布版本的安装包并上传到 GitHub Releases。

---

## 前置依赖详解

### Visual Studio 2022 Build Tools

**为什么必需？**
- Rust 的某些依赖项（如 `whisper-rs-sys`、`lib-sodium-sys`）需要编译 C/C++ 代码
- CEF（Chromium Embedded Framework）的 native bindings 需要 C++ 编译器和 Windows SDK

**最小安装选项**：
```
☑ 使用 C++ 的桌面开发
  ☑ MSVC v143 - VS 2022 C++ x64/x86 生成工具（最新）
  ☑ Windows 11 SDK (10.0.22621.0 或更新)
```

**验证安装**：
```powershell
# 在 VS Developer Command Prompt 中运行
cl.exe
lib.exe
rc.exe
```

### Windows SDK

**为什么必需？**
- 链接 Windows 系统库（如 `kernel32.lib`、`user32.lib`）
- 编译资源文件（.rc）

**验证安装**：
```powershell
# 检查 SDK 安装目录
dir "C:\Program Files (x86)\Windows Kits\10\Lib"
```

### Rust 和 Cargo

**版本要求**：1.93.0

**安装 rustfmt 和 clippy**：
```powershell
rustup component add rustfmt clippy
```

**验证**：
```powershell
rustc --version  # 1.93.0
cargo --version  # 应显示 cargo 1.93.0 或更高
```

### CEF 运行时

**说明**：CEF（Chromium Embedded Framework）是 Tauri 的 WebView 运行时，约 270MB。

**自动下载**：构建脚本会自动从 `app/src-tauri/vendor/tauri-cef` 下载。

**手动缓存位置**：
- Windows：`%LOCALAPPDATA%\tauri-cef`
- macOS：`~/Library/Caches/tauri-cef`
- Linux：`~/.cache/tauri-cef`

---

## 常见问题排查

### 问题 1：cl.exe 找不到

**错误信息**：
```
link.exe: extra operand '...rcgu.o'
```

**原因**：Git Bash 的 PATH 中有 Git 的 `link.exe`（GNU coreutils）而非 MSVC 的 `link.exe`。

**解决方案**：运行 `run-dev-win.sh`，脚本会自动修复 PATH 并设置 `CARGO_TARGET_X86_64_PC_WINDOWS_MSVC_LINKER`。

### 问题 2：缺少 Windows SDK

**错误信息**：
```
LINK : fatal error LNK1181: cannot open input file 'kernel32.lib'
```

**原因**：Windows SDK 未正确安装或环境变量未设置。

**解决方案**：
1. 重新安装 Visual Studio 2022 Build Tools
2. 确保勾选了 Windows SDK 组件
3. 或手动安装 Windows SDK：https://developer.microsoft.com/windows/downloads/windows-sdk/

### 问题 3：CEF 下载失败

**错误信息**：
```
Could not download CEF runtime
```

**解决方案**：
1. 检查网络连接
2. 设置代理（如果需要）：
```bash
export HTTPS_PROXY=http://your-proxy:port
bash ./scripts/run-dev-win.sh
```

### 问题 4：内存不足

**错误信息**：
```
out of memory
```

**解决方案**：构建时限制 Node.js 内存：
```powershell
$env:NODE_OPTIONS="--max-old-space-size=4096"
cargo tauri build --target x86_64-pc-windows-msvc
```

### 问题 5：pnpm 安装失败

**错误**：找不到 pnpm 命令

**解决方案**：
```powershell
# 方案 1：使用 npm 全局安装
npm install -g pnpm

# 方案 2：使用 winget 安装
winget install pnpm.pnpm

# 方案 3：验证 PATH
echo $PATH
# 确保包含 npm 或 pnpm 的安装路径
```

### 问题 6：Git 子模块初始化失败

**错误信息**：
```
fatal: not a git repository
```

**解决方案**：
```bash
git clone https://github.com/tinyhumansai/openhuman.git
cd openhuman
git submodule update --init --recursive
```

### 问题 7：构建时间过长

**说明**：首次构建需要编译大量 Rust 依赖和 CEF native 代码，可能需要 30-60 分钟。

**优化建议**：
- ✅ 使用 SSD 硬盘
- ✅ 保持网络连接以便下载 CEF
- ✅ 确保至少有 16GB RAM
- ✅ 关闭其他占用资源的程序

### 问题 8：杀毒软件误报

**说明**：某些杀毒软件可能将构建过程中的临时文件标记为可疑。

**解决方案**：
- ✅ 将项目目录添加到杀毒软件白名单
- ✅ 临时禁用实时防护（不推荐）
- ✅ 使用 Windows Defender（通常不会误报）

---

## 快速参考命令

### 常用命令汇总

```powershell
# 克隆仓库
git clone https://github.com/tinyhumansai/openhuman.git
cd openhuman

# 初始化子模块
git submodule update --init --recursive

# 安装依赖
pnpm install

# 开发构建（推荐）
bash ./scripts/run-dev-win.sh

# 仅构建 UI
cd app
pnpm run build:app

# 发布构建
cargo tauri build --target x86_64-pc-windows-msvc

# 检查 Rust 代码
cargo check --manifest-path Cargo.toml

# 运行测试
pnpm test
```

### 环境变量说明

| 变量 | 说明 | 默认值 |
| ---- | ---- | ------ |
| `CEF_PATH` | CEF 运行时路径 | `~/.cache/tauri-cef` (Windows: `%LOCALAPPDATA%\tauri-cef`) |
| `OPENHUMAN_CORE_PORT` | Core RPC 端口 | `7788` |
| `OPENHUMAN_DEV_PORT` | 开发服务器端口 | `1420` |
| `CARGO_TARGET_DIR` | Cargo 构建输出目录 | `target/` |
| `NODE_OPTIONS` | Node.js 内存限制 | `--max-old-space-size=8192` |

### 文件路径参考

```
E:\MyProject\github-projects\openhuman\
├── app\                              # 应用目录
│   ├── src\                          # React 源码
│   ├── src-tauri\                    # Tauri/Rust 源码
│   │   ├── src\                      # Rust 源码
│   │   ├── target\                   # 构建输出
│   │   └── vendor\                   # Vendored 依赖
│   └── package.json
├── src\                              # Rust Core 源码
├── scripts\                          # 构建脚本
└── Cargo.toml                        # Rust workspace 配置
```

---

## 技术支持

- **官方文档**：https://github.com/tinyhumansai/openhuman
- **问题反馈**：https://github.com/tinyhumansai/openhuman/issues
- **讨论区**：https://github.com/tinyhumansai/openhuman/discussions

---

**祝你构建顺利！** 🚀
