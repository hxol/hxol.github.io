---
title: FreshRSS 部署与配置
date: 2026-09-30 15:30:00
tags: [笔记, FreshRSS, 自托管]
---


# FreshRSS 部署与配置

## 1. 项目参考信息

- **项目主页 (GitHub)**: [FreshRSS/FreshRSS](https://github.com/FreshRSS/FreshRSS)
- **Docker 镜像地址**: [freshrss/freshrss Tags](https://hub.docker.com/r/freshrss/freshrss/tags)

---

## 2. 准备工作

首先，创建 FreshRSS 的工作目录并进入该目录：

```bash
sudo mkdir -p /opt/freshrss
cd /opt/freshrss
```

---

## 3. 配置环境变量

使用 `.env` 文件管理敏感信息和变量，能有效提升部署的安全性。

**第一步：创建 `.env` 文件**

```bash
sudo nano /opt/freshrss/.env
```

**第二步：填入环境配置**

将以下内容粘贴到文件中，并**务必根据实际情况修改相关值**：

```env
# ==========================================
# 站点基础配置
# ==========================================
# FreshRSS 的实际访问地址，请修改为你自己的域名或 IP 地址及端口（如 http://192.168.1.100:1067）
BASE_URL=https://freshrss.lab.io

# ==========================================
# 数据库连接信息 (PostgreSQL)
# ==========================================
# DB_HOST 必须与 docker-compose.yml 文件中的数据库服务名称保持一致
DB_HOST=freshrss-db
DB_BASE=freshrss
DB_USER=freshrss
# 请务必修改为强密码
DB_PASSWORD=8PvhN5D9uJYRmR

# ==========================================
# FreshRSS 管理员账户信息
# ==========================================
# 此处设置的账户将在首次启动容器时自动创建
ADMIN_USER=admin
# 请务必修改为更安全的密码
ADMIN_PASSWORD=qdzZS4P28guPmw
```

保存并退出（在 nano 中按 `Ctrl+O`, `Enter`, `Ctrl+X`）。

---

## 4. 编写编排文件 (Docker Compose)

**第一步：创建配置文件**

```bash
sudo nano /opt/freshrss/docker-compose.yml
```

**第二步：填入 `docker-compose.yml` 内容**

```yaml
services:
  freshrss:
    image: freshrss/freshrss:latest
    container_name: freshrss
    restart: unless-stopped
    environment:
      # 设置时区
      TZ: 'Asia/Shanghai'
      # 设置定时任务，表示在每小时的第 2 和第 32 分钟自动在后台刷新 RSS 源
      CRON_MIN: '2,32'
      
      # 自动安装参数 (仅在首次运行、未初始化时生效)
      # 从 .env 文件读取数据库信息进行自动化配置
      FRESHRSS_INSTALL: |-
        --base-url ${BASE_URL}
        --db-type pgsql
        --db-host ${DB_HOST}
        --db-base ${DB_BASE}
        --db-user ${DB_USER}
        --db-password ${DB_PASSWORD}
        --default-user ${ADMIN_USER}
        --language zh-cn
        
      # 自动创建用户参数 (仅在首次运行时生效)
      # 从 .env 文件读取管理员账户信息
      FRESHRSS_USER: |-
        --user ${ADMIN_USER}
        --password ${ADMIN_PASSWORD}
    volumes:
      # 将 FreshRSS 的数据持久化到数据卷中
      - freshrss-data:/var/www/FreshRSS/data
    ports:
      - 1036:80 # 左侧的暴露端口可根据宿主机实际情况修改
    depends_on:
      # 确保数据库服务先于 FreshRSS 主程序启动
      freshrss-db:
        condition: service_started

  # PostgreSQL 数据库服务
  freshrss-db:
    image: postgres:18-alpine # 使用 alpine 版本减小镜像体积并降低资源占用
    container_name: freshrss-db
    restart: unless-stopped
    environment:
      # 数据库初始化所需参数，自动从 .env 文件读取
      POSTGRES_DB: ${DB_BASE}
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      TZ: 'Asia/Shanghai'
    volumes:
      #  重要修正 PostgreSQL 18 容器数据默认路径是 /var/lib/postgresql
      - postgres-data:/var/lib/postgresql

# 定义数据卷
volumes:
  freshrss-data:
  postgres-data:
```

---

## 5. 启动运行

确保你处于工作目录下，使用以下命令在后台启动服务：

```bash
cd /opt/freshrss
sudo docker compose up -d
```

启动后，可以等待 10~30 秒让数据库和主程序完成初始化配置，然后即可通过设置的 `BASE_URL` 或 `http://宿主机IP:1036` 访问您的 FreshRSS 实例。
