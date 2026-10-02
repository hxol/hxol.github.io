---
title: nftables 与 UFW 实战
date: 2026-10-01 15:30:36
tags: [笔记, 防火墙, nftables, UFW, Linux, 安全]
---

# nftables 与 UFW 实战

在现代 Linux 系统中，网络包过滤主要依赖于内核的 Netfilter 子系统。本文将详细介绍新一代防火墙工具 `nftables` 的核心概念与高阶实战，并在文末提供极简防火墙 `UFW` 的快速入门指南。

---

## 第一部分：nftables 简明教程与核心概念

`nftables` 是替代 iptables、ip6tables、arptables 和 ebtables 的下一代包过滤框架。它提供了统一的语法、更高的性能以及原生的集合（Sets）和字典（Maps）支持。

### ⚡️ 快速参考 (Cheat Sheet)

| 操作 | 命令 |
| :--- | :--- |
| **列出所有规则** | `nft list ruleset` |
| **列出特定表** | `nft list table inet my_table` |
| **清空所有规则** | `nft flush ruleset` |
| **加载配置文件** | `nft -f /etc/nftables.conf` |
| **进入交互模式** | `nft -i` |
| **备份当前规则** | `nft list ruleset > backup.nft` |
| **添加表** | `nft add table inet my_table` |
| **添加链(基础)** | `nft add chain inet my_table my_chain '{ type filter hook input priority 0; policy accept; }'` |
| **添加规则(追加)**| `nft add rule inet my_table my_chain tcp dport 22 accept` |
| **删除指定规则** | `nft delete rule inet my_table my_chain handle <句柄号>` |
| **实时监控事件** | `nft monitor` |

### 1. 安装与服务管理

在主流现代 Linux 发行版（Debian 10+, Ubuntu 18.04+, RHEL/CentOS 8+）中，`nftables` 已是默认后端。

```bash
# Debian/Ubuntu 体系
sudo apt update && sudo apt install nftables -y

# 启用并开机自启
sudo systemctl enable --now nftables
```

### 2. 核心概念：表、链与规则

nftables 的层级结构为：**表 (Table) -> 链 (Chain) -> 规则 (Rule)**。

*   **地址簇 (Families)**：创建表时需指定。最常用的是 `inet`，可同时处理 IPv4 和 IPv6，极大简化双栈配置；此外还有 `ip` (IPv4), `ip6` (IPv6), `arp`, `bridge` 等。
*   **钩子 (Hooks)**：定义链在内核网络栈的拦截位置。如 `prerouting` (DNAT), `input` (发给本机), `forward` (转发), `output` (本机发出), `postrouting` (SNAT)。
*   **优先级 (Priority)**：数值越小，越早执行。常用别名：`filter` (0), `srcnat` (100), `dstnat` (-100)。

### 3. 高级特性：集合 (Sets) 与 字典 (Maps)

`nftables` 的强项在于处理大量 IP 或端口时性能极高。

**命名集合 (Named Sets) 示例：**
```bash
# 1. 创建黑名单集合
nft add set inet filter blacklist { type ipv4_addr; }

# 2. 动态添加元素（无需重载整个防火墙）
nft add element inet filter blacklist { 192.168.1.100, 10.0.0.5 }

# 3. 在规则中引用 (使用 @ 符号)
nft add rule inet filter input ip saddr @blacklist drop
```

---

## 第二部分：nftables 实战演练

此部分提供两个真实场景的配置流程，从基础搭建到高阶应用。

### 实战场景一：使用命令行构建基础服务器防火墙

以下步骤演示如何从零开始，使用命令行逐步构建一个安全的服务器防火墙。

