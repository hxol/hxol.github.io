---
title: 自建 DNS 服务器全指南：AdGuard Home 与 Technitium DNS 部署及高可用集群
date: 2026-09-30 15:30:00
tags: [笔记, DNS, AdGuard Home, Technitium DNS, 集群, 自托管]
---

# 自建 DNS 服务器全指南：AdGuard Home 与 Technitium DNS 部署及高可用集群教程

## ⚠️ 严重警告（前置准备）

**在进行本文所有操作前，请务必先拉取所需的 Docker 镜像！**
释放 53 端口会更改系统的 DNS 行为，如果未提前拉取镜像，可能会导致宿主机断网，从而无法下载镜像。

```bash
# 拉取 AdGuard Home 镜像
docker pull adguard/adguardhome:latest

# 拉取 Technitium DNS 镜像
docker pull technitium/dns-server:15.1.0
```

---

## 一、释放 Linux 宿主机的 53 端口

默认情况下，很多 Linux 发行版的 `systemd-resolved` 服务会占用 53 端口。我们需要将其释放，以便容器使用。您可以选择**脚本自动化**或**手动配置**。

### 方法 A：使用 Bash 脚本一键释放
```bash
cd ~ && nano setup_adguard_resolved.sh
```
填入以下内容：
```bash
#!/bin/bash

# 脚本出错时立即退出
set -e

echo "开始配置 systemd-resolved 释放 53 端口..."
echo "此脚本需要 sudo 权限执行。"
echo ""

# 1. 创建配置目录
echo "步骤 1: 创建 /etc/systemd/resolved.conf.d 目录..."
sudo mkdir -p /etc/systemd/resolved.conf.d

# 2. 写入配置文件
echo "步骤 2: 停用 DNSStubListener..."
CONF_FILE="/etc/systemd/resolved.conf.d/adguardhome.conf"
sudo tee "$CONF_FILE" > /dev/null <<EOF
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
EOF
echo "配置文件已创建。"

# 3. 重定向 resolv.conf
echo "步骤 3: 重新链接 /etc/resolv.conf..."
RESOLV_CONF="/etc/resolv.conf"
RESOLV_CONF_BACKUP="/etc/resolv.conf.backup"
TARGET_LINK_FILE="/run/systemd/resolve/resolv.conf"

if [ -L "$RESOLV_CONF" ] && [ "$(readlink -f "$RESOLV_CONF")" = "$TARGET_LINK_FILE" ]; then
    echo "$RESOLV_CONF 已经是正确的符号链接，跳过备份。"
else
    if [ -e "$RESOLV_CONF" ]; then
        sudo mv -f "$RESOLV_CONF" "$RESOLV_CONF_BACKUP"
    fi
    sudo ln -sf "$TARGET_LINK_FILE" "$RESOLV_CONF"
fi

# 4. 重启服务
echo "步骤 4: 重启 systemd-resolved 服务..."
sudo systemctl reload-or-restart systemd-resolved

echo "配置完成！53 端口已释放。"
```
赋予执行权限并运行：
```bash
chmod +x setup_adguard_resolved.sh && sudo ./setup_adguard_resolved.sh
```

### 方法 B：手动执行（等效于上方脚本）

