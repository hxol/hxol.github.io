---
title: Windows 10 11 KMS 激活与管理指南
date: 2026-10-01 15:30:29
tags: [笔记, Windows, KMS]
---

# Windows 10 11 KMS 激活与管理指南

### ⚠️ 注意事项与前提准备
1. **使用命令提示符 (CMD)**：请务必以**“管理员身份”**运行传统的“命令提示符 (cmd.exe)”。若使用新版 Windows 终端 (Windows Terminal) 或 PowerShell，可能会因参数解析机制不同而导致提示“命令参数无效”。
2. **命令输出方式**：
   * 直接运行 `slmgr.vbs` 命令会弹出窗口显示结果。
   * 如果希望结果直接在命令行界面输出，可以在命令前加上 `cscript //nologo C:\Windows\System32\` （如下文备选命令所示）。

---

### 第一阶段：KMS 激活核心步骤

#### 1. 安装产品密钥
根据您安装的系统版本，输入对应的 KMS 客户端安装密钥（GVLK）。
```cmd
slmgr.vbs /ipk NPPR9-FWDCX-D2C8J-H872K-2YT43
```
*(命令行静默输出版)*
```cmd
cscript //nologo C:\Windows\System32\slmgr.vbs /ipk NPPR9-FWDCX-D2C8J-H872K-2YT43
```

#### 2. 配置 KMS 服务器地址
将 KMS 服务器指向您所在的内网服务器或自建服务器（请将 `<KMS服务器地址>` 替换为实际的 IP 或域名，例如 `192.168.x.x` 或 `kms.example.com`）。
```cmd
slmgr.vbs /skms <KMS服务器地址>
```
*(命令行静默输出版)*
```cmd
cscript //nologo C:\Windows\System32\slmgr.vbs /skms <KMS服务器地址>
```

#### 3. 执行激活
应用密钥和服务器地址后，向 KMS 服务器发起激活请求。
```cmd
slmgr.vbs /ato
```
*(命令行静默输出版)*
```cmd
cscript //nologo C:\Windows\System32\slmgr.vbs /ato
```

> **💡 激活机制提示**：以上步骤完成后，您的 Windows 客户端即会使用指定的 KMS 服务器进行激活。KMS 激活的有效期通常为 **180 天**。系统客户端会默认每 **7 天**自动尝试与 KMS 服务器通信并续期。只要设备能确保持续或定期连接到该服务器网络，即可避免激活过期。

---

### 第二阶段：状态查询与维护命令

完成激活后或日常维护时，您可以使用以下命令检查系统授权状态：

* **查看激活过期日期**
  ```cmd
  slmgr.vbs /xpr
  ```
* **查看简要许可信息**
  ```cmd
  slmgr.vbs /dli
  ```
* **查看详细许可信息**（排错首选）
  ```cmd
  slmgr.vbs /dlv
  ```
* **卸载当前产品密钥**
  ```cmd
  slmgr.vbs /upk
  ```
* **重置激活倒计时**（最多通常允许重置 3~5 次）
  ```cmd
  slmgr.vbs /rearm
  ```

---

### 附录 A：常用 Windows 10/11 商业版 GVLK 密钥

在执行第一步（安装产品密钥）时，请根据系统版本选择对应的密钥：

| 系统版本 | KMS 客户端安装密钥 (GVLK) |
| :--- | :--- |
| **Windows 10 / 11 商业企业版** (普通企业版) | `NPPR9-FWDCX-D2C8J-H872K-2YT43` |
| **Windows 10 企业版 LTSC 2021 / 2019** | `M7XTQ-FN8P6-TTKYV-9D4CC-J462D` |
| **Windows 10 企业版 LTSB 2016** | `DCPHK-NFMTC-H88MJ-PFHPY-QJ4BJ` |
| **Windows 10 企业版 LTSB 2015** | `WNMTR-4C88C-JK8YV-HQ7T2-76DF9` |

---

### 附录 B：常见错误排查

若在执行 `/ato` 激活命令时出现任何错误提示，可先借助 `slmgr.vbs /dlv` 查看详细信息，并根据错误代码进行排查。

* ❌ **错误代码：`0xC004F069`**
  * **原因**：输入的密钥与当前安装的 Windows 系统版本不匹配（例如在专业版系统上强行输入企业版的密钥）。
  * **解决**：请检查“设置 > 系统 > 关于”中的 Windows 版本，并替换为对应的正确 GVLK 密钥。