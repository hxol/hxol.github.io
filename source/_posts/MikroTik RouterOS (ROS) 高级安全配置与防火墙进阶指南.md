---
title: MikroTik RouterOS (ROS) 高级安全配置与防火墙进阶指南
date: 2026-10-01 15:30:31
tags: [笔记, MikroTik, RouterOS, 路由器]
---

# MikroTik RouterOS (ROS) 高级安全配置与防火墙进阶指南

> **前言**：
> * 本文基于 RouterOS v7 语法编写（大部分向下兼容 v6）。
> * **核心原则**：防火墙配置应遵循**“RAW 表拦截攻击，Filter 表控制访问”**的原则。能用 RAW 表丢弃的垃圾流量绝不进入 Filter 表，以最大限度减轻 CPU 和连接跟踪（Conntrack）的负担。
> * **操作建议**：强烈推荐使用 CLI（命令行）进行配置，效率更高且不易遗漏。

---

## 第一部分：系统初始化与安全加固

### 1. 系统重置与恢复
* **软复位（命令行）**：不加载默认配置且不备份（空白开局）
  ```routeros
  /system reset-configuration no-defaults=yes skip-backup=yes
  ```
* **硬复位（物理按钮）**：断电 -> 按住 Reset 键 -> 通电。观察 LED 灯：
  * **闪烁**：重置配置（最常用）。
  * **常亮**（约 5 秒后）：进入 CAPs 模式（无线 AP 受控模式）。
  * **熄灭**（约 20 秒后）：进入 Netinstall 模式（线刷救砖）。

### 2. 基础环境与服务加固
在配置网络前，先锁死路由器的管理权限。
```routeros
# 1. 创建新管理员并删除默认 admin 账户
/user add name=myadmin group=full password="YourStrongPasswordHere"
/user remove admin

# 2. 禁用不必要的服务 (仅保留 SSH 和 Winbox)
/ip service disable telnet,ftp,www,api,api-ssl,www-ssl

# 3. 修改高危端口
/ip service set ssh port=22022
/ip service set winbox port=8291

# 4. 限制管理服务仅限内网访问 (假设内网为 192.168.88.0/24)
/ip service set winbox address=192.168.88.0/24
/ip service set ssh address=192.168.88.0/24

# 5. 定义接口列表 (Interface Lists) - 这是后续防火墙的基础
/interface list add name=WAN
/interface list add name=LAN
/interface list member add list=WAN interface=pppoe-out1
/interface list member add list=LAN interface=bridge1

# 6. MAC 登录限制与邻居发现限制 (禁止外网扫描和 MAC 登录)
/tool mac-server set allowed-interface-list=LAN
/tool mac-server mac-winbox set allowed-interface-list=LAN
/ip neighbor discovery-settings set discover-interface-list=LAN
```

---

## 第二部分：IPv6 与网络高级配置

### 1. 强制 Cloud DDNS 仅解析 IPv6
如果您的宽带没有公网 IPv4，只有公网 IPv6，为了让 ROS 自带的 DDNS 正确工作，可以通过 DNS 劫持将官方域名的 IPv4 解析作废。
* **方法**：在 DNS 静态解析中，将 `cloud2.mikrotik.com` 的 IPv4 设置为本地回环。
  ```routeros
  /ip dns static add name=cloud2.mikrotik.com address=127.0.0.1
  ```

### 2. IPv6 部署的三种方案
* **方案 A：标准家庭宽带 (SLAAC)** —— 适用于 ROS 下挂终端直接上网。
  ```routeros
  /ipv6 dhcp-client add interface=pppoe-out1 request=prefix pool-name=ipv6-pool pool-prefix-length=60 add-default-route=yes
  /ipv6 address add address=::1 from-pool=ipv6-pool interface=bridge1 advertise=yes
  ```
* **方案 B：二级路由专用** —— ROS 仅下发前缀，不分配网关地址，避免下级 OpenWrt 出现双网关。
  ```routeros
  /ipv6 address add address=::1 from-pool=ipv6-pool interface=bridge1 advertise=no
  /ipv6 dhcp-server add name=server1 interface=bridge1 address-pool=ipv6-pool lease-time=1d
  ```
* **方案 C：NAT66** —— 适用于运营商仅提供 `/64` 前缀，无法向下委派的情况。
  ```routeros
  /ipv6 pool add name=lan-private-pool prefix=fd00:192:168:1::/64 prefix-length=64
  /ipv6 address add address=fd00:192:168:1::1 interface=bridge1 from-pool=lan-private-pool advertise=yes
  /ipv6 firewall nat add chain=srcnat out-interface-list=WAN action=masquerade
  ```

---

## 第三部分：高级防火墙构建 (IPv4 & IPv6)

> **防火墙基本原则 (Default Drop)：**
> * **Input (进入路由器)**: 允许已建立/相关连接 -> 允许管理员/内网 -> 拒绝其他所有。
> * **Forward (穿过路由器)**: 允许内网发起连接 -> 允许已建立连接 -> 拒绝其他所有。
> * **Output (路由器发出)**: 默认全放行。

