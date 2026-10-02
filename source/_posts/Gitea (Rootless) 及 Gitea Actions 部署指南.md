---
title: Gitea (Rootless) 及 Gitea Actions 部署指南
date: 2026-09-30 15:30:00
tags: [笔记, Gitea, Gitea Actions, 自托管]
---


# Gitea (Rootless) 及 Gitea Actions 部署指南

[Gitea 官方 GitHub 仓库](https://github.com/go-gitea/gitea)

本文档将指导您使用 Docker Compose 部署高安全的 Rootless 版 Gitea 实例，并为其配置独立的 Gitea Actions 运行器（Act Runner）及缓存系统。

---

## 第一部分：部署 Gitea (Rootless 版)

### 1. 创建目录与赋权

Rootless 版本的 Gitea 镜像为了提高安全性，要求以非 root 用户运行，其容器内默认的 UID 为 `1000`。PostgreSQL 官方镜像默认的 UID 为 `999`。

创建数据存放目录并配置正确的权限：

```bash
# 创建目录
sudo mkdir -p /opt/gitea/{gitea_config,gitea_data,postgres_data}
cd /opt/gitea

# 为 Gitea 目录分配 UID/GID 1000
sudo chown -R 1000:1000 /opt/gitea/gitea_config
sudo chown -R 1000:1000 /opt/gitea/gitea_data

# 为 PostgreSQL 目录分配 UID/GID 999
sudo chown -R 999:999 /opt/gitea/postgres_data
```

### 2. 配置环境变量

将配置分离到 `.env` 文件中，方便后期统一修改。

```bash
sudo nano /opt/gitea/.env
```
填入以下内容（**请将占位符替换为您自己的实际信息**）：
```env
# ================= 应用配置 =================
SERVICE_NAME=gitea-server
# 服务域名 (您的 Gitea 将通过此域名访问，务必填写准确)
SERVICE_DOMAIN=gitea.example.com
# 应用镜像 (指定 Rootless 版本)
DOCKER_IMAGE=gitea/gitea:1.22.0-rootless

# ================= 端口配置 =================
# 宿主机映射端口
WEBUI_PORT=3000
SSH_PORT=2222

# ================= 权限与功能 =================
UID=1000
GID=1000
# 是否启用 Git LFS (大文件存储)
LFS=true
# 访客是否必须登录才能查看代码仓库
SIGNIN_VIEW=false
# 是否禁用初始安装界面 (设为 true 则自动跳过安装向导，但不影响注册)
INSTALL_LOCK=true

# ================= 数据库配置 =================
POSTGRES_USER=gitea_user
POSTGRES_PASSWORD=YOUR_SECURE_PASSWORD
POSTGRES_DB=gitea_db
```

### 3. 编写编排文件

```bash
sudo nano /opt/gitea/docker-compose.yml
```
填入以下内容：
```yaml
name: gitea-stack

services:
  gitea:
    image: ${DOCKER_IMAGE}
    container_name: gitea_server
    restart: unless-stopped
    ports:
      - "${WEBUI_PORT}:3000"       # Web UI 端口
      - "${SSH_PORT}:${SSH_PORT}"  # SSH 服务端口
    user: "${UID}:${GID}"          # 指定非 root 用户运行
    environment:
      - APP_NAME=${SERVICE_NAME}
      # 外部访问 URL，务必与反向代理配置的域名及协议一致
      - GITEA__server__ROOT_URL=https://${SERVICE_DOMAIN}
      - GITEA__server__LFS_START_SERVER=${LFS}
      - GITEA__service__REQUIRE_SIGNIN_VIEW=${SIGNIN_VIEW}
      - GITEA__security__INSTALL_LOCK=${INSTALL_LOCK}

      # PostgreSQL 数据库连接配置
      - GITEA__database__DB_TYPE=postgres
      - GITEA__database__HOST=gitea_db:5432
      - GITEA__database__NAME=${POSTGRES_DB}
      - GITEA__database__USER=${POSTGRES_USER}
      - GITEA__database__PASSWD=${POSTGRES_PASSWORD}

      # SSH 相关配置
      - SSH_DOMAIN=${SERVICE_DOMAIN}
      - SSH_PORT=${SSH_PORT}         # 告诉用户克隆时使用的端口
      - SSH_LISTEN_PORT=${SSH_PORT}  # 容器内实际监听的 SSH 端口
      - HTTP_PORT=3000               # 容器内监听的 HTTP 端口

    volumes:
      - ./gitea_config:/etc/gitea:rw   # 配置文件持久化
      - ./gitea_data:/var/lib/gitea:rw # 数据持久化 (仓库文件等)
      - /etc/timezone:/etc/timezone:ro # 同步时区
      - /etc/localtime:/etc/localtime:ro
    depends_on:
      gitea_db:
        condition: service_healthy
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
    healthcheck:
      test: ["CMD-SHELL", "wget -q --spider --proxy off localhost:3000 || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "2048M"

  gitea_db:
    image: postgres:18-alpine # 建议使用当前稳定版
    container_name: gitea_postgres_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./postgres_data:/var/lib/postgresql
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1024M"
```

### 4. 运行与验证

```bash
cd /opt/gitea 
sudo docker compose up -d

# 查看运行状态与日志
sudo docker compose ps
sudo docker compose logs -f gitea
```

---

## 第二部分：Gitea 高级配置调整

针对大项目或特定网络环境，我们需要修改 Gitea 的核心配置文件 `app.ini`。（依据上述映射，宿主机路径为 `/opt/gitea/gitea_config/app.ini`）。

```bash
sudo nano /opt/gitea/gitea_config/app.ini
```

### 1. 延长任务超时时间
默认的 3 小时对于大型 CI/CD 任务可能过短，建议延长：
```ini
[actions]
# 增加或修改以下行
ENDLESS_TASK_TIMEOUT = 24h
# ZOMBIE_TASK_TIMEOUT = 30m
```

### 2. 允许迁移局域网仓库
如果您需要从内网的其他 Git 服务（如 GitLab）迁移代码：
```ini
[migrations]
# 增加或修改以下行
ALLOW_LOCALNETWORKS = true
```
*修改完毕后，需重启 Gitea 服务：`sudo docker compose restart gitea`*

---

## 第三部分：部署 Gitea Actions (Act Runner)

为了将 CI/CD 任务的负载从 Gitea 主程序分离，我们将使用独立的 `docker-compose.yml` 部署 `act_runner`。

### 1. 获取 Runner 注册令牌
1. 使用管理员账号登录 Gitea 网页端。
2. 进入 **站点管理** -> **Actions** -> **Runners**（如果只针对特定组织/仓库，请去对应的设置页面）。
3. 点击 **Create new Runner (创建新运行器)**，复制生成的 **注册令牌 (Registration Token)**。

### 2. 创建目录与生成默认配置
```bash
sudo mkdir -p /opt/gitea-runner/data
cd /opt/gitea-runner

# 提取官方默认配置文件
sudo docker run --entrypoint="" --rm -it docker.io/gitea/act_runner:latest act_runner generate-config > /tmp/config.yaml
sudo mv /tmp/config.yaml /opt/gitea-runner/
```

### 3. 配置 Runner 缓存 (任选其一)

为了让 Actions 中的 `actions/cache` 正常工作，必须配置缓存服务端。

#### 方案 A：缓存在宿主机本地（推荐中小型项目）
打开 `config.yaml`：
```bash
sudo nano /opt/gitea-runner/config.yaml
```
找到 `cache:` 并修改如下：
```yaml
cache:
  enabled: true
  dir: ""
  # 【重要】填入该 Runner 宿主机的局域网 IP（如 192.168.x.x），不可填 127.0.0.1
  host: "YOUR_LAN_IP" 
  port: 8088
```
*(注：为保证 Job 容器能访问到缓存端口，需配合后文 compose 文件中的端口映射。)*

---

#### 方案 B：缓存在 S3 对象存储（推荐大型或多节点项目）
思路：使用 S3 配合权限管控，通过 `s3fs-fuse` 挂载到宿主机。

**步骤 1：配置 S3 存储桶与权限**
1. 创建私有桶，例如命名为 `gitea-runner-cache`。
2. 创建 IAM 策略（仅允许访问该桶）：
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:*"],
         "Resource": [
           "arn:aws:s3:::gitea-runner-cache",
           "arn:aws:s3:::gitea-runner-cache/*"
         ]
       }
     ]
   }
   ```
3. 创建子用户（如 `s3-cache-bot`），绑定上述策略，并获取 Access Key (AK) 和 Secret Key (SK)。

**步骤 2：安装并挂载 s3fs**
```bash
sudo apt update && sudo apt install s3fs -y

