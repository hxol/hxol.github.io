---
title: 针对老旧电脑 Lenovo M93p 的 Linux 系统优化指南
date: 2026-10-01 15:30:28
tags: [笔记, Linux, m93p]
---


# 针对老旧电脑 Lenovo M93p 的 Linux 系统优化指南

Lenovo M93p 搭载的是第四代 Intel (Haswell) 处理器。在现代 Linux 系统下，该平台的老旧核显（Intel HD Graphics 4600）常会遇到闪屏、休眠唤醒黑屏、或因持续检测显示接口引发的系统周期性微卡顿。

以下方案通过对内核及驱动参数的调整，系统性地解决了这些痛点。

## 核心参数优化 (GRUB)

为了方便日后随时增删，建议将优化参数分类定义为独立的变量，再进行集中引用。这不仅提升了可读性，还能避免因为参数过长导致配置错误。

**编辑 GRUB 配置文件**
```bash
sudo nano /etc/default/grub
```

**修改与补充**
找到 `GRUB_CMDLINE_LINUX_DEFAULT` 与 `GRUB_CMDLINE_LINUX` 配置项，将其替换为如下结构：

```bash
# i915 核显专用优化参数
I915_OPTS="drm_kms_helper.poll=0 i915.enable_psr=0 i915.enable_fbc=0 i915.enable_dc=0"

# 虚拟化与 IOMMU 优化参数
IOMMU_OPTS="intel_iommu=on,igfx_off"

# 集中拼接到默认启动项中
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash $I915_OPTS $IOMMU_OPTS"
GRUB_CMDLINE_LINUX=""
```

**参数原理解析（方便您随时按需增删）：**
* **`drm_kms_helper.poll=0`**：关闭显示接口的轮询检测。可彻底解决因主板不断检测未连接的显示器接口而引发的 `kworker` CPU 占用飙升及系统微卡顿。
* **`i915.enable_psr=0`**：关闭面板自刷新（Panel Self Refresh）。这是修复 Haswell 平台屏幕频繁闪烁的最有效手段。
* **`i915.enable_fbc=0`**：关闭帧缓冲压缩（Framebuffer Compression）。可防止高负载下偶发的画面撕裂与花屏。
* **`i915.enable_dc=0`**：关闭显示电源深度休眠状态（Display C-states）。修复电脑从睡眠状态唤醒时，显示器无法点亮（黑屏死机）的顽疾。
* **`intel_iommu=on,igfx_off`**（*替代您原有的 `intel_iommu=off`*）：原方案直接一刀切禁用了整体的 VT-d 虚拟化，会导致无法正常使用虚拟机直通等高级功能。新参数的作用是**仅针对集成显卡关闭 IOMMU**，既防止了老核显引发的 DMAR 报错与内核崩溃，又完美保留了系统整体的虚拟化性能。

---

## 应用并使配置生效

无论修改了 GRUB 还是其他底层配置，都需要重新生成引导菜单与内核初始化映像。在终端执行以下命令：

**1. 更新 GRUB 引导菜单**
```bash
sudo update-grub
```

**2. 更新内核初始化环境**
（*注：日常通过包管理器升级内核时系统会自动执行，但手动优化后需立即执行一次*）
```bash
sudo update-initramfs -u -k all
```

执行完毕后，**重启系统**，所有针对 Lenovo M93p 的优化即可生效。