### 1. 准备地址列表 (Address Lists)
集中管理保留 IP、内网 IP 和公网 DDNS。
```routeros
# IPv4 列表
/ip firewall address-list
add list=lan_addresses address=192.168.88.0/24
add list=admin_access address=192.168.88.100-192.168.88.200
add list=my_public_ipv4 address=你的域名.sn.mynetname.net comment="DDNS 公网 IPv4"
# Bogon IP (非法公网地址 / RFC6890)
add list=bogon_ipv4 address=0.0.0.0/8
add list=bogon_ipv4 address=10.0.0.0/8
add list=bogon_ipv4 address=100.64.0.0/10
add list=bogon_ipv4 address=127.0.0.0/8
add list=bogon_ipv4 address=169.254.0.0/16
add list=bogon_ipv4 address=172.16.0.0/12
add list=bogon_ipv4 address=192.168.0.0/16
add list=bogon_ipv4 address=224.0.0.0/4

# IPv6 列表
/ipv6 firewall address-list
add list=my_public_ipv6 address=你的域名.sn.mynetname.net comment="DDNS 公网 IPv6"
add list=bogon_ipv6 address=::/128
add list=bogon_ipv6 address=::1/128
add list=bogon_ipv6 address=fc00::/7
add list=bogon_ipv6 address=fe80::/10
```

### 2. RAW 表过滤 (抵御基础网络攻击)
在 Prerouting 链直接丢弃垃圾数据包，极大降低 CPU 负载。
```routeros
# IPv4 RAW 规则
/ip firewall raw
add action=drop chain=prerouting comment="丢弃源或目的为 Bogon IP 的流量" src-address-list=bogon_ipv4 in-interface-list=WAN
add action=drop chain=prerouting dst-address-list=bogon_ipv4 in-interface-list=WAN
add action=drop chain=prerouting protocol=tcp tcp-flags=!fin,!syn,!rst,!ack comment="丢弃无效 TCP 标志"
add action=drop chain=prerouting port=0 protocol=tcp comment="丢弃 TCP 0 端口"
add action=drop chain=prerouting port=0 protocol=udp comment="丢弃 UDP 0 端口"

# IPv6 RAW 规则
/ipv6 firewall raw
add action=accept chain=prerouting protocol=icmpv6 comment="放行基础 ICMPv6"
add action=drop chain=prerouting src-address-list=bogon_ipv6 in-interface-list=WAN comment="丢弃无效 IPv6 源"
add action=drop chain=prerouting dst-address=ff00::/8 comment="丢弃非本地多播"
```

### 3. PSD 端口扫描防护 (RAW 表)
识别恶意扫描器并拉黑，必须放在 RAW 表且顺序为“先拦截后识别”。
```routeros
/ip firewall raw
# 1. 直接丢弃已识别的扫描器 IP
add chain=prerouting src-address-list=port_scanners action=drop comment="Drop detected scanners"

# 2. 识别扫描行为并加入黑名单 (21分阈值，3秒统计周期)
add chain=prerouting protocol=tcp psd=21,3s,3,1 in-interface-list=WAN action=add-src-to-address-list address-list=port_scanners address-list-timeout=2w comment="Detect Port Scans"
```

### 4. Filter 表：保护路由器自身 (Input 链)
```routeros
/ip firewall filter
add action=accept chain=input connection-state=established,related,untracked comment="放行已建立的连接"
add action=drop chain=input connection-state=invalid comment="丢弃无效连接"
add action=accept chain=input protocol=icmp comment="允许 Ping"
add action=accept chain=input src-address-list=admin_access comment="允许管理员 IP 完全访问"
add action=drop chain=input in-interface-list=!LAN comment="拦截非局域网的所有其他请求"
```

### 5. Filter 表：保护内网设备 (Forward 链)
```routeros
/ip firewall filter
add action=fasttrack-connection chain=forward connection-state=established,related comment="FastTrack 硬件加速"
add action=accept chain=forward connection-state=established,related,untracked comment="放行已建立的连接"
add action=drop chain=forward connection-state=invalid comment="丢弃无效连接"
# IPsec 用户务必添加此规则绕过 NAT/Drop 限制：
# add action=accept chain=forward ipsec-policy=in,ipsec comment="允许 IPsec 流量"
add action=drop chain=forward connection-nat-state=!dstnat connection-state=new in-interface-list=WAN comment="拦截所有来自外网且未经端口映射的新请求"
```

---

## 第四部分：高级安全策略防护

