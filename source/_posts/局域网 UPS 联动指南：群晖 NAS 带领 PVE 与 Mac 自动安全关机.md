---
title: 局域网 UPS 联动指南：群晖 NAS 带领 PVE 与 Mac 自动安全关机
date: 2026-09-30 15:30:00
tags: [笔记, UPS, 群晖, PVE, Mac, 自托管]
---



# 局域网 UPS 联动指南：群晖 NAS 带领 PVE 与 Mac 自动安全关机

## 📖 核心思路

在拥有多台设备却只有一个 UPS（不间断电源）的场景下，我们可以利用 NUT (Network UPS Tools) 服务来实现局域网内的联动关机。
具体方案是：将**群晖 NAS 作为 NUT 服务器 (Master)** 直接通过 USB 连接 UPS；然后将局域网内的 **PVE 宿主机和 Mac 电脑配置为 NUT 客户端 (Slave)**。当停电发生时，客户端会通过局域网接收到群晖发来的断电信号，并自动触发安全关机。

---

## ⚙️ 第一步：配置群晖 NAS（服务端）

首先，确保你的 UPS 已经通过 USB 数据线连接到群晖，且群晖已经能正确识别 UPS。

为了让其他设备能连上群晖获取 UPS 状态，必须开启网络服务器并进行权限设置：
1. 登录群晖 DSM，进入 **控制面板 -> 硬件和电源 -> UPS**。
2. 勾选 **“启用网络 UPS 服务器”**。
3. 点击 **“允许的 DiskStation 设备”**（部分版本称为“允许的设备”）。
4. 在弹出的列表中，**添加你 PVE 和 Mac 的局域网 IP 地址**（例如：`192.168.1.20` 和 `192.168.1.30`）。
5. 点击“应用”保存设置。
6. 在群晖的防火墙放行 UPS 相关端口与IP。

> **💡 核心要点：** 
> 群晖内置的 NUT 账号密码是固定的：**用户名为 `monuser`，密码为 `secret`，UPS 设备名称默认叫 `ups`**。接下来的客户端配置中会频繁用到这三个参数。

---

## 🐧 第二步：配置 PVE (Linux) 接收断电信号

PVE 基于 Debian，其包管理器自带 NUT，安装和配置非常简单。

1. **SSH 登录到你的 PVE 宿主机终端**。
2. **安装 NUT 客户端：**
   ```bash
   apt update && apt install nut-client -y
   ```
3. **修改运行模式为客户端：**
   编辑配置文件 `nut.conf`：
   ```bash
   nano /etc/nut/nut.conf
   ```
   将文件中的 `MODE=none` 修改为：
   ```ini
   MODE=netclient
   ```
   *(保存并退出：按 `Ctrl+O`，回车确认，按 `Ctrl+X` 退出)*

4. **配置监控参数：**
   编辑配置文件 `upsmon.conf`：
   ```bash
   nano /etc/nut/upsmon.conf
   ```
   在文件末尾另起一行，添加以下内容（**请将 `<NAS_IP>` 替换为你群晖的实际 IP**，如 `192.168.1.10`）：
   ```ini
   MONITOR ups@<NAS_IP> 1 monuser secret slave
   ```
   *(参数解释：监控名为 ups 的设备，`1` 代表电源数量，用户 `monuser`，密码 `secret`，身份为 `slave`)*
   保存并退出。

5. **重启服务并设置开机自启：**
   ```bash
   systemctl restart nut-client
   systemctl enable nut-client
   ```

6. **测试连接：**
   在 PVE 终端输入：
   ```bash
   upsc ups@<NAS_IP>
   ```
   如果屏幕上输出了一大堆关于 UPS 的状态信息（如 `battery.charge`, `ups.load` 等），说明 PVE 已经成功连上群晖！

> **🛠 补充提示 (PVE 虚拟机安全关机)：**
> PVE 在收到关机信号后，会按顺序关闭其中的虚拟机。为了防止虚拟机被强制断电，请务必在重要的虚拟机（如 Windows、Linux 客户机）中**安装 QEMU Guest Agent**，并在 PVE 的虚拟机“选项”中开启 QEMU Guest Agent 功能，这样 PVE 才能优雅地软关机虚拟机。

