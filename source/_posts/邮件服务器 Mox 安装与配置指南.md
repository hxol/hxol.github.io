---
title: 邮件服务器 Mox 安装与配置指南
date: 2026-09-30 15:30:00
tags: [笔记, Mox, 邮件, 自托管]
---

# 邮件服务器 Mox 安装与配置指南

**参考来源**：[Mox 官方文档](https://www.xmox.nl/install/)

## 第一步：系统基础准备工作

### 1. 配置 DNS A 记录
在您的域名解析平台（如 Cloudflare）处，为邮件服务器添加一条 A 记录：
* **名称 (Name)**: `mail` (即 `mail.example.com`)
* **IPv4 地址**: `198.51.100.82`
* *(注意：请确保关闭 Cloudflare 的代理状态，即将其设置为“仅 DNS”的灰色云图标。)*

### 2. 修改主机名 (Hostname)
确保系统主机名与邮件服务器的域名一致。
```bash
sudo hostnamectl set-hostname mail.example.com
```

### 3. 修改 Hosts 文件
编辑系统的 hosts 文件：
```bash
sudo nano /etc/hosts
```
**注意**：在 `127.0.0.1` 或 `::1` 所在的行中，**不要**出现 `mail.example.com`。您需要在下方单独添加一行，将公网 IP 映射到主机名。

**修改示例：**
```text
127.0.0.1       localhost

# The following lines are desirable for IPv6 capable hosts
::1             localhost ip6-localhost ip6-loopback
ff02::1         ip6-allnodes
ff02::2         ip6-allrouters

# 邮件服务器公网 IP 与主机名映射
198.51.100.82   mail.example.com
```

### 4. 配置支持 DNSSEC 的本地解析器 Unbound
Mox 对 DNSSEC 有极高的要求。我们将禁用系统自带的 DNS 解析，改用 Unbound。

**安装 Unbound 和 DNS 调试工具：**
```bash
sudo apt update && sudo apt install unbound dnsutils -y
```

**写入 Mox 推荐的 EDE 配置文件**（这能让 DNS 报错更加清晰）：
```bash
sudo bash -c 'cat <<EOF >/etc/unbound/unbound.conf.d/ede.conf
server:
    ede: yes
    val-log-level: 2
EOF'
```

**停用系统自带的 systemd-resolved，强制使用本地 Unbound：**
```bash
# 停止并禁用自带的 resolved
sudo systemctl disable --now systemd-resolved

# 删除旧的 resolv.conf 软链接并重新创建 (使用 tee 避免 sudo 权限重定向问题)
sudo rm -f /etc/resolv.conf
echo "nameserver 127.0.0.1" | sudo tee /etc/resolv.conf > /dev/null

# 重启 Unbound 并设置开机自启
sudo systemctl restart unbound
sudo systemctl enable unbound
```

**验证 Unbound 是否生效：**
执行以下命令：
```bash
dig com. ns
```
在返回的结果中寻找 `;; flags:` 这一行。如果标志中包含了 **`ad`** (Authentic Data，意为认证数据)，例如 `;; flags: qr rd ra ad;`，则说明本地的 DNSSEC 验证已经成功生效！

### 5. 在域名提供商处开启 DNSSEC
Mox 强烈依赖 DNSSEC。以下以 Cloudflare 与 Dynadot 配合为例：

1. 在 [Cloudflare 域名设置面板] 中启用 **DNSSEC**，获取相关的密钥信息。
2. 登录您的域名注册商（如 [Dynadot]）的 DNSSEC 设置页面，点击**更改**。
3. 选择 **“◉ 设置DNSSEC记录”**（不要选择“使用现有记录”）。
4. 对应填写以下四个参数（以 Cloudflare 提供的值为准）：
   * **密钥标签 (Key Tag):** 填写 Cloudflare 提供的 `密钥标记` (例如：**`12345`**)
   * **摘要类型 (Digest Type):** 选择包含 **`2`** 或 **`SHA-256`** 的选项（通常显示为 `2 - SHA-256`）
   * **摘要 (Digest):** 复制 Cloudflare 提供的 `摘要`，粘贴到此处（一长串类似 **`A1B2C3D4...`** 的字符）
   * **算法 (Algorithm):** 选择包含 **`13`** 或 **`ECDSA Curve P-256 with SHA-256`** 的选项
5. 核对无误后，点击 **“保存DNSSEC记录”**。*(注意：DNSSEC 生效可能需要几十分钟到几个小时不等)*。

### 6. 创建 Mox 专用用户与目录
出于安全考虑，建议为 Mox 创建一个独立的系统用户，并禁止其作为普通用户登录。
```bash
sudo mkdir -p /opt/mox
# 创建 mox 用户，指定主目录，并禁止 shell 登录
sudo useradd -r -m -d /opt/mox -s /usr/sbin/nologin mox
```

---

## 第二步：二进制程序安装

*建议：以下命令中的下载链接版本（如 `v0.0.17`）可能会随官方更新而失效。建议去 [Mox 官方下载页](https://beta.gobuilds.org/github.com/mjl-/mox/) 复制最新版本的 Linux-amd64 链接。*

下载二进制文件并赋予执行权限：
```bash
cd /tmp
# 请替换为最新的官方下载链接
wget https://beta.gobuilds.org/github.com/mjl-/mox@v0.0.17/linux-amd64-go1.27.1/0BcooIyz1Z91f8-dFhIsp6fMzqxY/mox-v0.0.17-go1.27.1 -O mox

# 移动到安装目录并赋予权限
sudo cp /tmp/mox /opt/mox/
cd /opt/mox
sudo chmod +x mox

# 将目录所有权交给 mox 用户
sudo chown -R mox:mox /opt/mox
```

---

## 第三步：根据场景初始化与运行

### 场景 1：本地开发与测试 (Local Testing)
如果您仅用于测试收发或开发，不需要真实的公网环境，可以使用内置的 `localserve` 命令。
```bash
cd /opt/mox
sudo -u mox ./mox localserve -dir /opt/mox/localserve
```
**本地测试模式特性:**
* 监听常规端口 **+1000** 的端口（例如：SMTP 1025, IMAP 1143 等）。
* 自动生成自签名 TLS 证书。
* 自动创建一个测试账号：邮箱 `mox@localhost`，密码 `moxmoxmox`。
* 接收所有发往本地的邮件，发送邮件时忽略真实的远端服务器，而是**自我投递**。
* **特殊调试地址**：向前缀为 `temperror@...` 发信会模拟临时错误，向 `permerror@...` 模拟永久错误，向 `timeout@...` 发信模拟无响应。

---

### 场景 2：正式生产环境使用 (Production) 推荐
生产环境中，Mox 极度依赖系统主机名（`mail.example.com`）和公网 IP，并会自动通过 Let's Encrypt 申请 443(HTTPS) 和 80(HTTP) 端口的 TLS 证书。

**1. 执行快速配置 (Quickstart)**
假设您的主理人邮箱是 `admin@example.com`，运行以下命令：
```bash
cd /opt/mox
sudo ./mox quickstart admin@example.com mox
```
*运行该命令后，Mox 会：*
* 生成 `mox.conf` 和 `domains.conf` 配置文件。
* 生成管理员账号和初始密码。
* **打印您需要去域名注册商/Cloudflare 处添加的所有 DNS 记录**（TXT, MX 等）。
* 打印将 Mox 安装为 Systemd 守护进程的命令。
*(提示：所有的输出也会保存在 `/opt/mox/quickstart.log` 中备查，请务必妥善保存该日志中的密码信息。)*

**2. 配置为系统服务并启动**
根据 quickstart 的提示，它会生成一个 `mox.service` 文件（您也可以随时用 `./mox config printservice > mox.service` 重新生成）。

将服务文件部署到系统中并启动：
```bash
cd /opt/mox
sudo cp mox.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now mox
```
*启动后，您即可通过浏览器访问服务器的 IP 或域名来进入 Web 管理界面。*

---

### 场景 3：已有 Web 服务器（如 Nginx/Caddy）共存
如果您的机器上 80/443 端口已被占用：
```bash
cd /opt/mox
sudo ./mox quickstart -existing-webserver admin@example.com mox
```
*⚠️ **强烈警告**：这种场景需要大量的额外配置。您需要自行提供 TLS 证书，并在现有 Web 服务器中配置反向代理以处理 MTA-STS 和 autoconfig 的流量。官方强烈建议让 Mox 独占 80/443 端口；如果您需要托管网站，Mox 内置了非常优秀的反向代理和静态 Web 服务功能。*

---

## 第四步：常用管理命令速查

安装完成后，您可以使用 mox 提供的丰富子命令来管理邮件服务器（注意：需在 `/opt/mox` 目录下使用 `sudo` 运行，以保证权限充足）：

* **服务启停：**
  * 前台临时启动测试：`sudo ./mox serve`
  * 平滑停止服务：`sudo ./mox stop` （最多给予当前连接 3 秒钟的断开时间）
  * 查看运行状态：`systemctl status mox`

* **账号与密码管理：**
  * 修改管理员 Web 密码：`sudo ./mox setadminpassword`
  * 修改普通用户邮箱密码：`sudo ./mox setaccountpassword 账户名`

* **DNS 与验证 (重要)：**
  * 检查域名 DNS 记录是否正确：`sudo ./mox config dnscheck example.com`
  * 查看需要配置的完整 DNS 区域记录文件：`sudo ./mox config dnsrecords example.com`

* **备份与恢复：**
  * 备份数据（支持硬链接，节省存储空间）：`sudo ./mox backup /opt/mox/backup_dir`
  * 验证备份数据的完整性：`sudo ./mox verifydata /opt/mox/backup_dir`
