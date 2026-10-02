---
title: 管理与删除 YubiKey 中的 FIDO2 常驻凭据
date: 2026-10-02 13:02:08
tags: [笔记, YubiKey, FIDO2]
---


# 管理与删除 YubiKey 中的 FIDO2 常驻凭据

FIDO2 常驻凭据（Discoverable Credentials）允许我们在登录时实现无密码体验（Passkey）。当 YubiKey 存储空间不足或需要清理废弃账号时，可以通过以下方式单独删除这些凭据。

### 📌 前置要求
- **固件要求**：YubiKey 固件版本必须 **≥ 5.2.x**（若低于此版本，由于硬件限制无法单独管理凭据，只能重置整个 FIDO 模块）。
- **软件要求**：已安装官方管理工具 [YubiKey Manager](https://www.yubico.com/support/download/yubikey-manager/)。
- **安全验证**：必须知道你为 YubiKey 设置的 FIDO2 PIN 码。

---

## 方案一：通过图形界面（GUI）删除（🌟 推荐）
现代版本的 YubiKey Manager 已经支持直接在界面中可视化管理 FIDO2 凭据，这是最直观且跨平台通用的方法。

*   **打开应用**：插入 YubiKey 并启动 YubiKey Manager。
*   **进入 FIDO2 设置**：在顶部导航栏选择 `Applications` -> `FIDO2`。
*   **验证身份**：点击 `Credentials`，系统会要求你输入 YubiKey 的 FIDO2 PIN 码。
*   **管理凭据**：在弹出的列表中，你可以直观地看到所有存储的常驻凭据。选中需要废弃的凭据，点击 `Delete` 即可。

---

## 方案二：通过命令行（CLI）删除（💻 适合进阶/极客）

如果你更喜欢终端操作，或需要编写自动化脚本，可以使用 `ykman` 命令行工具。

### 步骤 A：进入命令行环境

不同操作系统的 `ykman` 工具调用路径略有不同，请根据你的系统打开终端（Windows 请使用管理员身份打开 PowerShell 或 CMD）：

*   **Windows**
    默认安装路径下的调用方式：
    ```powershell
    cd "C:\Program Files\Yubico\YubiKey Manager"
    ```
*   **macOS**
    如果通过官方 pkg 安装，可直接调用：
    ```bash
    /Applications/YubiKey\ Manager.app/Contents/MacOS/ykman
    ```
*   **Linux**
    如果已将 `ykman` 添加到环境变量，可直接在终端使用 `ykman` 命令。

### 步骤 B：执行凭据管理命令

在上述路径下，你可以使用以下命令进行管理（以 Linux/已配置环境变量的直接调用为例，Windows 需在命令前加上 `.\ykman.exe`）：

*   **列出当前所有凭据**
    执行后会提示你输入 PIN 码，随后展示包含“凭据 ID (Credential ID)”或“关联用户名”的列表。
    ```bash
    ykman fido credentials list
    ```
    *(也可直接在命令后附带 PIN 码跳过提示，例如：`--pin <你的FIDO2_PIN>`，但在公共环境中需注意防范屏幕窥视)*

*   **精确删除指定凭据**
    找到需要删除的凭据后，通过复制其 `ID` 或 `用户名` 进行删除：
    ```bash
    ykman fido credentials delete <凭据ID_或_用户名>
    ```

*   **复查清理结果**
    再次列出凭据，确认目标已成功移除：
    ```bash
    ykman fido credentials list
    ```

---

## 💡 扩展提示与避坑指南

*   **关于固件版本过低**：
    如果你的 YubiKey 固件低于 5.2.x，终端会报错提示不支持该指令。此时唯一的清理方法是重置 FIDO 应用（`ykman fido reset`）。**注意：** 整体重置会清空所有 FIDO/FIDO2 凭据，请提前做好其他账号的备用登录措施。
*   **占位符说明**：
    在本指南的命令中，带有 `< >` 的部分（如 `<你的FIDO2_PIN>`）为占位符。实际使用时，请将其替换为你自己的真实信息，并且**不要保留 `< >` 符号**。
*   **跨设备一致性**：
    无论在 Windows、macOS 还是 Linux 上删除凭据，该操作是直接作用于 YubiKey 硬件芯片的，删除后在任何设备上该凭据均将失效。