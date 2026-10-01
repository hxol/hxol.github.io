---
title: 群晖 DSM 完美实现 Let's Encrypt 泛域名证书自动申请与续期
date: 2026-09-30 15:40:00
tags: [笔记, 群晖, 自托管]
---

# 群晖 DSM 完美实现 Let's Encrypt 泛域名证书自动申请与续期

**核心逻辑：** 利用 Docker 里的 Certbot 申请证书 ➡️ 手动导入 DSM 建立“户口”（生成系统证书 ID） ➡️ 利用脚本实现后续的定期自动“偷梁换柱”与服务重载。

**适用环境：** 群晖 DSM 7.x 系统（DSM 6.x 部分服务重载命令可能不同）、已安装 Container Manager (Docker)。

---

### 第一阶段：准备工作与环境搭建

我们需要先在群晖中创建存放配置文件、证书和脚本的目录结构。请确保你的真实存储空间路径（如 `/volume1`）与下文一致，如果不一致请自行替换。

#### 1. 创建文件夹
通过 SSH 登录群晖（使用 root 权限），或者使用 File Station 创建以下目录结构：

*   `/volume1/docker/ssl` (存放 Certbot 生成的证书数据)
*   `/volume1/docker/ssl-renewal` (存放 `docker-compose.yml` 和 `renew.sh`)
*   `/volume1/docker/cloudflare` (存放 Cloudflare API 凭证)

**命令行一键创建：**
```bash
mkdir -p /volume1/docker/{ssl,ssl-renewal,cloudflare}
```

