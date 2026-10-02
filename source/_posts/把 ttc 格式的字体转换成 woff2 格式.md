---
title: 把 ttc 格式的字体转换成 woff2 格式
date: 2026-10-01 15:30:23
tags: [笔记, 字体, jellyfin, ttc, woff2]
---

# 把 ttc 格式的字体转换成 woff2 格式

jellyfin 需要 woff2 格式的字体。

## 安装 woff2 工具
```bash
sudo apt-get install woff2
```

## 提取 `.ttc` 文件中的字体

`.ttc` 文件是字体集合文件，可能包含多个字体。你需要先将 `.ttc` 文件中的字体提取出来。

可以使用 `fonttools` 来完成这一步：
```bash
pip install fonttools
ttx -s <your-font-file.ttc>
```
这将生成多个 .ttf 文件。


## 将提取的 `.ttf` 文件转换为 `.woff2` 文件：
```bash
woff2_compress <your-font-file.ttf>
```




