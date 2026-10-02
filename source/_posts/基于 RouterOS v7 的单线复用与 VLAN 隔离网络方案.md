---
title: 基于 RouterOS v7 的单线复用与 VLAN 隔离网络方案
date: 2026-10-02 13:01:38
tags: [笔记, RouterOS, VLAN, 路由器]
---

# 基于 RouterOS v7 的单线复用与 VLAN 隔离网络方案

## 概况与需求分析

*   **核心路由**：MikroTik RB750Gr3 (系统版本：RouterOS v7.20.6 或以上)
*   **物理接口**：共 5 个以太网口。`ether1` 为 WAN 口；`ether2` 至 `ether5` 归属于同一个 `bridge` 作为 LAN 口使用（网段 `192.168.8.0/24`）。
*   **网络拓扑**：`ether2` 作为 Trunk 端口，下接一台 8 端口网管型交换机（交换机 A）。交换机 A 还可级联其他交换机（如交换机 B）。

**VLAN 规划：**
规划在 `ether2` 上划分 5 个独立的 VLAN，配合交换机实现物理端口的隔离分配：
*   **VLAN 20** (192.168.20.0/24)：访客/非授信网络 (Guest / Untrusted)
*   **VLAN 30** (192.168.30.0/24)：专属 WiFi 网段 (Private WiFi)
*   **VLAN 40** (192.168.40.0/24)：安防监控网络 (NVR / IPCam)
*   **VLAN 50** (192.168.50.0/24)：智能家居物联网 (IoT)
*   **VLAN 60** (192.168.60.0/24)：核心安全网络 (Core Network)

**安全隔离需求：**
*   各 VLAN 网段之间完全物理隔离，禁止互相通信。
*   各 VLAN 均不可访问路由器主局域网 LAN (`192.168.8.0/24`)。
*   所有网段（LAN 及所有 VLAN）均可正常访问公共互联网。

---

## 实施准备与核心思路

RB750Gr3 搭载硬件交换芯片，配合 RouterOS v7 的 **Bridge VLAN Filtering（网桥 VLAN 过滤）** 功能，可以在保持高性能硬件转发的同时实现灵活的 VLAN 划分。

**⚠️ 关键安全提示 (Safe Mode)：**
在进行网桥和 VLAN 过滤配置时，极易因配置不当导致与路由器的管理连接中断。**请务必在 Winbox 左上角开启 `Safe Mode` (安全模式)。**
若操作失误导致断连，系统会在超时后自动回滚上一步配置；若一切正常，请在配置完成后点击撤销安全模式以保存更改。另外，建议通过 `ether3`、`ether4` 或 `ether5` 进行管理操作，避开即将被配置为 Trunk 的 `ether2`。

---

## RouterOS 核心配置指南

### 建立虚拟接口与寻址
VLAN 接口必须依附于 `bridge`，而不是直接建立在物理网口上。创建后，需为每个虚拟接口分配网关 IP。

**创建 VLAN 接口：**
```routeros
/interface vlan
add interface=bridge name=vlan20 vlan-id=20
add interface=bridge name=vlan30 vlan-id=30
add interface=bridge name=vlan40 vlan-id=40
add interface=bridge name=vlan50 vlan-id=50
add interface=bridge name=vlan60 vlan-id=60
```

**配置网关 IP 地址：**
```routeros
/ip address
add address=192.168.20.1/24 interface=vlan20
add address=192.168.30.1/24 interface=vlan30
add address=192.168.40.1/24 interface=vlan40
add address=192.168.50.1/24 interface=vlan50
add address=192.168.60.1/24 interface=vlan60
```

### 部署 DHCP 服务
为各个 VLAN 建立地址池并配置 DHCP Server 发放 IP。
*(注：如果不习惯命令行，这一步在 Winbox 中使用 `IP -> DHCP Server -> DHCP Setup` 向导配置会更不易出错)*

