---
title: Docker 部署 Pocket ID 完整指南
date: 2026-09-30 15:31:00
tags: [笔记, Pocket ID, 自托管]
---

# Docker 部署 Pocket ID 完整指南

Pocket ID 是一个简单且安全的身份验证解决方案。本指南将带你使用 Docker Compose 从零开始部署、配置以及备份 Pocket ID。

- **项目地址**: [GitHub - pocket-id/pocket-id](https://github.com/pocket-id/pocket-id)
- **官方文档**: [Pocket ID Docs](https://pocket-id.org/docs/)

## 0. 准备工作

在开始之前，请确保你的服务器已安装 Docker 和 Docker Compose。

## 1. 创建目录与设置权限

首先，我们需要为项目创建存放数据和配置文件的目录。

```bash
# 创建并进入项目目录
sudo mkdir -p /opt/pocket-id/data && cd /opt/pocket-id
```

为了防止 Docker 挂载数据卷时出现读写权限问题，建议将目录的所有者更改为你当前的用户。你可以通过 `id` 命令查看当前的 `uid` 和 `gid`：

```bash
id
```

假设你的 `uid` 和 `gid` 都是 `1000`（大多数常规非 root 用户默认是这个），执行以下命令赋予权限：

```bash
# 注意：如果你的 uid 和 gid 不是 1000，请将 1000:1000 替换为你自己的
sudo chown -R 1000:1000 /opt/pocket-id
sudo chmod -R 755 /opt/pocket-id
```

## 2. 生成加密密钥

Pocket ID 需要一个高强度的密钥来加密数据（包括私钥）。请运行以下命令生成一个随机的 Base64 字符串：

```bash
openssl rand -base64 32
```

> 💡 **提示**：请复制并在安全的地方**妥善记录这串字符**。如果丢失此密钥，你将无法解密和访问已有数据。稍后我们需要将它填入 `.env` 文件中。

## 3. 配置环境变量

创建并编辑环境变量文件：

```bash
sudo nano /opt/pocket-id/.env
```

填入以下内容（请根据注释**修改为你自己的信息**）：

```env
# 环境变量官方文档: https://pocket-id.org/docs/configuration/environment-variables

# Pocket ID 的访问网址 (修改为你的实际域名)
APP_URL=https://login.example.com

# Pocket ID 容器内部侦听的端口（默认 1411，通常无需更改）
PORT=1411

# 如果 Pocket ID 位于反向代理（如 Nginx, Traefik, NPM）后面，请保持为 true
TRUST_PROXY=true

# GeoLite2 数据库的许可证密钥（可选，用于显示登录时的地理位置）
MAXMIND_LICENSE_KEY=

# 运行容器的 uid 和 gid（与第一步通过 id 命令查看到的值保持一致）
PUID=1000
PGID=1000

# 用于加密数据的密钥，填入刚才通过 openssl 命令生成的 32 位随机字符
ENCRYPTION_KEY=在此处填入你生成的密钥
```

## 4. 创建编排文件

创建并编辑 `docker-compose.yml` 文件：

```bash
sudo nano /opt/pocket-id/docker-compose.yml
```

填入以下内容：

```yaml
services:
  pocket-id:
    image: ghcr.io/pocket-id/pocket-id:v2
    container_name: pocket-id
    restart: unless-stopped
    env_file: .env
    ports:
      - "1411:1411"  # 格式为 "宿主机端口:容器端口"，按需修改左侧端口
    volumes:
      - "./data:/app/data"
      # 使用 tmpfs 可以提高性能并解决自定义 UID 的特定读写问题
      - type: tmpfs
        target: /tmp
        tmpfs:
          mode: 01777      # 赋予标准 tmp 权限
          size: "50M"      # 限制最大使用的内存大小，防止 OOM
    healthcheck:
      test: [ "CMD", "/app/pocket-id", "healthcheck" ]
      interval: 1m30s
      timeout: 5s
      retries: 2
      start_period: 10s
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: "200M"
```

## 5. 启动与查看状态

确认当前在 `/opt/pocket-id` 目录下，在后台启动服务：

```bash
cd /opt/pocket-id
docker compose up -d
```

**日常运维命令：**

```bash
# 查看容器运行状态
docker compose ps

# 查看实时运行日志（按 Ctrl+C 退出）
docker compose logs -f pocket-id
```

---

## 6. 数据备份与恢复 (导入/导出)

Pocket ID 提供了内置的导出与导入命令，这比直接复制文件更加安全可靠。

### 备份 (导出)

建议养成定期备份的习惯。首先在当前用户的家目录创建一个用于存放备份的文件夹：

```bash
mkdir -p ~/backups/pocket-id
```

执行以下命令，将数据导出为一个以当前日期命名的 `.zip` 文件：

```bash
# 确保在执行时 pocket-id 容器正在运行
docker compose exec pocket-id ./pocket-id export --path - > ~/backups/pocket-id/export-$(date +%F).zip
```
> 备份完成后，你可以使用 `ls -lh ~/backups/pocket-id/` 检查备份文件是否成功生成及文件大小。

### 恢复 (导入)

> ⚠️ **警告**：导入操作可能会覆盖现有数据，请谨慎操作。

恢复数据时，需要先停止正在运行的服务，然后通过一个临时容器执行导入命令。

**步骤 1：停止当前服务**
```bash
cd /opt/pocket-id
docker compose down
```

**步骤 2：执行导入**
假设你要恢复的备份文件路径为 `~/backups/pocket-id/export.zip`，请执行：

```bash
cat ~/backups/pocket-id/export.zip | docker compose run --rm pocket-id ./pocket-id import --yes --path -
```
*(注：如果你的备份文件带有日期后缀，如 `export-2023-10-25.zip`，请将上述命令中的文件名替换为实际的文件名)*

**步骤 3：重新启动服务**
导入成功后，重新拉起容器即可：
```bash
docker compose up -d
```
