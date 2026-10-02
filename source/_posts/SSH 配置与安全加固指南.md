---
title: SSH 配置与安全加固指南
date: 2026-10-01 15:30:36
tags: [笔记, SSH, Linux, 安全]
---

> ⚠️ **注意**：请不要使用 `root` 权限在本地生成个人密钥对，否则普通用户将由于权限问题无法读取和使用该密钥对。
{.is-warning}

# 一、本地密钥对的管理与配置

## 1. 生成密钥对
**用法：**
```bash
ssh-keygen [参数]
```

**常用参数解析：**
*   `-t`：指定要创建的密钥类型。推荐使用 `ed25519`（目前最安全、加解密速度最快的类型，且生成的密钥长度较短）。`rsa` 兼容性最好（推荐长度 $\ge$ 3072）。`dsa` 和 `ecdsa` 因安全或技术原因已不推荐使用。
*   `-b`：指定密钥长度。对于 `ed25519`，此标志会被忽略（固定长度）。对于 `rsa`，建议设为 `3072` 或 `4096`。
*   `-C`：添加注释（通常填入邮箱、设备名或日期，方便识别）。
*   `-f`：指定用来保存密钥的路径和文件名。
*   `-p`：更改现有私钥的密码密语（Passphrase）。
*   `-q`：静默模式。

**推荐生成方式（Ed25519）：**
```bash
ssh-keygen -t ed25519 -C "$(whoami)@$(hostname)-$(date -I)" -f ~/.ssh/my_vps_key
```
*生成过程中会提示输入密码密语（Passphrase），若不想设置直接回车即可（为了极致安全建议设置）。若需给已有的无密码密钥添加密码，可使用 `ssh-keygen -p -f ~/.ssh/私钥文件名`。*

**使用硬件密钥（如 YubiKey）生成：**
```bash
# 生成非常驻密钥
ssh-keygen -t ed25519-sk -C "硬件密钥-非常驻" -f ~/.ssh/my_hw_key

# 生成常驻密钥（密钥存储在硬件中）
ssh-keygen -t ed25519-sk -O resident -O application=ssh:MyCredentialID -C "硬件密钥-常驻" -f ~/.ssh/my_hw_key
```

## 2. 妥善设置本地密钥权限
为了防止私钥被恶意读取，必须严格限制 `~/.ssh` 目录及文件的权限：
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 400 ~/.ssh/my_vps_key      # 私钥文件，仅拥有者可读
chmod 444 ~/.ssh/my_vps_key.pub  # 公钥文件，所有人可读
```

## 3. 使用 `ssh-add` 管理密钥缓存
`ssh-add` 可以将私钥添加到 `ssh-agent` 的内存中，这样如果你为私钥设置了密码，只需在添加时输入一次，后续登录便无需重复输入。
```bash
# 启动 ssh-agent（如果未启动）
eval "$(ssh-agent -s)"

# 将私钥添加到缓存中
ssh-add ~/.ssh/my_vps_key

# 查看已缓存的密钥列表
ssh-add -l

# 从缓存中删除指定密钥
ssh-add -d ~/.ssh/my_vps_key

# 清空缓存中的所有密钥
ssh-add -D
```

## 4. 创建并编辑本地 `config` 文件
通过配置 `~/.ssh/config`，可以为不同服务器设置别名，简化登录命令。
```bash
nano ~/.ssh/config
```
**写入以下内容：**
```ssh-config
# 服务器 A 别名
Host server-a
    HostName <服务器A的IP或域名>
    User <登录用户名>
    Port <服务器端口号>
    IdentityFile ~/.ssh/my_vps_key

# 服务器 B 别名
Host server-b
    HostName <服务器B的IP或域名>
    User <登录用户名>
    Port <服务器端口号>
    IdentityFile ~/.ssh/another_key