```bash
# 1. 清空所有现有规则
nft flush ruleset

# 2. 创建名为 my_table 的 inet 表（设为 dormant 挂起状态，配置完再激活以防断网）
nft add table inet my_table '{flags dormant;}'

# 3. 创建基础链
# 创建入站链 (input)，默认放行 (后面再改为 drop)
nft add chain inet my_table my_input '{type filter hook input priority 0; policy accept;}'
# 创建转发链 (forward)，默认拒绝
nft add chain inet my_table my_forward '{ type filter hook forward priority 0 ; policy drop ; }'
# 创建出站链 (output)，默认放行
nft add chain inet my_table my_output '{ type filter hook output priority 0 ; policy accept ; }'

# 4. 创建常规链（类似于自定义函数，用于分类处理 tcp/udp）
nft add chain inet my_table my_tcp_chain
nft add chain inet my_table my_udp_chain

# 5. 配置入站核心规则
# 放行已建立的和相关的流量 (必须)
nft add rule inet my_table my_input ct state '{established,related}' accept
# 放行本地回环接口 (lo)
nft add rule inet my_table my_input iif lo accept
# 丢弃非法状态的流量
nft add rule inet my_table my_input ct state invalid drop
# 放行 ICMP (Ping)
nft add rule inet my_table my_input icmp type echo-request accept
nft add rule inet my_table my_input icmpv6 type '{echo-request,nd-neighbor-solicit}' accept

# 6. 流量分流与放行
# 将新的 UDP/TCP 流量跳转到专属链处理
nft add rule inet my_table my_input meta l4proto udp ct state new jump my_udp_chain
# 仅对 TCP SYN 包跳转到 TCP 链
nft add rule inet my_table my_input 'meta l4proto tcp tcp flags & (fin|syn|rst|ack) == syn ct state new jump my_tcp_chain'

# 在 TCP 链中放行 SSH 端口 (例如 22)
nft add rule inet my_table my_tcp_chain tcp dport 22 accept

# 7. 完善拒绝规则与激活
# 拒绝 admin-prohibited 的 icmp 流量
nft add rule inet my_table my_input reject with icmpx type admin-prohibited
# 优雅地拒绝未被处理的流量 (返回 reject 而不是直接丢弃)
nft add rule inet my_table my_input meta l4proto udp reject
nft add rule inet my_table my_input meta l4proto tcp reject with tcp reset
nft add rule inet my_table my_input counter reject with icmpx port-unreachable

# 将入站链的默认策略改为 drop (收尾)
nft add chain inet my_table my_input '{ policy drop;}'

# 激活该表（移除 dormant 状态）
nft add table inet my_table

# 8. 持久化保存规则
nft list ruleset > /etc/nftables.conf
systemctl reload nftables
```

### 实战场景二：高阶应用 - 仅允许 Cloudflare 节点访问 Web 端口

如果你的网站套了 Cloudflare CDN，为了防止源站 IP 泄漏被直接攻击，可以通过脚本自动拉取 CF 的节点 IP，并限制只有这些 IP 能访问 443 端口（保留 SSH 端口供自己维护）。

