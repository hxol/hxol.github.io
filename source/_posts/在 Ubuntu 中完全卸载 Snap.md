---
title: 在 Ubuntu 中完全卸载 Snap
date: 2026-10-01 15:30:18
tags: [笔记, Linux, Ubuntu, Snap]
---

# 在 Ubuntu 中完全卸载 Snap


### 步骤 1：列出已安装的 Snap 软件包
首先，检查系统中当前安装了哪些 Snap 软件包：
```bash
snap list
```
这会显示所有通过 Snap 安装的应用程序，例如 `core`、`snapd` 等。

### 步骤 2：删除所有 Snap 软件包
逐一删除列出的 Snap 软件包。使用以下命令移除每个软件包（将 `<package_name>` 替换为实际的软件包名称，例如 `core` 或 `firefox`）：
```bash
sudo snap remove <package_name>
```
例如：
```bash
sudo snap remove firefox
sudo snap remove core
```
重复此步骤，直到 `snap list` 返回类似“没有安装的 Snap 软件包”的提示。

### 步骤 3：卸载 snapd
删除所有 Snap 软件包后，可以卸载 `snapd`（Snap 的后台服务和核心组件）：
```bash
sudo apt purge snapd -y
```
`purge` 选项会同时删除配置文件，确保清理更彻底。

### 步骤 4：删除 Snap 残留文件
即使卸载了 `snapd`，系统中可能仍有一些残留的缓存或配置文件。运行以下命令清理：
```bash
rm -rf ~/snap
sudo rm -rf /var/cache/snapd/
sudo rm -rf /var/lib/snapd/
```
这些命令会删除用户目录下的 Snap 数据以及系统级别的 Snap 文件。

### 步骤 5：检查是否还有 Snap 相关进程
确保没有 Snap 相关的进程在运行：
```bash
ps aux | grep snap
```
如果有任何 Snap 相关的进程，可以用 `kill` 命令结束它们，或者直接重启系统。

### 步骤 6：阻止 Snap 重新安装

#### 方法1 使用 `apt-mark hold` 锁定 snapd

```bash
sudo apt-mark hold snapd
```
- 这会告诉 `apt` 包管理器不要更新或安装 `snapd`，即使它是某些依赖项的一部分。
- 如果将来需要解锁，可以使用：
  ```bash
  sudo apt-mark unhold snapd
  ```

#### 方法2 创建一个 APT 偏好文件

通过设置 APT 的优先级规则，可以永久阻止 `snapd` 的安装。按以下步骤操作：
1. 创建一个偏好文件：
   ```bash
   sudo nano /etc/apt/preferences.d/nosnap.pref
   ```
2. 在文件中添加以下内容：
   ```
   Package: snapd
   Pin: release *
   Pin-Priority: -10
   ```
3. 保存并退出（按 `Ctrl+O`，回车，然后 `Ctrl+X`）。
- 这将为 `snapd` 设置一个负优先级，阻止其安装。

### 步骤 7：验证 Snap 已完全移除
运行以下命令，确认 Snap 已不再存在：
```bash
snap version
```
如果返回“command not found”或类似提示，说明 Snap 已成功移除。

### 注意事项
- 如果你依赖某些通过 Snap 安装的软件（例如 Firefox 在某些 Ubuntu 版本中默认通过 Snap 提供），请确保提前通过其他方式（如 `apt` 或 Flatpak）安装替代版本。
- 这些步骤适用于 Ubuntu 及其衍生版本，具体行为可能因版本而异。

完成以上步骤后，你的 Ubuntu 系统将不再包含 Snap。如果有其他问题，欢迎随时提问！






































