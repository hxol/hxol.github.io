---
title: Grimmory 部署指南
date: 2026-09-30 15:30:00
tags: [笔记, Grimmory, 自托管]
---

# Grimmory 部署指南

Grimmory 是一款优秀的图书管理工具。本文档将指导您使用 Docker Compose 快速、安全地部署 Grimmory。

- **项目地址**：[grimmory-tools/grimmory](https://github.com/grimmory-tools/grimmory)

---

## 1. 准备目录结构

首先，创建用于持久化存储数据和配置的目录，并进入工作目录：

```bash
# 创建 grimmory 基础目录及子目录
sudo mkdir -p /opt/grimmory/{grimmory_data,bookdrop,mariadb_config} 
cd /opt/grimmory
```

## 2. 配置目录权限

为了避免 Docker 容器运行产生权限问题，建议将目录所有者修改为当前非 root 用户。

### 2.1 获取当前用户的 UID 和 GID
在终端输入以下命令获取您的用户 ID (UID) 和组 ID (GID)：
```bash
id -u  # 查看 UID (例如返回 1000)
id -g  # 查看 GID (例如返回 1000 或 1001)
```

### 2.2 赋予目录权限
假设您的 UID 为 `1000`，GID 为 `1000`（请根据实际情况替换下方的数字）：
```bash
sudo chown -R 1000:1000 /opt/grimmory
```

---

## 3. 配置环境变量

创建并编辑 `.env` 文件，将敏感信息（如密码）与配置文件分离。

```bash
sudo nano /opt/grimmory/.env
```

填入以下内容（**注意：请务必将 `<YOUR_SECURE_PASSWORD>` 替换为您自定义的强密码**）：

```env
# =============== 数据库设置 ===============
# 数据库连接 URL（容器内部通信）
DATABASE_URL=jdbc:mariadb://mariadb:3306/grimmory
DB_USER=grimmory
# 请将下方替换为您的强密码
DB_PASSWORD=<YOUR_SECURE_PASSWORD>

# =============== 应用设置 ===============
# 可选：是否启用 API 文档和导出 OpenAPI JSON (默认为 false)
API_DOCS_ENABLED=false

# 存储类型：LOCAL (默认) 或 NETWORK
# (如果使用网络存储挂载，请设为 NETWORK，会禁用部分文件操作)
DISK_TYPE=LOCAL

# =============== MariaDB 数据库设置 ===============
# MariaDB 的 Root 密码（建议与 DB_PASSWORD 不同，此处为演示）
MYSQL_ROOT_PASSWORD=<YOUR_SECURE_PASSWORD>
MYSQL_DATABASE=grimmory
```

---

## 4. 创建编排文件

创建并编辑 `docker-compose.yml` 文件：

```bash
sudo nano /opt/grimmory/docker-compose.yml
```

填入以下内容：

```yaml
services:
  grimmory:
    image: grimmory/grimmory:latest
    container_name: grimmory
    environment:
      # 请确保这里的 ID 与您前面设置的权限 ID 一致
      - USER_ID=1000
      - GROUP_ID=1000
      - TZ=Asia/Shanghai
      - DATABASE_URL=${DATABASE_URL}
      - DATABASE_USERNAME=${DB_USER}
      - DATABASE_PASSWORD=${DB_PASSWORD}
      - API_DOCS_ENABLED=${API_DOCS_ENABLED}
      - DISK_TYPE=${DISK_TYPE}
    depends_on:
      mariadb:
        condition: service_healthy
    ports:
      - "1083:6060" # 左侧 1083 为宿主机访问端口，可按需修改
    volumes:
      - ./grimmory_data:/app/data
      - ./bookdrop:/bookdrop
      # 【重要提示】请将 /mnt/books 替换为您宿主机存放电子书的真实路径
      - /mnt/books:/books
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1800M"
    tmpfs:
      - /tmp:mode=1777,size=1024m,exec

  mariadb:
    image: ghcr.io/linuxserver/mariadb:11.4.5
    container_name: grimmory_db
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${DB_USER}
      - MYSQL_PASSWORD=${DB_PASSWORD}
    volumes:
      - ./mariadb_config:/config
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mariadb-admin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 5s
      retries: 10
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "500M"
```

> **补充说明**：
> 1. `grimmory` 容器中的 `/mnt/books:/books` 是将本机的图书文件夹映射到容器内。部署前请确保 `/mnt/books` 在您的机器上存在，或者将其修改为您实际存放图书的路径。
> 2. `USER_ID` / `GROUP_ID` 和 `PUID` / `PGID` 请保持一致，并对应步骤 2 中查到的权限。

---

## 5. 运行与管理

确认在 `/opt/grimmory` 目录下，执行以下命令启动项目：

```bash
cd /opt/grimmory
sudo docker compose up -d
```

### 常用运维命令：

- **查看容器运行状态**：
  ```bash
  sudo docker compose ps
  ```

- **查看实时日志**（排查启动报错时非常有用）：
  ```bash
  sudo docker compose logs -f grimmory
  ```

- **停止并移除容器**：
  ```bash
  sudo docker compose down
  ```

---

## 6. 访问与初始化设置

待容器启动完毕且日志无报错后，打开浏览器访问：

`http://<您的服务器IP>:1083`

登录系统后，建议根据个人喜好进行系统配置。

### 推荐设置：文件命名模式
为保证导出的文件整理规范，请在 Web 界面依次进入：
**设置** -> **命名模式 (Naming pattern)** -> **文件命名模式 (File naming pattern)** -> **默认模式**

将其修改为以下格式：
```text
<{series}/>{authors}/{title}/{title}<({year})>< - {authors}>
```
*(该格式的效果通常为：`系列名/作者/书名/书名(年份) - 作者`，可使书库结构极其清晰)*
