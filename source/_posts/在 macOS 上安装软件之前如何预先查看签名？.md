---
title: 在 macOS 上安装软件之前如何预先查看签名？
date: 2026-10-01 15:30:25
tags: [笔记, macOS, 签名]
---


# 在 macOS 上安装软件之前如何预先查看签名？

在 macOS 中，你可以使用 `codesign` 工具来验证其他程序的签名。

以下是一个基本的步骤：

1. **打开终端**：你可以通过 Spotlight 搜索“终端”来打开它。
2. **运行 `codesign` 命令**：使用以下命令来验证程序的签名：

   ```bash
   codesign --verify --deep --strict /path/to/your/application.app
   ```

   其中 `/path/to/your/application.app` 是你要验证的应用程序的路径。

3. **检查输出**：如果签名有效，你会看到类似于 `valid on disk` 和 `satisfies its Designated Requirement` 的输出。如果签名无效，终端会显示错误信息。

你也可以使用 `spctl` 命令来验证签名：

```bash
spctl --assess --type execute /path/to/your/application.app
```

这个命令会返回应用程序是否被允许执行的信息。

参考信息

`codesign` 命令有许多选项，可以用来执行不同的操作和修改其行为。以下是一些常用的选项：

1. **签名相关选项**：
   - `-s identity`：使用指定的身份（证书）对文件进行签名。
   - `-f`：强制覆盖现有签名。
   - `-v`：详细模式，显示更多信息。

2. **验证相关选项**：
   - `-v`：验证签名。
   - `-R=<req string>`：使用指定的要求字符串进行验证。
   - `--deep`：递归验证嵌套的代码签名。

3. **显示相关选项**：
   - `-d`：显示签名信息。
   - `-vv`：详细显示签名信息。
   - `--entitlements`：显示签名中的授权信息。

4. **其他选项**：
   - `-o flags`：设置签名标志。
   - `-r reqs`：指定签名要求。
   - `-i ident`：设置签名标识符。

你可以使用 `codesign --help` 命令来查看所有可用的选项和详细说明。