### 1. SSH 暴力破解防御阶梯
通过记录连接频率，将恶意爆破 IP 封禁。以下规则允许每 5 分钟尝试 3 次，超过则封禁 1 天。
```routeros
/ip firewall filter
# 4. 彻底拉黑在黑名单中的 IP
add action=drop chain=input src-address-list=ssh_blacklist comment="拦截 SSH 爆破黑名单"

# 3. 第 3 次新连接 -> 加入彻底黑名单 (1天)
add action=add-src-to-address-list address-list=ssh_blacklist address-list-timeout=1d chain=input connection-state=new dst-port=22022 protocol=tcp src-address-list=ssh_stage2 comment="Stage 3: Blacklist"

# 2. 第 2 次新连接 -> 加入 Stage 2 列表 (保存 15 分钟)
add action=add-src-to-address-list address-list=ssh_stage2 address-list-timeout=15m chain=input connection-state=new dst-port=22022 protocol=tcp src-address-list=ssh_stage1 comment="Stage 2"

# 1. 首次新连接 -> 加入 Stage 1 列表 (保存 5 分钟)
add action=add-src-to-address-list address-list=ssh_stage1 address-list-timeout=5m chain=input connection-state=new dst-port=22022 protocol=tcp comment="Stage 1"

# 放行正常 SSH 请求
add action=accept chain=input dst-port=22022 protocol=tcp src-address-list=!ssh_blacklist
```

### 2. Port Knocking (端口敲门)
隐藏真实服务端口。只有依次访问特定端口（如 `888 -> 555 -> 222`），真实服务端口才会向该 IP 开放。

* **配置敲门规则：**
```routeros
/ip firewall filter
# 敲门步骤 1
add action=add-src-to-address-list address-list=knock_1 address-list-timeout=30s chain=input dst-port=888 in-interface-list=WAN protocol=tcp
# 敲门步骤 2 (必须已完成步骤 1)
add action=add-src-to-address-list address-list=knock_2 address-list-timeout=30s chain=input dst-port=555 in-interface-list=WAN protocol=tcp src-address-list=knock_1
# 敲门步骤 3 -> 获得放行许可 (授权 30 分钟)
add action=add-src-to-address-list address-list=secured_ip address-list-timeout=30m chain=input dst-port=222 in-interface-list=WAN protocol=tcp src-address-list=knock_2

# 最终放行规则 (将此规则置于 Drop All 之前)
add action=accept chain=input in-interface-list=WAN src-address-list=secured_ip comment="允许敲门成功的 IP 访问"
```
* **客户端敲门命令 (Linux/macOS)**:
  `for x in 888 555 222; do nmap -p $x -Pn 你的公网IP; done`

---

## 第五部分：实用网络控制与 NAT 场景

### 1. NAT 端口转发与内网回流 (Hairpin NAT)
解决局域网内设备无法通过公网域名访问内部服务器的问题。
```routeros
/ip firewall nat
# 1. 端口映射 (使用公网 IP 列表而不是 IN-Interface，兼容内外网访问)
add chain=dstnat protocol=tcp dst-port=80 dst-address-list=my_public_ipv4 action=dst-nat to-addresses=192.168.88.200 to-ports=80 comment="Web 服务器端口映射"

# 2. 内网回流 (Hairpin NAT) - 源地址必须伪装，否则回包无法原路返回
add chain=srcnat src-address=192.168.88.0/24 dst-address=192.168.88.200 protocol=tcp dst-port=80 out-interface=bridge1 action=masquerade comment="内网回流"
```

### 2. 两个 IP 网段单向访问控制
**需求**：`192.168.60.0/24` (管理网) 能访问 `192.168.40.0/24` (访客网)，但访客网不能主动访问管理网。
```routeros
/ip firewall filter
# 1. 维护连接状态：只要是由特权网段发起的连接，后续回包全部放行 (关键)
add action=accept chain=forward connection-state=established,related comment="放行已建立的连接"

# 2. 允许特权网段发起所有请求
add action=accept chain=forward src-address=192.168.60.0/24 comment="允许特权网段发起请求"

# 3. 拦截受限网段主动发起的新连接请求
add action=drop chain=forward connection-state=new src-address=192.168.40.0/24 dst-address=192.168.60.0/24 comment="禁止受限网段主动发起访问"
```

### 3. DHCP 静态指定特定设备的网关/DNS (旁路由场景)
**需求**：让电视盒子走旁路由 (`192.168.88.2`) 代理，其余设备走主路由 (`192.168.88.1`)。
```routeros
# 1. 定义 DHCP Options (Code 3=网关, Code 6=DNS。注意 IP 需要被单引号包裹)
/ip dhcp-server option
add code=3 name=gateway_bypass value="'192.168.88.2'"
add code=6 name=dns_bypass value="'192.168.88.2'"

# 2. 在 IP -> DHCP Server -> Leases 中将该设备设为 Static (静态)
# 然后双击该条目，在 DHCP Options 字段选择 gateway_bypass 和 dns_bypass 即可生效。
```

*(提示：如果您配置了 WireGuard 接口，可以将其直接加入 LAN 接口列表（`/interface list member add list=LAN interface=wireguard1`），即可与内网互通，免去复杂的 NAT 配置。)*