#### 步骤 1：创建 Cloudflare IP 自动更新脚本
```bash
sudo nano /usr/local/sbin/update-cloudflare-ips.sh
```
填入以下内容：
```bash
#!/bin/bash
# 脚本：自动更新 nftables 中的 Cloudflare IP 地址集合

# --- 配置 ---
NFT_TABLE="inet global"          # nftables 表名 (与配置文件对应)
NFT_SET_V4="cloudflare_ipv4"     # IPv4 集合名称
NFT_SET_V6="cloudflare_ipv6"     # IPv6 集合名称

CLOUDFLARE_IPV4_URL="https://www.cloudflare.com/ips-v4"
CLOUDFLARE_IPV6_URL="https://www.cloudflare.com/ips-v6"

TMP_DIR="/tmp"
TMP_V4_FILE="${TMP_DIR}/cloudflare_ips_v4.$$.txt"
TMP_V6_FILE="${TMP_DIR}/cloudflare_ips_v6.$$.txt"
NFT_CMD_FILE="${TMP_DIR}/update_cf_sets.$$.nft"
LOG_TAG="cloudflare_nft_update"

# --- 函数 ---
log_message() {
    logger -t "$LOG_TAG" "$1"
    echo "$1"
}

cleanup() {
    rm -f "$TMP_V4_FILE" "$TMP_V6_FILE" "$NFT_CMD_FILE"
}

handle_error() {
    log_message "错误: $1"
    cleanup
    exit 1
}

trap cleanup EXIT
log_message "开始更新 Cloudflare IP 地址..."

# 下载 IP 列表
curl -sSL --fail -o "$TMP_V4_FILE" "$CLOUDFLARE_IPV4_URL" || handle_error "下载 IPv4 失败"
curl -sSL --fail -o "$TMP_V6_FILE" "$CLOUDFLARE_IPV6_URL" || handle_error "下载 IPv6 失败"

[ -s "$TMP_V4_FILE" ] || handle_error "IPv4 文件为空"
[ -s "$TMP_V6_FILE" ] || handle_error "IPv6 文件为空"

# 格式化 IP 列表
formatted_ips_v4=$(paste -sd, "$TMP_V4_FILE")
formatted_ips_v6=$(paste -sd, "$TMP_V6_FILE")

# 准备 nft 脚本 (原子操作)
cat > "$NFT_CMD_FILE" << EOF
flush set ${NFT_TABLE} ${NFT_SET_V4}
add element ${NFT_TABLE} ${NFT_SET_V4} { ${formatted_ips_v4} }
flush set ${NFT_TABLE} ${NFT_SET_V6}
add element ${NFT_TABLE} ${NFT_SET_V6} { ${formatted_ips_v6} }
EOF

# 应用更新
if sudo nft -f "$NFT_CMD_FILE"; then
    log_message "成功更新 nftables 中的 Cloudflare IP 地址集合。"
else
    mv "$NFT_CMD_FILE" "${NFT_CMD_FILE}.failed"
    handle_error "应用失败。请检查: ${NFT_CMD_FILE}.failed"
fi

exit 0
```
赋予执行权限：
```bash
sudo chmod +x /usr/local/sbin/update-cloudflare-ips.sh
```

#### 步骤 2：配置 nftables 规则文件
```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.bak
sudo nano /etc/nftables.conf
```
写入主规则（**注意替换 `<你的SSH端口>` 为实际端口，如 22**）：
```bash
#!/usr/sbin/nft -f
flush ruleset

table inet global {
    # 预先定义集合（由脚本自动填充）
    set cloudflare_ipv4 { type ipv4_addr; flags interval; }
    set cloudflare_ipv6 { type ipv6_addr; flags interval; }

    chain input {
        type filter hook input priority filter; policy drop;

        # 1. 允许本地回环
        iifname "lo" accept

        # 2. 允许已建立连接
        ct state { established, related } accept

        # 3. 允许 SSH (请将 <你的SSH端口> 替换为实际端口数字，如 22 或 2222)
        tcp dport <你的SSH端口> accept

        # 4. 核心：只允许 Cloudflare IP 访问 443/80 端口
        ip saddr @cloudflare_ipv4 tcp dport { 80, 443 } accept
        ip6 saddr @cloudflare_ipv6 tcp dport { 80, 443 } accept

        # 5. ICMP (Ping) 限速放行
        ip protocol icmp icmp type echo-request limit rate 5/second accept
        ip6 nexthdr icmpv6 icmpv6 type echo-request limit rate 5/second accept

        # 6. 跳转到 Fail2ban (防 SSH 爆破)
        jump f2b-sshd
    }

    # Fail2ban 专用链
    chain f2b-sshd {}
    chain forward { type filter hook forward priority filter; policy drop; }
    chain output { type filter hook output priority filter; policy accept; }
}
```