```routeros
# 定义 IP 地址池
/ip pool
add name=pool_vlan20 ranges=192.168.20.10-192.168.20.254
add name=pool_vlan30 ranges=192.168.30.10-192.168.30.254
add name=pool_vlan40 ranges=192.168.40.10-192.168.40.254
add name=pool_vlan50 ranges=192.168.50.10-192.168.50.254
add name=pool_vlan60 ranges=192.168.60.10-192.168.60.254

# 配置 DHCP 网络参数 (网关与 DNS)
/ip dhcp-server network
add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=192.168.20.1
add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=192.168.30.1
add address=192.168.40.0/24 gateway=192.168.40.1 dns-server=192.168.40.1
add address=192.168.50.0/24 gateway=192.168.50.1 dns-server=192.168.50.1
add address=192.168.60.0/24 gateway=192.168.60.1 dns-server=192.168.60.1

# 绑定 DHCP 服务到对应接口
/ip dhcp-server
add name=dhcp_vlan20 interface=vlan20 address-pool=pool_vlan20 disabled=no
add name=dhcp_vlan30 interface=vlan30 address-pool=pool_vlan30 disabled=no
add name=dhcp_vlan40 interface=vlan40 address-pool=pool_vlan40 disabled=no
add name=dhcp_vlan50 interface=vlan50 address-pool=pool_vlan50 disabled=no
add name=dhcp_vlan60 interface=vlan60 address-pool=pool_vlan60 disabled=no
```

### 配置网桥 VLAN 过滤
这是数据包正确打标签（Tag）的核心逻辑：
*   **bridge 本身**：必须包含在 Tagged 中，以便 CPU 参与这些 VLAN 的路由。
*   **ether2 (下联交换机)**：作为 Trunk 端口，必须包含在 Tagged 中透传所有 VLAN 数据。
*   **ether3,4,5**：保持默认配置，作为主 LAN 的 Untagged 端口。

```routeros
/interface bridge vlan
add bridge=bridge tagged=bridge,ether2 vlan-ids=20
add bridge=bridge tagged=bridge,ether2 vlan-ids=30
add bridge=bridge tagged=bridge,ether2 vlan-ids=40
add bridge=bridge tagged=bridge,ether2 vlan-ids=50
add bridge=bridge tagged=bridge,ether2 vlan-ids=60

# 开启网桥 VLAN 过滤（执行此命令前，请确保不在 ether2 端口上操作，并已开启安全模式）
/interface bridge set bridge vlan-filtering=yes
```

### 配置高扩展性防火墙策略
RouterOS 默认路由互通，必须通过防火墙规则阻断。为了保证**极高的可扩展性**，我们利用**接口列表 (Interface List)** 来管理策略。日后新增 VLAN，只需将其加入列表即可，无需修改任何防火墙规则。

**定义接口列表：**
```routeros
# 创建 VLAN 集合列表
/interface list add name=VLAN_GUESTS

# 将所有 VLAN 批量加入该列表
/interface list member
add interface=vlan20 list=VLAN_GUESTS
add interface=vlan30 list=VLAN_GUESTS
add interface=vlan40 list=VLAN_GUESTS
add interface=vlan50 list=VLAN_GUESTS
add interface=vlan60 list=VLAN_GUESTS

# 确保默认的 LAN 和 WAN 列表已存在并包含对应接口 (按需检查)
/interface list add name=LAN
/interface list add name=WAN
/interface list member add interface=bridge list=LAN
/interface list member add interface=ether1 list=WAN
```

**应用防火墙规则：**
*注意：在 RouterOS v7 中，新建规则默认在最底部。请在 Winbox 的 `IP -> Firewall` 中，将下列允许 DNS/DHCP/ICMP 的规则拖动到 `defconf: drop all not coming from LAN` 之前。*

```routeros
# 1. 允许 VLAN 设备向路由器请求 IP (DHCP) 和 DNS 解析
/ip firewall filter 
add chain=input action=accept protocol=udp dst-port=53,67 in-interface-list=VLAN_GUESTS comment="Allow VLAN DNS & DHCP"
add chain=input action=accept protocol=tcp dst-port=53 in-interface-list=VLAN_GUESTS comment="Allow VLAN DNS TCP"

# 2. 允许 VLAN Ping 测试路由器连通性
add chain=input action=accept protocol=icmp in-interface-list=VLAN_GUESTS comment="Allow VLAN Ping Router"

# 3. 拒绝 VLAN 访问路由器管理后台 (Winbox/SSH/Web 等)
add chain=input action=drop in-interface-list=VLAN_GUESTS comment="Block VLAN to Router Management"

# 4. 允许 VLAN 访问互联网 (出站到 WAN)
add chain=forward action=accept in-interface-list=VLAN_GUESTS out-interface-list=WAN comment="Allow VLANs to Internet"

# 5. 阻断 VLAN 访问其他所有网段 (实现了 VLAN 之间、以及 VLAN 到主 LAN 的隔离)
add chain=forward action=drop in-interface-list=VLAN_GUESTS comment="Drop VLAN to any other network"

# 6. 确认全局 NAT 伪装开启 (通常系统默认已存在)
/ip firewall nat add chain=srcnat out-interface-list=WAN action=masquerade comment="Default masquerade"
```

