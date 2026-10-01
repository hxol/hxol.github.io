---
title: 使用 Docker 部署 msmtpd：搭建轻量级安全邮件中继服务
date: 2026-09-30 15:35:00
tags: [笔记, msmtpd, 邮件, SMTP中继, 自托管]
---

# 使用 Docker 部署 msmtpd：搭建轻量级安全邮件中继服务

在局域网中，我们经常需要让各种服务或脚本发送通知邮件。然而，许多老旧应用只支持基础的 25 端口，不支持 TLS 加密或复杂的密码验证。

通过部署 `msmtpd` (基于 msmtp 的 SMTP 代理)，我们可以在本地开放一个无密码验证的 25 端口，由它负责接收本地应用的邮件，并**自动使用 TLS 加密和账号密码**中继转发给第三方邮件服务商（如飞书、阿里云、腾讯企业邮箱等）。

- **项目地址**: [crazy-max/docker-msmtpd](https://github.com/crazy-max/docker-msmtpd)

## 1. 环境准备

首先，创建项目目录并进入该目录：

```bash
sudo mkdir -p /opt/msmtpd && cd /opt/msmtpd
```

为了确保 Docker 容器内的非特权用户（UID 1000）有权限读取该目录，我们需要设置正确的所属权和权限：

```bash
# 查看当前用户的 uid 与 gid (确保与下方配置的 1000 匹配)
id

# 赋予权限
sudo chown -R 1000:1000 /opt/msmtpd
sudo chmod -R 755 /opt/msmtpd
```

## 2. 配置安全密钥 (Secrets)

为了避免在编排文件中明文暴露邮箱账号和密码，我们使用 Docker Secrets 来管理敏感信息。

创建账号和密码文件（**请将引号内的内容替换为你实际的 SMTP 用户名和密码**）：

```bash
echo "your_email@example.com" > smtp_user.txt
echo "your_smtp_password_or_auth_code" > smtp_password.txt
```

*注：部分邮件服务商（如 QQ、网易）需要使用“授权码”而非登录密码。*

为了保护文件安全，限制其权限仅限属主读写：

```bash
sudo chmod 600 smtp_user.txt smtp_password.txt
```

## 3. 编写 Docker Compose 文件

创建并编辑 `docker-compose.yml` 文件：

```bash
sudo nano /opt/msmtpd/docker-compose.yml
```

填入以下内容。请注意修改 `SMTP_HOST` 等环境变量以匹配你的邮件服务商：

```yaml
name: msmtpd

services:
  msmtpd:
    image: crazymax/msmtpd:1.8.34
    container_name: msmtpd
    ports:
      - target: 2500
        published: "25" 
        protocol: tcp
    environment:
      - "TZ=Asia/Shanghai"
      - "PUID=1000"
      - "PGID=1000"
      
      # SMTP 服务器配置 (以下以飞书 / 465端口 为例)
      - "SMTP_HOST=smtp.example.com" # 替换为实际的 SMTP 地址，如 smtp.larksuite.com
      - "SMTP_PORT=465"
      - "SMTP_TLS=on"
      - "SMTP_STARTTLS=off"          # 465 端口通常使用隐式 TLS，无需 STARTTLS；若使用 587 端口则设为 on
      - "SMTP_TLS_CHECKCERT=on"
      - "SMTP_AUTH=on"
      - "SMTP_USER_FILE=/run/secrets/smtp_user"
      - "SMTP_PASSWORD_FILE=/run/secrets/smtp_password"
      - "SMTP_DOMAIN=localhost"      # 宣告的主机名
    secrets:
      - smtp_user
      - smtp_password
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "300M"

secrets:
  smtp_user:
    file: ./smtp_user.txt
  smtp_password:
    file: ./smtp_password.txt
```

## 4. 运行与查看状态

启动服务并在后台运行：

```bash
cd /opt/msmtpd && sudo docker compose up -d
```

检查容器是否正常运行：

```bash
sudo docker compose ps
```

查看实时运行日志，确认是否有报错：

```bash
sudo docker compose logs -f 
```

## 5. 发信测试 (可选)

部署完成后，你可以通过 `swaks`（类似邮件版的 curl）来测试本地 25 端口是否能够正常发信：

```bash
# 安装 swaks (以 Debian/Ubuntu 为例)
sudo apt update && sudo apt install swaks -y

# 发送测试邮件
# 这里的 --from 需要和你 smtp_user.txt 中的邮箱保持一致
swaks --to target_email@example.com \
      --from your_email@example.com \
      --server 127.0.0.1 --port 25
```

如果日志中显示 `250 Message accepted` 或类似字样，并成功收到邮件，说明你的邮件中继服务器已经完美运行！

## ⚠️ 常见排错 (FAQ)

- **端口 25 被占用 (`bind: address already in use`)**：Linux 系统通常会默认安装 `postfix` 或 `sendmail` 占用 25 端口。你可以通过 `sudo netstat -tulpn | grep :25` 找出占用进程，并使用 `sudo systemctl disable --now postfix` 停用自带服务。
