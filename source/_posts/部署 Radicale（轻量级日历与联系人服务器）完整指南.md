---
title: 部署 Radicale（轻量级日历与联系人服务器）完整指南
date: 2026-09-30 15:36:00
tags: [笔记, Radicale, 自托管]
---

# 部署 Radicale（轻量级日历与联系人服务器）完整指南

## 1. 项目简介

[Radicale](https://github.com/Kozea/Radicale) 是一款使用 Python 编写的开源、轻量级 CalDAV 和 CardDAV 服务器，非常适合个人或小团队用来同步日历、待办事项和联系人。

## 2. 准备工作

首先，在宿主机上创建存放配置文件和数据的目录，并进入该目录：

```bash
sudo mkdir -p /opt/radicale/{config,data}
cd /opt/radicale
```

## 3. 创建 Radicale 配置文件

新建并编辑配置文件：

```bash
sudo nano /opt/radicale/config/config
```

填入以下内容：

```ini
[server]
# 必须绑定 0.0.0.0，否则 Docker 容器外部无法访问
hosts = 0.0.0.0:5232, [::]:5232

[auth]
# 开启 htpasswd 账号密码验证
type = htpasswd

# 指定密码文件的位置（注意：这里写的是容器内的绝对路径）
htpasswd_filename = /etc/radicale/users

# 密码加密方式：自动检测
htpasswd_encryption = autodetect

[storage]
# 数据存储目录（注意：这里写的是容器内的绝对路径）
filesystem_folder = /var/lib/radicale/collections
```

## 4. 创建账号密码文件

您可以从以下两种方法中选择一种来生成账号密码文件（`users`）。推荐使用**方法一**，因为无需在宿主机上额外安装软件包。

> **⚠️ 注意：** 
> 下方命令中的 `your_username` 请替换为您自己想设置的登录用户名。
> 首次创建文件时需带有 `-c` 参数；**如果后续需要添加更多用户，请务必去掉 `-c` 参数再执行**，否则会覆盖原有的密码文件！

### 方法一：使用 Docker 临时容器创建（推荐）

通过运行一次性的 `httpd:alpine` 容器来调用 `htpasswd` 工具，并使用 bcrypt 强加密（`-B`）：

```bash
sudo docker run --rm -it -v /opt/radicale/config:/config httpd:alpine htpasswd -B -c /config/users your_username
```
*执行后，终端会提示您输入并确认密码。*

### 方法二：手动安装 apache2-utils 创建

如果您更习惯在宿主机直接操作，可以安装相关工具：

```bash
sudo apt update && sudo apt install apache2-utils -y
```

创建账号及密码：

```bash
# 在 /tmp 下临时生成文件
htpasswd -B -c /tmp/users your_username

# 将生成的文件移动到配置目录中
sudo mv /tmp/users /opt/radicale/config/users

# 查看文件内容确认是否生成成功
cat /opt/radicale/config/users
```

## 5. 设置目录权限

Radicale 容器默认使用非 root 用户（UID 为 `1000`，GID 为 `1000`）运行以保证安全性。因为前面的目录是我们使用 `sudo` 创建的，默认所有者是 `root`，这会导致容器因权限不足而无法在 `/opt/radicale/data` 目录中写入日历数据。

我们需要将 `data` 和 `config` 目录的所有权赋予容器内的运行用户：

```bash
sudo chown -R 1000:1000 /opt/radicale/data
sudo chown -R 1000:1000 /opt/radicale/config
```

## 6. 编写 Docker Compose 文件

创建并编辑 `docker-compose.yml` 文件：

```bash
sudo nano /opt/radicale/docker-compose.yml
```

填入以下内容：

```yaml
services:
  radicale:
    image: ghcr.io/kozea/radicale:stable
    container_name: radicale
    restart: unless-stopped
    ports:
      # 宿主机端口:容器内端口，1102 可根据需求替换为其他未被占用的端口
      - "1102:5232"
    volumes:
      - ./config:/etc/radicale
      - ./data:/var/lib/radicale
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "500M"
```

## 7. 启动与管理服务

### 启动服务
在 `/opt/radicale` 目录下以后台模式启动容器：

```bash
sudo docker compose up -d
```

### 查看运行状态
确认容器是否正常启动：

```bash
sudo docker compose ps
```

### 查看实时日志
如果遇到问题，可以通过查看日志来排错（按 `Ctrl+C` 退出）：

```bash
sudo docker compose logs -f
```

## 8. 登录与使用

### Web 端登录
打开浏览器，访问：
`http://<您的服务器IP>:1102`
使用前面创建的用户名和密码即可登录 Web 界面，在其中您可以创建新的日历 (Calendar) 或通讯录 (Address Book)。

### 客户端同步设置
- **iOS/Mac (Apple)**：在设置中添加账户 -> 其他 -> 添加 CalDAV/CardDAV 账户，填入服务器地址、用户名及密码。
- **Android**：推荐使用 [DAVx⁵](https://www.davx5.com/)（开源应用）来连接您的 Radicale 服务器，同步到手机本地日历和通讯录。
- **Windows/Linux**：推荐使用 Thunderbird，原生支持 CalDAV 与 CardDAV。

---

## 💡 进阶补充建议 (Pro-Tips)

1. **配置 HTTPS 反向代理 (强烈推荐)**：
   绝大多数设备（尤其是 iOS 和 Android）在同步日历和通讯录时**强制要求或强烈建议使用 HTTPS 协议**。建议在 Radicale 前端配置 Nginx、Caddy 或 Traefik 进行反向代理，并申请免费的 SSL 证书（如 Let's Encrypt）。
2. **数据备份**：
   您的所有日历和联系人数据都存储在 `/opt/radicale/data` 目录中（以普通文本文件的形式）。建议定期通过 `cron` 和 `tar` 命令将该目录打包备份，以防数据丢失。