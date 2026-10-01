---
title: 使用 Pocket ID 为群晖 (Synology DSM) 配置 OIDC 单点登录指南
date: 2026-09-30 15:30:00
tags: [笔记, 群晖, Pocket ID, 自托管]
---


# 使用 Pocket ID 为群晖 (Synology DSM) 配置 OIDC 单点登录指南

本文将指导您如何使用 Pocket ID 作为身份提供商 (IdP)，为群晖 (Synology DSM) 配置基于 OpenID Connect (OIDC) 的单点登录 (SSO) 服务。

---

## 第一部分：Pocket ID 侧配置

### 1. 创建用户
首先，需要在 Pocket ID 中创建一个与群晖 NAS 本地账号对应的用户。
前往 **设置 -> 管理员选项 -> 用户**，点击 **“添加用户”**：

*   **名字**：`nas_user` *(⚠️ 注意：必须与您要在 NAS 上登录的本地账号名称完全一致)*
*   **显示名称**：`nas_user`（与名字相同即可）
*   **用户名**：`nas_user`（与名字相同即可）
*   **电子邮件**：`user@example.com` *(请替换为您真实的邮箱)*
*   点击 **“保存”**。

### 2. 创建用户组
为了方便权限管理，我们创建一个专门针对 NAS 访问的用户组。
前往 **设置 -> 管理员选项 -> 用户组**，点击 **“添加群组”**：

*   **显示名称**：`NAS Users`
*   **名称**：`nas_users`
*   点击 **“保存”**。

### 3. 将用户添加进组
*   在用户组列表中，点击刚刚创建的 **NAS Users** 进入详情页。
*   在用户列表中，勾选前面创建的用户（`nas_user`）。
*   点击 **“保存”**。

### 4. 创建 OIDC 客户端
接下来，为群晖 DSM 创建一个 OIDC 客户端应用。
前往 **设置 -> 管理员选项 -> OIDC 客户端**，点击 **“添加 OIDC 客户端”**：

*   **名称**：`Synology DSM`
*   **客户端启动网站**：`https://dsm.example.com/` *(请替换为您群晖的实际访问地址)*
*   **回调 URL (Callback URL)**：`https://dsm.example.com` *(注意：Pocket ID 现在已经不认带 `/#/signin` 的地址了)*
*   **登出回调 URL (Logout Callback URL)**：`https://dsm.example.com/`
*   **公共客户端 (Public Client)**：`取消勾选`
*   **公钥代码交换 (PKCE)**：`建议关闭 (取消勾选)`
*   **需要重新验证**：`关闭 (取消勾选)`
*   点击 **“保存”**。

> ⚠️ **非常重要**：保存成功后，请务必记录下页面上生成的 **Client ID (客户端 ID)** 和 **Client Secret (客户端密钥)**，这将在配置群晖时使用。

### 5. 为 OIDC 客户端分配群组权限
*   返回 **设置 -> 管理员选项 -> OIDC 客户端** 列表。
*   点击刚刚创建的 **Synology DSM** 客户端进入详情。
*   找到 **“允许的用户组” (Allowed Groups)** 选项。
*   勾选 **NAS Users**。
*   点击 **“保存”**。

---

## 第二部分：Synology DSM 侧配置

登录群晖 DSM 管理后台，前往 **控制面板 -> 域/LDAP -> SSO 客户端**。

### 1. 启用 SSO 服务
*   勾选 **“启用 OpenID Connect SSO 服务”**。
*   点击下方的 **“OpenID Connect SSO 设置”** 按钮。

### 2. 填写 OIDC 配置参数
在弹出的设置窗口中，按以下信息进行填写：

*   **配置文件**：选择 `OIDC`
*   **帐户类型 (Account type)**：⚠️ **非常重要！选择 `域/LDAP/本地 (Domain/LDAP/local)`**。选择此项既能使用 SSO 登录，也能在 SSO 出现故障时保留本地账号登录的后路。
*   **名称**：`Pocket ID`（此名称将显示在群晖的登录按钮上）
*   **Well-known URL**：`https://login.example.com/.well-known/openid-configuration` *(请将 `login.example.com` 替换为您 Pocket ID 的实际域名)*
*   **应用程序 ID**：填写在 Pocket ID 中生成的 **Client ID**。
*   **应用程序密钥 (Application Secret)**：填写在 Pocket ID 中生成的 **Client Secret**。
*   **重定向 URI (Redirect URI)**：`https://dsm.example.com` *(必须与 Pocket ID 中配置的回调 URL 一字不差)*。
*   **授权范围 (Authorization scope)**：填写 `openid profile email`（各单词间用空格隔开）。
*   **用户名声明 (Username claim)**：填写 `preferred_username` 或 `name`（这决定了群晖如何提取 Pocket ID 中的用户名去匹配本地账号）。

检查无误后，点击 **“应用”** 或 **“保存”**。

---

## 💡 补充提示与注意事项 (Troubleshooting)

1. **账号匹配原则**：由于您在群晖中选择了“域/LDAP/本地”账户类型，群晖会提取 OIDC 传过来的 `Username claim`（如 `nas_user`），并去匹配 NAS 的**本地同名账号**。如果 NAS 本地没有这个账号，或者名称大小写不一致，将导致 SSO 登录失败。
2. **HTTPS 证书要求**：无论是 Pocket ID 还是 Synology DSM，强烈建议（通常也是 OIDC 协议强制要求的）双方都配置并使用有效的 HTTPS 证书。如果使用未受信任的自签证书，可能导致 DSM 无法正确请求 Well-known URL。
3. **安全测试建议**：配置完成后，**请不要立即注销当前管理员账号**。建议打开浏览器的“无痕/隐身模式”或者使用另一个浏览器，测试通过 Pocket ID 登录群晖是否成功。如果配置有误导致无法登录，您还可以使用原浏览器的管理员会话进行修改或关闭 SSO 功能。
