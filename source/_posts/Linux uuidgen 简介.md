---
title: Linux uuidgen 简介
date: 2026-10-01 15:30:16
tags: [笔记, Linux, uuidgen]
---

# Linux uuidgen 简介

uuidgen 是一个用于生成 128 位通用唯一标识符（UUID）的命令行工具。

## 基本用法

在终端中直接输入命令即可生成一个基于随机数或时间戳的 UUID：
```bash
uuidgen
```
输出示例：
```
1b4e28ba-2fa1-11d2-88f5-0080c7f54f6e
```

## 常用选项

* -r 或 --random：基于随机数生成 UUID（通常为 v4 版本）。
* -t 或 --time：基于当前时间和 MAC 地址生成 UUID（通常为 v1 版本）。
* -h 或 --help：显示帮助信息。
* -V 或 --version：显示版本信息。