#### 步骤 3：验证与自动化
```bash
# 检查语法是否正确
sudo nft -c -f /etc/nftables.conf

# 应用新配置
sudo systemctl reload nftables

# 首次运行脚本拉取 CF 节点 IP
sudo /usr/local/sbin/update-cloudflare-ips.sh

# 验证集合是否已填充
sudo nft list set inet global cloudflare_ipv4

# 设置定时任务，每天凌晨 3:30 自动更新 CF 节点池
sudo crontab -e
# 添加以下行：
30 3 * * * /usr/local/sbin/update-cloudflare-ips.sh > /var/log/update-cloudflare-ips.log 2>&1
```

---

## 第三部分：UFW 简明教程 (轻量级替代方案)

如果你觉得 `nftables` 过于复杂，且只需进行简单的端口放行/拒绝，推荐使用 `UFW` (Uncomplicated Firewall)。它是 iptables/nftables 的前端配置工具，非常适合新手。

### 1. 安装与基本设置

```bash
# Debian/Ubuntu
sudo apt update -y && sudo apt install ufw -y

# 设置默认规则：拒绝所有传入，允许所有传出
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

### 2. 端口与 IP 管理

```bash
# 放行/关闭普通端口
sudo ufw allow 22/tcp       # 放行 22 端口 (TCP)
sudo ufw deny 80/tcp        # 拒绝 80 端口 (TCP)

# 放行特定 IP 或网段
sudo ufw allow from 192.168.1.100
sudo ufw allow from 10.0.0.0/24

# 组合参数：限制特定 IP 访问特定端口
# 示例：仅允许 192.168.1.100 通过 TCP 访问 22 端口
sudo ufw allow from 192.168.1.100 proto tcp to any port 22
```

### 3. 查看状态与删除规则

```bash
# 启用防火墙 (注意：启用前确保已放行 SSH 端口！)
sudo ufw enable

# 查看状态 (带序号，方便删除)
sudo ufw status numbered
```
输出示例：
```text
To                         Action      From
--                         ------      ----
[ 1] 22/tcp                ALLOW IN    192.168.1.100
[ 2] 80/tcp                ALLOW IN    Anywhere
```
```bash
# 根据序号删除规则 (如删除上面的第 2 条规则)
sudo ufw delete 2

# 重载规则使更改生效
sudo ufw reload

# 重置并禁用防火墙（清空所有配置）
sudo ufw reset
```

### 4. 高阶配置与日志

**高级配置文件：**
复杂的 UFW 规则（如 NAT 转发、自定义 ICMP 规则）无法通过命令行直接添加，需要编辑底层配置文件：
*   `/etc/ufw/before.rules`：在用户自定义规则之前执行的规则（配置 Ping、回环接口等）。
*   `/etc/ufw/after.rules`：在用户自定义规则之后执行的规则。
*   `/etc/default/ufw`：UFW 的主配置（如开启/关闭 IPv6 支持）。

**日志管理：**
```bash
# 开启日志并设置级别 (low/medium/high)
sudo ufw logging on
sudo ufw logging low

# 查看拦截日志
cat /var/log/ufw.log
```
> **日志示例解析：**
> `[UFW BLOCK] IN=eth0 OUT= MAC=... SRC=203.0.113.5 DST=198.51.100.2 LEN=40 PROTO=TCP SPT=48247 DPT=22 ...`
> *   `[UFW BLOCK]`：动作是拒绝连接。
> *   `IN/OUT`：网络接口，有值代表流量方向。
> *   `SRC/DST`：来源 IP / 目标 IP。
> *   `DPT`：目标端口（此处是 22，说明有人在尝试连接 SSH）。

---

## 总结

*   **UFW**：适合桌面系统或只需要简单开启/关闭端口的常规 Web 服务器。操作傻瓜化，不易出错。
*   **nftables**：适合高并发、需要复杂流量路由（NAT）、CDN IP 白名单动态更新等企业级场景。虽然学习曲线陡峭，但性能强悍、语法优雅。