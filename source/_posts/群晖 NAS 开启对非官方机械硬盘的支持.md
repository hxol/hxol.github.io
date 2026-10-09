---
title: 群晖 NAS 开启对非官方机械硬盘的支持
date: 2026-10-04 18:57:49
tags: [笔记, 群晖, NAS, 机械硬盘]
---


# 群晖 NAS 开启对非官方机械硬盘的支持

群晖 plus 系列（所有的所谓企业级）的 NAS 现在只支持群晖贴牌的所谓官方机械硬盘。虽然对此痛恨不已，也已经下定决心下一台 NAS 绝不选群晖。但是没坏之前还是得解决问题。

## 1. 备份关键文件

```bash
cp -a /etc.defaults/synoinfo.conf /etc.defaults/synoinfo.conf.bak && cp -a /var/lib/disk-compatibility /var/lib/disk-compatibility.bak
```

## 2. 关闭硬盘兼容性检查

```bash
/usr/syno/bin/synosetkeyvalue /etc.defaults/synoinfo.conf support_disk_compatibility no
```

## 3. 让 DSM 重新检查磁盘兼容性

```bash
/usr/syno/sbin/synostgdisk --check-all-disks-compatibility
```

## 4. 重启 DSM

## 5. 打开“存储管理器”，到“存储池”里添加硬盘。
