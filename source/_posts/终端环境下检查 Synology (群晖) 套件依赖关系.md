---
title: 终端环境下检查 Synology (群晖) 套件依赖关系
date: 2026-10-02 13:02:08
tags: [笔记, 群晖, Synology, DSM, 依赖关系]
---



# 终端环境下检查 Synology (群晖) 套件依赖关系

在管理 Synology NAS 时，某些核心套件（如 `SynologyApplicationService` 或 `Node.js`）往往被其他高级应用所依赖。在进行系统清理、套件卸载或故障排查前，理清套件间的依赖关系尤为重要。

本指南记录了如何通过终端命令行快速查询 Synology 套件的依赖关系。

### 前置准备：通过 SSH 访问终端
出于安全考虑，现已不推荐使用老旧的 Telnet 协议。请使用 SSH 登录您的 NAS：
* 在 DSM 控制面板的“终端机和 SNMP”中开启 SSH 功能。
* 使用终端工具（如 Terminal、PuTTY 等）连接至 NAS：
  ```bash
  ssh <您的用户名>@<您的NAS_IP> -p <SSH端口号>
  ```
  > **安全提示**：完成排查操作后，建议及时在控制面板中关闭 SSH 功能，或更改默认端口（22）以防范网络攻击。

### 核心查询命令

掌握以下两条核心指令，即可轻松梳理套件关系。您可以根据需要随时执行：

**获取所有已安装套件的内部名称**
在查询依赖之前，您需要知道目标套件的准确系统名称。运行以下命令可以列出当前系统内所有已安装套件的名称清单：
```bash
synopkg list --name
```

**检查特定套件的依赖关系**
当您确认了需要查询的套件名称后，通过附加 `--depend-on` 参数，即可查看哪些套件依赖于该核心套件：
```bash
synopkg list --name --depend-on <套件系统名称>
```

### 实际应用场景示例

以群晖的核心应用服务 `SynologyApplicationService` 为例。如果您想知道系统中哪些应用依赖它运行，可以执行：

```bash
synopkg list --name --depend-on SynologyApplicationService
```

终端将返回依赖于该服务的应用列表（基于 DSM 7.x 及以上版本的典型返回结果）：
```text
SynologyOffice
SynologyPhotos
SynologyDrive
Chat
```
*注：返回结果会根据您 NAS 上实际安装的套件动态变化。*

### 进阶备忘录（可扩展区域）

*这里为您预留了扩展空间，方便后续记录更多实用的 `synopkg` 相关命令：*

* **查看特定套件的详细运行状态**
  ```bash
  synopkg status <套件系统名称>
  ```
* **手动启动/停止特定套件** *(可能需要 sudo 权限)*
  ```bash
  sudo synopkg start <套件系统名称>
  sudo synopkg stop <套件系统名称>
  ```

