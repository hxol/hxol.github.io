---
title: Linkwarden 完整部署向导
date: 2026-09-30 15:30:00
tags: [笔记, Linkwarden, 自托管]
---

# Linkwarden 完整部署向导（基于 Docker Compose）

Linkwarden 是一个自托管的书签和网页存档工具。本文将指导你如何通过 Docker Compose 快速部署并配置该服务。

## 📌 参考资料与相关链接
- **Docker 镜像库**：[getmeili/meilisearch](https://hub.docker.com/r/getmeili/meilisearch/tags)
- **开源代码仓库**：[Linkwarden GitHub](https://github.com/linkwarden/linkwarden)
- **官方参考文档**：[官方文档首页](https://docs.linkwarden.app/) | [环境变量说明](https://docs.linkwarden.app/self-hosting/environment-variables)
- **官方配置文件**：[docker-compose.yml 模板](https://raw.githubusercontent.com/linkwarden/linkwarden/refs/heads/main/docker-compose.yml) | [.env 模板](https://raw.githubusercontent.com/linkwarden/linkwarden/refs/heads/main/.env.sample)

---

## 🛠️ 第一步：创建目录
首先，为 Linkwarden 创建工作目录并进入该目录：
```bash
sudo mkdir -p /opt/linkwarden && cd /opt/linkwarden
```

---

## ⚙️ 第二步：配置环境变量 (.env)
创建并编辑环境变量文件：
```bash
sudo nano /opt/linkwarden/.env
```

将以下内容填入文件（**⚠️ 注意：生产环境中请务必修改所有的密码和密钥，切勿直接使用示例中的配置！**）：

```env
# ==========================================
# 🌐 基础与认证配置
# ==========================================
# 公开访问 URL (若使用反向代理，请填写完整域名；若使用内网 IP，请填入对应的 IP 和端口)
NEXTAUTH_URL=https://linkwarden.lab.io/api/v1/auth

# 会话加密密钥 (用于签名/加密 Cookie 及安全令牌)
# 可通过命令生成: openssl rand -hex 32
NEXTAUTH_SECRET=a17ccb46959739bd9d4f1331c582JYEdriJwCwoH4c304590b0ivr2gDtce2yA6o

# ==========================================
# 🗄️ 数据库配置 (PostgreSQL)
# ==========================================
# 手动安装时的数据库连接字符串 (当前使用 Docker，此项可留空，将自动组合)
DATABASE_URL=
# 数据库 root 用户密码
POSTGRES_PASSWORD=uJ3T7qisxH7vyk

# ==========================================
# 🔍 搜索引擎配置 (Meilisearch)
# ==========================================
# 搜索引擎服务的内部地址
MEILI_HOST=http://meilisearch:7700
# 搜索引擎主密钥 (用于保护 API 访问)
MEILI_MASTER_KEY=mPfgv44ZvFZU55r9bL

# ==========================================
# 🤖 AI 功能配置 (可选)
# ==========================================
# --- 方案 A: Ollama 配置 ---
NEXT_PUBLIC_OLLAMA_ENDPOINT_URL=
OLLAMA_MODEL=

# --- 方案 B: OpenAI 或第三方兼容 API 配置 ---
# 参考: https://ai-sdk.dev/providers/openai-compatible-providers
OPENAI_API_KEY=sk-ye1RYspQaYACYLABU0v6Qw
OPENAI_MODEL=openrouter/openrouter/free
CUSTOM_OPENAI_BASE_URL=https://litellm.lab.io
CUSTOM_OPENAI_NAME=
```

---

## 🐳 第三步：配置编排文件 (docker-compose.yml)
创建并编辑 Docker Compose 文件：
```bash
sudo nano /opt/linkwarden/docker-compose.yml
```

写入以下配置：

```yaml
services:
  postgres:
    image: postgres:18
    env_file: .env
    restart: always
    volumes:
      # 注意：postgres:18 官方镜像结构发生变更，数据需挂载至 /var/lib/postgresql
      - ./postgresql_data:/var/lib/postgresql
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "100M"

  meilisearch:
    image: getmeili/meilisearch:v1.13.3
    restart: always
    env_file:
      - .env
    volumes:
      - ./meilisearch_data:/meili_data
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "1024M"

  linkwarden:
    image: ghcr.io/linkwarden/linkwarden:latest
    restart: always
    env_file: .env
    environment:
      # 动态拼接数据库连接串
      - DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@postgres:5432/postgres
    ports:
      - 1099:3000
    volumes:
      - ./linkwarden_data:/data/data
      - type: tmpfs
        target: /tmp
        tmpfs:
          mode: 01777      # 赋予标准 tmp 权限，解决自定义 UID 的读写问题
          size: "512M"     # (强烈建议) 限制最大使用的内存大小，防止 OOM
    depends_on:
      - postgres
      - meilisearch
    deploy:
      resources:
        limits:
          cpus: "1.5"
          memory: "1500M"
```

---

## 🚀 第四步：启动服务与验证
在工作目录下，拉取镜像并在后台启动所有服务：
```bash
cd /opt/linkwarden && docker compose up -d
```

**访问与验证：**
服务启动后，若你配置了反向代理，可通过 `https://linkwarden.lab.io` 访问；若在局域网内直接测试，请访问 `http://<服务器IP>:1099`。

---

## 📖 附录：核心环境变量解析

为了确保 Linkwarden 正常运行，了解各参数的含义非常重要。

### 🚨 必须修改的环境变量

| 变量名 | 作用解析与配置建议 |
| :--- | :--- |
| `NEXTAUTH_URL` | **作用**：定义 Linkwarden 的公开访问 URL，用于生成回调链接、认证验证及系统邮件中的跳转链接。<br>**配置**：<br>- **反代域名访问**：需带有 `/api/v1/auth` 后缀，例如 `https://linkwarden.lab.io/api/v1/auth`<br>- **本地 IP 访问**：需指定对应端口，例如 `http://192.168.251.221:1099/api/v1/auth` (注意端口需与映射端口一致)。 |
| `NEXTAUTH_SECRET` | **作用**：NextAuth（认证库）正常运行的安全基石，用于签名和加密会话 Cookie 及安全令牌。<br>**警告**：如果为空，应用将无法启动或无法登录。<br>**建议**：使用命令 `openssl rand -hex 32` 生成一个 64 位的随机强密码串填入。 |
| `POSTGRES_PASSWORD` | **作用**：设置 PostgreSQL 数据库中 `postgres` 用户的密码。<br>**配置**：Linkwarden 主服务会通过拼接此密码来连接数据库。请设置复杂的随机字符串以防爆破。 |
| `MEILI_HOST` | **作用**：告知 Linkwarden 主程序 Meilisearch 搜索引擎服务的内网地址。<br>**配置**：由于我们使用的是 Docker 网络，直接使用服务名和端口即可：`http://meilisearch:7700`。 |
| `MEILI_MASTER_KEY` | **作用**：Meilisearch API 访问的主密钥，保障搜索服务的安全。<br>**配置**：Linkwarden 与 Meilisearch 实例通信的凭据。请生成一个独一无二的随机字符串。 |

---

### ✨ 进阶配置：AI 功能环境变量 (可选)

Linkwarden 支持接入 AI 模型来实现自动网页摘要或标签建议。你可以根据自己使用的后端选择配置 Ollama 或是 OpenAI 兼容接口。

#### 方案一：接入 Ollama (本地部署的大模型)
```env
NEXT_PUBLIC_OLLAMA_ENDPOINT_URL=http://192.168.251.222:11434
OLLAMA_MODEL=phi3:mini-4k
```
> **💡 注意事项**：示例中使用的 `phi3:mini-4k` 模型仅对英文有良好支持，若需要处理中文网页，建议更换为如 `qwen`、`llama3` 等具备较好中文能力的本地模型。

#### 方案二：接入 OpenAI / 第三方兼容 API (如 LiteLLM)
```env
CUSTOM_OPENAI_BASE_URL=https://litellm.lab.io
OPENAI_MODEL=gemini/gemini-2.0-flash
OPENAI_API_KEY=sk-iOFL0U8blm8SifZzPoCV3A
```
> **💡 适用场景**：适合使用官方 OpenAI，或者通过 LiteLLM、OneAPI 等中间件转接 Claude、Gemini 等模型的场景。只需修改 `BASE_URL` 并填入对应平台的 `MODEL` 标签及 `API_KEY` 即可。