---
title: Stalwart 邮件服务器部署与维护手册
date: 2026-10-02 22:59:09
tags: [笔记, Stalwart, 邮件服务器, 自托管]
---




# Stalwart 邮件服务器部署与维护手册

## 📚 参考资源

*   **项目主页**: [Stalwart GitHub](https://github.com/stalwartlabs/stalwart)
*   **官方文档**: [安装指南](https://stalw.art/docs/install/)
*   **安全建议**: [保护您的服务器](https://stalw.art/docs/install/security/)
*   **访问控制**: [存取控制 (Access Control)](https://stalw.art/docs/http/access-control/)

---

## 🛠️ 前期网络准备

### 申请设置 rDNS (PTR 记录)
rDNS 是提升邮件送达率（避免被拒收或进垃圾箱）的关键步骤。需向 VPS 提供商提交工单申请。

**工单模板（英文）：**
> **Subject:** Request to Setup rDNS (PTR Record) for IP `<YOUR_VPS_IP>`
>
> **Body:**
> Dear Support Team,
> 
> I would like to request a Reverse DNS (PTR record) update for my server. Please set the rDNS as follows:
> 
> *   **IP Address:** `<YOUR_VPS_IP>`
> *   **PTR / Hostname:** `mail.example.com`
> 
> I am setting up a mail server strictly for personal use and administrative notifications. I guarantee that this server will not be used for bulk mailing or spamming activities.
> 
> All technical prerequisites, including SPF, DKIM, and DMARC records, have already been correctly configured. The rDNS record is the final step required to complete my setup.
> 
> Thank you for your assistance.
> 
> Best regards,
> `<YOUR_NAME>`

**验证 rDNS 是否生效：**
可以通过以下任一命令测试，若输出结果中包含 `mail.example.com` 即代表解析成功。
```bash
nslookup <YOUR_VPS_IP> 8.8.8.8
# 或
dig -x <YOUR_VPS_IP>
```

### 配置基础 DNS 记录
在您的 DNS 服务商（如 Cloudflare）处，为域名添加以下基础指向：
*   **类型**: `A 记录`
*   **名称**: `mail` (即 `mail.example.com`)
*   **IPv4 地址**: `<YOUR_VPS_IP>`
*   *(注意：请确保关闭 Cloudflare 的小黄云代理，仅作为 DNS 解析)*

---

## 🐳 环境部署

### 配置系统防火墙 (UFW)
Stalwart 自身接管 Web 与邮件协议，需要开放相关端口以确保正常通信及后续的证书申请。

```bash
# 开放 Web 及 ACME 证书申请端口
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# 开放邮件协议收发端口
sudo ufw allow 25/tcp    # SMTP (服务器间通信)
sudo ufw allow 465/tcp   # SMTPS (客户端发送)
sudo ufw allow 993/tcp   # IMAPS (客户端接收)

# 重载并检查状态
sudo ufw reload
sudo ufw status
```
*补充提示：除了系统防火墙，请务必检查您的云服务商（如 AWS, Oracle, 阿里云）控制台的安全组规则，特别是 **25 端口**是否已被服务商默认封禁，如有需要需单独向服务商申请解封。*

### 安装 Stalwart 容器

创建工作目录及编排文件：
```bash
sudo mkdir -p /opt/stalwart && cd /opt/stalwart
sudo nano docker-compose.yml
```

写入 `docker-compose.yml` 配置：
```yaml
services:
  stalwart:
    image: stalwartlabs/stalwart:v0.16.6
    container_name: stalwart
    restart: unless-stopped
    network_mode: "host" # 使用 host 模式，容器直接绑定宿主机端口
    volumes:
      # （可选）若需挂载外部 Let's Encrypt 证书可保留此行，若使用 Stalwart 内置 ACME 申请则非必须
      - /etc/letsencrypt:/etc/letsencrypt:ro
      # 配置文件持久化目录
      - stalwart-etc:/etc/stalwart
      # 应用程序数据持久化目录
      - stalwart-data:/var/lib/stalwart
    tty: true
    stdin_open: true

volumes:
  stalwart-etc:
  stalwart-data:
```

启动服务：
```bash
cd /opt/stalwart && sudo docker compose up -d
```

---

## ⚙️ 系统初始化配置

### 获取初始管理员凭据
服务首次启动后，系统会随机生成一个管理员账号和密码。
```bash
cd /opt/stalwart && sudo docker logs stalwart 2>&1 | grep -A8 'bootstrap mode'
```

### 通过 Web 向导完成初始设置
由于此时暂未配置好 TLS 证书，为了安全访问，建议先使用 Cloudflare Tunnel（或其他内网穿透/SSH 端口转发方式）代理宿主机的 `127.0.0.1:8080` 端口。

访问临时向导地址：`https://<YOUR_PROXY_DOMAIN>/admin`

**向导配置参数建议：**

*   **第一页：Server identity**
    *   Server Hostname: `mail.example.com`
    *   Default Email Domain: `example.com`
    *   Automatically Obtain TLS Certificate: 勾选 (启用)
    *   Generate Email Signing Keys: 勾选 (启用)
*   **第二页：Storage** -> 保持默认
*   **第三页：Account Directory** -> 保持默认
*   **第四页：Logging**
    *   Log destination: **Console** *(Docker 部署环境必须修改为此项)*
    *   其余选项保持默认
*   **第五页：Automatic DNS management**
    *   Automatic DNS Management: 选择 `Cloudflare`
    *   API Token: 选择 `Secret Value`
    *   Secret: 填入具备 DNS 编辑权限的 Cloudflare API Token

*重要提示：向导完成后，页面会重新打印一个最终版的管理员账号和密码，这是永久管理员凭据，请务必妥善保存。*

### 切换至正式管理界面
向导配置完成并保存后，Stalwart 会自动去申请 TLS 证书。此时我们需要重启容器让新配置生效：
```bash
cd /opt/stalwart && sudo docker compose restart
```
等待片刻，证书申请成功后，即可**关闭/删除**之前的 Cloudflare Tunnel 代理。
直接通过 443 端口访问正式管理后台：**`https://mail.example.com/admin`**

---

## 👥 域与账号管理

### 创建邮件账号
路径：`Management` -> `Directory` -> `Accounts` -> `Create Account`

*   **Details 选项卡**:
    *   Login name: `username`
    *   Name: `自定义显示名称`
    *   Email: `username@example.com`
    *   Aliases (别名): 若希望该账号作为 Catch-all (接收该域名下所有未匹配前缀的邮件)，可在此处添加一个 `@example.com`。
*   **Authentication 选项卡**: 设定该账号的登录密码。
*   **Permissions 选项卡**: 若需要赋予此账号管理后台的权限，点击 `+ Assign roles` 并选择管理员角色。

### 添加多域名支持 (扩展域名)
如需使用同一个服务器托管其他域名的邮件（例如 `second-domain.com`）：
1. 登录管理后台。
2. 导航至 `Management` -> `Directory` -> `Domains`。
3. 点击右上角 `Create domain`。
4. 填写 **Domain name** (`second-domain.com`) 和 **Description** (如：Secondary Domain)。
5. 点击保存。系统及内部自动管理的 DNS (如果联动了 Cloudflare) 会协助处理部分解析。

### 开启 MTA-STS 策略
MTA-STS 强制要求邮件服务器之间的通信使用 TLS 加密，可提升安全性。
路径：`Settings` -> `SMTP` -> `Inbound` -> `MTA-STS` -> `MTA-STS Policy` 页面

*   **Policy Application**: 设置为 `enforce` (强制执行)
*   **MX Patterns (override)**: 点击 Add，添加您的所有邮件域名
    *   `*.example.com`
    *   `*.second-domain.com`

*检测配置是否生效：访问 `https://mta-sts.example.com/.well-known/mta-sts.txt` 查看策略文件。*

---

## 💻 客户端接入与网络测试

### 端口与连通性测试
在您的**本地电脑**（外部网络环境）打开终端，测试 SMTP 25 端口是否被双向打通：

```bash
# 测试端口是否开放
nc -vz mail.example.com 25

# 手动验证 SMTP 协议响应
nc mail.example.com 25
```
正常情况下会输出类似：`220 stalw.art ESMTP Service Ready`。
此时输入 `EHLO gmail.com` 并回车，应返回一系列 `250-` 开头的功能列表（如 STARTTLS）。
*(排错：如果卡住无响应，通常是因为 Docker 容器内部进程未正常响应，或者 VPS 服务商在云端防火墙隐式阻断了 25 端口出入站流量。)*

### 邮件客户端配置 (以 Thunderbird 为例)
*   **收件服务器 (IMAP)**: `mail.example.com` | 端口: `993` | 加密: `SSL/TLS`
*   **发件服务器 (SMTP)**: `mail.example.com` | 端口: `465` | 加密: `SSL/TLS`
*   **用户名**: `username@example.com`
*   **密码**: 设置的账号密码（或专门生成的 App 密码）

---

## 🛡️ 安全与日常维护

### IP 黑白名单管理 (防误封)
如果您在配置客户端时多次输错密码导致本地 IP 被 Fail2ban 机制封禁：
*   前往管理后台：检查 `Blocked IP addresses` 列表，找到并移除自己的 IP。
*   为防止未来再次被封禁，可将自己的常用 IP 或内网 IP 频段加入 `Allowed IPs` 列表。

### 管理员安全加固
*   及时修改系统最初生成的默认 `admin` 账号的密码。
*   或创建一个拥有管理员权限的新账号，测试无误后，停用或删除默认的 `admin` 账号以防爆破。

### 拒收高危附件策略 (Sieve 脚本)
通过配置 Sieve 脚本，可以在 SMTP 的 DATA 阶段直接拒收包含 `.exe`, `.bat`, `.vbs` 等可执行文件的危险邮件。

> **⚠️ 版本更新注意 (To-Do)**：
> 自 Stalwart 版本更新后，Sieve 脚本的作用域和配置路径发生了变动。以下脚本逻辑供参考，实际部署前需查阅当前版本的文档，确认最新的脚本挂载位置。

**预期操作流程：**
1. 路径（可能发生变动）：`Settings` -> `Scripting` -> `System Scripts` -> `Create script`
2. **Script Id**: `filter_virus`
3. **Contents** 写入以下规则：
```sieve
require ["mime", "foreverypart", "reject"];

# 遍历邮件的每一个 MIME 部分 (即每一个附件或正文段落)
foreverypart {
    # 检查 Content-Type 和 Content-Disposition 中的文件名
    if anyof (
        header :mime :param "filename" "Content-Disposition" :matches ["*.exe", "*.com", "*.bat", "*.cmd", "*.msi", "*.scr", "*.pif", "*.vbs", "*.js", "*.iso", "*.jar", "*.cpl"],
        header :mime :param "name" "Content-Type" :matches ["*.exe", "*.com", "*.bat", "*.cmd", "*.msi", "*.scr", "*.pif", "*.vbs", "*.js", "*.iso", "*.jar", "*.cpl"]
    ) {
        # 若匹配高危后缀，直接拒收并断开连接，返回 550 状态码
        reject "550 5.7.1 Security Policy: Executable attachments are not allowed.";
        stop;
    }
}
```
4. 保存后，前往 `Settings` -> `SMTP` -> `Inbound` -> `DATA Stage`，将 **Spam filtering** 设置为 `true`，并在 **Run Script** 中填入 `'filter_virus'` 以应用该策略。