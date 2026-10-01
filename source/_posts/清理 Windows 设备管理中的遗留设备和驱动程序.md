---
title: 清理 Windows 设备管理中的遗留设备和驱动程序
date: 2026-10-01 15:30:27
tags: [笔记, Windows, 驱动程序]
---

# 清理 Windows 设备管理中的遗留设备和驱动程序


运行
```
rundll32.exe C:\windows\system32\pnpclean.dll,RunDLL_PnpClean /Devices /Maxclean
```

这条命令用于清理 Windows 设备管理中的遗留设备和驱动程序。具体来说，它会：

- /Devices：移除系统中已丢失的设备。

- /Maxclean：执行最大程度的清理，使所有未连接的设备都被处理并移除。

这个命令通常用于维护系统的设备管理，确保不必要的设备和驱动不会影响系统性能或占用资源。微软曾在 Windows 更新可靠性工具中使用过这个命令，但有时可能会导致 USB 设备驱动被意外删除。

如果你想更安全地使用它，可以先运行 
```
rundll32.exe C:\windows\system32\pnpclean.dll,RunDLL_PnpClean /Devices
```
然后检查系统是否有任何影响，再决定是否使用 `/Maxclean` 选项。


[微软文档](https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/usb-device-drivers-removed-unexpectedly-after-windows-10-update)