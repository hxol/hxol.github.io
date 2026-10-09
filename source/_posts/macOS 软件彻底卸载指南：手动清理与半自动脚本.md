---
title: macOS 软件彻底卸载指南：手动清理与半自动脚本
date: 2026-10-09 13:02:03
tags: [笔记, macOS, bash 脚本, 卸载软件]
---




# macOS 软件彻底卸载指南：手动清理与半自动脚本

在 macOS 中，仅将软件拖入“废纸篓”通常会遗留大量的偏好设置、缓存和运行日志。本指南提供手动和半自动两种清理方式，帮助你找回被占用的磁盘空间。

## 一、 基础篇：手动清理流程

适用于偶尔卸载个别软件，原理是通过系统目录寻找并删除相关文件。

### 1. 退出应用及后台进程
*   **常规退出**：若软件仍在运行，按 `Command + Q` 退出。
*   **强制退出**：若软件卡死，按 `Option + Command + Esc` 呼出强制退出窗口，结束该程序。
*   **清理残留进程**：打开**活动监视器**（可在启动台中搜索），搜索该软件名称，若有残留进程，点击上方“X”强制结束。
*   **清理启动项**：打开 **系统设置 -> 通用 -> 登录项**，检查并移除与该软件相关的开机启动或后台允许项目。

### 2. 删除应用主体
打开 **访达（Finder）**，进入 **应用程序** 文件夹，找到要卸载的软件，右键选择“移到废纸篓”（或直接拖入废纸篓）。

### 3. 清理深层残留文件（核心步骤）
应用产生的数据通常保存在隐藏的 `Library`（资源库）文件夹中。

在 **访达** 中按快捷键 `Command + Shift + G` 呼出“前往文件夹”，依次输入以下路径。在对应目录中，寻找包含该软件名称（或英文名、开发商名）的文件夹或文件，确认后删除：

**当前用户目录（以 `~/` 开头，共 14 个）：**
*   `~/Library/Application Support/` （核心数据，通常占空间最大）
*   `~/Library/Preferences/` （偏好设置，通常为 `.plist` 配置文件）
*   `~/Library/Caches/` （运行缓存文件）
*   `~/Library/Logs/` （运行日志）
*   `~/Library/Containers/` （沙盒应用的数据容器）
*   `~/Library/Group Containers/` （应用组共享的数据）
*   `~/Library/Saved Application State/` （应用关闭前的状态保存）
*   `~/Library/LaunchAgents/` （用户级开机或后台启动项）
*   `~/Library/WebKit/` （内嵌网页视图的缓存与数据）
*   `~/Library/HTTPStorages/` （网络请求和 HTTP 缓存）
*   `~/Library/Cookies/` （软件保存的登录凭证与网络 Cookies）
*   `~/Library/Application Scripts/` （应用的自动化脚本及权限配置）
*   `~/Library/CloudStorage/` （云盘服务的本地同步目录，如卸载网盘类软件需留意）
*   `~/Library/Autosave Information/` （软件自动保存的临时文档或数据）

**系统级目录（以 `/` 开头，共 4 个）：**
*   `/Library/Application Support/` （系统级核心数据）
*   `/Library/Preferences/` （系统级全局配置）
*   `/Library/LaunchDaemons/` （系统级后台守护进程）
*   `/Library/LaunchAgents/` （系统级启动项）

> **💡 注意事项：** 
> *   由于目录较多，如果是大型软件，以上 18 个路径可能都有残留；如果是轻量级小软件，通常只会出现在 `Application Support`、`Preferences` 和 `Caches` 这三个最主要的文件夹中。
> *   在 `~/Library` 中，按 `Command + Shift + .` 可切换显示/隐藏文件。遇到不确定的文件，**宁可保留也不要误删**。


---

## 二、 进阶篇：半自动化扫描脚本

如果需要一次性清理多个软件，或者觉得手动查找太繁琐，可以使用以下 Bash 脚本。

**安全性说明：** 这是一个**只读**脚本。它仅负责在系统目录中“搜索”并“列出”匹配的文件路径，**绝不会自动删除任何文件**。你可以根据输出的清单，再去访达中精准删除。

