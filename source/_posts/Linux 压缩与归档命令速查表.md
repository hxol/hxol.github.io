---
title: Linux 压缩与归档命令速查表
date: 2026-10-01 15:30:10
tags: [笔记, Linux, 压缩, tar]
---


# Linux 压缩与归档命令速查表

## 1. tar (最常用：打包 + 压缩)
`tar` 本身只是打包工具（归档），但常结合 `gzip`, `bzip2`, `xz` 共同使用。
**特点**：保留文件权限，支持多文件/目录。现代 `tar` 命令通常能自动识别压缩格式。

### 常用参数说明
| 参数 | 含义 |
| :--- | :--- |
| `-c` | **C**reate，建立新的归档文件 |
| `-x` | E**x**tract，解压/提取文件 |
| `-v` | **V**erbose，显示详细过程 |
| `-f` | **F**ile，指定归档文件名 (通常必选且放在最后) |
| `-z` | 调用 **gzip** 压缩/解压 (`.tar.gz`) |
| `-j` | 调用 **bzip2** 压缩/解压 (`.tar.bz2`) |
| `-J` | 调用 **xz** 压缩/解压 (`.tar.xz`) |
| `-C` | **C**hange directory，切换解压到的目录 |
| `-t` | Lis**t**，查看压缩包内容 |

### 1.1 智能解压 (推荐)
无需记忆 `-z/j/J`，tar 会自动根据后缀识别格式。
```bash
# 解压到当前目录
tar -xvf filename.tar.gz
tar -xvf filename.tar.bz2
tar -xvf filename.tar.xz

# 解压到指定目录 (目录需存在)
tar -xvf filename.tar.gz -C /path/to/target_dir
```

### 1.2 压缩 (打包)
```bash
# 打包为 .tar.gz (速度快，兼容性好，最常用)
tar -zcvf filename.tar.gz dir_name

# 打包为 .tar.bz2 (压缩率比 gzip 高，速度稍慢)
tar -jcvf filename.tar.bz2 dir_name

# 打包为 .tar.xz (压缩率极高，速度最慢)
tar -Jcphvf filename.tar.xz dir_name
```

### 1.3 查看包内容 (不解压)
```bash
tar -tvf filename.tar.gz
```

### 1.4 GNU tar 打包方案
打包并压缩
```bash
tar --xattrs --acls --selinux --numeric-owner -cpzf archive.tar.gz /path/to/dir
```
解压
```bash
tar --xattrs --acls --selinux -xpf archive.tar
```

### 1.5 
---

## 2. zip / unzip (跨平台通用)
**特点**：Windows/Mac/Linux 通用格式，支持压缩目录，保留原文件。

### 2.1 解压 (unzip)
```bash
# 解压到当前目录
unzip file.zip

# 解压到指定目录 (-d)
unzip file.zip -d /path/to/directory

# 查看压缩包内容 (不解压)
unzip -l file.zip
```

### 2.2 压缩 (zip)
```bash
# 压缩单个文件
zip file.zip file.txt

# 递归压缩目录 (-r)
zip -r file.zip folder_name
```

---

## 3. p7zip (高压缩率工具)
**特点**：支持 `.7z` (高压缩率) 及解压 `.zip`, `.rar` 等多种格式。
需安装：`sudo apt install p7zip-full`

### 3.1 解压 (7z)
```bash
# 解压并保持目录结构 (推荐，eXtract)
7z x file.7z

# 解压到指定目录 (-o 紧跟目录名，无空格)
7z x file.7z -o/path/to/output

# 解压并将所有文件平铺到当前目录 (不保留目录结构，慎用)
7z e file.7z
```

### 3.2 压缩
```bash
# 压缩文件或目录 (Add)
7z a file.7z file_or_folder
```

### 3.3 查看
```bash
# 查看档案内容
7z l file.7z
```

---

## 4. 单文件压缩工具 (gzip, bzip2, xz)
**特点**：通常**只能压缩单个文件**，不能压缩目录。默认**不保留**源文件（除非加参数）。

### 4.1 gzip (.gz)
最快，压缩率一般。
```bash
# 压缩 (file.txt -> file.txt.gz，源文件消失)
gzip file.txt

# 压缩但保留源文件
gzip -k file.txt

# 解压
gzip -d file.txt.gz
# 或者
gunzip file.txt.gz
```

### 4.2 bzip2 (.bz2)
压缩率优于 gzip。
```bash
# 压缩 (file.txt -> file.txt.bz2)
bzip2 file.txt

# 压缩但保留源文件
bzip2 -k file.txt

# 解压
bzip2 -d file.txt.bz2
# 或者
bunzip2 file.txt.bz2
```

### 4.3 xz (.xz)
压缩率最高，耗时最长。
```bash
# 压缩
xz -z file.txt

# 压缩但保留源文件
xz -k file.txt

# 解压
xz -d file.txt.xz
```

---

## 5. rar
**特点**：Windows 常用格式。Linux 下常用 `unrar` 解压。
需安装：`sudo apt install unrar`

### 5.1 解压
```bash
# 解压并保持目录结构 (推荐)
unrar x file.rar

# 解压到指定目录
unrar x file.rar /path/to/output

# 解压带密码的文件
unrar x -pYOUR_PASSWORD file.rar
# 或者 (回车后输入密码)
unrar x -p file.rar
```

### 5.2 分卷解压
只需指定第一个分卷（如 `.part1.rar` 或 `.r00`），程序会自动处理后续分卷。
```bash
unrar x file.part1.rar
```