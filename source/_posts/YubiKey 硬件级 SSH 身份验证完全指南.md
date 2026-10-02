---
title: YubiKey 硬件级 SSH 身份验证完全指南
date: 2026-10-02 13:01:39
tags: [笔记, YubiKey, SSH, 安全]
---

# YubiKey 硬件级 SSH 身份验证完全指南

通过硬件安全密钥（如 YubiKey）保护 SSH 连接，可有效抵御私钥被窃取或复制的风险。私钥始终保存在硬件内部，无法被导出或提取。

## 方案总览与对比

根据现有环境和安全需求，可选择以下不同的实现方案：

*   **FIDO2 / U2F (强烈推荐)**
    *   **优势**：配置极为简便，原生集成于现代 OpenSSH（无需额外安装中间件），支持多设备共享额度。
    *   **劣势**：部分老旧系统或自带的过时 OpenSSH 版本可能不支持，部分 VPS 提供商的控制面板不支持导入该种 SSH 公钥。
*   **PIV (智能卡)**
    *   **优势**：集中管理密钥，支持 PKCS#11 标准，非常适合具备现有 PKI (公钥基础设施) 部署的组织。
    *   **劣势**：配置相对繁琐，需要安装额外的库 (ykcs11)。
*   **OpenPGP**
    *   **优势**：在单机上易于管理，适合个人极客用户。
    *   **劣势**：密钥丢失无法恢复，缺乏企业级凭据管理支持。
*   **OTP (Yubico PAM)**
    *   **优势**：支持极为老旧的 SSH 环境。
    *   **劣势**：安全性不及非对称密钥，技术框架已近生命周期末期。

---

## 核心推荐方案：FIDO2 身份验证

FIDO2 验证可分为**不可发现凭证（非驻留 / U2F）**与**可发现凭证（驻留）**两种模式。

*   **不可发现凭证（推荐极高安全需求）**：凭据引用（Key Handle）保存在本地电脑的 `~/.ssh` 中。他人即使拿到 YubiKey 也无法在其他机器上使用。
*   **可发现凭证（推荐便携需求）**：凭据引用存储在 YubiKey 内部（共享 25 个额度）。更换电脑时，只需插入 YubiKey 即可自动恢复登录环境。

### 前置环境准备

*   **OpenSSH 版本**：不可发现凭证需 `8.2p1` 及以上；可发现凭证需 `8.3` 及以上。
    *   *注：某些操作系统的自带版本（如旧版 macOS 或 Windows）可能不支持 FIDO2。若遇到限制，可通过 Homebrew (macOS) 或官方发布页 (Windows) 更新 OpenSSH。*
*   **设备固件**：推荐使用 `ed25519-sk` 曲线，要求 YubiKey 固件版本在 5.2.3 及以上。
*   **PIN 码**：必须已通过 YubiKey Manager 或本地系统工具为 YubiKey 设置了 FIDO2 PIN。

### 服务端配置（远程目标机）

虽然 OpenSSH 默认支持 FIDO2，但为了强制校验用户身份（输入 PIN 或生物识别），建议在服务端开启强验证。

**修改 SSH 配置文件**
编辑远程服务器的 `/etc/ssh/sshd_config` 文件：
```bash
# 强制要求验证
PubkeyAuthOptions verify-required
# (可选) 彻底禁用密码登录以提升安全性
PasswordAuthentication no
ChallengeResponseAuthentication no
```
完成后重启 SSH 服务：`sudo systemctl restart sshd`。

*(进阶)* 如果不想全局限制，也可以仅针对特定公钥开启强制验证，在 `~/.ssh/authorized_keys` 的公钥条目前面加上 `verify-required` 即可。

### 客户端配置：不可发现凭证（非驻留）

**生成密钥对**
将 YubiKey 插入电脑，打开终端执行：
```bash
ssh-keygen -t ed25519-sk -C "您的注释信息" -f ~/.ssh/<自定义密钥文件名>
```
期间终端会提示输入 PIN，且 YubiKey 会闪烁，请触摸设备确认。生成后将得到私钥引用文件与 `.pub` 公钥文件。

**分发公钥**
使用工具将公钥发送至远程服务器：
```bash
ssh-copy-id -i ~/.ssh/<自定义密钥文件名>.pub -p <服务器端口> <用户名>@<服务器地址>
```

**多设备迁移**
若更换本地电脑，需将本地的私钥引用文件（`~/.ssh/<自定义密钥文件名>`）和公钥文件手动复制到新电脑的同级目录下，插入 YubiKey 即可继续使用。

### 客户端配置：可发现凭证（驻留）

**生成并驻留密钥**
将 YubiKey 插入系统，执行以下命令：
```bash
ssh-keygen -t ed25519-sk -O resident -O application=ssh:<自定义标识符> -O verify-required -C "您的注释信息" -f ~/.ssh/<自定义密钥文件名>
```
*参数解析：`-O resident` 声明写入硬件，`-O application=` 用于多密钥区分，`-O verify-required` 强制要求校验 PIN。*

按提示输入 PIN 并触摸闪烁的 YubiKey。随后进行与上述相同的“分发公钥”操作。

**多设备迁移（极简恢复）**
在新电脑上插入该 YubiKey，直接通过命令拉取凭证：
```bash
cd ~/.ssh
ssh-keygen -K
```
触摸设备后，SSH 会自动将存储在 YubiKey 中的凭证引用导出为文件（如 `id_ed25519_sk`），立即可用于身份验证。

### 客户端本地配置优化 (SSH Config)
为了避免每次连接都手动指定密钥文件，建议在本地 `~/.ssh/config` 中配置映射关系：
```ssh-config
Host <连接别名>
    HostName <服务器IP或域名>
    User <远程用户名>
    Port <远程端口号>
    IdentityFile ~/.ssh/<对应的密钥文件名>
```
此后，只需在终端输入 `ssh <连接别名>` 即可发起安全连接。