#### 2. 获取 Cloudflare API Token
1.  登录 [Cloudflare 官网](https://dash.cloudflare.com/)。
2.  点击右上角头像 ➡️ **My Profile (我的个人资料)** ➡️ **API Tokens**。
3.  点击 **Create Token (创建令牌)**。
4.  在模板列表中，找到并选择 **Edit zone DNS (编辑区域 DNS)**。
5.  在 **Zone Resources (区域资源)** 中选择：`Include` ➡️ `Specific zone` ➡️ `example.com`（选择你的域名）。
6.  点击继续并生成，**复制并妥善保存这个 Token**（它只显示一次）。

#### 3. 创建 Cloudflare 凭证文件
在 `/volume1/docker/cloudflare/` 目录下创建一个名为 `cloudflare.ini` 的凭证文件。

**创建并编辑文件：**
```bash
vi /volume1/docker/cloudflare/cloudflare.ini
```

**填入以下内容：**
```ini
# 将等号后面的内容替换为你刚刚复制的 Token
dns_cloudflare_api_token = 你的Cloudflare_API_Token_粘贴在这里
```

**设置安全权限（非常重要，否则 Certbot 会报错并拒绝运行）：**
```bash
chmod 600 /volume1/docker/cloudflare/cloudflare.ini
```

---

### 第二阶段：配置 Docker 环境

配置 `docker-compose.yml` 文件，它不仅用于后续的自动续期，也用于首次申请证书。

**文件路径：** `/volume1/docker/ssl-renewal/docker-compose.yml`

**文件内容：**
```yaml
services:
  certbot:
    image: certbot/dns-cloudflare:latest
    container_name: certbot_renewal
    network_mode: "host" 
    volumes:
      # 映射证书存储目录
      - /volume1/docker/ssl:/etc/letsencrypt
      # 映射 Cloudflare 凭据（只读）
      - /volume1/docker/cloudflare/cloudflare.ini:/etc/cloudflare/cloudflare.ini:ro
    command: ["certificates"] # 默认命令，实际运行时会被脚本覆盖

  # “工具人”容器：用于在不污染群晖宿主机环境的情况下，临时安装并使用 jq 解析 JSON 文件
  jq-helper:
    image: alpine:latest
    container_name: jq_helper
    network_mode: "host"
    volumes:
      # 只读挂载群晖的证书配置文件
      - /usr/syno/etc/certificate/_archive/INFO:/tmp/INFO:ro
    command: ["sh", "-c", "echo 'Waiting for command...'"]
```

---

### 第三阶段：首次申请证书

自动化脚本只能用于“续期”已经存在的证书。如果你是从零开始，必须先**手动运行一次申请命令**来生成证书。

1.  进入项目目录：
    ```bash
    cd /volume1/docker/ssl-renewal
    ```

2.  运行申请命令（此处以申请泛域名 `*.example.com` 和主域名 `example.com` 为例）：
    ```bash
    docker compose run --rm certbot certonly \
      --dns-cloudflare \
      --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini \
      --dns-cloudflare-propagation-seconds 60 \
      -d example.com \
      -d *.example.com
    ```

3.  **交互与验证**：
    *   首次运行会要求输入邮箱（用于接收证书即将过期等紧急通知）。
    *   同意服务条款 (输入 `Y` 并回车)。
    *   是否分享邮箱 (输入 `N` 即可)。
    *   等待约一分钟（DNS 验证中），若看到 `Successfully received certificate` 即代表成功。
    *   证书文件已生成并保存在 `/volume1/docker/ssl/live/example.com/` 目录下。

---

### 第四阶段：首次导入 DSM (建立“户口”)

为了让群晖识别并使用这个证书，我们需要手动导入一次。这一步会在群晖系统内部生成一个**随机的证书 ID**（例如 `xYzBa1`），后续的脚本就是依据这个 ID 精准定位并进行“偷梁换柱”的。

1.  先使用 SFTP 软件或群晖的 File Station，将 `/volume1/docker/ssl/live/example.com/` 目录下的证书文件下载到你的电脑本地。
2.  打开 DSM **控制面板** ➡️ **安全性** ➡️ **证书**。
3.  点击 **新增** ➡️ **添加新证书** ➡️ **导入证书**。
4.  分别上传刚刚下载的文件：
    *   **私钥 (Private Key)**: `privkey.pem`
    *   **证书 (Certificate)**: `cert.pem`
    *   **中间证书 (Intermediate Certificate)**: `chain.pem`
5.  点击 **确定** 完成导入。
6.  **最关键的一步**：导入后，在证书列表中选中新导入的 `example.com` 证书，点击 **设置**。
7.  将所有需要使用该证书的服务（特别是 **系统默认 (System Default)**）的证书都勾选为 `example.com`。
    > **原理说明：** 我们的自动化脚本是通过查找标记为 `default` 的证书 ID 来定位替换目标的，这一步绝对不能漏掉。

---

### 第五阶段：配置自动续期脚本

现在配置自动化脚本，让它接管后续的所有工作。Let's Encrypt 证书有效期为 90 天，该脚本会自动判断是否需要续期。

#### 1. 保存脚本文件
在 `/volume1/docker/ssl-renewal/` 目录下创建 `renew.sh` 文件。

**文件内容（已对变量和权限进行完善补充）：**
```bash
#!/bin/bash
set -e

# --- 配置区 (请根据实际情况修改) ---
LOG_FILE="/volume1/docker/ssl-renewal/renewal.log"
PROJECT_DIR="/volume1/docker/ssl-renewal"
CERT_LIVE_DIR="/volume1/docker/ssl"
DOMAIN="example.com"  # 替换为你的真实域名

# --- 脚本主体 ---
exec >> "${LOG_FILE}" 2>&1
echo "=========================================="
echo "--- 证书续期任务开始于: $(date) ---"

if [ ! -d "$PROJECT_DIR" ]; then
    echo "错误: Docker Compose 项目目录 '${PROJECT_DIR}' 不存在。"
    exit 1
fi

cd "$PROJECT_DIR"

# 1. 调用 Docker Certbot 进行续期检查
# Certbot 默认只在证书有效期少于 30 天时才会真正向服务器发起续期请求
echo "开始通过 Certbot 检查并续期证书..."
docker compose run --rm certbot renew \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 120

# 2. 定位 DSM 默认证书 ID
echo "正在通过 'jq-helper' 服务定位群晖默认证书 ID..."
# 巧妙之处：利用 alpine 容器临时安装 jq 读取宿主机的 INFO 文件，避免污染群晖系统
DEFAULT_CERT_ID=$(docker compose run --rm jq-helper sh -c "apk add --no-cache jq >&2 && jq -r 'to_entries[] | select(.value.services[]?.service == \"default\") | .key' /tmp/INFO")

if [ -z "$DEFAULT_CERT_ID" ]; then
    echo "错误：无法定位到默认证书 ID。请检查 DSM 控制面板中是否已将该证书设为系统默认。" >&2
    exit 1
fi
echo "成功定位到 DSM 默认证书 ID: ${DEFAULT_CERT_ID}"

ARCHIVE_DIR="/usr/syno/etc/certificate/_archive/${DEFAULT_CERT_ID}"
SOURCE_CERT_DIR="${CERT_LIVE_DIR}/live/${DOMAIN}"

# 3. 复制证书 (偷梁换柱)
if [ ! -d "$SOURCE_CERT_DIR" ]; then
    echo "错误: 找不到新证书源目录 '${SOURCE_CERT_DIR}'。可能是首次申请失败或路径配置错误。" >&2
    exit 1
fi

echo "正在将新证书覆盖到群晖系统存档目录..."
# 群晖内部所需的 fullchain.pem 对应 Let's Encrypt 的 fullchain.pem
cp -f -v "${SOURCE_CERT_DIR}/fullchain.pem" "${ARCHIVE_DIR}/fullchain.pem"
cp -f -v "${SOURCE_CERT_DIR}/privkey.pem"   "${ARCHIVE_DIR}/privkey.pem"
cp -f -v "${SOURCE_CERT_DIR}/chain.pem"     "${ARCHIVE_DIR}/chain.pem"
cp -f -v "${SOURCE_CERT_DIR}/cert.pem"      "${ARCHIVE_DIR}/cert.pem"

# 修正文件权限，确保群晖系统的安全性要求
chmod 400 ${ARCHIVE_DIR}/*.pem

# 4. 重载服务 (适配 DSM 7.x)
echo "正在通知 DSM 重载证书相关服务..."
synow3tool --gen-all
systemctl reload nginx

echo "脚本执行成功！"
echo "--- 任务结束于: $(date) ---"
echo "=========================================="
```

#### 2. 赋予执行权限
执行以下命令，让脚本拥有可运行的权限：
```bash
chmod +x /volume1/docker/ssl-renewal/renew.sh
```

#### 3. 设置 DSM 任务计划 (定时任务)
1.  打开 DSM **控制面板** ➡️ **任务计划**。
2.  点击 **新增** ➡️ **计划的任务** ➡️ **用户定义的脚本**。
3.  **常规** 设置：
    *   **任务名称**：`SSL Certificate Renewal`
    *   **用户账号**：选择 **`root`**（⚠️ 必须是 root，否则没有权限覆盖系统证书目录和重载 Nginx）。
4.  **计划** 设置：
    *   建议设置为 **每月运行一次**（例如每月的 1 号凌晨 2 点）。*注：Certbot 会自动判断，未临近过期时即便运行脚本也只是一次健康检查，并不会消耗 Let's Encrypt 的 API 额度。*
5.  **任务设置** ➡️ **用户定义的脚本** 中填入：
    ```bash
    bash /volume1/docker/ssl-renewal/renew.sh
    ```
6.  点击 **确定** 保存。你可以右键该任务点击“运行”来立即测试一次。

---

### 总结：为什么这个方案近乎完美？

1. **环境极度洁癖（零污染）：** 群晖系统底层精简，直接在宿主机安装软件经常面临依赖缺失或升级失效的痛点。本方案将续期核心（Certbot）和 JSON 解析核心（jq）全部容器化，用完即焚 (`--rm`)，完全不碰群晖底层依赖。
2. **支持通配符（泛域名）证书：** 传统的群晖内置 Let's Encrypt 申请方式不支持 DNS 质询，只能申请单域名证书，且要求暴露 80 端口。本方案利用 Cloudflare DNS 验证，无需开放 NAS 的任何外部端口，安全且轻松拿下 `*.example.com` 泛域名证书。
3. **闭环的自动化体验：** “续期判断 ➡️ 获取最新证书 ➡️ 定位群晖证书库 ➡️ 强制替换 ➡️ 重启 Web 服务”这五个动作被一个脚本完全串联，设置一次任务计划后，证书的更新将完全“无感”进行。
