---
title: 阻止 Mac 在外接磁盘或网络共享里面拉屎
date: 2026-10-01 15:30:22
tags: [笔记, Mac]
---

# 阻止 Mac 在外接磁盘或网络共享里面拉屎



macOS 在连接外接磁盘或网络共享（SMB/NFS）时，由于文件系统兼容性问题，确实非常喜欢到处“拉屎”。这些“屎”主要分为两类：

1. **`.DS_Store`**：记录文件夹的图标大小、位置、背景等视图设置。
2. **`._*` 隐藏文件（AppleDouble）**：如果服务器的硬盘（比如 ext4 / NTFS）不支持 Mac 原生的扩展属性（标签、资源分支等），Mac 就会强行生成一个同名但以 `._` 开头的文件来保存这些属性。

要想彻底根治这个“随地大小便”的毛病，需要**Mac 客户端**和**SMB 服务端（NAS/路由器）**双管齐下。以下是完整的解决指南：

## 第一步：阻止 Mac 产生 `.DS_Store`（在 Mac 上操作）

Mac 官方提供了一个隐藏的开关，可以直接禁止系统在网络共享上写入 `.DS_Store`。

1. 打开 Mac 上的 **终端 (Terminal)**。
2. 粘贴并回车执行以下命令：
   ```bash
   defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
   ```
   *(可选)* 如果你想连 U 盘和移动硬盘上的屎也一起禁了，可以再加一行：
   ```bash
   defaults write com.apple.desktopservices DSDontWriteUSBStores -bool true
   ```
3. 重启 Finder 让设置生效（或者直接重启电脑）：
   ```bash
   killall Finder
   ```

> **注意：** 这招只能阻止新的 `.DS_Store` 产生，但**无法阻止** `._*` 文件的生成。

## 第二步：阻止 `._*` 文件的生成（在 SMB 服务端 / NAS 上操作）

对于 `._*` 文件，最简单粗暴且一劳永逸的方法是：**在 SMB 服务端直接拉黑这些文件，让 Mac 根本写不进去**。

**情况 A：如果你使用的是群晖 (Synology) 或 威联通 (QNAP)**
* **群晖**：打开 `控制面板` -> `文件服务` -> `SMB` -> `高级设置`。在“常规”选项卡中找到 **否决文件 (Veto files)** 或者 **隐藏 Mac 文件** 的选项，输入 `/*.DS_Store/._*/` 并保存。
* **威联通**：打开 `控制台` -> `网络&文件服务` -> `Win/Mac/NFS/WebDAV` -> `高级设置`，同样寻找 **否决文件 (Veto files)** 选项，填入 `/*.DS_Store/._*/`。

**情况 B：如果你使用的是自己搭建的 Linux / OpenWrt / Unraid Samba 服务**
打开你的 Samba 配置文件（通常在 `/etc/samba/smb.conf`），在你对应的共享版块 `[Share]` 或者全局版块 `[global]` 中加入下面两行：

```ini
veto files = /._*/.DS_Store/
delete veto files = yes
```
*注：`delete veto files = yes` 的作用是，当你在 Windows 上删除一个包含 Mac 隐藏文件的文件夹时，Samba 会自动把里面的“屎”一起删掉，否则会报错“文件夹不为空”。*

修改后，重启 SMB 服务生效：
```bash
sudo systemctl restart smbd
```

## 第三步：清理以前拉下的“旧屎”

上面的设置只能用来“防范于未然”，已经存在 SMB 里的垃圾文件需要手动清理。
在 Mac 上连好你的 SMB 共享盘（假设它在 Finder 里挂载的名字叫 `ShareName`），然后在终端执行：

**1. 一键清理 `._*` 附属文件（Mac 官方提供的命令）：**
```bash
dot_clean -m /Volumes/ShareName
```

**2. 暴力扫除所有的 `.DS_Store` 和 `._*`（需谨慎确认挂载路径正确）：**
```bash
# 删除所有 .DS_Store
find /Volumes/ShareName -name ".DS_Store" -type f -delete

# 删除所有 ._ 开头的文件
find /Volumes/ShareName -name "._*" -type f -delete
```

## 懒人备选方案：BlueHarvest

如果你连接的是公司的 SMB 服务器，**没有权限修改服务端的设置**，上述第二步你就做不了。这种情况下，Mac 端依旧会顽固地生成 `._*` 文件。
如果是这种情况，可以考虑使用 Mac 上的著名第三方工具 **BlueHarvest**。这个软件的唯一功能就是“自动擦屁股”——它会在后台静默运行，只要你连上了网络共享盘或者插上 U 盘，Mac 刚拉出一点屎，它就会在毫秒级自动帮你清理掉，整个过程你和 Windows 同事都毫无察觉。