1. 创建目录：
```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
```
2. 停用 DNSStubListener：
```bash
sudo nano /etc/systemd/resolved.conf.d/adguardhome.conf
```
输入以下内容：
```ini
[Resolve]
DNS=127.0.0.1
DNSStubListener=no
```
3. 激活新的 `resolv.conf` 并重启服务：
```bash
sudo mv /etc/resolv.conf /etc/resolv.conf.backup
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl reload-or-restart systemd-resolved
```
*参考资料：[AdGuard Home 官方 FAQ (端口被占用)](https://adguard-dns.io/kb/zh-CN/adguard-home/faq/#bindinuse)*

---

## 二、AdGuard Home 安装部署详解

### 1. Docker Compose (Host 网络模式)
Host 模式是最简单、性能最好的方式，容器直接使用宿主机网络，无需映射端口。

```bash
sudo mkdir -p /opt/adguardhome/adguardhome/{work,conf}
cd /opt/adguardhome && sudo nano docker-compose.yml
```
```yaml
services:
  adguardhome:
    container_name: adguardhome
    image: adguard/adguardhome:latest
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./adguardhome/work:/opt/adguardhome/work
      - ./adguardhome/conf:/opt/adguardhome/conf
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "3"
```
启动容器：
```bash
docker compose up -d
```
启动后，访问 `http://<宿主机IP>:3000` 进行初始化设置。

### 2. Macvlan 网络模式（推荐在群晖 NAS 等设备使用）
Macvlan 让容器拥有局域网内独立的 IP 和 MAC 地址，避免与宿主机端口冲突。

#### 群晖实战方案（以 `192.168.1.x` 网段为例）
假设您的路由器 IP 为 `192.168.1.1`，我们预留 `192.168.1.240 - 247` 这个小网段给 Docker 专用，避免 IP 冲突。

**第一步：创建 Macvlan 网络 (SSH 操作)**
```bash
# 注意：群晖的网卡通常为 ovs_eth0 或 eth0，请根据实际情况修改 parent
docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  --ip-range=192.168.1.240/29 \
  -o parent=ovs_eth0 \
  ag_network
```

**第二步：编写 docker-compose.yml**
```bash
mkdir -p /volume1/docker/adguardhome/{work,conf}
cd /volume1/docker/adguardhome && nano docker-compose.yml
```
```yaml
services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
        max-file: "1"
    networks:
      ag_network:
        ipv4_address: 192.168.1.240 # 分配固定 IP
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf

networks:
  ag_network:
    external: true
```
启动：`docker compose up -d`

**第三步：让群晖宿主机与 Macvlan 容器互通 (桥接脚本)**
Docker 出于安全限制，宿主机默认无法直接访问 Macvlan 容器的 IP。如果您希望群晖本身也能使用此 DNS，需创建虚拟接口（Shim）。我们将使用 `192.168.1.241` 作为群晖的通讯马甲。

编写脚本：
```bash
#!/bin/bash
# 1. 开启 ovs_eth0 的混杂模式
ip link set ovs_eth0 promisc on
# 2. 建立虚拟接口 macvlan-shim
ip link add macvlan-shim link ovs_eth0 type macvlan mode bridge
# 3. 分配中间人 IP
ip addr add 192.168.1.241/32 dev macvlan-shim
# 4. 启动接口
ip link set macvlan-shim up
# 5. 添加路由，让群晖找 240 时走此接口
ip route add 192.168.1.240/32 dev macvlan-shim
```
*(注：在群晖中，建议将此脚本加入“控制面板” -> “任务计划” -> “开机触发的脚本”中，以便重启后依然生效。)*

---

## 三、AdGuard Home 优化增强设置

### 1. 上游 DNS 服务器推荐
进入设置 -> DNS 设置，根据网络环境填入：

**中国大陆网络环境：**
```text
tls://dns.pub
https://dns.pub/dns-query
tls://dns.alidns.com
https://dns.alidns.com/dns-query
```
**国际网络环境（需配置分流）：**
```text
tls://dns.google
https://dns.google/dns-query
tls://dns11.quad9.net
https://dns11.quad9.net/dns-query
```
*建议勾选：负载均衡（或并行请求）、使用 EDNS、使用 DNSSEC。*

### 2. Bootstrap 引导 DNS 服务器
用于解析 DoT/DoH 域名的传统 IP：
```text
# 国内
119.29.29.29
223.5.5.5

# 国际
8.8.8.8
9.9.9.11
```

### 3. DNS 缓存配置
适当增加缓存时间可显著提升解析速度：
*   **覆盖最小 TTL 值**：`600` (10分钟)
*   **覆盖最大 TTL 值**：`3600` (1小时)

### 4. 过滤器（去广告规则）
默认规则对国内网络效果有限，建议在“过滤器 -> DNS 封锁清单”中添加以下高质量规则：

| 规则名称 | 适用场景 | 链接地址 |
| :--- | :--- | :--- |
| **anti-AD** | 命中率高、兼容性强 (推荐) | `https://anti-ad.net/easylist.txt` |
| **halflife** | 涵盖多种国内防反感、视频过滤 (推荐) | `https://cdn.jsdelivr.net/gh/o0HalfLife0o/list@master/ad.txt` |
| **AdAway** | 官方去广告 Host 规则 | `https://adaway.org/hosts.txt` |
| **EasyList China** | 面向中文用户的去广告规则 | `https://easylist-downloads.adblockplus.org/easylistchina.txt` |
| **EasyPrivacy** | 反隐私跟踪、挖矿规则 | `https://easylist-downloads.adblockplus.org/easyprivacy.txt` |

---

## 四、Technitium DNS 服务器部署

Technitium DNS 是一款强大的、面向企业和高级用户的开源 DNS 服务器。

### 1. 常规 Linux 部署 (Host 模式)

创建目录与环境变量：
```bash
sudo mkdir -p /opt/technitium-dns && cd /opt/technitium-dns
sudo nano .env
```
`.env` 内容：
```env
IMAGE=technitium/dns-server:15.1.0
HOST_NAME=dns-server-1
DNS_SERVER_DOMAIN=dns-server-1
TZ=Asia/Shanghai
```
编写 `docker-compose.yml`：
```yaml
services:
  dns-server:
    container_name: dns-server
    hostname: ${HOST_NAME}
    image: ${IMAGE}
    network_mode: host
    environment:
      - DNS_SERVER_DOMAIN=${DNS_SERVER_DOMAIN}
      - DNS_SERVER_LOG_USING_LOCAL_TIME=true
      - TZ=${TZ}
    volumes:
      - config:/etc/dns
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "300M"
    # 注意：Host 模式下需注释掉 sysctls 的端口范围限制
    # sysctls:
    #   - net.ipv4.ip_local_port_range=1024 65535
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://127.0.0.1:5380/api/dnsClient/healthCheck | grep '\"status\":\"ok\"'"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

volumes:
    config:
```
启动：`docker compose up -d`

### 2. 群晖 Synology 部署 (Macvlan 模式)

*(假设使用 `192.168.2.x` 网段，网关 `192.168.2.1`，容器 IP `192.168.2.13`)*

**创建网络：**
```bash
sudo docker network create -d macvlan \
    --subnet=192.168.2.0/24 \
    --gateway=192.168.2.1 \
    -o parent=ovs_eth0 \
    --ip-range=192.168.2.13/32 \
    technitium_dns_network
```

**编写 Compose 文件：**
```yaml
services:
  dns-server:
    container_name: dns-server
    hostname: ${HOST_NAME}
    image: ${IMAGE}
    environment:
      - DNS_SERVER_DOMAIN=${DNS_SERVER_DOMAIN}
      - DNS_SERVER_LOG_USING_LOCAL_TIME=true
      - TZ=${TZ}
    volumes:
      - config:/etc/dns
    restart: unless-stopped
    sysctls:
      - net.ipv4.ip_local_port_range=1024 65535
    networks:
      technitium_dns_network:
        ipv4_address: 192.168.2.13
    dns:
      - 192.168.2.1
    deploy:
      resources:
        limits:
          cpus: "1"
          memory: "300M"
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://127.0.0.1:5380/api/dnsClient/healthCheck | grep '\"status\":\"ok\"'"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
    config:

networks:
  technitium_dns_network:
    external: true
```

**群晖 Macvlan 宿主互通开机脚本（极其重要）：**
群晖的 Nginx 反代或其他服务若要访问该容器，必须打通网络，否则会出现 `No route to host` 错误。

进入 **控制面板 -> 任务计划 -> 新增 -> 触发的任务 -> 用户定义的脚本**（使用 root 账号，开机触发）：
```bash
#!/bin/bash
# 等待群晖网络底层初始化完成
sleep 30

# 创建虚拟网卡并分配 IP (例如 250 为群晖通信IP)
ip link add mac0 link ovs_eth0 type macvlan mode bridge
ip addr add 192.168.2.250/32 dev mac0
ip link set mac0 up

# 添加静态路由，将目标容器 IP 导向 mac0
ip route add 192.168.2.13/32 dev mac0
```

---

## 五、Technitium 高可用集群配置实战

构建集群可以实现 DNS 的冗余容灾。
假设我们有两台服务器：
*   **主节点 (Primary Node)**: `dns1.example.lan` (`192.168.2.32`)
*   **从节点 (Secondary Node)**: `dns2.example.lan` (`192.168.2.219`)

### 1. 基础配置（双机均需执行）
通过 `http://<IP>:5380` 登录两台服务器后台。
*   **开启 HTTPS**：`Settings` -> `Web Service` -> 勾选 `Enable HTTPS`, `Enable HTTP/3`, 并勾选 `Use A Self Signed TLS Certificate...` -> 保存。
*   **修改 DNS Domain**：`Settings` -> `General` -> 主机设为 `dns-server-1`，从机设为 `dns-server-2`。
*   **关闭 DNSSEC**：(如有需要) 在 `General` 取消勾选 `Enable DNSSEC Validation`。
*   **配置上游转发**：`Settings` -> `Proxy & Forwarders` 设置好可靠的局域网或公网网关。

### 2. 初始化主节点 (`dns1.example.lan`)
1. 导航到 **Administration** -> **Cluster**。
2. 点击 **Initialize** 下拉按钮，选择 **New Cluster**。
3. 填写：
   * **Cluster Domain**: 集群专属域名，例如 `cluster.example.lan`（**注意：一旦初始化无法更改！**）
   * **Primary Node IP Addresses**: 点击 "Quick Add" 添加本机静态 IP (`192.168.2.32`)。
4. 保存。记下界面上生成的 **Primary Node URL**。

### 3. 加入从节点 (`dns2.example.lan`)
1. 导航到 **Administration** -> **Cluster**。
2. 点击 **Initialize**，选择 **Join Cluster**。
3. 填写信息：
   * **Secondary Node IP Addresses**: 点击 "Quick Add" 填入本机 IP (`192.168.2.219`)。
   * **Primary Node URL**: 填入主节点的 HTTPS URL(例如 `https://dns1.example.lan:53443` 注意端口号默认是 https 的端口 `53443` )。
   * **Primary Node IP Address**: 强烈建议手动填入主节点 IP `192.168.2.32`，防止初次解析失败。
   * **Certificate Validation**: 选择 **Ignore Certificate Validation Errors**（忽略自签名证书报错）。
   * **Credentials**: 填入主节点的账号密码。
4. 点击 **Join** 开始同步。

### 4. 进阶核心：如何同步 DNS 区域 (Catalog Zones)
在 Technitium 集群中，**普通的 DNS 记录默认是单机独立的！** 若要实现同步，必须将其放入“目录区域（Catalog Zone）”。

当集群名为 `cluster.example.lan` 时，系统会自动生成一个同步文件夹，通常名为 `cluster-catalog.cluster.example.lan`。

**新建并同步域名的步骤：**
1. 登录**主节点**，进入 **Zones** -> **Add Zone**。
2. 填入你想解析的域名（例如 `test.local`）。
3. **最关键的一步**：在下方的 **Catalog Zone** 下拉菜单中，不要选 `None`，**必须选择 `cluster-catalog.cluster.example.lan`**。
4. 保存。该域名下的所有 A 记录、CNAME 记录将自动同步至从节点。

### 💡 避坑指南（强烈建议阅读）

1. **从机数据覆盖警告**：
   从节点加入集群的瞬间，主节点的黑白名单（Allowed/Blocked lists）、Apps、用户权限会**完全覆盖**从节点！请务必提前备份从机数据。
2. **拒绝反向代理 (TLS Termination) 破坏**：
   集群通信使用的是基于 `DANE-EE` 的节点间端到端加密。如果您在 DNS 节点前部署了 Nginx/Traefik，**绝对不能在反代层卸载 HTTPS 证书**。必须使用 TCP 层直接转发 (TCP Stream)，否则节点间的证书握手会失败，导致集群断开。
3. **DHCP 暂不支持集群**：
   如果您启用了内置的 DHCP 服务器，两台机器的 DHCP 作用域不会互相感知和同步，目前需要您手动分配不同的 IP 池以避免冲突。
4. **强依赖静态 IP**：
   主从节点必须绑定静态 IP。如果路由器 DHCP 变更了它们的 IP，基于 A 记录和 DANE 的安全校验会立刻失效，导致集群崩溃。
5. **单向修改机制**：
   集群组建后，所有的全局设置只能在**主节点**修改。如果主节点宕机，您需要登录从节点，右键点击自身并选择 **"Promote To Primary" (提升为主节点)** 来接管控制权。

