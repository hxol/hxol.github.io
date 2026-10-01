---
title: PVE 虚拟机轻量级桌面与 Synology Drive Client 部署指南
date: 2026-10-01 15:30:21
tags: [笔记, Linux, Openbox, Synology Drive, X2Go]
---

# PVE 虚拟机轻量级桌面与 Synology Drive Client 部署指南

本指南旨在为已运行 `code-server` 的轻量级 Debian/Ubuntu 虚拟机（2GB RAM, 4 Core, 12GB Disk）添加极简图形界面（Openbox + X2Go），以便远程配置和运行 Synology Drive Client (SDC)。

不想在群晖里运行 `code-server` ，又不想让群晖里的目录通过 NFS 挂载到 安装`code-server` 的机器里，可以尝试这个方案。其实如果选用  linuxserver 版的 `code-server` docker 镜像，能以非 root 权限、只读系统运行，也可以尝试在群晖里安装 `code-server` 。

## 📌 当前虚拟机环境评估
*   **服务**：已运行 `code-server`（占用一定资源）
*   **资源**：2GB 内存 / 4 CPU 核心 / 剩余 12GB 硬盘
*   ⚠️ **重要存储提示**：您提到“SDC同步的数据应存放在NAS上，不占用VM的12GB”。请注意：**Synology Drive Client 本质是同步工具，默认会将 NAS 上的文件下载到本地（即虚拟机硬盘中）**。如果同步的文件夹大于 12GB，会直接撑爆虚拟机。
    *   *建议*：在后续配置 SDC 时，请仅选择极小的核心文件夹进行同步，或者配置为**单向上传**（Backup 模式），避免大量 NAS 数据同步到虚拟机本地。

---

## 🛠️ 实施步骤

### 1. SSH 登录到 PVE 虚拟机
打开本地终端，使用 SSH 连接到虚拟机（请替换尖括号内的信息）：
```bash
ssh <你的用户名>@<虚拟机IP>
```

### 2. 更新系统与基础环境
保持系统最新，以防依赖冲突。
```bash
sudo apt update && sudo apt upgrade -y
```

### 3. 安装轻量级图形核心组件
使用 `--no-install-recommends` 参数，坚决不安装冗余的推荐包，保持系统极简。
```bash
sudo apt install --no-install-recommends xorg openbox
```
*   `xorg`: X 窗口系统的核心。
*   `openbox`: 极度轻量、资源占用极低的窗口管理器。

### 4. 安装必备图形工具和中文字体
确保能在图形界面输入命令，且 SDC 客户端中文不乱码。
```bash
sudo apt install --no-install-recommends lxterminal fonts-wqy-zenhei fonts-noto-cjk
```

### 5. 安装任务栏面板 (推荐)
Openbox 默认只有右键菜单，没有任务栏。为了能看到 Synology Drive 的系统托盘图标，建议安装轻量级面板 `tint2`。
```bash
sudo apt install --no-install-recommends tint2
```

### 6. 安装 X2Go Server
X2Go 采用 NX 协议，在低带宽和轻量级环境下体验远超 VNC。
```bash
sudo apt install x2goserver x2goserver-xsession
```

### 7. 下载并安装 Synology Drive Client

由于群晖官网下载链接经常带有重定向，在无图形界面的虚拟机里直接 `wget` 容易失败，推荐以下 **两种方式** 之一：

