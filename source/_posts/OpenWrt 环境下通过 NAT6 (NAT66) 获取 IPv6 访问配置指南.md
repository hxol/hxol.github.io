---
title: OpenWrt 环境下通过 NAT6 (NAT66) 获取 IPv6 访问配置指南
date: 2026-10-02 18:25:09
tags: [笔记, OpenWrt, NAT66, IPv6]
---



# OpenWrt 环境下通过 NAT6 (NAT66) 获取 IPv6 访问配置指南

**前言与机制说明**
在默认情况下，IPv6 网络推荐通过前缀授权（DHCPv6-PD）为内网设备分配公网地址。但在某些场景下（如上游未下发可用的公网前缀、校园网限制设备数量、或是使用 ULA 本地地址时），我们需要借助 IPv6 伪装（NAT66），让局域网设备共享路由器的 IPv6 地址访问公网。

> **版本提示**：自 OpenWrt 22.03 起，系统防火墙已全面升级为基于 `nftables` 的 `fw4`，原生完美支持 IPv6 网络地址转换，**不再需要**像旧版本那样手动安装 `kmod-ipt-nat6` 及依赖包或编写复杂的 iptables 脚本。

---

#### 基础模块：放行上游接口的源地址过滤

无论您使用哪种 NAT6 规则，当内网使用本地局域网前缀（如以 `fd` 开头的 ULA 地址），而上游无法提供全局单播前缀时，OpenWrt 默认的安全路由策略会拦截并丢弃这些“未被授权”的出站数据包。

因此，配置 NAT 的第一步是禁用 WAN6 接口的源地址过滤：

```bash
uci set network.wan6.sourcefilter="0"
uci commit network

# 相比重启整个路由器，仅重载网络服务更为优雅和高效
service network restart
```

---

#### 方案 A：启用全局 IPv6 伪装（全局 NAT66）

如果您希望局域网内的**所有**设备都能通过 WAN 口的 IPv6 地址进行伪装上网，这是最快捷的配置方式。直接在防火墙的出站区域（WAN 区域）开启 `masq6` 即可：

```bash
# 注意：@zone[1] 默认通常对应系统自带的 wan 区域。如果您的区域有增删，请确保索引或名称正确
uci set firewall.@zone[1].masq6="1"
uci commit firewall
service firewall restart
```

---

#### 方案 B：配置 IPv6 选择性伪装（策略 NAT66）

如果您只希望对特定的局域网子网（例如访客网络或特定 VLAN）启用 IPv6 伪装，而其他子网保持原生路由，可以使用更精准的自定义 NAT 规则替代全局伪装。

> **参数解析**：在 OpenWrt 的 NAT 规则中，`src="wan"` 指代的是出站（POSTROUTING）时应用的区域，而 `src_ip` 则是需要被匹配转换的局域网源 IP 段。

```bash
# 清理可能存在的同名旧规则，防止冲突
uci -q delete firewall.nat6

# 建立新的 IPv6 专用伪装规则
uci set firewall.nat6="nat"
uci set firewall.nat6.name="Custom_NAT6"  # 为规则命名，方便在 LuCI 界面中辨识
uci set firewall.nat6.family="ipv6"
uci set firewall.nat6.proto="all"
uci set firewall.nat6.src="wan"

# [修改提示] 将下方地址替换为您实际需要进行 NAT 的局域网 IPv6 前缀（如 fd12:3456:7890::/64）
uci set firewall.nat6.src_ip="<你的自定义IPv6子网前缀>" 
uci set firewall.nat6.target="MASQUERADE"

uci commit firewall
service firewall restart
```

---

#### 维护与验证

*   **状态验证**：配置生效后，终端设备可访问诸如 `test-ipv6.com` 等测试网站，若显示路由器的 WAN 口 IPv6 地址，即说明 NAT6 转换成功。
*   **规则排查**：在较新的 OpenWrt 系统中排查防火墙时，请使用 `nft list ruleset` 命令来检视底层规则，这已取代了过时的 `ip6tables` 命令。
*   **按需切换**：方案 A 与方案 B 根据业务需求二选一即可。若需要调整策略，只需复原 `masq6="0"` 或删除对应的 `firewall.nat6` 节点并重启防火墙（`service firewall restart`）。