---
title: Docker 部署 Code-Server 及进阶配置指南
date: 2026-09-30 15:30:00
tags: [笔记, Code-Server, Git, Gitea, 自托管]
---


# Docker 部署 Code-Server 及进阶配置指南

本文档记录了使用 Docker 部署 `code-server`（基于 LinuxServer 镜像）的完整流程，包含 NAS 挂载、权限配置、容器编排以及 SSH 密钥连接私有 Git 仓库（如 Gitea）的进阶配置。

## 1. 项目信息参考

*   [Code-Server 官方 GitHub 仓库](https://github.com/coder/code-server)
*   [Code-Server 官方 ghcr.io 镜像](https://github.com/coder/code-server/pkgs/container/code-server)
*   [Code-Server 官方 Docker Hub 镜像](https://hub.docker.com/r/codercom/code-server/tags)
*   [LinuxServer 版定制镜像](https://github.com/linuxserver/docker-code-server/pkgs/container/code-server) *(本文档默认使用此版本，该版本对权限管理和常用工具集成了较好的支持)*

---

## 2. 环境准备

### 2.1 创建项目目录
首先在宿主机创建项目所需的基础目录，用于存放配置和 SSH 密钥。

```bash
sudo mkdir -p /opt/code-server/config/.ssh
cd /opt/code-server
```

### 2.2 挂载 NAS 目录 (可选)
如果需要将 NAS 上的共享文件夹作为代码工作区，可以通过 NFS 进行挂载。

1. **安装 NFS 客户端工具**：
   ```bash
   sudo apt update && sudo apt install nfs-common -y
   ```

2. **创建本地挂载点**：
   ```bash
   sudo mkdir -p /mnt/docs
   ```

3. **配置开机自动挂载**：
   编辑 `/etc/fstab` 文件：
   ```bash
   sudo nano /etc/fstab
   ```
   在文件末尾追加以下内容（**请将 `<NAS_IP>` 和 `<NAS_PATH>` 替换为实际的 IP 和共享路径**）：
   ```text
   # 格式: <NAS_IP>:<NAS_PATH> /mnt/docs nfs 参数 0 0
   192.168.x.x:/volumeX/Documents  /mnt/docs  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=15s,soft,timeo=50,retrans=2,rw,nfsvers=4.1  0  0
   ```
   > **Tip**: 这里使用了 `systemd.automount`，可以避免因 NAS 未启动导致宿主机开机卡死的情况，它会在首次访问该目录时自动触发挂载。

4. **重载守护进程并测试挂载**：
   ```bash
   # 如果之前已经挂载过，可以先卸载 (可选)
   # sudo umount /mnt/docs
   
   # 重新加载 fstab 并启动自动挂载目标
   sudo systemctl daemon-reload
   sudo systemctl restart remote-fs.target
   
   # 验证挂载是否成功
   ls -la /mnt/docs
   ```

---

## 3. 权限配置

为了避免 Docker 产生读写权限问题，我们需要指定容器以特定的用户身份运行。

### 3.1 获取当前用户的 UID 和 GID
运行以下命令查看当前用户的 `uid` 和 `gid`（通常普通用户均为 `1000`）：
```bash
id 
```

### 3.2 设置目录权限
将目录的所有权赋予对应的用户（假设您的 UID 为 `1000`，GID 为 `1000`，请根据上一步的输出结果进行调整）：
```bash
sudo chown -R 1000:1000 /opt/code-server
sudo chmod -R 755 /opt/code-server
```

---

## 4. 编写 Docker 配置

### 4.1 环境变量文件 (`.env`)
将敏感信息和可变参数提取到环境变量中。

```bash
sudo nano /opt/code-server/.env
```
写入以下内容（**请修改为您自己的密码**）：
```env
# 镜像版本
CODE_SERVER_IMAGE=ghcr.io/linuxserver/code-server:latest

# 权限配置 (与上一步查询到的 id 一致)
APP_USER_ID=1000
APP_GROUP_ID=1000

# 时区
TZ=Asia/Shanghai

# Web 界面登录密码
APP_PASSWORD=your_secure_password
# 可选：哈希加密后的密码（更安全，若配置此项会覆盖 APP_PASSWORD）
# HASHED_PASSWORD=your_hashed_password

# PWA 应用名称
PWA_APPNAME=code-server
```

### 4.2 容器编排文件 (`docker-compose.yaml`)
创建 `docker-compose` 文件：

```bash
sudo nano /opt/code-server/docker-compose.yaml
```
写入以下内容：
```yaml
services:
  code-server:
    image: ${CODE_SERVER_IMAGE}
    container_name: code-server
    environment:
      - PUID=${APP_USER_ID}
      - PGID=${APP_GROUP_ID}
      - TZ=${TZ}
      - PASSWORD=${APP_PASSWORD}            # 必填或选填 HASHED_PASSWORD
      # - HASHED_PASSWORD=${HASHED_PASSWORD} # 可选，Web 图形用户界面密码的哈希值
      # - SUDO_PASSWORD=password             # 可选，用于容器内需要 sudo 权限的场景
      - PWA_APPNAME=${PWA_APPNAME}
    volumes:
      # 配置目录，由于前面已经创建了 .ssh，因此 .ssh 会被自动包含在内
      - ./config:/config
      # 将 NAS 挂载的目录映射为工作区
      - /mnt/docs:/config/workspace
    ports:
      - 1038:8443
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: "4G" # 注意：单位是 G 或 M，不能是 GM
```

---

## 5. 启动服务

在启动容器前，如果您已有现成的私钥需要使用，请将其上传至 `/opt/code-server/config/.ssh` 目录下（若不配置则跳过）。

启动容器：
```bash
cd /opt/code-server
sudo docker compose up -d
```
> 服务启动后，您可以通过浏览器访问 `http://<宿主机IP>:1038`，输入在 `.env` 中设置的密码即可进入网页版 VS Code。

---

## 6. 进阶：配置 SSH 连接私有 Git (以 Gitea 为例)

如果您有一台私有 Git 服务器（如 Gitea/GitLab）并在非标准端口运行 SSH 服务，可以通过配置 `~/.ssh/config` 让 `code-server` 中的 Git 操作免密且顺畅。

1. **配置公钥**：
   将您的公钥（如 `id_rsa.pub`）添加到 Gitea 服务器的 SSH 密钥设置中。

2. **配置私钥**：
   将对应的私钥（如 `id_rsa_gitea`）放入 code-server 所在宿主机的 `/opt/code-server/config/.ssh` 目录。
   ```bash
   # 确保私钥权限安全
   sudo chmod 600 /opt/code-server/config/.ssh/id_rsa_gitea
   ```

3. **创建/修改 SSH Config 文件**：
   在 `/opt/code-server/config/.ssh` 目录下创建 `config` 文件：
   ```bash
   sudo nano /opt/code-server/config/.ssh/config
   ```
   填入以下内容（**请将 IP、端口和域名替换为您实际的 Gitea 信息**）：
   ```ssh-config
   Host gitea.yourdomain.com
       HostName 192.168.y.y           # SSH 服务器的实际内网 IP 或域名
       Port 2222                      # SSH 服务使用的端口 (默认22时可省略)
       User git
       IdentityFile /config/.ssh/id_rsa_gitea
       IdentitiesOnly yes             # 仅尝试指定的密钥，不尝试其他默认密钥，避免冲突
   ```

4. **测试 SSH 连接并生成 `known_hosts`**：
   在 Code-Server 的网页端打开内部终端（Terminal），执行：
   ```bash
   ssh -T git@gitea.yourdomain.com
   ```
   *首次连接会提示验证指纹，输入 `yes` 即可生成 `known_hosts` 文件。*

5. **更新现有项目的远程仓库地址**：
   如果您的工作区中已有 Git 仓库，并在 code-server 中终端运行：
   ```bash
   # 确保 URL 使用您的 SSH Config 中配置的 Host 别名
   git remote set-url origin git@gitea.yourdomain.com:Username/Repository.git
   ```