---

## 下联交换机配置方案

基于 802.1Q 协议，交换机配置的核心在于设定 **Tagged（Trunk，透传端口）** 与 **Untagged（Access，接入端口）**，以及正确的 **PVID**。

*   `Tagged (打标)`：通常用于交换机与路由器互联，或交换机之间的级联。
*   `Untagged (去标)`：用于连接不支持 VLAN 的终端设备（如电脑、AP、摄像头等）。
*   `PVID (缺省 VLAN)`：当端口接收到没有 VLAN 标签的普通数据时，为其打上的默认标签。

### 交换机 A 配置表

交换机 A 的**端口 1**连接路由器 `ether2`，**端口 8**预留作为级联口连接下一级交换机。

**VLAN 成员与端口划分：**

| VLAN ID | 职能描述 | 成员端口 | Tagged 端口 (Trunk) | Untagged 端口 (Access) |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 默认局域网 | 1, 7, 8 | - | 1, 7, 8 |
| 20 | 访客网络 | 1, 2, 8 | 1, 8 | 2 |
| 30 | 专属 WiFi | 1, 3, 8 | 1, 8 | 3 |
| 40 | 安防监控 | 1, 4, 8 | 1, 8 | 4 |
| 50 | 智能家居 | 1, 5, 8 | 1, 8 | 5 |
| 60 | 核心网络 | 1, 6, 8 | 1, 8 | 6 |

**PVID (默认端口 ID) 设置：**

| 物理端口 | 对应 PVID | 接入设备说明 |
| :--- | :--- | :--- |
| 端口 1 | 1 | 上联至 RouterOS |
| 端口 2 | 20 | 访客设备接入 |
| 端口 3 | 30 | 无线 AP 接入 |
| 端口 4 | 40 | 录像机 / 摄像头接入 |
| 端口 5 | 50 | 智能网关接入 |
| 端口 6 | 60 | 安全设备 / 私人电脑接入 |
| 端口 7 | 1 | 主 LAN 设备接入 |
| 端口 8 | 1 | 下联至交换机 B |

### 交换机 B 配置表 (级联扩展)

假设交换机 B 的**端口 1**连接到交换机 A 的**端口 8**。

**VLAN 成员与端口划分：**

| VLAN ID | 职能描述 | 成员端口 | Tagged 端口 (Trunk) | Untagged 端口 (Access) |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 默认局域网 | 1 | - | 1 |
| 20 | 访客网络 | 1, 2 | 1 | 2 |
| 30 | 专属 WiFi | 1, 3 | 1 | 3 |
| 40 | 安防监控 | 1, 4 | 1 | 4 |
| 50 | 智能家居 | 1, 5 | 1 | 5 |

**PVID (默认端口 ID) 设置：**

| 物理端口 | 对应 PVID | 接入设备说明 |
| :--- | :--- | :--- |
| 端口 1 | 1 | 上联至交换机 A 的端口 8 |
| 端口 2 | 20 | 对应 VLAN20 设备 |
| 端口 3 | 30 | 对应 VLAN30 设备 |
| 端口 4 | 40 | 对应 VLAN40 设备 |
| 端口 5 | 50 | 对应 VLAN50 设备 |

---

## 💡 扩展与维护指南 (可扩展性设计说明)

得益于上文逻辑的优化，您的网络目前具备极高的可维护性。如果您未来需要**新增一个 VLAN (例如 VLAN 70)**，只需遵循以下精简步骤，无需再碰复杂的防火墙策略：

*   **创建基础组件**：新增 `/interface vlan` (VLAN 70)、为其配置 `/ip address` 和 DHCP 服务（Pool、Network、Server）。
*   **配置二层网桥**：在 `/interface bridge vlan` 中，将 VLAN 70 添加，并将 `bridge` 和 `ether2` 设置为 Tagged。
*   **一键应用安全策略**：将 `vlan70` 接口加入到 `VLAN_GUESTS` 列表中 (`/interface list member add interface=vlan70 list=VLAN_GUESTS`)。系统会自动继承网络隔离和外网访问权限。
*   **交换机放行**：在对应交换机的 Trunk 口配置 Tagged，在 Access 口配置 Untagged 及对应 PVID 即可。

2026.10.2