---

## 🍏 第三步：配置 Mac (macOS) 接收断电信号

macOS 系统不自带 NUT 客户端，最简单且规范的方法是通过 Homebrew 来安装。

1. **打开 Mac 的“终端 (Terminal)”**。
2. **安装 Homebrew**（如已安装请跳过此步）：
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
3. **使用 Homebrew 安装 NUT：**
   ```bash
   brew install nut
   ```

4. **配置监控参数：**

   > **⚠️ 注意：** 以下路径基于 Apple Silicon (M1/M2/M3) 芯片的 Mac（默认路径为 `/opt/homebrew`）。如果你使用的是 **Intel 芯片** 的老款 Mac，请将下方所有的 `/opt/homebrew` 替换为 `/usr/local`。

   **① 复制配置文件模板**
   把 `sample` 模板复制为正式配置文件：
   ```bash
   cp /opt/homebrew/etc/nut/nut.conf.sample /opt/homebrew/etc/nut/nut.conf
   cp /opt/homebrew/etc/nut/upsmon.conf.sample /opt/homebrew/etc/nut/upsmon.conf
   ```

   **② 设置客户端模式**
   打开 `nut.conf` 文件：
   ```bash
   nano /opt/homebrew/etc/nut/nut.conf
   ```
   找到 `MODE=none`（如果没有就直接加在最下面），修改为：
   ```ini
   MODE=netclient
   ```

   **③ 添加监控器配置**
   打开 `upsmon.conf` 文件：
   ```bash
   nano /opt/homebrew/etc/nut/upsmon.conf
   ```
   用方向键翻到文件最末尾，另起一行，添加以下内容（同样将 `<NAS_IP>` 换成群晖的真实 IP）：
   ```ini
   MONITOR ups@<NAS_IP> 1 monuser secret slave
   ```

5. **启动 NUT 守护进程：**
   **必须以系统管理员 (root) 身份设置开机自启并运行：**
   ```bash
   sudo brew services start nut
   ```
   *查看服务状态命令：`sudo brew services list`*

6. **测试连接：**
   运行测试命令，查看是否能获取到数据：
   ```bash
   upsc ups@<NAS_IP>
   ```
   如果有数据刷出，说明 Mac 已经成功连上 UPS 服务器！

> **❓ 为什么 macOS 下启动服务必须用 `sudo`？**
> 在 macOS 中，`brew services` 有两种运行级别：
> 1. **普通运行 (`brew services start nut`)**：在当前用户的 `LaunchAgent` 级别运行。缺点是：只有当你开机并**输入密码登录进桌面后**，NUT 服务才会启动。且普通用户权限在触发强制关机指令时，极易被 macOS 拦截或要求确认。
> 2. **系统级运行 (`sudo brew services start nut`)**：在系统底层 `LaunchDaemon` 运行。**只要 Mac 一通电开机，即使停留在锁屏登录界面，NUT 也会默默在后台运行**。它拥有最高权限，收到群晖的断电信号后，可以毫无阻碍地直接执行关机指令。

---

## 🚨 第四步：检查与断电演习（非常重要）

配置完成后，请务必进行以下检查与测试，以确保在真正的灾难发生时系统能如期工作：

1. **防火墙检查**：
   群晖的 NUT 服务使用 TCP 端口 **`3493`**。如果你在群晖（或路由器）中开启了严格的防火墙策略，请务必放行 `3493` 端口，否则 PVE 和 Mac 会连接超时。
2. **关机策略机制**：
   * 群晖默认在进入“安全模式”（可通过群晖控制面板设置停电多久后进入）时，会向所有从机（PVE 和 Mac）发送 `FSD` (Forced Shutdown) 强制关机信号。
   * PVE 和 Mac 收到信号后会立刻开始关机流程，随后群晖自身会进入安全模式停止硬盘转动，最后 UPS 会自动切断自身电源。
3. **拔电演习**：
   在所有机器空闲且**没有重要数据写入时**，手动拔掉 UPS 的市电插头，模拟真实停电。
   * 观察 UPS 是否切换至电池供电。
   * 观察群晖到达设定时间后，PVE 和 Mac 是否自动触发了关机指令。
   * 确认所有设备均能安全关闭。