### 第一步：为终端授予“完全磁盘访问权限”
脚本需要扫描沙盒和深层系统目录，macOS 的隐私保护机制会拦截此行为。若不授权，脚本会报错或漏扫。

1. 打开 **系统设置 -> 隐私与安全性**。
2. 找到并进入 **完全磁盘访问权限 (Full Disk Access)**。
3. 找到 **终端 (Terminal)**，打开右侧开关（需指纹或密码解锁）。
   *(若列表中没有“终端”，点击下方 “+” 号，在“应用程序 -> 实用工具”中添加即可)*。
4. 授权后，按 `Command + Q` **彻底退出终端 App**，准备重新打开。

### 第二步：创建并运行脚本
1. 在桌面新建一个纯文本文件，命名为 `clean_search.sh`。
2. 将以下代码复制进去，**修改 `APPS` 列表为你需要清理的软件英文名**，然后保存：

```bash
#!/bin/bash

# ============================================================
# 变量区域：要清理的软件名（支持同时搜索多个）
# 建议填写软件准确的英文名、开发商名称或 Bundle ID （如 com.microsoft.OneDrive）或准确的英文名
# ============================================================
APPS=(
  "AppName1"      # 替换为你要卸载的软件名，例如 "OneDrive"
  "AppName2"      # 替换为你要卸载的软件名，例如 "Tencent"
  "DeveloperName" # 替换为开发商名称
)

# 要扫描的系统和用户资源库目录
SEARCH_PATHS=(
  "$HOME/Library/Application Support"
  "$HOME/Library/Preferences"
  "$HOME/Library/Caches"
  "$HOME/Library/Logs"
  "$HOME/Library/Containers"
  "$HOME/Library/Group Containers"
  "$HOME/Library/LaunchAgents"
  "$HOME/Library/Saved Application State"
  "$HOME/Library/WebKit"
  "$HOME/Library/HTTPStorages"
  "$HOME/Library/Cookies"
  "$HOME/Library/Application Scripts"
  "$HOME/Library/CloudStorage"
  "$HOME/Library/Autosave Information"
  "/Library/Application Support"
  "/Library/Preferences"
  "/Library/LaunchDaemons"
  "/Library/LaunchAgents"
)

echo "=============================================="
echo "  macOS 软件残留文件搜索脚本（仅搜索，不删除）"
echo "=============================================="
echo "搜索关键词: ${APPS[*]}"
echo ""

# 拼接正则表达式
PATTERN=$(IFS='|'; echo "${APPS[*]}")
FOUND_COUNT=0

for path in "${SEARCH_PATHS[@]}"; do
  if [ ! -d "$path" ]; then
    continue
  fi

  echo "----------------------------------------------"
  echo "正在搜索: $path"
  echo "----------------------------------------------"

  # 限制搜索深度为 3 (-maxdepth 3)，确保列出应用主目录，同时防止文件夹内海量细碎文件刷屏
  results=$(find "$path" -maxdepth 3 2>/dev/null | grep -iE "$PATTERN")

  if [ -n "$results" ]; then
    echo "$results"
    count=$(echo "$results" | grep -c "^")
    FOUND_COUNT=$((FOUND_COUNT + count))
  else
    echo "（无匹配）"
  fi
  echo ""
done

echo "=============================================="
echo "搜索完成。共找到约 $FOUND_COUNT 条结果。"
echo "请仔细审查上面的列表，确认无误后再前往访达手动删除。"
echo "=============================================="
```

### 第三步：执行与清理
1. 打开**终端**应用。
2. 输入 `bash `（注意 `bash` 后面有一个空格）。
3. 将桌面的 `clean_search.sh` 文件拖入终端窗口中，按下回车。
4. 终端会输出一份“残留文件清单”。
5. 根据清单上的路径，在访达中找到对应文件并手动移入废纸篓。

### 第四步：收尾
所有文件清理完毕后，请**清空废纸篓**，并**重启一次 Mac**。这能确保相关的系统底层守护进程和常驻缓存被彻底释放。