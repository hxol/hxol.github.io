---
title: Fcitx5 + Rime + 雾凇拼音
date: 2026-10-01 15:30:20
tags: [笔记, Linux, Debian, Fcitx5, Rime, 雾凇拼音, 输入法]
---


# Fcitx5 + Rime + 雾凇拼音

## 一、安装 Fcitx5 等基础软件

```bash
sudo dnf update && sudo dnf install fcitx5 fcitx5-rime librime-lua
```

## 二、安装 plum

```bash
cd ~ && git clone https://github.com/rime/plum.git plum
```

> Rime 用户目录是`~/.local/share/fcitx5/rime`


## 三、安装雾凇拼音到默认客户端（Weasel，Squirrel，iBus-rime）

更新词库只要执行这一步，4、5 步不需要。 如使用其他客户端请手动指定 rime_dir 变量为对应的用户目录。

```bash
cd ~/plum && rime_dir="$HOME/.local/share/fcitx5/rime" bash rime-install iDvel/rime-ice
```

## 四、(可选)双拼用户额外执行，全拼用户跳过

替换「double_pinyin_flypy」为你使用的方案
```bash
rime_dir="$HOME/.local/share/fcitx5/rime" bash rime-install iDvel/rime-ice:others/recipes/config:schema=double_pinyin_flypy
```

## 五、(可选)如需万象语法模型，额外执行

替换「rime_ice」为你使用的方案
```bash
rime_dir="$HOME/.local/share/fcitx5/rime" bash rime-install iDvel/rime-ice:others/recipes/grammar:schema=rime_ice
```

## 六、最后重新部署

```bash
fcitx5-remote -r
```

## 七、参考信息
```
# 方案名称
# rime_ice（雾凇拼音、全拼）
# double_pinyin（自然码双拼）
# double_pinyin_flypy（小鹤双拼）
# double_pinyin_mspy（微软双拼）
# double_pinyin_sogou（搜狗双拼）
# double_pinyin_abc（智能 ABC 双拼）
# double_pinyin_jiajia（拼音加加双拼）
# double_pinyin_ziguang（紫光双拼）
```

[官方文档](https://github.com/iDvel/rime-ice/blob/main/others/docs/Installation.md)