```
配置完成后，只需输入 `ssh server-a` 即可直接登录。

---

# 二、服务器端基础配置

## 1. 将公钥上传到远程服务器
在**本地终端**执行以下命令，将公钥安全地复制到服务器的 `~/.ssh/authorized_keys` 中：
```bash
ssh-copy-id -i ~/.ssh/my_vps_key.pub -p <服务器端口号> <用户名>@<服务器IP>
```
*(如果服务器上已有旧密钥想强制替换，可以添加 `-f` 参数。)*

## 2. 检查/修复服务器端权限
登录远程服务器，确保 SSH 目录权限正确（如果通过 `ssh-copy-id` 上传，通常会自动设置好）：
```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

## 3. 启用密钥登录并禁用密码登录 (Debian)
在服务器上编辑 SSH 配置文件：
```bash
sudo nano /etc/ssh/sshd_config
```
确保以下配置正确：
```ssh-config
# 开启公钥登录
PubkeyAuthentication yes

# 禁用密码登录（确保你已经测试过可以通过密钥成功登录！）
PasswordAuthentication no
```
保存后，重启 SSH 服务生效：
```bash
sudo systemctl restart ssh
```

---

# 三、SSH 服务端安全加固 (Debian 适用)

编辑 `/etc/ssh/sshd_config` 文件进行以下加固配置：

## 1. 禁止 Root 登录与空白密码
```ssh-config
PermitRootLogin no
PermitEmptyPasswords no
```

## 2. 更改默认端口
将默认的 22 端口改为高位端口（如 1024 - 65535 之间）：
```ssh-config
Port <自定义端口号，例如 27005>
```
> **注意**：修改端口前，请务必在服务器防火墙（如 UFW 或 iptables）以及云服务商的安全组中放行该新端口！
> `sudo ufw allow 27005/tcp`

## 3. 限制允许登录的用户
通过白名单严格限制哪些用户可以 SSH 登录：
```ssh-config
# 仅允许 user1 和 user2 登录
AllowUsers user1 user2

# 或者通过用户组限制
# AllowGroups admins
```

## 4. 仅监听 IPv4 或 IPv6
如果你不需要 IPv6 登录，可以强制 SSH 仅监听 IPv4：
```ssh-config
AddressFamily inet   # 仅 IPv4
# AddressFamily inet6  # 仅 IPv6
```

## 5. 设置超时自动断开
防止 SSH 会话长时间挂起被恶意利用（300秒 = 5分钟无操作自动断开）：
```ssh-config
ClientAliveInterval 300
ClientAliveCountMax 0
```

## 6. 删除短 Diffie-Hellman 密钥
根据 OpenSSH 安全指南，使用的 Diffie-Hellman 模数应至少为 3072 位长。
```bash
# 1. 备份原 moduli 文件
sudo cp -a /etc/ssh/moduli /etc/ssh/moduli-bak-$(date +"%Y%m%d%H%M%S")

# 2. 过滤掉长度小于 3071 位的密钥并替换
sudo awk '$5 >= 3071' /etc/ssh/moduli | sudo tee /etc/ssh/moduli.tmp
sudo mv /etc/ssh/moduli.tmp /etc/ssh/moduli
```

完成上述所有配置后，**重启 SSH 服务**：
```bash
sudo systemctl restart ssh
```

---

# 四、进阶安全防护

## 1. 使用 Fail2Ban 防范暴力破解
Fail2Ban 可以监控日志，当某个 IP 尝试登录失败次数过多时，自动通过防火墙（UFW/iptables）将其封禁。

**安装与启动：**
```bash
sudo apt update && sudo apt install fail2ban python3-systemd ufw -y
sudo systemctl enable --now fail2ban
```

**创建 SSH 专属配置：**
```bash
sudo nano /etc/fail2ban/jail.local
```
写入以下内容（请根据实际情况修改端口）：
```ini
[DEFAULT]
# 使用 UFW 进行封禁
banaction = ufw

[sshd]
enabled = true
# 你的 SSH 自定义端口
port = <自定义端口号，例如 27005>
# 指定后端为 systemd, 直接读取 journalctl
backend = systemd
# 放宽 journal 的匹配条件, 只认服务名不管进程名
journalmatch = _SYSTEMD_UNIT=ssh.service
# 开启激进模式, 对付只连接不输密码或频繁断开的扫描器
mode = aggressive
filter = sshd
# 允许失败次数
maxretry = 3
# 封禁时间（86400秒 = 1天）
bantime = 86400
# 统计时间范围（1小时内的失败次数）
findtime = 3600
```
**应用配置并验证：**
```bash
sudo systemctl restart fail2ban
# 查看 sshd 监控状态
sudo fail2ban-client status sshd
# 验证 UFW 拦截规则
sudo ufw status
```

