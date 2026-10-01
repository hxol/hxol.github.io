---
title: 从 Debian 12 (Bookworm) 升级到 Debian 13 (Trixie) 的具体步骤
date: 2026-10-01 15:30:18
tags: [笔记, Linux, Debian]
---

# 从 Debian 12 (Bookworm) 升级到 Debian 13 (Trixie) 的具体步骤

⚠️ **重要警告**：
1.  **备份数据**：虽然升级过程通常很稳定，但如文档 4.1.1 所述，强烈建议在操作前对云服务器进行**快照**或全量备份。
2.  **防止断连**：如文档 4.1.5 所述，如果你是通过 SSH 远程升级，建议使用 `tmux` 或 `screen` 会话，防止网络中断导致升级过程中途挂起。

---

## 第一步：确保当前系统已是最新状态 (文档 4.2.2)

在开始升级大版本前，必须确保你当前的 Debian 12 已经安装了所有补丁。

```bash
apt update
apt upgrade
apt autoremove
```

检查是否有软件包被“保持”（Hold）住，如果有，必须先处理掉，否则会导致升级失败（文档 4.2.12）：
```bash
apt-mark showhold
# 如果没有任何输出，说明一切正常。
```

---

## 第二步：修改软件源 (文档 4.3)

虽然文档推荐使用新的 `deb822` 格式（`.sources` 文件），但为了简便和兼容你当前的云环境，直接修改现有的 `/etc/apt/sources.list` 是最稳妥的。

1.  **备份当前的源文件**：
    ```bash
    cp /etc/apt/sources.list /etc/apt/sources.list.bak
    ```

2.  **修改源文件**：
    你可以手动编辑文件，将所有的 `bookworm` 替换为 `trixie`。
    或者使用以下 `sed` 命令一键替换（请确保执行前已备份）：

    ```bash
    sed -i 's/bookworm/trixie/g' /etc/apt/sources.list
    ```

    **注意**：有些云厂商的配置中，`sources.list.d/` 目录下可能也有文件，也需要检查一下：
    ```bash
    sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/*.list
    ```

3.  **检查修改结果**：
    ```bash
    cat /etc/apt/sources.list
    ```
    确认输出中的 URL 依然是 `mirrors.tencentyun.com`，但代号都变成了 `trixie`（对于安全更新可能是 `trixie-security`）。

---

## 第三步：更新软件包列表 (文档 4.4.2)

让 APT 获取 Debian 13 (Trixie) 的软件包信息：

```bash
apt update
```
此时你应该看到所有源都已指向 `trixie`。

---

## 第四步：最小化系统升级 (文档 4.4.5)

**这是关键的一步。** 不要直接进行完整升级，先进行最小化升级，以避免一次性删除太多软件包导致冲突。

```bash
apt upgrade --without-new-pkgs
```
*   如果系统询问是否重启服务（Restart services...），选择 **Yes**。
*   如果遇到配置文件冲突（询问是保留旧配置还是使用维护者的新配置），通常建议：
    *   如果你没改过该文件，输入 `Y`（使用新版）。
    *   如果你改过，可以输入 `D` 查看差异，或者选 `N` 保留你的版本（如果不确定，通常保留旧版本比较安全，以后再手动合并）。

---

## 第五步：执行完整系统升级 (文档 4.4.6)

完成最小化升级后，执行完整的版本升级。这将升级内核并解决复杂的依赖关系。

```bash
apt full-upgrade
```

*   这个过程可能需要一些时间，取决于你的网络和磁盘速度。
*   期间可能会再次询问配置文件覆盖的问题，请按需选择。
*   如果报错提示 `Could not perform immediate configuration`，请尝试运行：`apt full-upgrade -o APT::Immediate-Configure=0` (文档 4.5.1)。

---

## 第六步：重启并验证 (文档 4.6)

升级完成后，必须重启以加载新内核（Linux Kernel 6.x）。

1.  重启系统：
    ```bash
    systemctl reboot
    ```

2.  重新连接 SSH，验证版本：
    ```bash
    cat /etc/debian_version
    # 输出应该显示 13.x 或 trixie/sid
    ```

3.  验证内核：
    ```bash
    uname -r
    ```

---

## 第七步：清理 (文档 4.7 & 4.8)

新系统稳定运行后，清理不再需要的旧软件包和依赖：

```bash
apt autoremove
apt clean
```

如果你发现有不再维护的旧包（Obsolete），可以使用 `apt list '?obsolete'` 查看并手动决定是否清除。
```
apt purge '?obsolete'
```