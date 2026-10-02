---
title: 基于 Docker Compose 与 PostgreSQL 部署 Vaultwarden
date: 2026-09-30 15:35:00
tags: [笔记, Vaultwarden, 自托管]
---

# 基于 Docker Compose 与 PostgreSQL 部署 Vaultwarden

本文档介绍了如何使用 Docker Compose 部署 Vaultwarden（第三方 Bitwarden 服务端）并配合 PostgreSQL 作为后端数据库。该方案使用非 root 用户运行，提升了系统的安全性。

## 项目信息

- [Vaultwarden GitHub 仓库](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Docker 镜像](https://hub.docker.com/r/vaultwarden/server/tags)
- [PostgreSQL Docker 镜像](https://hub.docker.com/_/postgres/tags) *(注：原 databasus 镜像建议替换为官方稳定版镜像)*

---

## 1. 目录与权限准备

首先，创建用于存放应用数据和数据库数据的目录。

```bash
sudo mkdir -p /opt/vaultwarden/{vw-data,pg-data} 
cd /opt/vaultwarden
```

为了确保容器能以非 root 用户正常读写数据，需要对宿主机的目录进行权限划分：

```bash
# 授权 Vaultwarden 数据目录 (与下文 .env 中的 APP_USER_ID 和 APP_GROUP_ID 保持一致)
sudo chown -R 1000:1000 /opt/vaultwarden/vw-data

# 授权 PostgreSQL 数据目录 (官方 Postgres 镜像默认使用 UID/GID 999 运行)
sudo chown -R 999:999 /opt/vaultwarden/pg-data
```

---

## 2. 配置环境变量

将配置单独写入 `.env` 文件，方便统一管理和后续修改。

```bash
sudo nano /opt/vaultwarden/.env
```

填入以下内容（**注意：请将示例域名、数据库密码和 Token 替换为您自己的信息**）：

```env
# ==============================
# 镜像版本配置
# ==============================
VW_IMAGE=vaultwarden/server:latest
DB_IMAGE=postgres:18-alpine

# ==============================
# 权限配置 (UID 与 GID)
# ==============================
# 与上一步 vw-data 目录赋权的 UID:GID 一致
APP_USER_ID=1000
APP_GROUP_ID=1000

# ==============================
# 基础服务配置
# ==============================
# 您的访问域名
DOMAIN=https://vault.example.com
# 时区
TZ=Asia/Shanghai

# ==============================
# PostgreSQL 数据库配置
# ==============================
PG_USER=your_db_user
PG_PASSWORD=YourStrongPasswordHere
PG_PORT=5432
PG_DB=vaultwarden_db

# ==============================
# 端口与网络配置
# ==============================
# 容器内部端口: 因为是非 root 运行,无法绑定 80 端口,必须修改为 >1024 的端口
VW_PORT_ROCKET_PORT=8080
# 宿主机映射端口 (WebUI 端口)
VW_WEBUI_PORT=1070

# 是否启用 WebSocket
# 注: 从 Vaultwarden 1.29.0 开始, WebSocket 已默认永久开启,并合并到了 HTTP 端口(8080)中
WEBSOCKET_ENABLED=true

# ==============================
# 日志与管理配置
# ==============================
# 日志等级 (trace, debug, info, warn, error)
LOG_LEVEL=warn
# 是否开启扩展日志记录
EXTENDED_LOGGING=true

# 管理面板密码 Token (建议先留空，按本文第 5 步生成后再填入)
ADMIN_TOKEN=
```

---

## 3. 编写编排文件

创建 `docker-compose.yml` 文件。

```bash
sudo nano /opt/vaultwarden/docker-compose.yml
```

填入以下内容：

```yaml
services:
  vaultwarden:
    image: ${VW_IMAGE}
    container_name: vaultwarden
    restart: unless-stopped
    user: "${APP_USER_ID}:${APP_GROUP_ID}"
    environment:
      # --- 基础设置 ---
      - DOMAIN=${DOMAIN}
      - TZ=${TZ}
      # --- 数据库设置 (PostgreSQL) ---
      # 格式: postgresql://用户名:密码@服务名:端口/数据库名
      - DATABASE_URL=postgresql://${PG_USER}:${PG_PASSWORD}@db:${PG_PORT}/${PG_DB}
      # --- 网络设置 ---
      - ROCKET_PORT=${VW_PORT_ROCKET_PORT}
      # - WEBSOCKET_ENABLED=${WEBSOCKET_ENABLED} # 1.29.0+ 版本可省略此项
      # --- 日志与杂项 ---
      - LOG_LEVEL=${LOG_LEVEL}
      - EXTENDED_LOGGING=${EXTENDED_LOGGING}
      - ADMIN_TOKEN=${ADMIN_TOKEN}
    volumes:
      - ./vw-data:/data
    ports:
      - ${VW_WEBUI_PORT}:${VW_PORT_ROCKET_PORT}
    depends_on:
      db:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: "256M"

  db:
    image: ${DB_IMAGE}
    container_name: vaultwarden-db
    restart: unless-stopped
    # 数据库容器由内部 postgres 用户(UID 999)管理权限，建议保持默认，不要随意修改 user 参数
    environment:
      - POSTGRES_USER=${PG_USER}
      - POSTGRES_PASSWORD=${PG_PASSWORD}
      - POSTGRES_DB=${PG_DB}
    volumes:
      - ./pg-data:/var/lib/postgresql
    healthcheck:
      # 使用环境变量进行健康检查，防止密码修改后检查失败
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: "256M"
```

---

## 4. 启动与状态检查

确保您当前位于 `/opt/vaultwarden` 目录下，在后台启动服务：

```bash
cd /opt/vaultwarden
sudo docker compose up -d
```

**常用运维命令：**

- **查看容器运行状态**：
  ```bash
  sudo docker compose ps
  ```
- **查看 Vaultwarden 实时日志**（按 `Ctrl+C` 退出）：
  ```bash
  sudo docker compose logs -f vaultwarden
  ```
- **查看数据库实时日志**：
  ```bash
  sudo docker compose logs -f db
  ```

---

## 5. 开启并保护 Admin 管理页面

Vaultwarden 提供了一个 `/admin` 页面，用于管理用户、配置 SMTP 等。为了安全起见，必须使用 Argon2 哈希加密的 Token 才能开启。

### 步骤 1：生成 Argon2 哈希密码
运行以下命令（请根据您实际使用的镜像版本调整 `vaultwarden/server:latest`）：

```bash
sudo docker run --rm -it vaultwarden/server:latest /vaultwarden hash
```
系统会提示您输入并确认一个管理密码。完成后，终端会输出一长串类似如下的哈希字符串：
`ADMIN_TOKEN=$$argon2id$v=19$m=65536,t=3,p=4$XXXXXXXXXXXXXXXXXXXXX`

### 步骤 2：填入环境变量
复制上面生成的整段字符串。打开 `.env` 文件，将其覆盖到 `ADMIN_TOKEN` 字段处。

```bash
sudo nano /opt/vaultwarden/.env
```

### 步骤 3：重启容器生效
配置修改后，需要重新拉起容器以应用新的环境变量：

```bash
cd /opt/vaultwarden
sudo docker compose down
sudo docker compose up -d
```

### 步骤 4：访问面板
容器重启完成后，通过浏览器访问 `https://vault.example.com/admin`，输入您在步骤 1 中设置的明文密码，即可进入管理后台。