**方式 A（推荐）：在本地电脑下载后上传**
1. 访问 [群晖下载中心](https://www.synology.cn/zh-cn/support/download) 下载 `Linux (deb)` 版本的 SDC（文件名通常为 `synology-drive-client-XXXX.x86_64.deb`）。
2. 在本地电脑的终端执行 `scp` 命令传到虚拟机：
   ```bash
   scp /本地路径/synology-drive-client-*.deb <你的用户名>@<虚拟机IP>:~
   ```

**方式 B：在虚拟机内获取下载直链**
如果您能获取到真实的 `.deb` 直链：
```bash
wget -O synology-drive-client.deb "替换为你的真实下载链接"
```

**开始安装：**
```bash
sudo dpkg -i synology-drive-client*.deb
```
> 💡 **注意**：此处极大概率会提示缺少依赖。请紧接着运行以下命令自动修复并完成安装：
```bash
sudo apt --fix-broken install -y
```

### 8. 配置 X2Go Client (在本地电脑操作)
1. 下载并安装 [X2Go Client](https://wiki.x2go.org/doku.php/download:start)。
2. 创建新会话（Session）：
   *   **Session name**: `PVE-CodeServer-GUI` (自定义)
   *   **Host**: `<虚拟机IP>`
   *   **Login**: `<你的用户名>`
   *   **SSH port**: `22` (若修改过请对应更改)
   *   **Session type**: 下拉选择 **Custom desktop**，并在后面的 Command 框中输入 `openbox-session`
   *   **Media/Sound**: 建议在设置中**关闭**声音（Sound）和打印（Printing）映射，以节省性能。
3. 保存并双击连接。

### 9. 首次远程连接并配置 SDC
1. 登录 X2Go 后，您将看到一个全黑桌面（底部可能有 `tint2` 任务栏）。
2. **右键点击桌面空白处**，在菜单中选择并打开终端 (`LXTerminal`)。
3. 在终端中启动群晖客户端：
   ```bash
   synology-drive
   ```
   *(如果提示找不到命令，请尝试完整路径：`/opt/Synology/SynologyDrive/bin/launcher`)*
4. SDC 的图形向导会弹出，请按照指引登录 NAS。
5. ⚠️ **再次提醒**：配置同步路径时，请务必注意虚拟机的剩余磁盘空间（12GB）。

### 10. 配置 SDC 和任务栏开机自启 (可选)
为了下次连接 X2Go 时无需手动敲命令启动，我们可以配置 Openbox 自启动脚本。

```bash
# 创建配置目录
mkdir -p ~/.config/openbox
# 编辑自启动文件
nano ~/.config/openbox/autostart
```
将以下内容粘贴进去（注意末尾的 `&` 符号必不可少，代表后台运行）：
```sh
# 启动任务栏面板
tint2 &

# 启动群晖 Drive 客户端
synology-drive &
```
按下 `Ctrl+O` 保存，`Enter` 确认，`Ctrl+X` 退出。

### 11. 安装 QEMU Guest Agent (PVE 宿主机联动优化)
确保 PVE 能正确读取虚拟机状态并执行软关机。
```bash
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
```
> **注意**：安装完成后，请到 PVE 网页后台 -> 选中该虚拟机 -> **Options (选项)** -> 找到 **QEMU Guest Agent** 并勾选启用。然后建议在 PVE 重启一次该虚拟机生效。

---

## 🔧 常见故障排除 (Troubleshooting)

1. **X2Go 连接后秒退，或只有鼠标黑屏：**
   *   检查自定义命令是否准确拼写为 `openbox-session`。
   *   查看日志定位问题：在 SSH 中运行 `cat ~/.xsession-errors`。
2. **SDC 启动报错，提示 `qt.qpa.plugin: Could not load the Qt platform plugin "xcb"`：**
   *   这是精简版 Linux 安装 GUI 软件的通病（缺少图形渲染库）。请在 SSH 中补齐以下依赖：
   ```bash
   sudo apt install libxcb-xinerama0 libxcb-cursor0 libx11-xcb1
   ```
3. **资源占用过高：**
   *   `code-server` 和 `Xorg` 会争抢 2GB 内存。如果在 SDC 同步大量零碎文件时系统卡顿，可在 SSH 中运行 `htop` 查看瓶颈。
   *   若内存长期吃紧，建议在 PVE 中为该虚拟机分配一定量的 Swap 交换空间，或者将内存临时提升至 3GB。