## 2. 启用基于时间的一次性密码（TOTP 2FA/MFA）
开启双因素认证后，即便私钥/密码泄露，攻击者没有动态验证码也无法登录。
> **场景说明**：通常配置为**仅密码登录时要求 2FA**，或者**强制公钥 + 2FA 双重验证**。以下展示标准配置。

**安装依赖：**
```bash
sudo apt update && sudo apt install libpam-google-authenticator -y
```

**生成密钥（切换到需要登录的用户执行，不要用 root）：**
```bash
google-authenticator
```
程序会询问多个问题（如基于时间、防重放攻击、速率限制等），一路输入 `y` 回车即可。
**重要：请务必保存终端输出的二维码、Secret Key 以及 Emergency scratch codes（紧急恢复代码）！**

**配置 PAM 模块：**
```bash
sudo nano /etc/pam.d/sshd
```
在文件末尾添加：
```text
# nullok 表示如果某个用户没配置谷歌验证码，也允许其登录。如果要求强制所有用户使用，请去掉 nullok。
auth required pam_google_authenticator.so nullok
```

**配置 SSH 服务端：**
```bash
sudo nano /etc/ssh/sshd_config
```
修改以下内容（注意：较新的 Debian 中使用 `KbdInteractiveAuthentication`）：
```ssh-config
# 开启键盘交互认证
KbdInteractiveAuthentication yes
# (如果是老系统，可能使用的是 ChallengeResponseAuthentication yes)
```
重启 SSH 服务：
```bash
sudo systemctl restart ssh
```

---

# 五、Windows 环境下的 OpenSSH 管理

在 Windows 10/11 系统中，可以使用 Winget 或 MSI 管理原生 OpenSSH 客户端与服务端。

## 1. 通过 MSI 方式管理 Win32-OpenSSH
MSI 安装包会将其安装到 `ProgramFiles\OpenSSH` 目录下，可以在 `cmd` 或 `PowerShell` 中无头运行。

*   **安装 SSH 客户端和服务器（默认）：** `msiexec /i <openssh.msi路径>`
*   **仅安装 SSH 客户端：** `msiexec /i <openssh.msi路径> ADDLOCAL=Client`
*   **仅安装 SSH 服务器：** `msiexec /i <openssh.msi路径> ADDLOCAL=Server`
*   **卸载指定的组件：** `msiexec /i <openssh.msi路径> REMOVE=Server`
*   **完全卸载：** `msiexec /x <openssh.msi路径>`

**更新系统环境变量 PATH（供 SCP/SFTP 使用）：**
在管理员 PowerShell 中运行：
```powershell
[Environment]::SetEnvironmentVariable("Path", [Environment]::GetEnvironmentVariable("Path",[System.EnvironmentVariableTarget]::Machine) + ';' + ${Env:ProgramFiles} + '\OpenSSH', [System.EnvironmentVariableTarget]::Machine)
```

## 2. 替代方法（通过系统内置组件或 Winget）
*   **删除旧版内置 OpenSSH：**
    ```powershell
    Remove-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
    ```
*   **使用 Winget 安装较新版本：**
    ```powershell
    winget install Microsoft.OpenSSH.Beta --override ADDLOCAL=Client
    ```

**验证安装状态：**
```powershell
Get-Service -Name ssh*
```

---

# 附录：常用运维排错命令

如果遇到 SSH 无法连接的情况，可以使用以下命令在服务端排查日志：
```bash
# 查看 SSH 服务的运行状态
sudo systemctl status ssh

# 实时查看 SSH 的最近 100 行日志 (Debian 系统)
sudo journalctl -u ssh.service -n 100 --no-pager
```