# 启用 allow_other 以允许非 root 用户访问 fuse 挂载点
sudo sed -i 's/#user_allow_other/user_allow_other/g' /etc/fuse.conf

# 配置凭证 (替换为您的 AK:SK)
echo "YOUR_AK:YOUR_SK" | sudo tee /opt/gitea-runner/.passwd-s3fs
sudo chmod 600 /opt/gitea-runner/.passwd-s3fs

# 创建本地挂载点
sudo mkdir -p /opt/gitea-runner/s3-cache
```

**步骤 3：配置开机自动挂载 (/etc/fstab)**
```bash
sudo nano /etc/fstab
```
添加以下内容（**注意替换您的桶名、URL 和宿主机的固定 UID/GID，通常为 1000**）：
```text
s3fs#gitea-runner-cache /opt/gitea-runner/s3-cache fuse _netdev,allow_other,nonempty,passwd_file=/opt/gitea-runner/.passwd-s3fs,url=https://s3.example.com,use_path_request_style,uid=1000,gid=1000 0 0
```
测试挂载：
```bash
sudo systemctl daemon-reload
sudo mount -a
```

**步骤 4：修改 Runner 配置**
修改 `/opt/gitea-runner/config.yaml`：
```yaml
cache:
  enabled: true
  dir: "/opt/gitea-runner/s3-cache"
  host: "YOUR_LAN_IP"
  port: 8088
