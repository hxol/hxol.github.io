---
title: Dawarich (私人位置轨迹记录) 部署与配置
date: 2026-09-30 15:39:00
tags: [笔记, Dawarich, 自托管]
---

# Dawarich (私人位置轨迹记录) 部署与配置

**Dawarich** 是一款开源的自托管位置历史记录应用，可作为 Google Location History (谷歌时间轴) 的替代方案。

- **GitHub 仓库**: [Freika/dawarich](https://github.com/Freika/dawarich)
- **官方文档**: [Dawarich 官方配置文档](https://dawarich.app/docs/category/configuration/)

---

## 1. 环境准备与目录创建

在开始之前，请确保您的 Linux 系统已安装 **Docker** 和 **Docker Compose**。

首先，为 Dawarich 创建专属的运行目录并进入该目录：

```bash
sudo mkdir -p /opt/dawarich && cd /opt/dawarich 
```

## 2. 环境变量配置 (.env)

在项目目录下创建环境变量文件：

```bash
sudo nano /opt/dawarich/.env
```

将以下内容粘贴进去（请务必修改带有 `YOUR_` 前缀的自定义项）：

```env
# =============================================================================
# Dawarich Docker Compose 核心配置
# =============================================================================

# Rails 运行环境
RAILS_ENV=production

# =============================================================================
# 数据库配置 (PostgreSQL)
# =============================================================================
# 请确保此处与下方 DATABASE_ 设置中的密码保持一致
POSTGRES_USER=postgres
POSTGRES_PASSWORD=YOUR_SECURE_DB_PASSWORD
POSTGRES_DB=dawarich_production

# 供 Rails 应用连接使用的数据库设置
DATABASE_HOST=dawarich_db
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=YOUR_SECURE_DB_PASSWORD
DATABASE_NAME=dawarich_production

# =============================================================================
# Redis 配置
# =============================================================================
REDIS_URL=redis://dawarich_redis:6379

# =============================================================================
# 应用基础配置
# =============================================================================
# 运行权限 (可通过 id 命令查看当前用户的 uid 和 gid)
PUID=1000
PGID=1000

# 对外暴露的端口号
DAWARICH_APP_PORT=1063

# 允许访问的域名/IP (以逗号分隔)
# 生产环境示例: dawarich.yourdomain.com,localhost,127.0.0.1
APPLICATION_HOSTS=dawarich.yourdomain.com,localhost,::1,127.0.0.1

# 应用连接协议 (http 或 https)
# 注意：如果您使用 Nginx 或 Nginx Proxy Manager 等反向代理来处理 SSL 证书，
# 这里必须保持为 http，否则应用内部重定向会发生故障。
APPLICATION_PROTOCOL=http

# 时区
TIME_ZONE=Asia/Shanghai

# 自托管标志
SELF_HOSTED=true

# 存储地理数据 (反向地理编码结果)
STORE_GEODATA=true

# 存储后端 (local 或 s3)
STORAGE_BACKEND=local

# =============================================================================
# SMTP 邮件配置 (用于密码重置、通知等)
# =============================================================================
DOMAIN=dawarich.yourdomain.com
SMTP_SERVER=smtp.your-email-provider.com
SMTP_PORT=25 # 或 465 / 587，根据服务商而定
SMTP_FROM=notify@yourdomain.com
SMTP_AUTHENTICATION=plain
SMTP_OPEN_TIMEOUT=25
SMTP_READ_TIMEOUT=25
SMTP_USERNAME=notify@yourdomain.com
SMTP_PASSWORD=YOUR_SMTP_PASSWORD

# =============================================================================
# 安全相关 (Secret Key)
# =============================================================================
# 生产环境密钥，用于加密会话和 Cookie。
# 【必填】生成命令: openssl rand -hex 64
SECRET_KEY_BASE=YOUR_GENERATED_SECRET_KEY_BASE_HERE

# =============================================================================
# 性能、监控与日志
# =============================================================================
# Sidekiq 后台任务并发数 (处理大量导入时可临时调大)
BACKGROUND_PROCESSING_CONCURRENCY=10

# Prometheus 监控 (按需开启)
PROMETHEUS_EXPORTER_ENABLED=false

# Rails 及 Docker 日志设置
RAILS_LOG_TO_STDOUT=true
LOG_MAX_SIZE=100m
LOG_MAX_FILE=5

# 容器硬件资源限制
APP_CPU_LIMIT=0.50
APP_MEMORY_LIMIT=4G

# =============================================================================
# 身份验证控制 (可选)
# =============================================================================
# 是否允许用户使用邮箱/密码进行注册 (设为 false 则仅限管理员邀请)
ALLOW_EMAIL_PASSWORD_REGISTRATION=false
# 是否显示邮箱/密码登录表单
ALLOW_EMAIL_PASSWORD_LOGIN=true

# (OIDC 单点登录配置如不需要可留空，详见官方文档)
```

## 3. 编写 Docker Compose 编排文件

创建 `docker-compose.yml` 文件：

```bash
sudo nano /opt/dawarich/docker-compose.yml
```

填入以下内容：

```yaml
networks:
  dawarich:

services:
  dawarich_redis:
    image: redis:alpine
    container_name: dawarich_redis
    command: >
      redis-server
      --save 900 1
      --save 300 10
      --appendonly no
    networks:
      - dawarich
    volumes:
      - dawarich_shared:/data
    restart: unless-stopped
    healthcheck:
      test: [ "CMD", "redis-cli", "--raw", "incr", "ping" ]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: "100M"

  dawarich_db:
    image: postgis/postgis:17-3.5-alpine
    # 如果您使用的是 ARM 架构(如树莓派/Mac M系列)，请替换为:
    # image: imresamu/postgis:17-3.5-alpine 
    shm_size: 1G
    container_name: dawarich_db
    volumes:
      - dawarich_db_data:/var/lib/postgresql/data
      - dawarich_shared:/var/shared
    networks:
      - dawarich
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    restart: unless-stopped
    healthcheck:
      test: [ "CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}" ]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: "100M"

  dawarich_app:
    image: freikin/dawarich:latest
    container_name: dawarich_app
    volumes:
      - dawarich_public:/var/app/public
      - dawarich_watched:/var/app/tmp/imports/watched
      - dawarich_storage:/var/app/storage
      - dawarich_db_data:/dawarich_db_data
    networks:
      - dawarich
    ports:
      - "${DAWARICH_APP_PORT:-3000}:3000"
    stdin_open: true
    tty: true
    entrypoint: web-entrypoint.sh
    command: ['bin/rails', 'server', '-p', '3000', '-b', '::']
    restart: unless-stopped
    env_file:
      - .env
    logging:
      driver: "json-file"
      options:
        max-size: ${LOG_MAX_SIZE:-100m}
        max-file: ${LOG_MAX_FILE:-5}
    healthcheck:
      test: [ "CMD-SHELL", "wget -qO - http://127.0.0.1:3000/api/v1/health | grep -q '\"status\"\\s*:\\s*\"ok\"'" ]
      interval: 10s
      retries: 30
      start_period: 30s
      timeout: 10s
    depends_on:
      dawarich_db:
        condition: service_healthy
        restart: true
      dawarich_redis:
        condition: service_healthy
        restart: true
    deploy:
      resources:
        limits:
          cpus: ${APP_CPU_LIMIT:-0.50}
          memory: ${APP_MEMORY_LIMIT:-4G}
    tmpfs:
      - /tmp:mode=1777,size=800m,exec

  dawarich_sidekiq:
    image: freikin/dawarich:latest
    container_name: dawarich_sidekiq
    volumes:
      - dawarich_public:/var/app/public
      - dawarich_watched:/var/app/tmp/imports/watched
      - dawarich_storage:/var/app/storage
    networks:
      - dawarich
    stdin_open: true
    tty: true
    entrypoint: sidekiq-entrypoint.sh
    command: ['sidekiq']
    restart: unless-stopped
    env_file:
      - .env
    logging:
      driver: "json-file"
      options:
        max-size: ${LOG_MAX_SIZE:-100m}
        max-file: ${LOG_MAX_FILE:-5}
    healthcheck:
      test: [ "CMD-SHELL", "pgrep -f sidekiq" ]
      interval: 10s
      retries: 30
      start_period: 30s
      timeout: 10s
    depends_on:
      dawarich_db:
        condition: service_healthy
        restart: true
      dawarich_redis:
        condition: service_healthy
        restart: true
      dawarich_app:
        condition: service_healthy
        restart: true
    deploy:
      resources:
        limits:
          cpus: ${APP_CPU_LIMIT:-0.50}
          memory: ${APP_MEMORY_LIMIT:-4G}
    tmpfs:
      - /tmp:mode=1777,size=800m,exec

volumes:
  dawarich_db_data:
  dawarich_shared:
  dawarich_public:
  dawarich_watched:
  dawarich_storage:
```
*(注：这里精简了 `docker-compose.yml` 中冗余的环境变量映射，统一使用 `env_file: - .env` 读取，使得配置更加集中和清晰。)*

## 4. 启动服务

确保当前位于 `/opt/dawarich` 目录，执行以下命令在后台启动服务：

```bash
cd /opt/dawarich
docker compose up -d
```

*提示：初次启动时，数据库初始化和依赖拉取可能需要几分钟。可以使用 `docker compose logs -f` 查看实时日志。*

---

## 5. 账号配置与管理

当服务完全启动后，您可以通过配置的 IP/域名及端口（例如 `http://<您的IP>:1063`）访问应用。

### 默认测试账号

系统默认内置了一个演示账号：
- **用户名**: `demo@dawarich.app`
- **密码**: `safepassword`

> **安全警告**：在公网暴露服务前，强烈建议删除此默认账号或立即修改其密码！

### 使用 Rails 控制台创建专属管理员账号

为了安全和完全控制，推荐通过 Rails 终端直接在数据库中创建您的管理员账号：

1. **进入应用容器内部终端**：
   ```bash
   docker compose exec dawarich_app bin/rails c
   ```
   *(等待几秒钟，直到出现 `irb(main):001:0>` 提示符)*

2. **创建新用户**（将邮箱和密码替换为您自己的）：
   ```ruby
   User.create!(email: 'your_email@domain.com', password: 'YourSecurePassword123')
   ```
   *(如果输出绿色的 `#<User id: ...>` 信息，说明创建成功)*

3. **赋予该账号管理员权限**：
   ```ruby
   User.find_by(email: 'your_email@domain.com').update(admin: true)
   ```
   *(控制台会返回 `true` 代表更新成功)*

4. **退出控制台**：
   输入 `exit` 并回车。

现在，您可以使用刚刚创建的管理员账号登录 Dawarich，并开始您的位置追踪记录之旅了！
