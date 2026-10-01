---
title: Linux base64 命令
date: 2026-10-01 15:30:08
tags: [笔记, Linux, base64]
---

# Linux base64 命令

## 用法

```bash
base64 [选项]... [文件]
```

使用 Base64 编码/解码文件或标准输入输出。
```
  -d, --decode          解码数据
  -i, --ignore-garbag   解码时忽略非字母字符
  -w, --wrap=字符数     在指定的字符数后自动换行(默认为76)，0 为禁用自动换行

      --help            显示此帮助信息并退出
      --version         显示版本信息并退出
```

如果没有指定文件，或者文件为"-"，则从标准输入读取。

数据以 RFC 3548 规定的 Base64 字母格式进行编码。 解码时，输入数据(加密流)可能包含一些非有效 Base64 字符的新行字符。可以尝试用 --ignore-garbage 选项来恢复加密流中任何非 base64 字符。

## 标准输入读取加密

把字符Hello World使用base64加密：
```bash
echo Hello World | base64
```
结果输出：
```
SGVsbG8gV29ybGQK
```

## 标准输入读取解密

把base64字符SGVsbG8gV29ybGQK解密出来：
```bash
echo SGVsbG8gV29ybGQK | base64 -d
```
结果：
```
Hello World
```

## 文件加密

把写有明文的文件1.txt用base64加密后输出到2.txt：
```bash
base64 1.txt > 2.txt
```

## 文件解密
把写有加密字符的文件2.txt解密后输出到文件1.txt
```bash
base64 -d 2.txt > 1.txt
```

