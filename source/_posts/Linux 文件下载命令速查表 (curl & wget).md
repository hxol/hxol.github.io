---
title: Linux 文件下载命令速查表 (curl & wget)
date: 2026-10-01 15:30:11
tags: [笔记, Linux, curl, wget]
---

# Linux 文件下载命令速查表 (curl & wget)

## 1. curl (Client URL)
**特点**：支持协议多 (HTTP, FTP, SCP 等)，常用于 API 交互和脚本中的单文件下载。默认输出到屏幕（标准输出），**下载文件必须指定参数**。

### 1.1 常用参数速查
| 参数 | 含义 |
| :--- | :--- |
| `-o filename` | (小写) 将输出写入**指定文件名** |
| `-O` | (大写) 使用**远程文件名**保存 (Remote Name) |
| `-L` | 跟随重定向 (Location, 如下载链接跳转时必加) |
| `-C -` | 断点续传 (自动计算断点位置) |
| `-s` | 静默模式 (Silent, 不显示进度条) |
| `-f` | 连接失败时不显示 HTTP 错误 (Fail silently) |
| `-S` | 在静默模式下，如果有错误则显示错误 (Show error) |

### 1.2 常用场景
**场景 1：基础下载与重命名**
```bash
# 使用远程文件名保存 (file.iso)
curl -O http://example.com/file.iso

# 重命名保存为 my_file.iso (注意是小写 -o)
curl -o my_file.iso http://example.com/file.iso
```

**场景 2：脚本/安装常用组合 (推荐)**
在安装脚本中极常见的写法，确保下载静默但出错时有提示，且支持重定向。
```bash
# -fsSL: 失败不报错HTML(-f), 静默(-s), 显示报错(-S), 跟随跳转(-L)
curl -fsSL https://github.com/v2fly/v2ray-core/.../v2ray.zip -o v2ray.zip
```

**场景 3：断点续传与限速**
```bash
# -C - : 自动寻找断点开始续传
# --limit-rate : 限制下载速度
curl -O -C - --limit-rate 50k http://man.linuxde.net/text.iso
```

---

## 2. wget (Web Get)
**特点**：非交互式下载器，擅长**网络环境差**时的下载（自动重试），以及**递归下载**（下载整个网站）。默认直接保存文件。

### 2.1 常用参数速查
| 参数 | 含义 |
| :--- | :--- |
| `-O filename` | (大写) 将输出写入**指定文件名** |
| `-c` | 断点续传 (Continue) |
| `-b` | 后台下载 (Background) |
| `-P dir` | 指定下载目录 (Prefix) |
| `-r` | 递归下载 (Recursive) |
| `-p` | 下载页面所需所有元素 (Page requisites, 如图片/CSS) |
| `-m` | 镜像网站 (Mirror, 等同于 -r -N -l inf --no-remove-listing) |

### 2.2 常用场景
**场景 1：基础下载与重命名**
注意：wget 的重命名是**大写** `-O`，而 curl 是小写 `-o`。
```bash
# 直接下载
wget http://www.linuxde.net/text.iso

# 重命名保存
wget -O rename.iso http://www.linuxde.net/text.iso

# 下载到指定目录 (-P)
wget -P /var/www/html http://example.com/image.jpg
```

**场景 2：断点续传与限速**
非常适合大文件下载。
```bash
wget -c --limit-rate=50k http://www.linuxde.net/text.iso
```

**场景 3：后台下载**
对于超大文件，可以让其在后台运行，下载日志会写入 `wget-log` 文件。
```bash
wget -b http://example.com/big-file.iso
```

**场景 4：打包下载整个网站 (镜像)**
用于离线浏览或备份站点。
```bash
# --mirror: 开启镜像模式
# -p: 下载页面显示所需的所有文件 (图片, css, js)
# --convert-links: 将链接转换为本地链接，以便离线浏览
# -P: 指定保存目录
wget --mirror -p --convert-links -P /var/www/html http://man.linuxde.net/
```

---

## 3. 区别与总结 (curl vs wget)

| 功能 | **curl** | **wget** |
| :--- | :--- | :--- |
| **重命名参数** | `-o` (小写) | `-O` (大写) |
| **默认行为** | 输出到屏幕 (需加参数保存) | 保存为文件 |
| **断点续传** | `-C -` | `-c` |
| **递归/镜像** | 不支持 (主要用于单文件/API) | **强项** (支持 --mirror) |
| **API 调试** | **强项** (支持 POST, Header 等) | 较弱 |