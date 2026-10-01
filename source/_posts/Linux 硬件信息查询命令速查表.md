---
title: Linux 硬件信息查询命令速查表
date: 2026-10-01 15:30:13
tags: [笔记, Linux, 硬件信息查询命令]
---

# Linux 硬件信息查询命令速查表

## 1. 系统概览 (最常用)
快速查看系统软硬件总体情况。

| 命令 | 说明 | 备注 |
| :--- | :--- | :--- |
| `lshw -short` | **显示硬件摘要列表** | 需 root 权限，清晰列出 CPU、内存、磁盘、网卡 |
| `inxi -Fxz` | 显示极详细的软硬件报告 | `-z` 隐藏 IP 等隐私信息 (需安装 `inxi`) |
| `uname -a` | 显示内核版本、架构、主机名 | 基础命令 |
| `neofetch` | 终端显示系统 Logo 及配置概览 | 适合截图装逼 (需安装) |

---

## 2. CPU 信息
| 命令 | 说明 | 常用参数 |
| :--- | :--- | :--- |
| `lscpu` | **查看 CPU 架构信息** (最推荐) | 显示核心数、线程数、架构、频率 |
| `cat /proc/cpuinfo` | 查看 CPU 详细详情 | 脚本常用来分析每个核心的详情 |
| `lshw -C cpu` | 查看 CPU 硬件类信息 | 显示产品名称、供应商、功能特性 |

**常用过滤技巧：**
```bash
# 查看 CPU 型号
lscpu | grep "Model name"

# 查看 CPU 是否支持 64 位 (lm = Long Mode)
grep -o "lm" /proc/cpuinfo | uniq
```

---

## 3. 内存 (RAM)
| 命令 | 说明 | 备注 |
| :--- | :--- | :--- |
| `free -h` | **查看内存使用量** (最常用) | `-h` 以 GB/MB 显示，易读 |
| `dmidecode -t memory` | 查看**物理内存条**详情 | 需 sudo，显示插槽数、频率、类型(DDR4/5) |
| `vmstat -s` | 查看内存统计详情 | 包括分页、交换分区活动等 |

**常用场景：**
```bash
# 查看支持的最大内存容量
sudo dmidecode -t memory | grep "Maximum Capacity"

# 检查是否有空闲内存插槽 (升级内存用)
sudo lshw -short -C memory | grep -i empty
```

---

## 4. 磁盘与存储
| 命令 | 说明 | 备注 |
| :--- | :--- | :--- |
| `lsblk` | **列出块设备及分区树** (最推荐) | 清晰展示磁盘->分区->挂载点结构 |
| `df -h` | 查看文件系统**磁盘空间占用** | 显示已用/可用空间 |
| `fdisk -l` | 查看分区表详情 | 需 sudo，显示扇区、分区类型 ID |
| `blkid` | 查看分区的 UUID 和文件系统类型 | 挂载磁盘时常用 |
| `smartctl -a /dev/sda`| 查看磁盘健康状态 (SMART) | 需安装 `smartmontools` |

**常用场景：**
```bash
# 查看硬盘型号和序列号 (SATA盘)
sudo hdparm -i /dev/sda
```

---

## 5. 网络设备
*注：`ifconfig` 和 `netstat` 已逐渐被 `ip` 和 `ss` 取代。*

| 命令 | 说明 | 替代/备注 |
| :--- | :--- | :--- |
| `ip addr` | **查看 IP 地址和接口** | 替代 `ifconfig` |
| `ip route` | 查看路由表/网关 | 替代 `netstat -r` |
| `ip link` | 查看网卡物理状态 (MAC, UP/DOWN) | - |
| `lshw -C network` | 查看网卡硬件详情 | 显示驱动、芯片型号 |
| `ss -lntp` | 查看监听端口及对应进程 | 替代 `netstat -nalp` |
| `ethtool eth0` | 查看网卡底层参数 | 需安装，查看网线连接速度、双工模式 |

---

## 6. 其他硬件 (主板/USB/PCI)
| 命令 | 说明 | 备注 |
| :--- | :--- | :--- |
| `lspci` | **列出 PCI 设备** | 显卡、网卡、声卡通常走 PCI 总线 |
| `lsusb` | **列出 USB 设备** | 鼠标、键盘、外接存储 |
| `dmidecode -t bios` | 查看 BIOS/UEFI 版本 | 升级 BIOS 前查询 |
| `dmidecode -t baseboard` | 查看主板型号 | 攒机、升级驱动用 |

**查看显卡显存技巧：**
```bash
# 1. 找到显卡设备号
lspci | grep -i vga
# 2. 查看详细信息 (假设设备号是 00:02.0)
lspci -v -s 00:02.0 | grep -i "prefetchable"
```

---

## 7. 快速查询速查表 (Cheat Sheet)

| 想要查询的内容 | 推荐命令 |
| :--- | :--- |
| **整体硬件摘要** | `sudo lshw -short` |
| **CPU 型号/核心** | `lscpu` |
| **内存剩余/占用** | `free -h` |
| **内存条规格/插槽** | `sudo dmidecode -t memory` |
| **磁盘分区/挂载** | `lsblk` |
| **磁盘剩余空间** | `df -h` |
| **磁盘 UUID** | `sudo blkid` |
| **IP 地址** | `ip addr` |
| **网关/路由** | `ip route` |
| **主板型号** | `sudo dmidecode -t baseboard` |
| **BIOS 版本** | `sudo dmidecode -t bios` |
| **PCI 设备 (显卡等)** | `lspci` |
| **USB 设备** | `lsusb` |
| **内核版本** | `uname -r` |
| **发行版版本** | `cat /etc/os-release` |