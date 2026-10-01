---
title: 部署与配置 Nexus 3 全面指南：Docker、APT 与 Hugging Face 私有代理仓库
date: 2026-09-30 15:30:00
tags: [笔记, Nexus 3, 自托管]
---


# 部署与配置 Nexus 3 全面指南：Docker、APT 与 Hugging Face 私有代理仓库

**参考信息：** [官方 Docker 镜像 (sonatype/nexus3)](https://hub.docker.com/r/sonatype/nexus3/tags)

域名以 `domaim.com` 作为例子。

---

## 壹、 安装 Nexus 3 

### 方案 A：在标准 Linux 系统上安装

**1. 创建目录**
```bash
sudo mkdir -p /opt/nexus/{nexus-data,nexus-keys} && cd /opt/nexus
```

**2. 配置权限**
Nexus 3 容器内部运行用户的 UID 和 GID 固定为 `200:200`，必须赋予挂载目录相应的权限：
```bash
cd /opt/nexus
sudo chown -R 200:200 ./nexus-data ./nexus-keys
sudo chmod -R 770 ./nexus-data ./nexus-keys
```

**3. 创建 JSON 密钥文件**
```bash
sudo nano /opt/nexus/nexus-keys/nexus-secrets.json
```
填入以下内容：
```json
{
  "active": "my-custom-key-id-01",
  "keys": [
    {
      "id": "my-custom-key-id-01",
      "key": "UfySlSkVAkzsQcY9c8sX2SiudoXxiIHfSxFPKgh9Iok="
    }
  ]
}
```

**4. 编写 Docker Compose 文件**
```bash
sudo nano /opt/nexus/docker-compose.yml
```
填入以下内容：
```yaml
services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped
    ports:
      - 1028:8081
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: '6G'
        # 可选：软预留
        # reservations:
          # cpus: '0.25'
          # memory: '1G'
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2703m -Xmx2703m -XX:MaxDirectMemorySize=2703m -Djava.util.prefs.userRoot=/nexus-data/javaprefs
      - NEXUS_SECRETS_KEY_FILE=/nexus-secrets/nexus-secrets.json
    volumes:
      - ./nexus-data:/nexus-data
      - ./nexus-keys:/nexus-secrets:ro
```

**5. 运行与查看状态**
```bash
cd /opt/nexus
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs -f nexus
```

**6. 获取初始管理员密码**
```bash
sudo cat /opt/nexus/nexus-data/admin.password
```

---

### 方案 B：在群晖 (Synology) NAS 上安装

**1. 创建目录**
```bash
mkdir -p /volume1/docker/nexus/{nexus-data,nexus-keys} && cd /volume1/docker/nexus
```

**2. 配置权限**
```bash
cd /volume1/docker/nexus
sudo chown -R 200:200 ./nexus-data ./nexus-keys
sudo chmod -R 770 ./nexus-data ./nexus-keys
```

**3. 创建 JSON 密钥文件**
首先，生成一个随机密钥备用：
```bash
openssl rand -base64 32
```
创建文件并填入内容（将生成的密钥替换到下方的 `key` 字段）：
```bash
sudo vi /volume1/docker/nexus/nexus-keys/nexus-secrets.json
```
```json
{
  "active": "my-custom-key-id-01",
  "keys": [
    {
      "id": "my-custom-key-id-01",
      "key": "此处替换为上面命令生成的32位密钥"
    }
  ]
}
```

**4. 编写 Docker Compose 文件**
```bash
sudo vi /volume1/docker/nexus/docker-compose.yml
```
填入以下内容：
```yaml
services:
  nexus:
    image: sonatype/nexus3:latest
    container_name: nexus
    restart: unless-stopped
    ports:
      - 1028:8081
    mem_limit: 6g
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms2703m -Xmx2703m -XX:MaxDirectMemorySize=2703m -Djava.util.prefs.userRoot=/nexus-data/javaprefs
      - NEXUS_SECRETS_KEY_FILE=/nexus-secrets/nexus-secrets.json
    volumes:
      - ./nexus-data:/nexus-data
      - ./nexus-keys:/nexus-secrets:ro
    networks:
      - public-net

networks:
  public-net:
    external: true
```

**5. 运行与获取密码**
```bash
cd /volume1/docker/nexus && sudo docker compose up -d
# 获取初始密码
sudo cat /volume1/docker/nexus/nexus-data/admin.password
```

**6. 触发重新加密任务 (可选)**
系统通常会自动执行。若未执行，可通过 REST API 手动触发（等待 Nexus 完全启动后运行）：
```bash
curl -v -u "admin:<你的admin密码>" -X PUT \
  'https://nexus.domaim.com/service/rest/v1/secrets/encryption/re-encrypt' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "secretKeyId": "my-custom-key-id-01"
}'
```
> **注意**：`secretKeyId` 需与 `nexus-secrets.json` 中一致。请求成功会返回一个任务 ID，可在 UI 的 **System** -> **Tasks** 中查看进度。

**7. 访问 Nexus**
初次启动需等待 2-5 分钟（取决于 NAS 性能），随后在浏览器访问配置好的域名或 IP（例如：`https://nexus.domaim.com` 或 `http://<NAS_IP>:1028`）。

---

## 贰、 基础配置：创建 Blob Store (存储块)

创建任何仓库前，建议为其分配独立的存储块，以便于管理。

1. **Docker 专用 Blob Store**：
   - 导航至：⚙️ **Settings** -> **Repository** -> **Blob Stores** -> **Create blob store**
   - **Type**: `File`
   - **Name**: `docker-hub`
   - 点击 **Save**。
   - *(依次继续创建：`docker-group-hub`, `docker-hosted`, `gcr-hub`, `ghcr-hub`, `quay-hub` 等)*

2. **APT 专用 Blob Store**：
   - 同上操作，**Name** 设置为 `apt-hub`，点击 **Save**。

---

## 叁、 配置 Docker 仓库代理与托管

### 1. 代理 Docker Hub

- 导航至：⚙️ **Settings** -> **Repository** -> **Repositories** -> **Create repository**
- **Recipe**: `docker (proxy)`
- **Name**: `docker-hub-proxy`
- **Online**: 勾选 ✅
- **Repository Connectors**: 勾选 `Path based routing`
- **Allow anonymous docker pull**: 勾选 ✅
- **Remote storage**: `https://registry-1.docker.io`
- **Docker Index**: 选择 `Use Docker Hub`
- **Blob Store**: 选择 `docker-hub`
- 点击 **Save**。

**⚠️ 启用 Docker Bearer Token Realm** (重要)
导航至：⚙️ **Settings** -> **Security** -> **Realms**。将 `Docker Bearer Token Realm` 从左侧 **Available** 移至右侧 **Active** 列表。

**客户端使用 (Docker 镜像加速)**：
编辑 `/etc/docker/daemon.json`：
```json
{
  "registry-mirrors": ["https://nexus.domaim.com/repository/docker-hub-proxy"]
}
```

### 2. 代理其他主流 Registry (GHCR / GCR / Quay)

创建流程与 Docker Hub 类似，仅修改以下字段（**注意 Docker Index 均需选择 `Use proxy registry (specified above)`**）：

| 仓库名称 | Remote storage | Blob store |
| :--- | :--- | :--- |
| **docker-ghcr-proxy** | `https://ghcr.io` | `ghcr-hub` |
| **docker-gcr-proxy** | `https://gcr.io` | `gcr-hub` |
| **docker-quay-proxy** | `https://quay.io` | `quay-hub` |

**客户端使用示例**：
- **CLI**: `docker pull nexus.domaim.com/docker-ghcr-proxy/grafana/grafana`
- **Docker Compose**: 将 `image: gcr.io/xxx` 替换为 `image: nexus.domaim.com/docker-gcr-proxy/xxx`。

### 3. 创建本地托管仓库 (Docker Hosted)

用于推送和存储企业内部或个人自建的镜像。

- **Recipe**: `docker (hosted)`
- **Name**: `docker-local`
- **HTTP Connector**: 勾选 `Path based routing`
- **Allow redeploy**: 按需勾选（生产环境通常不推荐）
- **Blob store**: `docker-hosted`

**上传镜像到 Hosted 仓库**：
```bash
# 1. 登录
docker login nexus.domaim.com
# 2. 打标签
docker tag baikal:latest nexus.domaim.com/docker-local/baikal:latest
# 3. 推送
docker push nexus.domaim.com/docker-local/baikal:latest
```

### 4. 创建聚合仓库 (Docker Group)

将 Proxy 和 Hosted 仓库组合成一个统一的访问入口。

- **Recipe**: `docker (group)`
- **Name**: `docker-group`
- **Repository Connectors**: 勾选 `Path based routing`
- **Allow anonymous docker pull**: 勾选 ✅
- **Member repositories**: 将之前创建的 proxy 和 hosted 仓库移入右侧。
- **Blob Store**: `docker-group-hub`

**客户端使用**：
```yaml
image: nexus.domaim.com/docker-group/wazuh-manager:20260101
```

---

## 肆、 配置 APT 软件源代理与托管

所有 APT 代理仓库配置页面底部的 **HTTP/HTTPS 连接器端口**请**保持空白**，不要设置。

### 1. Debian (Trixie)
- **debian-trixie-proxy**: Remote `http://deb.debian.org/debian/`, Distribution `trixie`
- **debian-trixie-security-proxy**: Remote `http://security.debian.org/debian-security/`, Distribution `trixie-security`

**Debian 客户端配置 (DEB822 格式)**：
```bash
sudo mv /etc/apt/sources.list.d/debian.sources /etc/apt/sources.list.d/debian.sources.disabled
sudo nano /etc/apt/sources.list.d/nexus.sources
```
```ini
Types: deb
URIs: https://nexus.domaim.com/repository/debian-trixie-proxy/
Suites: trixie trixie-updates trixie-backports
Components: main contrib non-free-firmware non-free
Enabled: yes
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://nexus.domaim.com/repository/debian-trixie-security-proxy/
Suites: trixie-security
Components: main contrib non-free-firmware non-free
Enabled: yes
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

### 2. Ubuntu 24.04 LTS (Noble)
- **ubuntu-noble-proxy**: Remote `http://archive.ubuntu.com/ubuntu/`, Distribution `noble`
- **ubuntu-noble-security-proxy**: Remote `http://security.ubuntu.com/ubuntu/`, Distribution `noble-security`

**Ubuntu 客户端配置 (DEB822 格式)**：
```bash
sudo nano /etc/apt/sources.list.d/nexus.sources
```
```ini
# Ubuntu 24.04 Noble Main
Types: deb
URIs: https://nexus.domaim.com/repository/ubuntu-noble-proxy
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse

# Ubuntu 24.04 Noble Security
Types: deb
URIs: https://nexus.domaim.com/repository/ubuntu-noble-security-proxy
Suites: noble-security
Components: main restricted universe multiverse
```

### 3. 其他常用第三方软件源代理 (APT)

| 代理名称 (Name) | Remote storage | Distribution | 客户端源配置示例 (修改对应的 `.list` 或 `.sources`) |
| :--- | :--- | :--- | :--- |
| **LinuxMint Zara** | `http://packages.linuxmint.com` | `zara` | `deb https://nexus.../linuxmint-zara-main-proxy/ zara main upstream import backport` |
| **Proxmox VE (Trixie)** | `http://download.proxmox.com/debian/pve` | `trixie` | `URIs: https://nexus.../proxmox-pve-trixie-proxy/` (需保留官方 GPG) |
| **Wazuh Agent** | `https://packages.wazuh.com/4.x/apt` | `stable` | `deb [signed-by=...] https://nexus.../wazuh-apt-proxy/ stable main` |
| **Docker CE** | `https://download.docker.com/linux/debian` | `trixie` | `deb [arch=amd64 signed-by=...] https://nexus.../docker-debian-proxy/ trixie stable` |
| **Tailscale** | `https://pkgs.tailscale.com/stable/debian` | `trixie` | `deb [signed-by=...] https://nexus.../tailscale-trixie-proxy/ trixie main` |
| **Raspberry Pi** | `http://archive.raspberrypi.com/debian/` | `trixie` | `URIs: https://nexus.../raspberrypi-trixie-proxy/` (需保留官方 GPG) |

*(注意：Nexus 代理仅转发元数据，不重新签名，因此客户端必须保留 `signed-by` 字段以验证原始 GPG 密钥。)*

### 4. 创建内部托管仓库 (APT Hosted)
对于自行打包的 `.deb` 软件：
1. 在 Nexus 创建 `apt (hosted)`，填入自定义的 **Distribution** 名称，并粘贴您生成的 GPG 私钥 (Private Key) 用于签名。
2. 通过 API (`curl`) 或第三方工具上传 `.deb` 文件。
3. 客户端需信任您的公钥 (`apt-key add` 或放入 keyring 目录)，并将源指向该 Hosted 仓库。

---

## 伍、 代理 Hugging Face 模型仓库

借助 Nexus 缓存庞大的 AI 模型文件，极大提升内网下载速度。

### 1. 服务端配置
- **Recipe**: `huggingface (proxy)`
- **Name**: `huggingface-proxy`
- **Remote storage URL**: `https://huggingface.co`
- **HTTP request settings**: 将 Connection timeout 设置为 `120` 秒。
- **Authentication (可选)**: 如需下载需授权的模型，选择 `Preemptive Bearer Token` 并填入您的 HF Token。
- **Strict Content Type Validation**: 建议**取消勾选**，避免某些模型文件类型不匹配导致 400 错误。
*(建议将 Blob store 建立在高性能存储介质上。)*

### 2. 客户端配置
在运行 Python/Hugging Face 脚本的机器上注入环境变量：
```bash
# 替换默认端点
export HF_ENDPOINT="http://<nexus_ip>:8081/repository/huggingface-proxy"

# 如果 Nexus 配置了基础认证：
# export HF_ENDPOINT="http://<用户名>:<密码>@<nexus_ip>:8081/repository/huggingface-proxy"

# 增加下载超时时间以防首次拉取中断
export HF_HUB_DOWNLOAD_TIMEOUT=120
export HF_HUB_ETAG_TIMEOUT=1800
```

---

## 陆、 权限与用户安全设置

**核心目标**：局域网内允许匿名拉取 (Pull)；禁止匿名推送 (Push)；仅授权用户可推送。

### 1. 配置匿名拉取权限
1. ⚙️ **Settings** -> **Security** -> **Anonymous Access**，勾选 `Allow anonymous access to the server`，保存。
2. ⚙️ **Settings** -> **Security** -> **Roles**，检查默认的 `nx-anonymous` 角色。
3. **安全原则**：该角色应仅包含 `read` 和 `browse` 权限，**绝对不能**包含 `add`, `edit`, `delete` 等操作。如果你需要精细化控制，可以新建一个角色（如 `lan-pull-only`），仅赋予指定代理仓库的 read/browse 权限，然后分配给 anonymous 用户。

### 2. 为推送操作创建专用账号（强烈建议）
**请遵循最小权限原则，不要使用 `admin` 账户执行日常的 Docker push 任务。**

1. **创建角色**：⚙️ **Settings** -> **Security** -> **Roles** -> **Create role**
   - Role ID: `docker-pusher`
   - Privileges 加入：
     - `nx-repository-view-docker-docker-local-read`
     - `nx-repository-view-docker-docker-local-browse`
     - `nx-repository-view-docker-docker-local-add` (允许推送)
     - `nx-repository-view-docker-docker-local-edit` (允许覆盖标签)
2. **创建用户**：⚙️ **Settings** -> **Security** -> **Users** -> **Create local user**。
   - 填写信息，并在 Roles 选项卡中，将 `docker-pusher` 分配给该用户。

> **局域网安全提示**：Nexus 无法直接限制匿名访问的 IP 来源。若暴露在公网，请务必通过群晖防火墙或 Nginx 反向代理配置 IP 白名单，仅允许内网网段（如 `192.168.x.x/24`）访问 8081 端口。

---

## 柒、 自动清理旧缓存与释放磁盘空间

Nexus 的清理分为“逻辑删除 (Soft Delete)”和“物理删除 (Hard Delete)”。**若不配置最后一项的“压缩 (Compact)”任务，磁盘空间永远不会减少。**

### 步骤 1：创建清理策略 (Cleanup Policy)
1. ⚙️ **Settings** -> **Repository** -> **Cleanup Policies** -> **Create cleanup policy**。
2. **Format**: `docker`。
3. 勾选 **Published Before** 设置为 `180` 天。
4. （建议）开启 System -> Capabilities 中的下载统计后，勾选 **Last Downloaded Before** 设置为 `180` 天，防止误删老旧但常用的基础镜像。

### 步骤 2：将策略应用到仓库
在具体的仓库设置页面（如 `docker-hosted`），向下滚动找到 **Cleanup Policies**，将上述策略移入 **Applied policies**。

### 步骤 3：配置三个定时任务 (Scheduled Tasks)
前往 ⚙️ **Settings** -> **System** -> **Tasks**，依次创建以下三个任务，并确保执行时间有先后顺序：

1. **逻辑删除**
   - Task type: `Admin - Cleanup repositories`
   - Time: 每日 `02:00`
2. **清理孤儿层 (Docker 专属)**
   - Task type: `Docker - Delete unused manifests and images`
   - Time: 每日 `03:00`
3. **物理释放空间 (最关键的一步)**
   - Task type: `Admin - Compact blob store`
   - Time: 每日 `04:00`

---

## 捌、 反向代理配置 (以 Synology NAS 为例)

通过群晖的“登录门户” -> “高级” -> “反向代理服务器”配置 HTTPS 访问。

**关键设置：自定义标题 (Custom Headers)**
为了确保 Docker 客户端和 Nexus 之间通信正常（特别是认证和 IP 获取），必须添加以下 Header 信息：

1. 点击 **自定义标题** -> **新增** -> 选择 **WebSocket**。
2. 手动添加以下信息（点击“新增”）：

| 名称 (Header Name) | 值 (Value) | 说明 |
| :--- | :--- | :--- |
| `Host` | `nexus.domaim.com` | 填写你在“来源”中设置的主机名 |
| `X-Forwarded-Proto` | `https` | 声明外网使用的是 HTTPS |
| `X-Real-IP` | `$remote_addr` | 传递客户端真实 IP |
| `X-Forwarded-For` | `$proxy_add_x_forwarded_for` | 传递代理链路 IP |