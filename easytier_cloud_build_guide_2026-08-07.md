# EasyTier 订阅配置隐私保护与 GitHub Actions 云端打包指南

**文档日期**: 2026-08-07  
**当前 Git 分支**: `borUI`

---

## 一、 云端打包 (CI/CD Automated Build) 原理解析

### 1.1 什么是云端打包？
**云端打包**是指基于 GitHub Actions（GitHub 提供的持续集成/持续部署 CI/CD 服务），在 GitHub 托管的云端 64 位 Windows 虚拟机环境（`windows-latest`）中，自动执行代码拉取、环境依赖安装、编译打包并生成最终二进制安装程序（如 `.exe`）的全自动化过程。

### 1.2 为什么使用云端打包？
1. **免去本地复杂环境配置**：由于 EasyTier 桌面端采用 **Tauri 2.0 (Rust + Node.js + C++ MSVC)** 架构，本地编译需要安装庞大的 Rust 工具链（`cargo` / `rustc`）与 Visual Studio C++ 构建依赖。云端虚拟机预装了完整的 Rust 和 Visual Studio 编译环境，开箱即用。
2. **环境标准一致**：云端虚拟机环境纯净且标准化，可避免本地开发环境因 DLL 版本、环境变量、Rust 版本不匹配引发的各种奇样编译报错。
3. **自动化与归档**：只要向 GitHub 仓库提交/推送代码，云端就会自动运行编译构建，并将编译好的二进制 `.exe` 安装程序自动打包归档供随时下载。

---

## 二、 本次修改的全部功能与文件细节

我们对 EasyTier 的核心与 UI 前端进行了完整改造，使得**由订阅服务器配置的网络节点在本地客户端界面和 RPC 接口中均无法查看内部敏感参数**（如 `secret` 加密口令、Peer 加密链接等），同时配置了云端打包工作流。

### 2.1 核心底层与 RPC 权限保护 (`easytier-core`)
- **[easytier-core/src/instance/manager.rs](file:///d:/TempGitPro/EasyTier/easytier-core/src/instance/manager.rs)**
  - 在 `ConfigFilePermission` 结构中新增 `NO_VIEW = 1 << 2` 权限标记位，用于声明“禁止查看内部配置细节”。
- **[easytier-core/src/management/full/process_rpc.rs](file:///d:/TempGitPro/EasyTier/easytier-core/src/management/full/process_rpc.rs)**
  - 在 `get_network_instance_config` 服务层增加防泄露拦截：凡是来源为订阅服务器（`ConfigSource::Web`）或设置了 `NO_VIEW` 标记的实例，RPC 拦截并拒绝返回明文 `NetworkConfig`。
  - 在 `list_network_instance_meta` 中，向前端自动广播包含 `NO_VIEW | READ_ONLY | NO_DELETE` 的组合权限，确保客户端识别订阅节点。

### 2.2 前端与界面盲化保护 (`easytier-web/frontend-lib`)
- **[easytier-web/frontend-lib/src/modules/api.ts](file:///d:/TempGitPro/EasyTier/easytier-web/frontend-lib/src/modules/api.ts)**
  - 导出 `ConfigSource` 枚举，并在 `ConfigFilePermission` 命名空间中加入 `NO_VIEW` 常量与 `isViewable(perm)` 判断助手函数。
- **[easytier-web/frontend-lib/src/components/RemoteManagement.vue](file:///d:/TempGitPro/EasyTier/easytier-web/frontend-lib/src/components/RemoteManagement.vue)**
  - **操作菜单隐藏**：对订阅配置隐藏了“编辑网络”、“导出配置文件”和“以文件方式编辑”按钮。
  - **专属受保护界面卡片**：选中订阅配置时，主面板渲染“**🔒 订阅服务器配置（已受保护）**”专有卡片，提示用户内部敏感连接参数已由订阅服务器受保护隐藏。
  - **基础功能保留**：用户依然可以点击“启用网络/禁用网络”进行开关控制，并实时监控 P2P 连接拓扑与延迟。

### 2.3 云端打包工作流配置 (`.github/workflows/gui.yml`)
- **[.github/workflows/gui.yml](file:///d:/TempGitPro/EasyTier/.github/workflows/gui.yml)**
  - 补充添加 `workflow_dispatch:` 开关：解锁 GitHub 网页端的 **`Run workflow` 手动运行按钮**。
  - 增加分支监听 `"borUI"`：确保提交并推送到您当前的 `borUI` 分支时，能自动触发云端 Windows `.exe` 编译。

---

## 三、 本地 Git 提交记录

我们已在本地 `borUI` 分支上完成了所有修改与打包提交：

1. **Commit `ecae2667`**: `feat: 核心底层与 RPC 接口控制及 UI 盲化保护实现`
2. **Commit `7f292d8`**: `ci: add workflow_dispatch and borUI branch trigger to gui.yml`

---

## 四、 触发云端打包的完整操作步骤

在您的本地命令行终端（PowerShell 或 Git Bash）中执行以下命令即可触发云端打包：

```bash
# 1. 切换到项目根目录并推送到您自己的 GitHub 远程仓库
git push origin borUI
```

推送成功后的云端编译与下载流程：

1. **打开 GitHub 仓库 Actions 页面**：刷新您的 GitHub 仓库页面，点击 **Actions** 标签。
2. **查看自动构建状态**：您会看到标题为 `EasyTier GUI` 的 Workflow 已经自动开启并处于运行中状态（图标为黄色旋转圆圈）。
3. **手动触发方式（可选）**：如果以后想随时手动构建，可以在该页面点击 **EasyTier GUI** -> 点击右上角 **Run workflow** 按钮 -> 选择 `borUI` 分支 -> 点击 **Run workflow**。
4. **下载生成好的 `.exe` 文件**：
   - 编译完成后（约 5~10 分钟），构建状态会变为绿色勾号 `✓`。
   - 点击该构建记录进入详情页，拉到最下方的 **Artifacts (构建产物)** 区域。
   - 点击下载 `easytier-gui-x86_64-pc-windows-msvc` 压缩包，解压后即可直接获取可执行的 Windows `.exe` 安装程序！
