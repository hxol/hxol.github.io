---
title: Homebrew 简单用法
date: 2026-10-01 15:30:29
tags: [笔记, macOS, Homebrew]
---

# Homebrew 简单用法

## 安装

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## brew 常用命令

软件查询与搜索
* brew search <package>：搜索指定的命令行软件包。
* brew search --cask <app>：搜索带图形界面（GUI）的 macOS 应用程序。
* brew info <package>：查看某个包的详细信息（版本、依赖、主页等）。

软件安装
* brew install <package>：安装命令行工具或软件库。
* brew install --cask <app>：安装图形界面应用（如 Chrome、QQ 等）。
* brew install <package>@<version>：安装指定版本的软件。

软件卸载与清理
* brew uninstall <package>：卸载指定的软件包。
* brew uninstall --cask <app>：卸载图形界面应用。
* brew cleanup：清理所有已安装包的旧版本缓存和临时文件。

更新与升级
* brew update：更新 Homebrew 自身及其核心脚本。
* brew outdated：列出所有需要更新（过时）的软件。
* brew upgrade：升级所有过时的软件。
* brew upgrade <package>：升级指定的某一个软件。

查看已安装列表
* brew list：查看当前已安装的软件包列表。
* brew list --versions：查看已安装包的具体版本号。
* brew deps <package>：查看某个包的依赖关系。

后台服务管理
* brew services list：查看由 brew 管理的后台服务运行状态。
* brew services start <service>：启动指定的后台服务。
* brew services stop <service>：停止指定的后台服务。
* brew services restart <service>：重启指定的后台服务。

其他常用命令
* brew -v / brew --version：查看当前 Homebrew 的版本号。
* brew doctor：检查当前系统中的 Homebrew 配置是否有异常或冲突。 