```

---

### 4. Runner 其他进阶配置 (config.yaml)

在 `/opt/gitea-runner/config.yaml` 中，您还可以根据需求修改：

- **网络模式**：
  在 `container:` 模块下，您可以调整执行任务容器的网络。
  ```yaml
  container:
    network: "host" # 若需要 Job 容器共享宿主机网络可开启此项
  ```
- **超时时间**：
  ```yaml
  runner:
    timeout: 24h    # 增加任务执行的最大超时时间
  ```

### 5. 编排 Runner 服务

创建环境变量文件保护隐私数据：
```bash
sudo nano /opt/gitea-runner/.env
```
```env
# Gitea 的访问地址 (Runner 必须能通过此 URL 访问到 Gitea)
GITEA_INSTANCE_URL=https://gitea.example.com

# 第 1 步中获取的注册令牌
GITEA_RUNNER_REGISTRATION_TOKEN=YOUR_REGISTRATION_TOKEN

# 运行器名称 (将在 Gitea UI 中显示)
GITEA_RUNNER_NAME=docker-runner-01
```

编写编排文件：
```bash
sudo nano /opt/gitea-runner/docker-compose.yml
```
```yaml
name: gitea-runner

services:
  runner:
    image: docker.io/gitea/act_runner:latest
    container_name: gitea_act_runner
    restart: unless-stopped
    environment:
      - CONFIG_FILE=/config.yaml
      - GITEA_INSTANCE_URL=${GITEA_INSTANCE_URL}
      - GITEA_RUNNER_REGISTRATION_TOKEN=${GITEA_RUNNER_REGISTRATION_TOKEN}
      - GITEA_RUNNER_NAME=${GITEA_RUNNER_NAME}
      # 默认运行环境镜像标签配置
      # - GITEA_RUNNER_LABELS=ubuntu-latest:docker://node:20-bookworm,ubuntu-22.04:docker://node:20-bookworm
    volumes:
      - ./config.yaml:/config.yaml:ro
      - ./data:/data:rw
      # 挂载 Docker Socket 以实现 Docker-in-Docker (允许 Runner 创建执行任务的临时容器)
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
      # 如果使用了 S3 挂载，需要将宿主机挂载点映射到 Runner 容器内
      # - ./s3-cache:/opt/gitea-runner/s3-cache:rw 
    ports:
      # 暴露 cache 端口给 Job 容器 (对应 config.yaml 中的 port: 8088)
      - "8088:8088"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "8192M"
```

### 6. 启动并验证 Runner

```bash
cd /opt/gitea-runner
sudo docker compose up -d

# 查看日志，确认是否成功注册并连线
sudo docker compose logs -f
```
*预期日志：应出现 `Runner registered successfully` 及 `Starting runner daemon` 等字样。*

最后，回到 Gitea 网页端的 **Runners** 列表页，如果您的运行器 `docker-runner-01` 显示为 **在线 (Idle)**，则表示一切部署成功。

### 💡 维护建议
由于 Runner 在运行 Actions 时会频繁拉取镜像并创建临时容器，建议在宿主机设置定时任务（Cron），定期清理无用的 Docker 缓存以防磁盘占满：
```bash
# 建议每周执行一次
docker system prune -af --volumes
```