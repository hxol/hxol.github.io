---
title: 如何使用 Diskpart 命令彻底清理与重置U盘
date: 2026-10-01 15:30:30
tags: [笔记, Windows, Diskpart, U盘, 硬盘]
---

# 如何使用 Diskpart 命令彻底清理与重置U盘

⚠️ **高危操作警告**：
> `clean` 命令会**无条件彻底抹除**所选磁盘上的所有数据！在执行前，请**务必三连确认**您选择的磁盘编号是否真的是您的U盘，千万不要选成电脑的系统盘或数据盘，以免造成不可挽回的数据丢失！

## 详细操作步骤

### 1. 以管理员身份运行终端
右键点击开始菜单（或按 `Win + X`），选择“Windows PowerShell (管理员)” 或 “终端 (管理员)”。

### 2. 进入 Diskpart 工具并查找U盘
在命令行中输入 `diskpart` 并回车。接着输入 `list disk` 来查看当前电脑上连接的所有磁盘。请根据**磁盘大小（大小列）**来准确判断哪一个是你的U盘。

```powershell
PS C:\Users\Username> diskpart

Microsoft DiskPart 版本 10.0.19041.928
Copyright (C) Microsoft Corporation.
在计算机上: DESKTOP-XXXXXXX

DISKPART> list disk

  磁盘 ###  状态           大小     可用     Dyn  Gpt
  --------  -------------  -------  -------  ---  ---
  磁盘 0    联机             512 GB      0 B        *
  磁盘 1    联机               1 TB      0 B        *
  磁盘 2    联机              32 GB      0 B        *   <-- 假设这是你的U盘
```

### 3. 选择并清理U盘
确认你的U盘磁盘编号（本例中为 `磁盘 2`）后，选择该磁盘并执行清除命令。

```powershell
DISKPART> select disk 2

磁盘 2 现在是所选磁盘。

DISKPART> clean

DiskPart 成功地清除了磁盘。
```
*(注：执行完 `clean` 后，U盘内所有数据和分区表已被彻底清空，此时U盘处于“未初始化/未分配”状态。)*

### 4. 重新建立分区并格式化（必做）
为了让U盘能够重新被电脑识别和使用，我们需要为它创建一个新的主分区，并进行格式化。在 `DISKPART>` 提示符下继续依次输入以下命令：

```powershell
# 1. 创建主分区
DISKPART> create partition primary
DiskPart 成功地创建了指定分区。

# 2. 格式化分区（此处以 exFAT 格式为例，支持大文件且跨平台兼容性好；如果仅在Windows下使用可改为 fs=ntfs）
DISKPART> format fs=exfat quick
  100 百分比已完成
DiskPart 成功格式化了该卷。

# 3. 为U盘分配驱动器号（盘符），让它出现在“我的电脑”中
DISKPART> assign
DiskPart 成功地分配了驱动器号或装载点。

# 4. 退出 Diskpart
DISKPART> exit
```

至此，您的U盘已经犹如出厂般干净，并且可以正常存储文件了！

