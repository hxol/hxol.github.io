---
title: YubiKey PGP 密钥存储完整指南
date: 2026-10-02 13:01:39
tags: [笔记, GunPG, YubiKey]
---

# YubiKey PGP 密钥存储完整指南

本指南旨在演示如何生成凭据并将其安全地存储到 [YubiKey](https://www.yubico.com/products/identifying-your-yubikey/) 中。私钥一旦写入设备便无法被复制导出；我们会在安全的离线环境中单独保留一份“认证（Certify）”主密钥，仅用于未来的密钥替换或续期。

## 核心硬件准备

*   **选择 YubiKey 模型：** 请选择支持 OpenPGP 应用的 [YubiKey 硬件](https://www.yubico.com/store/compare/)（注意：Security Key 系列和 Bio 系列不支持 OpenPGP，无法用于本指南）。
*   **验证正品：** 前往 [yubico.com/genuine](https://www.yubico.com/genuine/) 验证设备。按照提示触摸按键以识别设备型号，这有助于防范供应链篡改。
*   **准备备份介质：** 准备至少两个 USB 闪存盘或 microSD 卡，用于在不同的物理地点存放加密的离线备份。

---

## 搭建安全配置环境

生成密钥和备份时，请务必使用专用的加固环境。以下环境的安全性由低到高排列（建议根据自身威胁模型选择）：

*   他人拥有的公共或共享计算机（最不安全）
*   具备无限制网络访问权限的日常用个人操作系统
*   功能受限的虚拟机（如 virt-manager、VirtualBox 或 VMware）
*   经过安全加固的专用 Debian 或 OpenBSD 裸机系统
*   **拔除主存储器后运行的临时 Debian Live 或 Tails 系统（大多数人的实用基准线）**
*   加固过的硬件和固件（例如 Coreboot、移除 Intel ME）
*   无网络功能的物理隔离（Air-gapped）系统，最好是基于 ARM 的树莓派等异构架构（最安全）

### 验证与制作 Debian Live 启动盘

*   **下载镜像与签名：**
    ```bash
    imageUrl="https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/"
    curl -sfL -O "$imageUrl/SHA512SUMS" -O "$imageUrl/SHA512SUMS.sign"
    curl -sfLO "$imageUrl/$(awk '/xfce\.iso$/ {print $NF}' SHA512SUMS)"
    ```
*   **获取并验证 Debian 公钥：**
    ```bash
    gpg --keyserver hkps://keyring.debian.org \
        --recv DF9B9C49EAA9298432589D76DA87E80D6294BE9B
    ```
    *(若接收失败，可更换为 `hkps://keyserver.ubuntu.com:443`)*
*   **验证签名与哈希：**
    ```bash
    gpg --verify SHA512SUMS.sign SHA512SUMS
    grep $(sha512sum debian-live-*-amd64-xfce.iso) SHA512SUMS
    ```
*   **写入 USB 设备：**
    连接存储设备并确认路径（本指南通篇以 `/dev/sdc` 为例，**请务必根据实际情况修改以防数据丢失**）。
    **Linux 环境写入：**
    ```bash
    sudo dd if=debian-live-*-amd64-xfce.iso of=/dev/sdc bs=4M status=progress ; sync
    ```
    **OpenBSD 环境写入：**
    ```bash
    doas dd if=debian-live-*-amd64-xfce.iso of=/dev/rsd2c bs=4m
    ```

### 准备硬件机器

关闭用于生成密钥的计算机。断开内部硬盘以及不必要的外设（如无线网卡）。通过刚刚制作的 Debian Live USB 启动系统。

> [!TIP]
> 如果 Debian Live 锁屏，默认的解锁用户名为 `user`，密码为 `live`。

---

## 软件安装与环境初始化

配置好网络（下载完软件后建议断网）。打开终端，根据您的操作系统安装依赖：

**Debian/Ubuntu**
```bash
sudo apt update
sudo apt -y upgrade
sudo apt -y install wget gnupg2 gnupg-agent dirmngr cryptsetup scdaemon pcscd yubikey-personalization yubikey-manager
```

**macOS** (使用 Homebrew)
```bash
brew install gnupg yubikey-personalization ykman pinentry-mac wget
```

**Arch Linux**
```bash
sudo pacman -Syu --needed gnupg pcsclite ccid yubikey-personalization
```

---

## 配置 GnuPG 环境

创建临时目录存放 GnuPG 配置。如果存放在非内存文件系统中，配置完成后请务必删除该目录。
```bash
export GNUPGHOME=$(mktemp -d "${TMPDIR:-/tmp}/$(date +%Y.%m.%d)-XXXXXXXX")
printf "\n临时目录:\t%s\n\n" "$GNUPGHOME"
```

下载并应用推荐的安全配置（包含现代加密默认值与隐私保护选项）：
```bash
wget https://raw.githubusercontent.com/drduh/YubiKey-Guide/main/config/gpg.conf -P $GNUPGHOME
```

检查配置
```bash
grep -v "^#" $GNUPGHOME/gpg.conf
```
内容
```
personal-cipher-preferences AES256 AES192 AES
personal-digest-preferences SHA512 SHA384 SHA256
personal-compress-preferences ZLIB BZIP2 ZIP Uncompressed
default-preference-list SHA512 SHA384 SHA256 AES256 AES192 AES ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
no-comments
no-emit-version
no-greeting
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
require-cross-certification
require-secmem
no-symkey-cache
armor
use-agent
throw-keyids
```

> [!IMPORTANT]
> **在此之后的设置，建议全程断开网络连接。**

### 设置密钥参数

*   **身份标识 (Identity)：** 如果不需要公开身份，可生成随机标签；如需用于邮件或 Git，请填写真实名称和邮箱。
    ```bash
    export IDENTITY="YubiKey User <yubikey@example.com>"
    ```
*   **密钥算法 (Algorithm)：** 推荐使用 RSA 4096。
    ```bash
    export KEY_TYPE=rsa4096
    ```
*   **过期时间 (Expiration)：** 认证主密钥保持不过期；子密钥推荐设置 2 年有效期。过期后旧密钥仍可解密现有数据，但无法加密新数据。
    ```bash
    export KEY_EXPIRATION=2y
    ```
*   **主密钥密码 (Passphrase)：** 生成只包含大写字母和数字的强密码，便于手写记录。
    ```bash
    export CERTIFY_PASS=$(LC_ALL=C tr -dc "A-Z3-9" < /dev/urandom | tr -d "IOUS5" | head -c ${PASS_LENGTH:-24} | fold -w ${PASS_GROUPSIZE:-4} | paste -sd ${PASS_DELIMITER:--} -)
    printf "\n认证密钥密码:\t%s\n\n" "$CERTIFY_PASS"
    ```
    *请将此密码抄写在纸上，妥善保管。*

---

## 生成 OpenPGP 密钥体系

### 生成认证密钥 (Certify Key)
该密钥将永远离线保存，仅用于授权子密钥。
```bash
printf '%s' "$CERTIFY_PASS" |
  gpg --batch --passphrase-fd 0 \
      --quick-generate-key "$IDENTITY" "${KEY_TYPE-rsa4096}" \
      cert never
```
获取指纹与 ID：
```bash
export KEY_FP=$(gpg -k --with-colons $IDENTITY | awk -F: '/^fpr:/ { print $10; exit }')
export KEY_ID="${KEY_FP: -16}"
printf "\nKey ID/指纹:\t%s\n%s\n\n" "$KEY_ID" "$KEY_FP"
```

### 生成子密钥 (Subkeys)
分别生成用于签名 (Sign)、加密 (Encrypt) 和身份验证 (Auth, 可用于 SSH) 的子密钥：

**签名与加密子密钥：**
```bash
printf '%s' "$CERTIFY_PASS" |
  gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
      --quick-add-key "$KEY_FP" "${KEY_TYPE-rsa4096}" sign "${KEY_EXPIRATION-2y}"
printf '%s' "$CERTIFY_PASS" |
  gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
      --quick-add-key "$KEY_FP" "${KEY_TYPE-rsa4096}" encrypt "${KEY_EXPIRATION-2y}"
```

**身份验证子密钥：** (注：若部分 SSH 服务器拒收 RSA，可将 `$KEY_TYPE` 改为 `ed25519`)
```bash
printf '%s' "$CERTIFY_PASS" |
  gpg --batch --pinentry-mode loopback --passphrase-fd 0 \
      --quick-add-key "$KEY_FP" "${KEY_TYPE-ed25519}" auth "${KEY_EXPIRATION-2y}"
```

验证密钥状态 (`gpg -K`)，确保 `[C]`, `[S]`, `[E]`, `[A]` 均已成功生成。
```
sec   rsa4096/0xF0F2CFEB04341FB5 2026-08-01 [C]
      Key fingerprint = 4E2C 1FA3 372C BA96 A06A  C34A F0F2 CFEB 0434 1FB5
uid                   [ultimate] yk.pn8wgfx67khlhext
ssb   rsa4096/0xB3CD10E502E19637 2026-08-01 [S] [expires: 2028-08-01]
ssb   rsa4096/0x30CBE8C4B085B9F7 2026-08-01 [E] [expires: 2028-08-01]
ssb   rsa4096/0xAD9E24E1B8CB9600 2026-08-01 [A] [expires: 2028-08-01]
```
---

## 导出与加密备份

> [!IMPORTANT]
> 备份文件包含绝对机密。掌握备份文件和主密钥密码的人，将能够完全复刻您的身份。

**导出密钥到临时目录：**
```bash
gpg --output $GNUPGHOME/public-$KEY_ID-$(date +%F).asc --armor --export $KEY_ID
printf '%s' "$CERTIFY_PASS" | gpg --batch --pinentry-mode loopback --passphrase-fd 0 --output $GNUPGHOME/secret-$KEY_ID-Certify.key --armor --export-secret-keys $KEY_ID
printf '%s' "$CERTIFY_PASS" | gpg --batch --pinentry-mode loopback --passphrase-fd 0 --output $GNUPGHOME/secret-$KEY_ID-Subkeys.key --armor --export-secret-subkeys $KEY_ID
```

**创建 LUKS 加密 U 盘备份 (以 Linux 为例)：**
生成 LUKS 密码：
```bash
export LUKS_PASS=$(LC_ALL=C tr -dc "A-Z3-9" < /dev/urandom | tr -d "IOUS5" | head -c 24 | fold -w 4 | paste -sd - -)
printf "\nLUKS 密码:\t%s\n\n" "$LUKS_PASS"
```
格式化并加密 U 盘分区（危险操作，确认 `/dev/sdc1` 无误）：
```bash
printf '%s' "$LUKS_PASS" | sudo cryptsetup -q luksFormat /dev/sdc1
printf '%s' "$LUKS_PASS" | sudo cryptsetup -q luksOpen /dev/sdc1 gnupg-secrets
sudo mkfs.ext2 /dev/mapper/gnupg-secrets -L gnupg-$(date +%F)
```
挂载并复制文件，然后卸载：
```bash
sudo mkdir -p /mnt/encrypted-storage
sudo mount /dev/mapper/gnupg-secrets /mnt/encrypted-storage
sudo cp -av $GNUPGHOME /mnt/encrypted-storage/
sudo umount /mnt/encrypted-storage
sudo cryptsetup luksClose gnupg-secrets
```
*建议在多块 U 盘上重复此备份操作。*

**导出公钥（明文保存）：**
在 U 盘上建立一个非加密分区（如 `/dev/sdc2`），将公钥（`.asc` 文件）保存在此，以便日后在新电脑上导入。
```bash
gpg --armor --export $KEY_ID | sudo tee /mnt/public/$KEY_ID-$(date +%F).asc
```
---

## 配置与转移至 YubiKey

插入 YubiKey 并确认状态 (`gpg --card-status`)。

### 更改 YubiKey PIN 码
YubiKey OpenPGP 应用有独立的三种 PIN 码：
| 名称 | 默认值 | 权限说明 |
| :-: | :-: | - |
| User PIN | `123456` | 常规操作（解密、签名、验证），最短 6 位 |
| Admin PIN | `12345678` | 管理操作（重置 PIN、写入密钥），最短 8 位 |
| Reset Code | 无 | 用于重置 User PIN |

生成随机 PIN：
```bash
PINS=$(LC_ALL=C tr -dc '0-9' < /dev/urandom | head -c 14)
export ADMIN_PIN=${PINS:0:8}
export USER_PIN=${PINS:8:6}
printf "\nAdmin PIN:\t\t%s\nUser  PIN:\t\t%s\n\n" "$ADMIN_PIN" "$USER_PIN"
```
使用 `gpg --change-pin` 菜单分别修改 Admin PIN (选项 3) 和 User PIN (选项 1)。您也可以使用 `ykman` 修改重试次数限制。

### 设置智能卡属性
为避免物理接触带来的隐私泄露，请勿在智能卡内明文写入真实姓名，推荐使用随机标签：
```bash
export CARD_ATTR_LOGIN="yk.$(LC_ALL=C tr -dc 'a-z0-9' < /dev/urandom | head -c 16)"
```
通过 `gpg --edit-card` 的 `admin` 模式，使用 `login` 命令写入该属性。

### 转移子密钥 (单向操作)
> [!NOTE]
> 密钥一旦移入 YubiKey，本地私钥将被删除并替换为存根(stub)。**请确保上一步备份已完成！**

使用 `gpg --edit-key $KEY_FP`，依次选择并转移三个子密钥：
*   选择签名密钥：`key 1` -> `keytocard` -> 选择 `1`
*   选择加密密钥：`key 2` -> `keytocard` -> 选择 `2`
*   选择验证密钥：`key 3` -> `keytocard` -> 选择 `3`
*   输入 `save` 保存。

验证转移：运行 `gpg -K`，若三个子密钥前均显示 `ssb>` (带有 `>`)，即表示私钥已存在于智能卡中。清理临时工作目录并重启计算机。

---

## 日常配置与使用指南

在新电脑上使用时，您**只需导入公钥**即可。

*   **导入公钥：** 挂载包含公钥的 U 盘并运行 `gpg --import /mnt/public/public-*.asc`。然后使用 `gpg --edit-key $KEY_ID` 将信任级别(`trust`)设为极致 (`5`)。
*   **防止 GnuPG 弹窗干扰：** 在 `~/.gnupg/scdaemon.conf` 中添加 `disable-ccid`。

### 基础密码学操作

*   **文件加密：**
    ```bash
    printf "机密信息" | gpg --encrypt --armor --recipient $KEY_ID --output encrypted.txt
    ```
*   **文件解密 (将触发 YubiKey PIN 提示)：**
    ```bash
    gpg --decrypt --armor encrypted.txt
    ```
*   **签名与验证：**
    ```bash
    printf "文本" | gpg --armor --clearsign > signed.txt
    gpg --verify signed.txt
    ```

### 配置物理触摸确认 (Touch Policy)
使用 YubiKey Manager 强制要求在执行操作时触摸按键：
```bash
ykman openpgp keys set-touch dec on  # 加密
ykman openpgp keys set-touch sig on  # 签名
ykman openpgp keys set-touch aut on  # 身份验证
```

### SSH 登录与代理配置

GnuPG 的 `gpg-agent` 可以接管 SSH 认证，直接将请求发送给 YubiKey。

*   **配置 GPG Agent：** 在 `~/.gnupg/gpg-agent.conf` 中加入 `enable-ssh-support`。
*   **环境变量加载 (添加至 `~/.bashrc` 或 `~/.zshrc`)：**
    ```bash
    export GPG_TTY=$(tty)
    export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
    gpgconf --launch gpg-agent
    ```
*   **提取 SSH 公钥：**
    ```bash
    ssh-add -L > ~/.ssh/id_rsa_yubikey.pub
    ```
*   将该公钥内容添加到远程服务器的 `~/.ssh/authorized_keys` 中。
*   **避免 SSH 枚举密钥：** 在 `~/.ssh/config` 对应 Host 下配置 `IdentitiesOnly yes` 并指定 `IdentityFile ~/.ssh/id_rsa_yubikey.pub`。

*(注：Windows 平台支持 PuTTY 整合，WSL 平台可通过 weasel-pageant 或 usbipd-win 实现代理转发。)*

### SSH 代理转发 (Agent Forwarding)
> [!CAUTION]
> 代理转发存在一定的安全风险，请确保只对完全信任的服务器启用。

*   **使用 S.gpg-agent.ssh：** 在本地 `~/.ssh/config` 中配置 `RemoteForward` 指令，映射本地与远程的 agent 套接字路径。这使得在远程机器上执行 `ssh` 操作时可以直接调取本地的 YubiKey。

### Git 代码签名验证
```bash
git config --global user.signingkey $KEY_ID
git config --global commit.gpgsign true
git config --global tag.gpgSign true
```

### 邮件加密 (Email)
YubiKey 可与 Thunderbird（原生支持外挂 GPG）、Mailvelope（Chrome 插件支持 Gmail）以及 Mutt 配合使用，实现端到端的邮件加密。

---

## 密钥管理与维护

### 更新与轮换密钥 (Updating Keys)

当子密钥过期时，需要取出离线存放的认证主密钥。
1.  进入离线安全环境。
2.  解密并挂载 LUKS U 盘，将原密钥复制到临时目录。
3.  **续期：** 使用 `gpg --quick-set-expire "$KEY_FP" "$KEY_EXPIRATION"` 延长有效期。
4.  **轮换：** 撤销旧子密钥，生成新子密钥，并重新执行转移到 YubiKey 的步骤。
5.  导出更新后的公钥并导入日常使用的电脑中。

### 锁定重置 (Reset YubiKey)
如果 PIN 码输入错误次数超限，YubiKey OpenPGP 应用将被锁定。您可以使用 `ykman openpgp reset` 将其彻底清空，随后重新从加密备份中生成并导入子密钥。

---

## 附加安全建议与故障排查

**可选加固措施：**
*   **增强熵源：** 配置 `rng-tools` 或硬件随机数生成器提升密钥生成质量。
*   **启用 KDF：** 启用密钥派生函数以防止 PIN 被监听窃取。
*   **网络隔离：** 在生成密钥期间配置严格的防火墙（默认拒绝入站），或物理断开网络。

**常见问题速查表：**

| 症状或错误提示 | 推荐操作 |
| :--- | :--- |
| `Yubikey core error: no yubikey present` | 检查插入状态，或更新 `yubikey-personalization` 软件包。 |
| 公钥丢失 | 通过卡片内的 URL 或公共密钥服务器寻回。 |
| 解密或签名失败 `agent refused operation` | 确认 `gpg-agent` 成功接管，执行 `gpg-connect-agent updatestartuptty /bye`。 |
| SSH 报错 `Permission denied (publickey)` | 运行 `ssh -vvv host` 查看详细日志，确认卡内公钥是否已发送给服务器。 |
| `gpg: selecting card failed: No such device` | 检查 `pcscd` 守护进程是否启动，或检查 Polkit 权限规则配置。 |

> *(如需针对 macOS, Windows 等平台添加详细配置，请直接在相关模块下方补充。)*

## 参考文档

https://github.com/drduh/YubiKey-Guide
