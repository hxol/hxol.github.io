---
title: ChangeDetection.io 部署与内存调优
date: 2026-09-30 15:30:00
tags: [笔记, ChangeDetection.io, 自托管]
---

# ChangeDetection.io 部署与内存调优

## 1. 项目参考信息

- **项目主页 (GitHub)**: [dgtlmoon/changedetection.io](https://github.com/dgtlmoon/changedetection.io)
- **主程序 Docker 镜像**: [Changedetection.io Packages](https://github.com/dgtlmoon/changedetection.io/pkgs/container/changedetection.io)
- **浏览器环境 Docker 镜像**: [Sockpuppetbrowser Tags](https://hub.docker.com/r/dgtlmoon/sockpuppetbrowser/tags)

---

## 2. 安装与部署

我们将使用 Docker Compose 来部署 ChangeDetection.io 及其配套的 Playwright 浏览器容器。

**第一步：创建工作目录并编辑配置文件**

```bash
sudo mkdir -p /opt/changedetection
cd /opt/changedetection
sudo nano docker-compose.yml
```

**第二步：填入 `docker-compose.yml` 内容**

将以下配置粘贴到文件中（已限制资源并优化了相关权限）：

```yaml
services:
  changedetection:
    image: ghcr.io/dgtlmoon/changedetection.io:latest
    container_name: changedetection
    hostname: changedetection
    environment:
      - PLAYWRIGHT_DRIVER_URL=ws://browser-sockpuppet-chrome:3000
      - HIDE_REFERER=true
      - TZ=Asia/Shanghai
      - ALLOW_IANA_RESTRICTED_ADDRESSES=true # 允许访问本地/局域网地址
    volumes:
      - changedetection-data:/datastore
      - type: tmpfs
        target: /tmp
        tmpfs:
          mode: 01777      # 赋予标准 tmp 权限，解决自定义 UID 的读写问题
          size: "512M"     # (强烈建议) 限制最大使用的内存大小，防止 OOM
    ports:
      - 1036:5000
    depends_on:
      browser-sockpuppet-chrome:
        condition: service_started
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1000M"

  browser-sockpuppet-chrome:
    hostname: browser-sockpuppet-chrome
    image: dgtlmoon/sockpuppetbrowser:latest
    cap_add:
      - SYS_ADMIN
    environment:
      - SCREEN_WIDTH=1920
      - SCREEN_HEIGHT=1024
      - SCREEN_DEPTH=16
      - MAX_CONCURRENT_CHROME_PROCESSES=1  # 限制 Chrome 并发数，降低内存占用
      - FETCH_WORKERS=1                    # 限制抓取 Worker 数
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "2"
          memory: "2G"

volumes:
  changedetection-data:
```

**第三步：启动服务**

保存并退出编辑器（在 nano 中按 `Ctrl+O`, `Enter`, `Ctrl+X`），然后在该目录下启动容器：

```bash
sudo docker compose up -d
```

---

## 3. 防止 Playwright 容器内存泄漏

长时间运行 Playwright/Chrome 容器极易引发内存泄漏问题。我们可以通过设置一个定时任务（Cron），定期重启浏览器容器来释放内存。

**第一步：创建自动重启脚本**

```bash
sudo nano /opt/changedetection/restart-playwright.sh
```

填入以下内容：

```bash
#!/bin/bash
# 切换到 docker-compose.yml 文件所在的目录
cd /opt/changedetection

# 仅重启浏览器服务，不影响主程序的运行
docker compose restart browser-sockpuppet-chrome
```

**第二步：赋予脚本可执行权限**

```bash
sudo chmod +x /opt/changedetection/restart-playwright.sh
```

**第三步：添加系统定时任务**

编辑宿主机的 crontab：

```bash
sudo crontab -e
```

在文件末尾添加以下规则，设置为**每 30 分钟**执行一次自动重启：

```conf
*/30 * * * * /opt/changedetection/restart-playwright.sh >/dev/null 2>&1
```
*(注：执行日志已被重定向到黑洞，以避免产生多余的系统垃圾邮件。)*
