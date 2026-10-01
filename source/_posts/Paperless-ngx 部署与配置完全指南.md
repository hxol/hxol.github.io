---
title: Paperless-ngx 部署与配置完全指南
date: 2026-09-30 15:33:00
tags: [笔记, Paperless-ngx, 自托管]
---

# Paperless-ngx 部署与配置完全指南

## 项目信息
* **官方介绍**: Paperless-ngx 是一个开源的文档管理系统，它可以将您的物理文档转化为可搜索的在线归档，实现无纸化办公。
* **GitHub 仓库**: [paperless-ngx/paperless-ngx](https://github.com/paperless-ngx/paperless-ngx)
* **官方镜像**: [ghcr.io/paperless-ngx](https://github.com/paperless-ngx/paperless-ngx/pkgs/container/paperless-ngx)

---

## 方案一：部署在 Linux 环境

### 1. 创建目录
```bash
sudo mkdir -p /opt/paperless-ngx/{consume,export,data,media}
cd /opt/paperless-ngx
```

### 2. 权限设置
为了确保 Docker 容器内的用户有权限读写宿主机挂载的目录，需要调整所有者权限。
查看当前用户的 uid 与 gid：
```bash
id
```
假设您的 UID 为 `1000`，GID 为 `1001`（请根据实际情况调整）：
```bash
sudo chown -R 1000:1001 /opt/paperless-ngx
sudo chmod -R 755 /opt/paperless-ngx
```

### 3. 环境变量配置
```bash
sudo nano /opt/paperless-ngx/.env
```
写入以下内容（**请修改域名和密钥**）：
```env
# 0. 软件名字
COMPOSE_PROJECT_NAME=paperless-ngx

# 1. 用户权限映射 (必须与宿主机实际账号一致)
USERMAP_UID=1000
USERMAP_GID=1001

# 2. 外部访问 URL (用于生成正确的文档链接和安全校验)
# 修改为您实际的反向代理域名
PAPERLESS_URL=https://paperless.example.com 

# 3. 安全密钥 (非常重要，请不要使用弱口令)
# 可以使用命令 `openssl rand -hex 32` 在终端生成一串随机字符填入此处
PAPERLESS_SECRET_KEY=your_super_secret_key_please_change_me

# 4. 时区设置
PAPERLESS_TIME_ZONE=Asia/Shanghai

# 5. OCR 语言设置
# 默认识别语言
PAPERLESS_OCR_LANGUAGE=chi_sim
# 附加安装的 Tesseract 语言包 (简体中文、繁体中文)
PAPERLESS_OCR_LANGUAGES=chi-sim chi-tra

# 6. 数据库设置
POSTGRES_DB=paperless
POSTGRES_USER=paperless
# 只要数据库密码不等于 paperless，必然会报错，除非同步修改 compose 文件中的连接字符串
POSTGRES_PASSWORD=paperless
```

### 4. 编写 Compose 文件
```bash
sudo nano /opt/paperless-ngx/docker-compose.yml
```
写入以下内容（注：PostgreSQL 建议使用 `16` 版本以保证稳定性）：
```yaml
services:
  broker:
    image: valkey/valkey:9-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data
    env_file: .env
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "50M"

  db:
    image: postgres:18
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql
    env_file: .env
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "70M"

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - db
      - broker
      - gotenberg
      - tika
    ports:
      - "1040:8000"
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    env_file: .env
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBHOST: db
      PAPERLESS_DBENGINE: postgresql
      PAPERLESS_TIKA_ENABLED: 1
      PAPERLESS_TIKA_GOTENBERG_ENDPOINT: http://gotenberg:3000
      PAPERLESS_TIKA_ENDPOINT: http://tika:9998
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1024M"

  gotenberg:
    image: gotenberg/gotenberg:8.34
    restart: unless-stopped
    # Gotenberg 的 Chromium 路由用于转换 .eml 等文件。禁止外部内容 (如追踪像素或 JS) 以保证安全。
    command:
      - "gotenberg"
      - "--chromium-disable-javascript=true"
      - "--chromium-allow-list=file:///tmp/.*"
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1500M"

  tika:
    image: apache/tika:latest
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "700M"

volumes:
  data:
  media:
  pgdata:
  redisdata:
```

### 5. 运行与查看状态
```bash
# 启动服务
cd /opt/paperless-ngx && sudo docker compose up -d

# 查看容器状态
sudo docker compose ps

# 查看 Web 服务实时日志 (初始化可能需要几分钟)
sudo docker compose logs -f webserver
```

---

## 方案二：部署在 Synology NAS

*提示：群晖 NAS 部署流程与 Linux 类似，主要区别在于用户权限的获取和目录路径。*

### 1. 目录与权限
```bash
# 创建目录
mkdir -p /volume1/docker/paperless-ngx/{consume,export,data,media}
cd /volume1/docker/paperless-ngx

# 获取群晖用户的 UID (假设用户为 container-user)
id -u container-user  # 记录输出值，例如 1068

# 获取群晖无文件夹权限组的 GID
grep '^no-folder-group:' /etc/group | cut -d: -f3 # 记录输出值，例如 65573

# 赋予目录权限
sudo chown -R 1068:65573 /volume1/docker/paperless-ngx
sudo chmod -R 755 /volume1/docker/paperless-ngx
```

### 2. 第三方 Docker 网络与防火墙
```bash
sudo docker network create public-net
sudo docker network create privacy-net
```
*注意：请在群晖“控制面板” -> “安全性” -> “防火墙”中，放行 `public-net` 与 `privacy-net` 对应的 IP 段或 Docker 接口。*

### 3. 环境变量设置
```bash
vi /volume1/docker/paperless-ngx/.env
```
内容同上文 Linux 环境的 `.env`，只需将 `USERMAP_UID` 和 `USERMAP_GID` 改为您刚才获取的 `1068` 和 `65573`。

### 4. 选择并编写 Compose 文件
根据您 NAS 的性能（特别是内存大小），选择一种配置部署。

#### 选项 A：高性能完整版 (Postgres + Tika + Gotenberg)
*适用：内存 ≥ 4GB 的 NAS，支持解析 Office 文档和复杂 PDF。*
文件内容同上文 Linux 章节的 `docker-compose.yml`。

#### 选项 B：轻量版 (SQLite 数据库，无 Tika)
*适用：入门级 NAS，内存较小，仅处理常规图片和 PDF。*
```bash
vi /volume1/docker/paperless-ngx/docker-compose.yml
```
```yaml
services:
  broker:
    image: valkey/valkey:9-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data

  webserver:
    image: ghcr.io/paperless-ngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - broker
    ports:
      - "1040:8000"
    volumes:
      - data:/usr/src/paperless/data
      - media:/usr/src/paperless/media
      - ./export:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    env_file: .env
    environment:
      PAPERLESS_REDIS: redis://broker:6379
      PAPERLESS_DBENGINE: sqlite
      
volumes:
  data:
  media:
  redisdata:
```

### 5. 运行
```bash
cd /volume1/docker/paperless-ngx && sudo docker compose up -d
```

---

## 基本配置与工作流最佳实践

### 1. 基础工作流设置 (Inbox Zero 理念)
在服务启动并创建管理员账号后，建议按照以下步骤初始化标签和视图，建立标准化的文档处理流程。

#### 1.1 核心标签设置
进入系统后台 **管理 (Administration) > 标签 (Tags)**，创建以下标签：
1. **Inbox (收件箱)**
   * **用途**: 标记未处理的新文件。
   * **关键设置**: 
     * [x] 勾选“收件箱标签 (Inbox tag)”。
     * **匹配算法**: 选择“禁用匹配 (None)” （防止系统自动为已有文档乱打此标签）。
2. **TODO (待办)**
   * **用途**: 标记需要后续人工介入的文件（如需要报销的单据）。
   * **匹配算法**: 可选“自动分配 (Auto)”或“手动分配 (Any)”。

#### 1.2 基础分类体系
* **文档类型 (Document Types)**: 创建如 `发票`、`收据`、`合同`、`银行对账单`。
  * *提示*: 对于排版固定的文件（如 `银行对账单`），可开启“自动学习匹配”。
* **通讯录/联系人 (Correspondents)**: 创建常用往来对象，如 `招商银行`、`国家电网`、`物业`。

#### 1.3 仪表盘视图配置
在“文档 (Documents)”列表页面，筛选特定标签，然后点击右上角“保存视图 (Save view)”并勾选“在仪表盘显示 (Show on dashboard)”：
1. **收件箱视图**: 筛选标签包含 `Inbox`。
2. **待办事项视图**: 筛选标签包含 `TODO`。

### 2. 自动化工作流 (Workflow) 高级配置
为了确保自动化逻辑在“文件刚上传自动识别”和“人工手动修改元数据”时都能生效，**强烈建议在触发器中同时包含“添加 (Added)”和“更新 (Updated)”事件**。

#### 2.1 示例：自动归档银行对账单
**目标**: 当文档被标记为“招商银行”的“银行对账单”时，自动打上归档标签并移出收件箱。

* **名称**: 自动分类-招行对账单
* **触发器 (Triggers)**: 添加两个触发器以确保健壮性。
  1. **文档已添加 (Document Added)** -> 筛选条件：类型=`银行对账单`, 通讯录=`招商银行`。
  2. **文档已更新 (Document Updated)** -> 筛选条件：类型=`银行对账单`, 通讯录=`招商银行`。
* **操作 (Actions)**:
  1. **分配标签**: `bank-statements` (或您自定义的归档标签)
  2. **移除标签**: `Inbox`

#### 2.2 为什么需要“双触发器”？
* 如果**仅使用“已添加”**：只有在上传瞬间，系统立刻识别成功才会触发工作流。如果是模糊的扫描图片，刚上传时可能未能识别出类型，后续人工修正类型时**不会**再次触发此流程。
* 使用**双触发器**：无论是系统自动识别成功，还是人工后续修正了文档属性，工作流都会被激活，确保自动化不漏单。

---

## 3. 消息通知集成 (ntfy)

利用 Paperless-ngx 的 Webhook 功能，可以将文档处理结果推送到自建或公共的 ntfy 服务器，甚至可以包含直接打开文档的操作按钮。

### 3.1 工作流 Webhook 设置
在工作流的 **操作 (Actions)** 模块中添加 **Webhook**：

* **Webhook URL**: `https://ntfy.example.com` *(注意：这里填您的 ntfy 基础域名，Topic 会在下方的参数里指定)*
* **HTTP 方法**: `POST`
* **参数编码**: `JSON`
* **使用参数作为 webhook 负载**: [x] 勾选此项

### 3.2 JSON 参数配置 (包含占位符)
将以下 JSON 格式的内容填入配置框。Paperless 会自动替换 `{}` 中的占位符：

```json
{
  "topic": "paperless_alerts", 
  "title": "📄 {document_type} 已归档: {correspondent}",
  "message": "文件名: {original_filename}\n创建日期: {created_date}",
  "tags": "receipt,archive",
  "priority": 4,
  "actions": [
    {
      "action": "view",
      "label": "打开文档",
      "url": "{doc_url}"
    }
  ]
}
```

*说明：*
* `"topic"`: 对应您在 ntfy 客户端订阅的主题名称（需自行更改，建议设置得复杂一点以防别人猜测到）。
* `"tags"`: ntfy 支持的表情符号标签。

### 3.3 常用占位符参考手册
在配置 Webhook 负载时，您可以组合使用以下占位符：
* `{document_type}`: 文档类型（如：发票）
* `{correspondent}`: 通讯录/联系人（如：招商银行）
* `{doc_url}`: 文档直达链接 (依赖于 `.env` 中 `PAPERLESS_URL` 环境变量的正确配置)
* `{tags}`: 文档当前包含的所有标签
* `{created_date}`: 文档创建日期
* `{original_filename}`: 原始文件名
