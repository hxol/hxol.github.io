---
title: 使用 Docker 或 Podman 部署 Cloudflared Tunnel
date: 2026-09-30 15:34:00
tags: [笔记, Cloudflared Tunnel, Podman, 自托管]
---

# 使用 Docker / Podman 部署 Cloudflared Tunnel 指南

本文档介绍了如何使用 Docker Compose 或 Podman 在 Linux 环境中部署 Cloudflare Tunnel（`cloudflared`）。

> **💡 最佳实践提示（关于镜像版本）：**
> 示例中使用了 `latest` 标签。在生产环境中，建议前往 [Cloudflared Docker Hub Tags](https://hub.docker.com/r/cloudflare/cloudflared/tags) 查看最新稳定版，并将 `latest` 替换为具体的版本号（例如 `2023.10.0`），以避免自动更新带来不可预期的破坏。

---

## 方案一：通过 Docker Compose 安装

使用 Docker Compose 可以方便地管理容器配置，并通过 `.env` 文件保护您的隧道密钥。

### 1. 创建目录结构
首先，创建工作目录并进入该目录：
```bash
sudo mkdir -p /opt/cloudflaredtunnel
cd /opt/cloudflaredtunnel
```

### 2. 配置环境变量（保护隐私）
为了防止密钥泄露在配置文件中，我们将其写入隐藏的环境变量文件中：
```bash
sudo nano .env
```
在文件中填入您的 Cloudflare Tunnel Token（请将 `<您的隧道密钥>` 替换为真实 Token）：
```env
TUNNEL_TOKEN=<您的隧道密钥>
```

### 3. 创建配置文件
创建 `docker-compose.yml` 文件：
```bash
sudo nano docker-compose.yml
```
填入以下内容：
```yaml
services:
  cloudflared-tunnel:
    image: cloudflare/cloudflared:latest
    container_name: CloudflaredTunnel
    restart: unless-stopped
    network_mode: "host"
    # 使用环境变量注入 Token，避免在命令中直接明文暴露
    environment:
      - TUNNEL_TOKEN=${TUNNEL_TOKEN}
    command: tunnel --no-autoupdate run
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: "500M"
```

### 4. 启动服务
使用以下命令在后台启动并运行容器：
```bash
# 较新的 Docker 版本使用 docker compose，旧版本使用 docker-compose
sudo docker compose up -d
```
您可以通过以下命令查看运行日志，确认是否连接成功：
```bash
sudo docker compose logs -f
```

---

## 方案二：通过 Podman 安装

Podman 提供了无守护进程的架构，并能与 Systemd 完美结合，实现开机自启和进程守护。

### 1. 运行容器
使用环境变量传入 Token 启动容器。这里去除了不必要的交互模式参数（`-it`），直接以后台守护进程模式运行：

```bash
sudo podman run \
  --detach \
  --name cloudflaredtunnel \
  --network=host \
  --restart=unless-stopped \
  --env TUNNEL_TOKEN="<您的隧道密钥>" \
  docker.io/cloudflare/cloudflared:latest tunnel --no-autoupdate run
```

### 2. 生成 Systemd 服务单元文件
利用 Podman 内置命令生成标准的 Systemd 服务文件，以便系统接管容器的生命周期：
```bash
cd /opt  # 切换到一个临时工作目录
sudo podman generate systemd --name cloudflaredtunnel --files
```
*(注：在最新的 Podman v5+ 中推荐使用 Quadlet 配置，但上述命令在旧版 Podman 系统中依然有效且常用)*

### 3. 安装并配置服务
将生成的服务文件移动到 Systemd 系统目录，并重载系统守护进程：
```bash
sudo mv container-cloudflaredtunnel.service /etc/systemd/system/
sudo systemctl daemon-reload
```

### 4. 启动并设置开机自启
启用并立即启动该服务：
```bash
sudo systemctl enable --now container-cloudflaredtunnel.service
```

### 5. 检查运行状态
最后，检查服务的运行状态，如果显示 `active (running)` 则表示部署成功：
```bash
sudo systemctl status container-cloudflaredtunnel.service
```

按 `q` 键可退出状态查看界面。如需查看更详细的运行日志，可使用：
```bash
sudo journalctl -u container-cloudflaredtunnel.service -f
```
