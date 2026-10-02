---
title: PGP 基本用法
date: 2026-10-01 15:30:38
tags: [笔记, SSH, PGP, GnuPG, 安全, 隐私, 加密]
---


# PGP 基本用法

## PGP 核心概念与现代认知

### 基础释义
*   **PGP**：最初商业软件的名字（Pretty Good Privacy）。
*   **OpenPGP**：基于 PGP 技术提取的 IETF 开放标准。
*   **GnuPG (GPG)**：实现了 OpenPGP 标准的开源自由软件，也是我们日常使用的命令行工具 `gpg` 的本体。

### 核心能力
*   **加密与防篡改**：确保信息只有指定的接收者能解密，且传输途中连一个字节都未被篡改。
*   **身份认证 (Git/代码签名)**：防止他人在代码仓库中伪造你的提交记录（Commit Spoofing）。
*   **SSH 登录**：通过 `gpg-agent` 替代 `ssh-agent`，配合硬件智能卡实现物理级别的防窃取 SSH 登录。

### 算法的现代演进 (ECC 取代 RSA)
过去的教程常推荐 RSA 4096，但在现代密码学实践中，**椭圆曲线密码学 (ECC)** 才是绝对的主流：
*   **Ed25519**：用于数字签名的爱德华曲线算法，速度极快，安全性极高。
*   **Curve25519 (Cv25519)**：用于加密的曲线算法。
*   **优势**：同等安全强度下，ECC 密钥体积极小（导出后仅几行），加解密性能优异，且目前的硬件智能卡（YubiKey 等）均已完美支持。

---

## 快速部署与密钥生成

### 环境准备
各主流操作系统均已原生支持或提供便捷安装：
*   **macOS**: `brew install gnupg`
*   **Linux**: `sudo apt install gnupg2` (或其他包管理器)
*   **Windows**: 下载安装 Gpg4win

### 生成主密钥 (Master Key)
安全实践的核心是：**主密钥仅用于认证和签发，日常加解密使用子密钥**。

运行全功能生成向导：
```bash
gpg --expert --full-generate-key
```
交互流程关键点（选用 ECC 算法）：
```text
Please select what kind of key you want:
   (1) RSA and RSA
   (9) ECC (sign and encrypt) *default*
Your selection? 9

Please select which elliptic curve you want:
   (1) Curve 25519 *default*
Your selection? 1

Please specify how long the key should be valid.
Key is valid for? (0) 2y  # 建议设置 1~2 年有效期，防止私钥丢失后公钥永久生效，到期前可随时续期

Real name: Alice Zero                # 姓名或常用网名
Email address: alice@example.com     # 核心邮箱
Comment:                             # 可留空

# 确认无误后输入 o，并在弹出的密码框中设置一个强保护密码 (Passphrase)。
```

### 生成撤销证书 (Revocation Certificate)
如果你遗忘了密码，或主密钥失窃，撤销证书是你**唯一**能向世界宣告该密钥作废的凭证。务必第一时间生成并妥善离线保存。
```bash
gpg --gen-revoke -ao revoke.asc alice@example.com
```

### 生成专属子密钥 (Subkeys)
主密钥生成时会自动附带一个加密 `[E]` 子密钥。为实现权限隔离，我们通常还需要追加独立的签名 `[S]` 和认证 `[A]` 子密钥。

进入密钥编辑模式：
```bash
gpg --edit-key alice@example.com
```
交互流程：
```text
gpg> addkey

Please select what kind of key you want:
   (10) ECC (sign only)
   (11) ECC (set your own capabilities)
Your selection? 10  # 选择生成签名专用子密钥

# 按照提示继续选择 Curve 25519 和有效期，生成完成后：

gpg> save  # 【关键】必须输入 save 保存并退出！
```
*注：按照相同步骤，选择 `(11)` 可自定义权限，关闭 Sign 和 Encrypt，仅开启 Authenticate，即可获得 SSH 登录专用的 `[A]` 子密钥。*

---

## 密钥的查看与安全配置

### 安全显示配置
GPG 默认显示的短 ID 极易遭受碰撞伪造攻击。必须配置为显示长 ID 和完整指纹。

编辑 `~/.gnupg/gpg.conf`，添加以下两行：
```text
keyid-format 0xlong
with-fingerprint
```

### 列出密钥细节与指纹 (重点识别标识)
查看本地私钥库：
```bash
gpg -K --keyid-format long --fingerprint
```
输出细节解析：
```text
sec   ed25519/0x99F583599B7E31F1 2026-01-11 [SC]
      Key fingerprint = 7053 58AB 8536 6CAB 05C0  220F 99F5 8359 9B7E 31F1
uid                   [ultimate] Alice Zero <alice@example.com>
ssb   cv25519/0x6FE9C71CFED44076 2026-01-11 [E]
ssb   ed25519/0xFDB960B857D397F6 2026-01-11 [S]
```
*   `sec` (Secret Key)：主私钥。若带有 `#` 号（如 `sec#`），说明主私钥处于离线安全状态，当前设备仅存有子私钥（这是最佳实践）。
*   `ssb` (Secret Subkey)：子私钥。
*   `[SC]`, `[E]`, `[S]`：代表密钥具备的能力（Sign 签名, Certify 认证, Encrypt 加密）。
*   `0x99F5...`：此为长 Key ID。
*   `Key fingerprint`：40 位的完整指纹，用于在多渠道验证公钥真伪。

---

## 身份管理：UID 与头像操作

一个 PGP 密钥可以绑定多个身份（UID）。例如，你主要使用个人邮箱，但在 GitHub 上希望使用 `noreply` 隐私邮箱进行签名，此时只需追加一个新的 UID，无需生成新密钥。

