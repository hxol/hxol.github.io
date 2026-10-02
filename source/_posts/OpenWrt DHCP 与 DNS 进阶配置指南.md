---
title: OpenWrt DHCP 与 DNS 进阶配置指南
date: 2026-10-02 18:25:10
tags: [笔记, OpenWrt, DHCP, DNS, Dnsmasq]
---



# OpenWrt DHCP 与 DNS 进阶配置指南

本指南记录了在 OpenWrt 系统中自定义局域网设备的网关、DNS 派发规则，以及设置内网域名解析的常用方法。支持通过 Web 界面（LuCI）和命令行（CLI）两种方式进行配置。

## 核心概念：DHCP 选项 (DHCP Options)
OpenWrt 使用 Dnsmasq 来提供 DHCP 和 DNS 服务。在 DHCP 协议中，不同的网络参数由不同的“选项号”代表：
*   **选项 3 (Option 3)**：代表**默认网关**。
*   **选项 6 (Option 6)**：代表**DNS 服务器**。

---

## 模块一：自定义下发的网关与 DNS

当你网络中存在旁路由，或者希望设备使用特定的 DNS 服务器（如去广告 DNS）时，需要修改此项。

### 方式 A：通过 Web 界面 (LuCI) 配置
进入 OpenWrt 后台，导航至 **网络 -> 接口 -> LAN -> 编辑**。
在界面下方找到 **DHCP 服务器 -> 高级设置 -> DHCP 选项**，在空白框中填入你需要的规则并保存应用。

*   **修改自定义网关**
    格式为 `3,网关IP`。
    示例：将网关指向旁路由 `3,192.168.1.253`
*   **修改自定义 DNS**
    格式为 `6,DNS地址`。如果需要下发多个 DNS 备用，可以直接使用逗号分隔。
    示例：下发公共 DNS 服务器 `6,8.8.8.8,114.114.114.114`

### 方式 B：通过命令行 (UCI) 配置
通过 SSH 登录 OpenWrt，直接运行以下命令修改配置。此方式适合批量部署或编写脚本。

```bash
# 添加自定义网关
uci add_list dhcp.lan.dhcp_option='3,192.168.1.253'

# 添加自定义 DNS
uci add_list dhcp.lan.dhcp_option='6,8.8.8.8,114.114.114.114'

# 保存并应用 DHCP 配置
uci commit dhcp
/etc/init.d/dnsmasq restart
```
> **排错提示**：如果发现某些设备仍然优先获取了 IPv6 的 DNS，导致 IPv4 自定义 DNS 未生效，可以酌情删除 IPv6 的 SLAAC 宣告机制（仅限不需要纯正 IPv6 环境的情况）：
> `uci del dhcp.lan.ra_slaac`
> `uci commit dhcp`

---

## 模块二：内网自定义域名解析 (DNS 劫持/映射)

将特定的域名直接解析到局域网内的某台设备上（例如将 NAS 或内网服务器绑定一个好记的域名，无需公网解析即可在局域网内访问）。

### 方式 A：通过 Web 界面 (LuCI) 配置
进入 OpenWrt 后台，导航至 **网络 -> DHCP/DNS**。
这里有两种图形化添加方式（任选其一即可）：
*   **自定义域名 (Hostnames)**：在底部找到“主机名”选项卡，直接添加“主机名（如 `nas.local`）”与对应的“IP 地址（如 `192.168.1.100`）”。
*   **地址配置 (Addresses)**：在“常规设置”下的“地址”框中，使用 Dnsmasq 的原生语法格式填入：`/nas.local/192.168.1.100`。

配置完成后，点击底部的“保存并应用”即可立即生效。

### 方式 B：通过命令行配置
修改 Dnsmasq 的配置文件或利用 UCI 引擎动态添加：

```bash
# 进入自定义域名配置文件
vi /etc/dnsmasq.conf

# 在文件末尾追加映射关系，格式为 address=/域名/IP
address=/nas.local/192.168.1.100

# 重启 dnsmasq 服务使解析生效 (无需重启整台路由器)
/etc/init.d/dnsmasq restart
```

---

## 附录：日常维护
*   **服务重启建议**：涉及到网络参数的修改，**通常只需重启 `dnsmasq` 服务或 `network` 服务即可生效，完全不需要重启整台路由器**。这样不仅速度快，也不会中断其他无关业务的运行。