---

## 备选方案：PIV 身份验证

若需要利用 PKI 架构或使用 PIV 智能卡功能，可以通过配置 PKCS#11 模块实现。本方案主要适用于 macOS 和 Linux。

### 前置依赖环境
*   开启 PIV 应用的 YubiKey。
*   安装 `yubico-piv-tool`。
*   安装 `ykcs11` 模块。各系统默认路径通常为：
    *   **macOS**: `/usr/local/lib/libykcs11.dylib`
    *   **Linux**: `/usr/local/lib/libykcs11.so`
    *   **Windows**: `C:\Program Files\Yubico\Yubico PIV Tool\bin\libykcs11.dll`

### 方案 A：常规 PIV 登录
通过 YKCS11 模块直接导出 YubiKey 中的公钥进行常规验证。

**导出并部署公钥**
```bash
# 请根据操作系统替换库文件路径
ssh-keygen -D /usr/local/lib/libykcs11.so -e
```
将终端输出的公钥内容添加到远程服务器的 `~/.ssh/authorized_keys` 中。

**配置本地连接代理**
在本地电脑的 `~/.ssh/config` 顶部添加 PKCS#11 提供程序路径：
```ssh-config
PKCS11Provider /usr/local/lib/libykcs11.so
```
配置完成后即可进行 SSH 登录，终端将提示输入 PIV PIN。

### 方案 B：基于 PIV 的 SSH 用户证书
此方法通过本地生成的 CA 根密钥对 YubiKey 中的公钥进行签名认证，实现更高阶的证书颁发机制。

**生成 CA 证书并部署**
在本地生成证书并在本地账户中信任（或部署到远程服务器）：
```bash
ssh-keygen -N '' -C user-ca -f ~/.ssh/ca
sed 's/^/cert-authority /' ~/.ssh/ca.pub > ~/.ssh/authorized_keys
```

**在 PIV 硬件中生成密钥并自签**
向 PIV 的 `9c` 插槽写入策略为“需验证 PIN 且需触摸”的密钥：
```bash
yubico-piv-tool -a generate -s 9c -A RSA2048 --pin-policy=never --touch-policy=always -o public.pem
yubico-piv-tool -a selfsign-certificate -s 9c -S "/CN=SSH key/" -i public.pem -o cert.pem
yubico-piv-tool -a import-certificate -s 9c -i cert.pem
```

**通过 CA 签名硬件公钥**
挂载 YKCS11 库，并使用 CA 私钥签署从 PIV 导出的公钥：
```bash
# 清理并挂载 SSH Agent
ssh-add -D
ssh-add -s /usr/local/lib/libykcs11.so

# 导出硬件公钥并签名
ssh-add -L > ~/.ssh/id_rsa.pub
ssh-keygen -s ~/.ssh/ca -I identity -n "${LOGNAME}" ~/.ssh/id_rsa.pub
```
完成后将生成 `id_rsa-cert.pub`。使用此证书连接目标系统时，YubiKey 将直接闪烁等待触摸确认，无需再次输入 PIV PIN。

---

## 进阶应用：通过 ProxyJump 代理 SSH 连接

若需要通过跳板机（堡垒机）访问内部网络中的目标主机，可结合 YubiKey 的 PIV 功能实现端到端加密代理。

**配置说明**
确保本地已正确挂载 YKCS11 库（见上文 PIV 基础配置）。编辑 `~/.ssh/config`，利用 `ProxyJump` 指令构建跳板逻辑：

```ssh-config
# 全局挂载智能卡模块
PKCS11Provider /usr/local/lib/libykcs11.so

# 定义跳板机 (代理主机)
Host JumpServer
    Hostname <跳板机IP>
    User <跳板机用户名>
    Port 22

# 定义内网目标机
Host TargetServer
    Hostname <内网目标IP>
    ProxyJump JumpServer
    User <内网用户名>
    Port 22
```

配置生效后，执行 `ssh TargetServer`。系统会依次向您请求验证（要求输入 PIV PIN 或触摸设备），即可自动穿透跳板机完成内网服务器的安全登录。

---

## 故障排除与常见问题

若遇到连接失败或提示退化为密码登录，请按以下方向进行排查：

*   **版本兼容性确认**
    使用 `ssh -V` 检查本地 OpenSSH 版本。确保非驻留密钥满足 `>= 8.2p1`，驻留密钥满足 `>= 8.3`。
*   **权限问题排查**
    SSH 极其严格地要求私钥引用文件及配置目录的权限。请执行以下命令修复权限：
    ```bash
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/id_*
    chmod 600 ~/.ssh/authorized_keys # (远程服务器端)
    ```
*   **硬件读取状态**
    有时系统会提示无法读取 `id_ecdsa_sk` 或 `id_ed25519_sk`，通常是因为硬件握手失败。尝试拔下 YubiKey 并重新插入。
*   **调试模式查看日志**
    *   **客户端**：追加 `-vvvv` 参数运行 SSH（如 `ssh -vvvv <别名>`），观察输出中哪一步被拒绝。
    *   **服务端 (Linux)**：查看 SSH 守护进程日志，寻找拒绝原因。
        *   Ubuntu/Debian：`tail /var/log/syslog | grep sshd` 或 `journalctl -u ssh`
        *   CentOS/Fedora：`journalctl -r /usr/sbin/sshd`
*   **触摸或闪烁无响应**
    部分操作系统（或通过 WSL、特定终端模拟器访问时）的 USB 直通机制存在不一致，可能导致触摸提示不显示或硬件灯不闪烁。遇此情况请检查系统底层的 USB 重定向设置。