---
title: Ntfy 消息通知服务器部署与配置指南
date: 2026-09-30 15:33:00
tags: [笔记, Ntfy, 自托管]
---

# Ntfy 消息通知服务器部署与配置指南

## 项目简介
Ntfy (Pronounced "notify") 是一个基于 HTTP 的开源发布/订阅通知服务。它允许您通过简单的 HTTP PUT/POST 请求向手机或桌面发送推送通知。
- **官方仓库**: [https://github.com/binwiederhier/ntfy](https://github.com/binwiederhier/ntfy)
- **官方文档**: [https://docs.ntfy.sh](https://docs.ntfy.sh)

---

## 1. 基础环境配置 (Linux)

### 1.1 创建目录
创建用于存放数据和配置的目录，并进入该目录：
```bash
sudo mkdir -p /opt/ntfy/{data,config}
cd /opt/ntfy
```

### 1.2 配置权限
查看当前用户的 `uid` 与 `gid`：
```bash
id
```
> *注：结果一般是 1000:1000 或 1000:1001。以下假设为 1000:1001。*

赋予目录相应的权限，确保容器内部有权读写：
```bash
sudo chown -R 1000:1001 /opt/ntfy
sudo chmod -R 755 /opt/ntfy
```

### 1.3 环境变量设置
创建 `.env` 文件：
```bash
sudo nano /opt/ntfy/.env
```
填入以下内容：
```env
NTFY_IMAGE=binwiederhier/ntfy:v2.20.1
PUID=1000
PGID=1001
TZ=Asia/Shanghai
```

---

## 2. 配置文件与编排

### 2.1 主配置文件 (`server.yml`)
创建并编辑 Ntfy 的配置文件：
```bash
sudo nano /opt/ntfy/config/server.yml
```
填入以下内容（请根据需要修改域名）：
```yaml
# Ntfy 服务在容器内部监听的地址和端口
listen-http: ":8080"

# 是否启用反向代理暴露 Ntfy 到公网
behind-proxy: true

# Ntfy 被反代后的公网地址 (需替换为您自己的域名)
base-url: "https://ntfy.example.com"

# 缓存文件的路径 (位于容器内)
cache-file: /var/lib/ntfy/cache.db

# 定义消息存储在缓存中的持续时间 (默认 12h)
cache-duration: 72h

# 用户认证文件路径 (放在可读写的 data 目录下，解决容器只读限制)
auth-file: /var/lib/ntfy/users.db

# 默认访问权限。纯局域网测试时可设为 "read-write"，生产环境强烈建议设为 "deny-all"
auth-default-access: "deny-all"

# 附件相关的设置 (可选)
attachment-cache-dir: /var/lib/ntfy/attachments
attachment-total-size-limit: 1G
attachment-file-size-limit: 25M
attachment-expiry-duration: 24h

# Web PUSH 设置 (可选，如果未通过反向代理处理 HTTPS，需要配置此项以支持浏览器通知)
# web-push-public-key: "YOUR_VAPID_PUBLIC_KEY"
# web-push-private-key: "YOUR_VAPID_PRIVATE_KEY"
# web-push-email-address: "mailto:your-email@example.com"
```

### 2.2 Docker Compose 编排文件
创建 `docker-compose.yaml`：
```bash
sudo nano /opt/ntfy/docker-compose.yaml
```
填入以下内容：
```yaml
services:
  ntfy:
    image: ${NTFY_IMAGE}
    container_name: ntfy
    command:
      - serve
    ports:
      - 1030:8080
    user: "${PUID}:${PGID}"
    # 启用容器文件系统只读，提升安全性
    read_only: true
    volumes:
      # 配置文件目录 (只读)
      - ./config:/etc/ntfy:ro
      # 状态数据目录：包含主题订阅、缓存、附件及权限配置数据库 (必须可写)
      - ./data:/var/lib/ntfy
    # 配合文件系统的只读模式，将临时目录映射为内存(tmpfs)
    tmpfs:
      - /tmp
      - /run
    environment:
      TZ: "${TZ}"
    restart: unless-stopped
    healthcheck:
      # 健康检查：端口需与 server.yml 中的 listen-http 一致
      test: ["CMD-SHELL", "wget -q --tries=1 http://localhost:8080/v1/health -O - | grep -Eo '\"healthy\"\\s*:\\s*true' || exit 1"]
      interval: 60s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: '100M'
        # 可选：软预留
        # reservations:
          # cpus: '0.25'
          # memory: '50M'

```

---

## 3. 启动服务与权限配置

### 3.1 创建认证数据库并启动容器
因为我们在 `server.yml` 中开启了权限认证（`deny-all`），为了防止 Ntfy 找不到数据库文件报错，先创建一个空文件：
```bash
sudo touch /opt/ntfy/data/users.db
sudo chown 1000:1001 /opt/ntfy/data/users.db
```

启动服务：
```bash
cd /opt/ntfy
docker compose up -d
```
查看状态与日志：
```bash
docker compose ps
docker compose logs -f ntfy
```

### 3.2 账户与权限控制 (ACL)
为保证系统安全，我们需要进入容器内部创建专用账号，并分配读写权限。

1. **进入 Ntfy 容器：**
```bash
docker exec -it ntfy sh
```

2. **添加发信账号 (Sender)：**
```bash
ntfy user add sender_user
```
> *系统会提示您输入密码，请设置一个强密码并妥善保存（如 `SenderPass_123!`）。*

3. **赋予发信账号写入权限：**
```bash
ntfy access sender_user prometheus write
ntfy access sender_user wazuh write
ntfy access sender_user nas write
```

4. **添加查收账号 (Viewer)：**
```bash
ntfy user add viewer_user
```
> *同样设置并保存查收密码（如 `ViewerPass_456#`）。*

5. **赋予查收账号读取权限：**
```bash
ntfy access viewer_user prometheus read
ntfy access viewer_user wazuh read
ntfy access viewer_user nas read
```

6. **退出并重启生效：**
```bash
exit
docker compose restart ntfy
```

---

## 4. 客户端订阅与测试

### 4.1 测试发送消息
在任意终端测试发送一条需要认证的消息：
```bash
curl -v -X POST \
  -u "sender_user:<您的发信密码>" \
  -H "Title: Ntfy Test" \
  -d "这是一条来自 Curl 的加密测试消息" \
  "https://ntfy.example.com/nas"
```

### 4.2 客户端订阅 (Web & App)
1. **浏览器订阅**：
   访问 `https://ntfy.example.com`，点击左下角的账号图标登录您的 `viewer_user`，然后订阅 `prometheus`、`wazuh` 或 `nas` 主题，并允许浏览器通知。
2. **手机 App**：
   下载 Ntfy 客户端 (Android/iOS)。
   - 添加服务器：`https://ntfy.example.com`
   - 在设置中添加用户：`viewer_user` 及对应密码。
   - 订阅对应主题。

---

## 5. 进阶：配置 Synology NAS Webhook 推送

要让群晖 NAS 通过 Ntfy 发送系统通知，请在群晖控制面板的 "通知设置" -> "Webhook" 中按照以下参数配置：

### 5.1 Webhook URL
确保 URL 是干净的，**不要**包含 `?text=...` 等查询参数。格式必须带有认证信息：
```text
https://sender_user:<您的发信密码>@ntfy.example.com/nas
```
*(注意：请将 `<您的发信密码>` 替换为实际密码，并将域名替换为您自己的。)*

### 5.2 HTTP 请求设置
- **HTTP 方法 (HTTP Method)**: `POST`
- **HTTP 标头 (HTTP Headers)**:
  - `Content-Type`: `application/json`
- **HTTP 主体 (HTTP Body)**:
  请在文本框中粘贴以下 JSON 格式的内容：

```json
{
  "title": "[NAS系统通知]",
  "message": "@@TEXT@@"
}
```

**参数说明：**
- `"title"`: Ntfy 通知的标题。您可以自定义，例如写死为 `"[NAS系统通知]"`。
- `"message": "@@TEXT@@"`: 这是群晖系统内置的变量，群晖会自动将报警内容的文本替换掉 `@@TEXT@@`，并通过 Ntfy 的 `message` 字段发送出来。

### 5.3 保存并测试
配置完成后，点击“保存 (Save)”或“应用 (Apply)”。随后可以在群晖面板发送一条测试通知，您的手机或浏览器 Ntfy 客户端应当能立刻收到带有标题和内容的推送。
