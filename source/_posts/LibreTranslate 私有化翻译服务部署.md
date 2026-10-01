---
title: LibreTranslate 私有化翻译服务部署
date: 2026-09-30 15:30:00
tags: [笔记, LibreTranslate, 自托管]
---

# LibreTranslate 私有化翻译服务部署

## 1. 项目信息

- **开源仓库**: [LibreTranslate/LibreTranslate](https://github.com/LibreTranslate/LibreTranslate)
- **Docker 镜像**: [libretranslate/libretranslate Tags](https://hub.docker.com/r/libretranslate/libretranslate/tags)

> **💡 项目简介**：LibreTranslate 是一个免费的开源机器翻译 API，完全自建，不依赖 Google 或 Azure 等专有服务商。

---

## 2. 准备工作

创建 LibreTranslate 的工作目录并进入该目录：

```bash
sudo mkdir -p /opt/libretranslate
cd /opt/libretranslate
```

---

## 3. 编写编排文件 (Docker Compose)

```bash
sudo nano /opt/libretranslate/docker-compose.yml
```

填入以下内容

```yaml
services:
  libretranslate:
    image: libretranslate/libretranslate:latest
    container_name: libretranslate
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "python3 -c 'import urllib.request; urllib.request.urlopen(\"http://localhost:5000/health\")' || exit 1"]
      interval: 10s
      timeout: 4s
      retries: 4
      # 首次启动需下载模型，耗时较长，宽限期调高至 120 秒以防止误判为不健康
      start_period: 120s 
    ports:
      - 1057:5000
    volumes:
      # 持久化 API 密钥数据 (对应 LT_API_KEYS 功能)
      - lt-db:/app/db
      # 持久化下载的语言模型，避免容器重启后重新下载浪费时间
      - lt-local:/home/libretranslate/.local
      - type: tmpfs
        target: /tmp
        tmpfs:
          mode: 01777      # 赋予标准 tmp 权限，解决自定义 UID 的读写问题
          size: "512M"     # (建议) 限制 tmpfs 使用的最大内存大小
    # 为容器分配一个伪终端
    tty: true
    environment:
      # 开启 API 密钥功能，配合 lt-db 数据卷使用
      - LT_API_KEYS=true
      
      # 仅下载并加载指定的语言模型（如：中、英、日）
      # 如果不加此参数，默认会下载所有数十种语言，可能需要几十GB磁盘和极高内存！
      - LT_LOAD_ONLY=zh,en,ja
      
      # 开启调试模式 (如有需要可取消注释)
      # - LT_DEBUG=true
    deploy:
      resources:
        limits:
          cpus: "1"
          # 【内存警告】1024M 仅适合加载 2~3 个语言模型。
          # 若需加载更多语言，请务必将内存提升至 2G 甚至 4G 以上，否则极易发生 OOM 崩溃！
          memory: "1024M"

# 定义命名数据卷
volumes:
  lt-db:
  lt-local:
```

---

## 4. 启动服务

确认配置无误后，在后台启动服务：

```bash
cd /opt/libretranslate
sudo docker compose up -d
```

### ⏳ 启动注意事项：
首次运行容器时，LibreTranslate 会在后台**自动下载**你配置在 `LT_LOAD_ONLY` 中的语言模型（中、英、日等）。你可以通过查看日志来观察下载进度：

```bash
sudo docker compose logs -f libretranslate
```
当日志中出现类似 `Running on http://0.0.0.0:5000` 的字样时，说明服务已完全启动。此时你可以通过 `http://宿主机IP:1057` 访问 Web 界面。

---

## 5. 进阶：API 密钥管理 (ltmanage)

因为我们在环境变量中开启了 `- LT_API_KEYS=true`，所以你可以通过命令行来生成和管理调用 API 所需的密钥。

**生成一个新的 API 密钥：**
```bash
sudo docker exec -it libretranslate ltmanage keys add
```
*(系统会输出一串生成的 API Key，请妥善保存)*

**生成限制调用次数的 API 密钥（例如限制每月 1000 次请求）：**
```bash
sudo docker exec -it libretranslate ltmanage keys add 1000
```

**查看已生成的全部 API 密钥列表：**
```bash
sudo docker exec -it libretranslate ltmanage keys
```

**删除指定的 API 密钥：**
```bash
sudo docker exec -it libretranslate ltmanage keys remove <你的API_KEY>
```
