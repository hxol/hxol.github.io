---
title: Tailscale 部署与进阶网络配置
date: 2026-09-30 15:30:00
tags: [笔记, Tailscale, 自托管]
---


# Tailscale 部署与进阶网络配置指南

本文档涵盖了在 Linux 环境下通过**二进制文件**、**Docker** 以及 **Podman** 部署 Tailscale 的详细步骤，并包含子网路由（Subnet Routing）、出口节点（Exit Node）、MSS 钳制及网卡硬件加速等进阶配置。

> **注**：目前在某些特定内核环境下，容器版 Tailscale 在内核模式（Kernel Mode）下可能存在网络性能或稳定性波动，建议根据实际情况通过 `TS_USERSPACE=false/true` 进行调整。

---

## 目录
- [通过二进制文件安装](#一通过二进制文件安装)
- [通过 Docker 安装](#二通过-docker-安装)
- [通过 Podman 安装](#三通过-podman-安装)
- [官方参考文档](#四官方参考文档)

---

## 一、通过二进制文件安装

### 1. 注册账号
访问 [Tailscale 官网](https://tailscale.com)，点击 **Get Started**，按照提示使用你的身份提供商（如 Google、GitHub、Microsoft 等）注册并登录账号。

### 2. 安装 Tailscale
使用具有 `root` 权限的账号，运行官方提供的 Linux 通用安装脚本：
```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 3. 启用 IP 转发
如果需要将该设备作为“子网路由器（Subnet Router）”或“出口节点（Exit Node）”，必须开启系统 IP 转发功能：
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
```
应用配置使其生效
```bash
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 4. 启动 Tailscale
执行以下命令启动 Tailscale 并应用配置网络参数：
```bash
sudo tailscale up \
  --authkey=tskey-auth-YOUR_SECRET_KEY_HERE \
  --hostname=home-node \
  --advertise-routes=192.168.60.0/24 \
  --accept-routes \
  --advertise-exit-node
```
**参数详解：**
* `--authkey`：预身份验证密钥（请在后台 Settings -> Keys 中生成）。
* `--hostname`：在 Tailscale 网络中显示的设备名称。
* `--advertise-exit-node`：宣告本机为出口节点（允许其他设备通过本机代理上网）。
* `--advertise-routes=192.168.60.0/24`：宣告本地子网路由，允许异地设备访问本机的局域网设备。
* `--accept-routes`：接受其他 Tailscale 节点通告的路由。

### 5. 防火墙设置（补充内容）
如果你的 Linux 服务器开启了 `UFW` 或 `iptables` 防火墙，请放行 Tailscale 所需的端口及虚拟网卡：
```bash
# 放行 UDP 41641 端口（Tailscale P2P 通信端口）
sudo ufw allow 41641/udp
# 允许 tailscale0 虚拟网卡的所有流量
sudo ufw allow in on tailscale0
# 重载 UFW
sudo ufw reload
```

### 6. 局域网主路由静态规则设置
为了让家里（局域网）的其他设备也能不安装客户端直接访问 Tailscale 节点，需要在**主路由器（如 192.168.60.1）**上添加静态路由（Static Route）：

* **目标网络 (Destination)**: `100.64.0.0`
* **子网掩码 (Subnet Mask)**: `255.192.0.0` （CIDR 写法为 `/10`）
* **网关 (Gateway/Next Hop)**: `192.168.60.14` （此处为你部署 Tailscale 设备的真实内网 IP）
* **接口 (Interface)**: `LAN`

> **含义说明**：“凡是发往 `100.64.0.0/10`（Tailscale 虚拟网络）的流量，均交由 `192.168.60.14` 这个网关进行转发。”

### 7. 在 OpenWrt 设置 MSS 钳制
如果你使用 OpenWrt 作为网络网关，为了防止出现“连接建立成功，但大文件传输/网页加载卡死”的 MTU 黑洞现象，需要配置 MSS 钳制（基于 fw4 / nftables）：
```bash
nft add chain inet fw4 mangle_forward { type filter hook forward priority mangle \; }
nft add rule inet fw4 mangle_forward tcp flags syn tcp option maxseg size set rt mtu
```

### 8. 开启 Masquerade (SNAT)
若子网设备无法正常回程，可开启 SNAT 进行地址伪装。**注意：请将 `eth0` 替换为你实际的物理网卡名称**（可用 `ip a` 命令查看）：
```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```
*(注：建议将此命令写入系统的启动脚本，如 `rc.local` 或防火墙自定义规则中，以便开机自启生效。)*

### 9. 开启网卡的硬件加速（UDP GRO 优化）
为了压榨出局域网设备的极限网络转发性能（消除 Tailscale 启动时的 Warning），我们可以开启网卡的 UDP GRO (Generic Receive Offload) 功能。

**第1步：临时开启（立即生效）**
```bash
sudo apt update && sudo apt install ethtool -y
# 注意替换 eth0 为你的实际物理网卡名
sudo ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```
此时再运行一次 `sudo tailscale up ...`，警告将消失。

**第2步：配置持久化（开机自动应用）**
因为 `ethtool` 的设置在重启后会还原，我们需要创建一个 Systemd 服务：
```bash
sudo nano /etc/systemd/system/tailscale-gro.service
```
粘贴以下内容：
```ini
[Unit]
Description=Tailscale UDP GRO Optimization for eth0
After=network.target

[Service]
Type=oneshot
# 请确保此处的 eth0 与你的实际网卡一致
ExecStart=/sbin/ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
启用并启动服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable tailscale-gro.service
sudo systemctl start tailscale-gro.service
```
到这一步，你的 Tailscale 节点已达到**完美状态**，既完美实现跨网段访问，也释放了设备的极限网络吞吐能力。

### 10. 卸载 Tailscale
若需卸载，可参考官方指南：[Uninstall Tailscale](https://tailscale.com/kb/1069/uninstall)

---

## 二、通过 Docker 安装

### 1. 拉取镜像
```bash
docker pull tailscale/tailscale:latest
# 或使用 Github 镜像源
docker pull ghcr.io/tailscale/tailscale:latest
```

### 2. 启用 IP 转发
宿主机同样必须开启 IP 转发才能实现子网路由功能：
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 3. 创建环境变量文件
登录 Tailscale ​官网，在 Settings - Keys 中生成一次性授权密钥（Auth Key）。
新建目录并创建 `.env` 文件：
```bash
mkdir tailscale && cd tailscale
nano .env
```
写入以下内容（**注意保护 `.env` 中的密钥，防止泄露**）：
```ini
# Tailscale 网络中的主机名
TS_HOSTNAME=docker-router
# Tailscale 认证密钥（填入你生成的真实 Key）   
TS_AUTHKEY=tskey-auth-YOUR_SECRET_KEY_HERE
# 宣告局域网路由段（请根据实际网段修改）
TS_ROUTES=192.168.251.0/24
# 额外启动参数：宣告为出口节点并接受其他路由
TS_EXTRA_ARGS=--advertise-exit-node --accept-routes
# 禁用 Userspace 模式，启用内核模式以提升性能（若遇容器网络崩溃，可设为 true）
TS_USERSPACE=false
# 状态存储路径（需与下面 Compose 中的路径对应）
TS_STATE_DIR=/var/lib/tailscale
```

### 4. 创建 Docker Compose 配置
```bash
nano docker-compose.yml
```
填入以下内容：
```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    network_mode: host      # 使用宿主机网络栈
    cap_add:                # 赋予网络管理权限
      - NET_ADMIN
      - NET_RAW
    devices:
      - /dev/net/tun:/dev/net/tun  # 挂载 TUN 设备
    volumes:
      - tailscale-state:/var/lib/tailscale  # 持久化身份状态
    env_file:
      - .env
    restart: unless-stopped

volumes:
  tailscale-state: {}
```
启动容器：
```bash
docker compose up -d
```

---

## 三、通过 Podman 安装

### 1. 安装基础环境
```bash
sudo apt update && sudo apt install podman python3-pip -y
pip3 install podman-compose
```

### 2. 启用 IP 转发
```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

### 3. 创建环境变量文件
```bash
nano .env
```
填入：
```ini
TS_HOSTNAME=home-docker-tailscale-1
TS_AUTHKEY=tskey-auth-YOUR_SECRET_KEY_HERE
TS_ROUTES=192.168.251.0/24
# 参数中加入了关闭 SNAT 的选项（可根据需自行取舍）
TS_EXTRA_ARGS=--advertise-exit-node --accept-routes --snat-subnet-routes=false
TS_USERSPACE=false
TS_STATE_DIR=/var/lib/tailscale
```

### 4. 运行 Podman 容器
直接使用 CLI 启动：
```bash
sudo podman run \
  -d \
  --name tailscale \
  --network host \
  --cap-add NET_ADMIN \
  --cap-add NET_RAW \
  --device /dev/net/tun:/dev/net/tun \
  -v tailscale-state:/var/lib/tailscale \
  --env-file .env \
  docker.io/tailscale/tailscale:latest
```

### 5. 创建并配置 Systemd 守护服务
利用 Podman 的特性直接生成 Systemd 服务，保障开机自启与守护：
```bash
# 生成 systemd service 文件
sudo podman generate systemd --name tailscale --files
# 将文件移动至 systemd 目录
sudo cp container-tailscale.service /etc/systemd/system/
# 重新加载 systemd 并设置自启
sudo systemctl daemon-reload
sudo systemctl enable container-tailscale.service
sudo systemctl start container-tailscale.service
# 查看运行状态
sudo systemctl status container-tailscale.service
```

---

## 四、官方参考文档
在部署或进阶配置遇到问题时，建议查阅以下官方资料：
* [Tailscale Quickstart / Install](https://tailscale.com/kb/1017/install)
* [Linux Installation Guide](https://tailscale.com/kb/1347/installation)
* [Using Tailscale as an Exit Node](https://tailscale.com/kb/1408/quick-guide-exit-nodes)
* [Subnet Routing (通告子网路由)](https://tailscale.com/kb/1019/subnets)
* [Using Tailscale with Docker](https://tailscale.com/kb/1282/docker)
