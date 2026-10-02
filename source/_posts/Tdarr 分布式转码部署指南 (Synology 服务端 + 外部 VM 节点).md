---
title: Tdarr 分布式转码部署指南 (Synology 服务端 + 外部 VM 节点)
date: 2026-09-30 15:30:00
tags: [笔记, Tdarr, Synology, 转码, 分布式, 自托管]
---


# Tdarr 分布式转码部署指南 (Synology 服务端 + 外部 VM 节点)

## 项目简介

[Tdarr](https://docs.tdarr.io/docs/welcome/what) 是一款分布式的媒体转码与优化自动化工具。本指南介绍了如何在 Synology (群晖) NAS 上部署 Tdarr 服务端与本地节点，并通过 NFS 挂载的方式，引入性能更强的外部虚拟机（如 Intel N100 小主机）作为独立转码节点。

> **⚠️ 核心避坑说明 (关于 Unmapped 节点)：**
> Tdarr 的 `Unmapped` 节点（无需路径映射，通过网络传输文件进行转码）属于 **Pro 收费功能**。对于免费用户，**必须**使用 `Mapped` 模式。这意味着：
> 1. 外部节点必须通过 NFS/SMB 等方式挂载 NAS 上的媒体目录。
> 2. **所有节点容器内的挂载路径（即 `docker-compose.yml` 中 `:` 右侧的路径），必须与服务端的路径完全一致**。

---

## 目录

- [第一部分：Synology DSM (服务端 + 本地节点)](#第一部分synology-dsm-服务端--本地节点)
  - [1. 配置 NFS 服务](#1-配置-nfs-服务)
  - [2. 配置防火墙规则](#2-配置防火墙规则)
  - [3. 目录与权限准备](#3-目录与权限准备)
  - [4. Docker 部署配置](#4-docker-部署配置)
- [第二部分：外部虚拟机节点配置 (LXC/Linux)](#第二部分外部虚拟机节点配置-lxclinux)
  - [1. 配置 NFS 客户端与挂载](#1-配置-nfs-客户端与挂载)
  - [2. 节点目录与权限准备](#2-节点目录与权限准备)
  - [3. Docker 部署配置](#3-docker-部署配置)

---

## 第一部分：Synology DSM (服务端 + 本地节点)

### 1. 配置 NFS 服务

为了让外部节点能够读取并写入转码后的视频，NAS 必须开启 NFS 服务并配置对应权限。

**开启 NFS 基础服务：**
1. 进入 `控制面板` > `文件服务` > `NFS`。
2. 勾选 **启用 NFS 服务**。
3. 单击 **应用** 保存设置。

**配置共享文件夹的 NFS 权限：**
1. 进入 `控制面板` > `共享文件夹`。
2. 选择存放视频的共享文件夹，点击 **编辑**。
3. 进入 **NFS 权限** 选项卡，点击 **新增**：
   - **主机名或 IP**：填写外部虚拟机节点的 IP 地址（例如 `192.168.x.x`）或整个网段（如 `192.168.x.0/24`）。
   - **权限**：可读写 (Read/Write)
   - **安全性**：`sys` (AUTH_SYS)
   - **Squash**：无映射 (No mapping)
   - **启用异步**：勾选
   - **允许用户访问装载的子文件夹**：勾选
4. 点击 **保存** 以应用规则。

### 2. 配置防火墙规则

如果您的群晖开启了防火墙，请确保放行以下端口：
- **Tdarr 服务端口**：`8130` (Server Port) 和 `8129` (WebUI Port)
- **NFS 相关端口**：默认通常包含 `111` (TCP/UDP), `892` (TCP/UDP), `2049` (TCP/UDP)

### 3. 目录与权限准备

**创建映射目录：**
（假设您的 Docker 数据存放在 `/volume1/docker`，媒体缓存存放在 `/volume1/docker_cache`，请根据实际情况替换卷标）

```bash
# 创建配置及日志目录
mkdir -p /volume1/docker/tdarr/{server,server_configs,server_logs,node_configs,node_logs}

# 创建转码缓存目录 (建议放置在 SSD 存储池中)
mkdir -p /volume1/docker_cache/tdarr-cache/{server_cache,node_cache}
```

**获取当前用户的 UID 和 GID：**
```bash
# 获取 uid
id -u
# 假设输出为：1026

# 获取所属组 gid
id -g
# 假设输出为：100
```

**赋予目录权限：**
```bash
sudo chown -R 1026:100 /volume1/docker/tdarr
sudo chown -R 1026:100 /volume1/docker_cache/tdarr-cache
```

### 4. Docker 部署配置

进入 Tdarr 配置目录：
```bash
cd /volume1/docker/tdarr
```

#### 创建 `.env` 环境变量文件

```bash
vi .env
```

填入以下内容：
```env
# 镜像版本
TDARR_SERVER_IMAGE=ghcr.io/haveagitgat/tdarr:latest
TDARR_NODE_IMAGE=ghcr.io/haveagitgat/tdarr_node:latest

# 资源限制
SERVER_MEM=2g
NODE_MEM=2g

# 端口配置
WEB_UI_PORT=8129
SERVER_PORT=8130

# 环境配置
TZ=Asia/Shanghai
PUID=1026  # 替换为您上一步查询到的真实 UID
PGID=100   # 替换为您上一步查询到的真实 GID

# 节点名称
NODE_NAME=Synology-Local-Node
```

#### 创建 `docker-compose.yml` 编排文件

```bash
vi docker-compose.yml
```

填入以下内容（请根据您的 NAS 实际路径调整 `:` 左侧的路径，**`:` 右侧的容器内路径切记不要随意更改，后续需保持统一**）：

```yaml
services:
  tdarr:
    container_name: tdarr
    image: ${TDARR_SERVER_IMAGE}
    restart: unless-stopped
    mem_limit: ${SERVER_MEM}
    ports:
      - ${WEB_UI_PORT}:${WEB_UI_PORT} # Web UI
      - ${SERVER_PORT}:${SERVER_PORT} # Server 端通讯端口
    environment:
      - TZ=${TZ}
      - PUID=${PUID}
      - PGID=${PGID}
      - UMASK_SET=002
      - serverIP=0.0.0.0
      - serverPort=${SERVER_PORT}
      - webUIPort=${WEB_UI_PORT}
      - internalNode=false
      - inContainer=true
      - ffmpegVersion=7
      - auth=false
      - openBrowser=false
      - maxLogSizeMB=10
    volumes:
      # 服务端配置与缓存
      - ./server:/app/server
      - ./server_configs:/app/configs
      - ./server_logs:/app/logs
      - /volume1/docker_cache/tdarr-cache/server_cache:/temp
      # 媒体目录映射 (左侧为主机实际路径，右侧为容器内路径)
      - /volume1/Knowledge:/media/Knowledge
      - /volume1/Temp:/media/Temp
      - /volume1/Videos-Movies:/media/Movies
      - /volume1/Videos-TV-series:/media/TV-series
      - /volume1/Personal:/media/Personal

  tdarr-node:
    container_name: tdarr-node
    image: ${TDARR_NODE_IMAGE}
    restart: unless-stopped
    mem_limit: ${NODE_MEM}
    depends_on:
      - tdarr
    environment:
      - TZ=${TZ}
      - PUID=${PUID}
      - PGID=${PGID}
      - UMASK_SET=002
      - nodeName=${NODE_NAME}
      - serverIP=tdarr
      - serverPort=${SERVER_PORT}
      - inContainer=true
      - ffmpegVersion=7
      - nodeType=mapped
      - priority=-1
      - startPaused=false
      - maxLogSizeMB=10
      
      # 任务进程配置 (请根据 NAS CPU 性能合理分配)
      - transcodecpuWorkers=2
      - healthcheckcpuWorkers=1

    volumes:
      # 节点配置与缓存
      - ./node_configs:/app/configs
      - ./node_logs:/app/logs
      - /volume1/docker_cache/tdarr-cache/node_cache:/temp
      
      # ⚠️ 注意：这里的媒体目录映射，冒号右侧必须与上方 tdarr 服务端的冒号右侧完全一致！
      - /volume1/Knowledge:/media/Knowledge
      - /volume1/Temp:/media/Temp
      - /volume1/Videos-Movies:/media/Movies
      - /volume1/Videos-TV-series:/media/TV-series
      - /volume1/Personal:/media/Personal
```

#### 启动服务

```bash
docker compose up -d
```
可通过 `docker compose ps` 或 `docker compose logs -f tdarr` 查看运行状态。

---

## 第二部分：外部虚拟机节点配置 (LXC/Linux)

利用外部设备（如 Intel N100）强大的核显性能作为独立转码节点，是提升 Tdarr 转码效率的最佳实践。

### 1. 配置 NFS 客户端与挂载

进入外部虚拟机的终端（以 Ubuntu/Debian 为例）：

```bash
# 安装 NFS 客户端依赖
sudo apt update && sudo apt install -y nfs-common
```

#### 方案 A：脚本一键挂载 (临时使用)
创建一个挂载脚本 `mount_nas.sh`：
```bash
#!/bin/bash
NAS_IP="192.168.x.x"  # 替换为您的群晖 IP

echo "1. 正在创建挂载目录..."
sudo mkdir -p /mnt/nfs/{Knowledge,Temp,Videos-Movies,Videos-TV-series,Personal}

echo "2. 开始挂载 NFS 共享..."
sudo mount -t nfs ${NAS_IP}:/volume1/Knowledge /mnt/nfs/Knowledge
sudo mount -t nfs ${NAS_IP}:/volume1/Temp /mnt/nfs/Temp
sudo mount -t nfs ${NAS_IP}:/volume1/Videos-Movies /mnt/nfs/Videos-Movies
sudo mount -t nfs ${NAS_IP}:/volume1/Videos-TV-series /mnt/nfs/Videos-TV-series
sudo mount -t nfs ${NAS_IP}:/volume1/Personal /mnt/nfs/Personal

echo "3. 挂载完成，查看状态："
df -h | grep /mnt/nfs
```
执行脚本：`sudo bash mount_nas.sh`

#### 方案 B：修改 fstab 实现开机自启挂载 (强烈推荐)
为了防止虚拟机重启后掉盘导致 Tdarr 转码失败，建议配置开机自动挂载：
```bash
sudo nano /etc/fstab
```
在末尾添加（注意替换您的 NAS_IP 和实际路径）：
```text
192.168.x.x:/volume1/Knowledge     /mnt/nfs/Knowledge       nfs defaults,timeo=900,retrans=5,_netdev 0 0
192.168.x.x:/volume1/Temp          /mnt/nfs/Temp            nfs defaults,timeo=900,retrans=5,_netdev 0 0
192.168.x.x:/volume1/Videos-Movies /mnt/nfs/Videos-Movies   nfs defaults,timeo=900,retrans=5,_netdev 0 0
192.168.x.x:/volume1/Videos-TV-series /mnt/nfs/Videos-TV-series nfs defaults,timeo=900,retrans=5,_netdev 0 0
192.168.x.x:/volume1/Personal      /mnt/nfs/Personal        nfs defaults,timeo=900,retrans=5,_netdev 0 0
```
保存后执行 `sudo mount -a` 即可生效。

### 2. 节点目录与权限准备

在外部虚拟机上创建工作目录：
```bash
sudo mkdir -p /opt/tdarr/{node_configs,node_logs,node_cache}
cd /opt/tdarr
```

赋予权限（这里的 `1000:1000` 需对应当前运行 Docker 的普通用户，但请**确保该用户对上述挂载的 `/mnt/nfs` 拥有读写权限**）：
```bash
sudo chown -R 1000:1000 /opt/tdarr
sudo chmod -R 755 /opt/tdarr
```

### 3. Docker 部署配置

#### 创建 `.env` 文件

```bash
sudo nano /opt/tdarr/.env
```
内容如下：
```env
TDARR_NODE_IMAGE=ghcr.io/haveagitgat/tdarr_node:latest
CONTAINER_NAME=tdarr-node-external
NODE_MEM=2g
SERVER_PORT=8130
TZ=Asia/Shanghai

# ⚠️ 此处建议与 NAS 上的 UID/GID 保持一致，避免产生权限问题。
# 若 NAS 上是 1026:100，这里也填 1026 和 100。
PUID=1026
PGID=100

NODE_NAME=External-VM-Node-N100
MASTER_IP=192.168.x.x  # 填入群晖 NAS 的 IP
```

#### 创建 `docker-compose.yml`

```bash
sudo nano /opt/tdarr/docker-compose.yml
```
内容如下：
```yaml
services:
  tdarr-node:
    container_name: ${CONTAINER_NAME}
    image: ${TDARR_NODE_IMAGE}
    restart: unless-stopped
    mem_limit: ${NODE_MEM}
    privileged: true # 开启特权模式以调用硬件 GPU

    environment:
      - TZ=${TZ}
      - PUID=${PUID}
      - PGID=${PGID}
      - UMASK_SET=002
      - nodeName=${NODE_NAME}
      - serverIP=${MASTER_IP}
      - serverPort=${SERVER_PORT}
      - inContainer=true
      - ffmpegVersion=7
      - nodeType=mapped
      - priority=-1
      - startPaused=false
      - maxLogSizeMB=10

      ######## GPU 及工作流相关设置 ########
      # N100 核显性能较强，通常可开 2-3 个 GPU 转码进程
      - transcodegpuWorkers=2
      # 保留 1 个 CPU 进程用于处理无需 GPU 的任务（如提取字幕、音频转码等）
      - transcodecpuWorkers=1
      # 健康检查进程
      - healthcheckgpuWorkers=1
      - healthcheckcpuWorkers=1

    ######## GPU 设备映射 ########
    ### 用于 Intel QuickSync 硬件加速 (N100 必备)
    devices:
      - /dev/dri:/dev/dri

    volumes:
      # 虚拟机本地工作目录 (配置、日志及缓存)
      - ./node_configs:/app/configs
      - ./node_logs:/app/logs
      # 建议在虚拟机的高速硬盘(SSD)上专门建立一个缓存目录
      - ./node_cache:/temp

      # ⚠️ 媒体库映射：至关重要！
      # 冒号左侧：指向本机刚挂载的 /mnt/nfs/... 路径
      # 冒号右侧：必须与群晖 NAS 上 docker-compose 配置的容器内路径【一字不差】
      - /mnt/nfs/Knowledge:/media/Knowledge
      - /mnt/nfs/Temp:/media/Temp
      - /mnt/nfs/Videos-Movies:/media/Movies
      - /mnt/nfs/Videos-TV-series:/media/TV-series
      - /mnt/nfs/Personal:/media/Personal
```

#### 启动节点

确认群晖服务端的 Tdarr 正在运行后，在外部虚拟机中执行：
```bash
cd /opt/tdarr && sudo docker compose up -d
```
启动后，打开 Tdarr 的 WebUI (群晖IP:8129)，在 `Nodes` 界面即可看到名为 `External-VM-Node-N100` 的新节点已上线并准备就绪。
