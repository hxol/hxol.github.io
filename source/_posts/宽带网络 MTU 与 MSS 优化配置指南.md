---
title: 宽带网络 MTU 与 MSS 优化配置指南
date: 2026-10-01 15:30:32
tags: [笔记, MikroTik, RouterOS, 路由器, MTU, MSS, WireGuard]
---

# 宽带网络 MTU 与 MSS 优化配置指南 (WireGuard / Phantun / ROS)

在配置隧道（如 WireGuard）或网络混淆工具（如 Phantun）时，合理设置 MTU（最大传输单元）和 MSS（最大报文段长度）是避免网络卡顿、断流和提高传输效率的关键。

## 一、 WireGuard 基础 MTU 计算 (最保守/标准环境)

WireGuard 的默认开销为：**IPv4 下 60 字节**（20 IP + 8 UDP + 32 WG），**IPv6 下 80 字节**（40 IP + 8 UDP + 32 WG）。
根据外层网络的不同，端点（Endpoint）的安全 MTU 计算如下：

*   **标准以太网 (MTU 1500)**
    *   IPv4 Endpoint MTU：`1440` (1500 - 60)
    *   IPv6 Endpoint MTU：`1420` (1500 - 80)
*   **普通 PPPoE 宽带 (MTU 1492，相比标准以太网减少 8 字节)**
    *   IPv4 Endpoint MTU：`1432` (1440 - 8)
    *   IPv6 Endpoint MTU：`1412` (1420 - 8)

---

## 二、 复杂网络环境下的 MTU 推荐值

结合不同的网络线路及是否套用 TCP 伪装工具（如 Phantun），推荐使用以下 MTU 值：
> **💡 补充说明**：Phantun 会将 UDP 流量伪装成 TCP，而 TCP 头部（20字节）比 UDP 头部（8字节）多出 12 字节开销，因此套用 Phantun 后，MTU 需要额外减去 12。

| 网络环境 | 推荐 MTU | 备注说明 |
| :--- | :--- | :--- |
| **正常普通 PPPoE** | `1412` | 基于 IPv6 PPPoE 的标准 WireGuard 安全值。 |
| **普通 PPPoE 套 Phantun** | `1400` | `1412 - 12` (TCP 与 UDP 的头部差值)。 |
| **精品网 (如 CN2 等)** | `1362` | 部分精品网由于运营商底层存在额外封装（如 VLAN/QoS），需进一步降低。 |
| **精品网 套 Phantun** | `1350` | `1362 - 12`。 |
| **极致保守 / 强迫症推荐** | `1344` 或 `1400`| `1280 + 32 + 32 = 1344`。整数对齐且几乎兼容世上所有劣质网络环境。 |
| **纯 IPv6 WireGuard 环境** | `1280` | `1280` 是 IPv6 协议规定的最低合法 MTU，能确保 100% 不发生分片丢失。 |

---

## 三、 RouterOS (ROS) 自动修改 MSS 设置

在路由器上配置隧道后，由于 MTU 缩小，若不配合修改 TCP MSS（MSS Clamping），可能会导致部分网页打不开或加载缓慢。

> **💡 MSS 计算公式**：
> * IPv4 MSS = MTU - 40 字节 (20 IP + 20 TCP)
> * IPv6 MSS = MTU - 60 字节 (40 IP + 20 TCP)

以下是 RouterOS 的 Firewall Mangle 规则，用于在 WAN 口（以 `pppoe-out1` 为例）自动钳制 TCP SYN 包的 MSS 值。请根据你实际的 WAN 口名称和所需 MSS 值进行替换。

### 1. IPv6 环境
将 IPv6 的 MSS 限制为 `1280`（非常保守且安全的值，能穿透极端的 IPv6 封装）：
```routeros
/ipv6 firewall mangle 
add chain=forward out-interface=pppoe-out1 protocol=tcp tcp-flags=syn action=change-mss new-mss=1280 comment="Clamp MSS to 1280 for IPv6"
```

### 2. IPv4 环境
将 IPv4 的 MSS 限制为 `1344`（对应 MTU 约为 1384，兼容绝大多数套了多层协议的代理或 VPN）：
```routeros
/ip firewall mangle 
add chain=forward out-interface=pppoe-out1 protocol=tcp tcp-flags=syn action=change-mss new-mss=1344 comment="Clamp MSS to 1344 for IPv4"
```

*(注：如果你希望 ROS 自动计算 MSS，也可以将 `action=change-mss` 搭配 `new-mss=clamp-to-pmtu` 使用，但在某些伪装隧道下，手动指定如 1280 或 1344 这样的固定保守值效果往往更稳定。)*