### 添加与修改 UID
进入编辑交互界面：
```bash
gpg --edit-key alice@example.com
```

交互流程：
```text
gpg> adduid

Real name: Alice Dev
Email address: alice-id@users.noreply.github.com
Comment: GitHub Account
# 确认后，列表会显示两个 UID。

# 切换主 UID（Primary UID）
gpg> uid 2      # 选中第二个 UID（前面会出现星号 *）
gpg> primary    # 将其设为主 UID

# 吊销旧 UID（注：已存在的 UID 无法直接修改，只能吊销后新建）
gpg> uid 1      # 选中旧 UID
gpg> revuid     # 吊销该 UID

gpg> save       # 务必保存退出
```

### 添加头像 (Photo ID)
可以在公钥中嵌入极小体积的头像，方便他人在验证时辨识。
```text
gpg> addphoto
# 输入本地 JPEG 图片的绝对路径（建议尺寸 128x128 左右，体积越小越好）
gpg> showphoto  # 查看当前头像
gpg> save
```

---

## 日常加解密与签名操作

### 加密与解密
**发送机密文件给 Bob：**
```bash
# -s 签名 (证明是你发的)
# -e 加密 (确保只有 Bob 能看)
# -r 指定接收者公钥 (需提前导入 Bob 的公钥)
gpg -se -o secret.txt -r bob@example.com source.txt
```
**解密收到的文件：**
```bash
gpg -d secret.txt
```

### 签名与验证
**对文件生成独立签名（不改变原文件）：**
```bash
gpg --armor --detach-sign document.pdf
# 将生成 document.pdf.asc 签名文件
```
**验证他人发来的签名：**
```bash
gpg --verify document.pdf.asc document.pdf
```

---

## 进阶集成场景

### 使用 PGP 为 Git Commit 签名
配置 Git 使用你的 PGP 密钥（填入具有 `[S]` 权限的长 Key ID）：
```bash
git config --global user.signingkey 0xFDB960B857D397F6
git config --global commit.gpgsign true
```

### 使用 PGP 替代 SSH 密钥
通过启用 `gpg-agent` 的 SSH 支持，可以让带 `[A]` 权限的子密钥负责 SSH 登录。

修改 `~/.gnupg/gpg-agent.conf`，追加：
```text
enable-ssh-support
```
将当前 Shell 环境接管（可写入 `~/.bashrc` 等）：
```bash
export GPG_TTY=$(tty)
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)
echo UPDATESTARTUPTTY | gpg-connect-agent 1> /dev/null
```
获取 `[A]` 子密钥的 Keygrip，并追加到 `~/.gnupg/sshcontrol` 中。最后通过 `ssh-add -L` 即可导出可用于服务器 `authorized_keys` 的公钥格式。

---

## 密钥的备份、迁移与吊销

### 离线备份策略
**主密钥极其重要，建议导出后存入全盘加密的脱机 U 盘，日常设备仅导入子密钥。**

```bash
# 1. 导出公钥 (用于发布)
gpg -ao public-key.asc --export alice@example.com

# 2. 导出所有私钥 (最高机密，包含主私钥)
gpg -ao master-secret.asc --export-secret-key alice@example.com

# 3. 仅导出子私钥 (用于日常工作设备)
gpg -ao subkeys.asc --export-secret-subkeys alice@example.com
```
*注：命令中 `!` 符号的妙用：如果你只想导出某一个特定的子私钥，可以在其 Key ID 后加 `!`，如 `--export-secret-subkeys 0xFDB960B857D397F6!`*

### 吊销子密钥或主密钥
如果工作电脑失窃，导致子私钥泄露，需使用离线保存的主私钥来吊销该子密钥。

```bash
gpg --edit-key alice@example.com

gpg> key 2      # 选择需要吊销的子密钥序号 (前面会出现星号 *)
gpg> revkey     # 声明吊销
gpg> save       # 保存退出
```
如果是**主密钥泄露或遗失密码**：
在一台新环境导入你的公钥，然后直接导入当初备份的 `revoke.asc` 撤销证书，该公钥即被标记为作废。随后请将作废后的公钥重新发布，通知世界。

---

## 公钥的分发与现代信任网络

### 抛弃过时的传统 KeyServer
过去常见的 SKS Keyserver Pool（如 `keyserver.ubuntu.com`）采用“不可删除、无需验证”的设计。这导致了严重的隐私泄漏（无法彻底撤回真实姓名）、UID 滥用（被塞入垃圾信息）以及证书 DoS 投毒攻击。**目前强烈不推荐新手向此类服务器上传公钥。**

### 现代公钥分发方案

#### keys.openpgp.org
这是现代 GPG 社区主推的新一代服务器：
*   **强制邮箱验证**：必须接收确认邮件，他人才能通过邮箱搜到你的公钥。
*   **抗 DoS 攻击**：自动剥离无关的第三方签名。
*   **随时可删**：支持用户自主撤下公钥。
*   **用法**：直接在网页端上传你的公钥文件，或通过命令行：
    `gpg --keyserver hkps://keys.openpgp.org --send-keys <KeyID>`

#### WKD (Web Key Directory)
对于拥有个人域名的用户，WKD 提供了无缝的分发体验。通过在域名下放置特定格式的公钥文件（如 `https://example.com/.well-known/openpgpkey/...`），现代邮件客户端（如 Enigmail、ProtonMail）在发信时会自动静默获取接收者的公钥，真正实现透明加密。

#### 自有网络发布
将公钥文件及指纹发布在你能够控制的平台上，是性价比最高的防伪方式：
*   代码托管平台（GitHub 个人资料仓库、Gist）。
*   个人博客、Notion 共享页的“关于我”页面（确保启用 HTTPS）。
*   去中心化社交网络（Mastodon）的简介栏。