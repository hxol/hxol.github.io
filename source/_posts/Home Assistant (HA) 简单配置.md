---
title: Home Assistant (HA) 简单配置
date: 2026-10-02 18:25:09
tags: [笔记, Home Assistant, 米家, 智能家居]
---



# Home Assistant (HA) 简单配置


## 一、 系统部署与安装

Home Assistant 提供多种运行方式，最常用的是基于官方操作系统的 HAOS（适合树莓派/微型主机）以及 Docker 容器化部署（适合 NAS/Linux 服务器）。

### 方式 A：通过 HAOS 镜像物理安装
HAOS 是官方高度定制的系统，包含 Supervisor 功能，支持插件商店，适合纯粹的智能家居网关设备。

*   **官方安装文档**: [Home Assistant Installation](https://www.home-assistant.io/installation)
*   **镜像下载发布页**: [HAOS Releases (GitHub)](https://github.com/home-assistant/operating-system/releases/)
*   **哈希校验信息**: [rpi-imager-haos.json](https://github.com/home-assistant/version/blob/master/rpi-imager-haos.json)

**镜像下载链接示例（请前往发布页获取最新稳定版）：**
*   树莓派 4 (64位): `https://github.com/.../download/<最新版本>/haos_rpi4-64-<最新版本>.img.xz`
*   x86-64 (通用PC): `https://github.com/.../download/<最新版本>/haos_generic-x86-64-<最新版本>.img.xz`

### 方式 B：通过 Docker 容器化安装
Docker 版（Home Assistant Container）更轻量，适合与其他服务共存的环境，但没有官方的“加载项 (Add-on)”商店功能。

创建项目目录并编辑 Compose 配置文件：
```bash
mkdir -p /opt/homeassistant/config
cd /opt/homeassistant
nano docker-compose.yaml
```

写入以下配置（建议使用 `stable` 标签以保持系统稳定）：
```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: "ghcr.io/home-assistant/home-assistant:stable"
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```

启动容器并访问：
```bash
docker compose up -d
```
启动后，在浏览器访问 `http://<服务器局域网IP>:8123` 即可进入初始化界面。

---

## 二、 核心系统配置

### SSH 与 Web 终端配置 (仅限 HAOS)
Advanced SSH & Web Terminal 插件不仅提供网页端命令行，还开放了底层的 SSH 访问权限，是高级运维的必备工具。

*   **安装途径**：在 HA 侧边栏进入 `设置` -> `加载项` -> `加载项商店`，搜索 `SSH` 或 `Terminal` 并安装官方推荐的 Advanced 版本。
*   **安全配置**：安装后务必进入 `配置` 面板。
    *   **推荐方案（密钥登录）**：在 `authorized_keys` 中填入你的公钥（`id_rsa.pub` 内容），并将 `password` 留空。
    *   **备用方案（密码登录）**：在 `password` 字段设置高强度密码。若留空将无法从外部 SSH 连接。
*   **启动设置**：建议勾选“启动时引导”、“监控(Watchdog)”以及“在侧边栏显示”，随后点击启动。
*   **外部连接测试**：
    ```bash
    # 使用你设置的用户名(默认root)和HA设备的IP连接
    ssh root@<HomeAssistant_IP>
    ```

### HACS 社区商店安装
HACS (Home Assistant Community Store) 提供了海量的第三方集成、卡片和主题，是进阶玩法的核心。

**HAOS 环境安装：**
直接点击官方重定向链接：[打开 HA 添加 HACS 仓库](https://my.home-assistant.io/redirect/supervisor_addon/?addon=cb646a50_get&repository_url=https%3A%2F%2Fgithub.com%2Fhacs%2Faddons)。在弹出的对话框中选择添加并安装。

**Docker 环境脚本安装：**
进入 Home Assistant 容器内部或在挂载的 `config` 目录下执行下载脚本：
```bash
wget -O - https://get.hacs.xyz | bash -
```

**后续激活步骤：**
重启 HA 后，**务必清空浏览器缓存**。前往 `设置` -> `设备与服务` -> `添加集成`，搜索 HACS。按照提示勾选风险确认，并点击链接前往 GitHub 输入设备验证码进行 OAuth 授权。完成后即可在侧边栏看到 HACS 入口。
*   *排错提示*：如果在 HACS 搜索界面无法使用 `Ctrl+V` 粘贴，请尝试 `Ctrl+Shift+V` 或鼠标右键粘贴。官方文档：[hacs.xyz](https://hacs.xyz/)

### 反向代理与真实 IP 透传
当使用 Nginx、Cloudflare 等反向代理暴露 HA 时，必须在 `configuration.yaml` 中配置信任代理，否则会导致外部请求被阻断或日志 IP 记录错误。

编辑 `configuration.yaml`，添加或修改 `http:` 模块：
```yaml
http:
  # 允许 HA 解析 X-Forwarded-For 请求头获取真实访客 IP
  use_x_forwarded_for: true
  trusted_proxies:
    # 填写反代服务器的局域网 IP
    - 192.168.x.x
    # 如果反代与 HA 在同一宿主机但不互通网络，可填写 Docker 网桥网关 IP (通过 ip addr show docker0 查看)
    - 172.17.0.1 
    # 如果反代是作为 HAOS 的插件运行，通常需要放行 HA 的内部 Docker 网段
    - 172.30.33.0/24  
```

---

## 三、 智能设备接入 (以小米生态为例)

小米设备接入 HA 有多种流派，需根据设备类型（WiFi/蓝牙/Zigbee）、对本地化控制的需求以及自身技术水平进行选择。

### Xiaomi MIoT Auto (社区推荐)
由社区维护的第三方集成，通过 HACS 安装。
*   **优势**：覆盖面极广，支持直接使用小米账号登录，自动同步云端设备，部分设备支持本地与云端混合控制，配置极简。
*   **劣势**：初次加载需依赖外网云端，自定义组件偶有报错。
*   **配置**：在 HACS 搜索安装重启后，在 `设备与服务` 中通过小米账号密码直接添加。

### Xiaomi Home (米家官方原生集成)
小米官方推出的集成，长期稳定性有保障。
*   **优势**：官方维护，安全性高（OAuth 2.0）。若家中配有“小米中枢网关”，可实现极佳的本地极速控制。
*   **劣势**：对老旧蓝牙设备或虚拟设备的支持不如 MIoT Auto 丰富。
*   **配置**：同样可在 HACS 中搜索安装或通过 GitHub 下载，在集成中进行网页端账号授权登录。

### Xiaomi Miio (HA 核心内置)
HA 自带的官方集成，主要针对较早的传统 WiFi 设备（如净化器、扫地机、老款插座）。
*   **优势**：免安装开箱即用，一旦配置成功即为纯本地控制，响应极速。
*   **劣势**：提取设备 Token 非常繁琐（需借助抓包、旧版 App 或专用提取工具）。
*   **配置**：UI 界面添加，或在 `configuration.yaml` 中通过 `host` 和 `token` 手动绑定。

### Xiaomi Gateway 3 (多模网关专属)
面向“小米多模网关”用户的极客方案（通过 HACS 安装）。
*   **优势**：完全本地化，将官方网关转变为本地 Zigbee/蓝牙协调器，接入子设备极其丰富。
*   **劣势**：对网关硬件型号及固件版本有严苛要求（新版固件常封堵 Telnet 漏洞），新手门槛高。

---

## 四、 监控与可观测性 (Prometheus + Loki + Grafana)

为了将 HA 的运行状态纳入家庭数据中心的全局监控台，需暴露其内部指标和底层硬件状态。

### 导出 Home Assistant 业务指标
HA 内置了 Prometheus 导出器，可直接输出实体状态变化。
在 `configuration.yaml` 中追加：
```yaml
prometheus:
```
重启 HA。前往用户个人资料页面，生成一个“长期访问令牌 (Long-lived access token)”。
在监控服务器的 `prometheus.yml` 中配置抓取：
```yaml
  - job_name: 'home-assistant'
    metrics_path: /api/prometheus
    static_configs:
      - targets: ['<HA_IP>:8123']
    bearer_token: '<此处填写刚才生成的长期访问令牌>'
```

### 采集宿主机/树莓派硬件指标
需在 HA 运行的宿主机上部署 `node_exporter` 收集 CPU、内存、磁盘等参数。
对于标准 Linux/Docker 环境：
```bash
# 基于 Debian 系的快捷安装
sudo apt-get update && sudo apt-get install prometheus-node-exporter
# 默认占用 9100 端口
```
随后在 Prometheus 中添加 `<主机IP>:9100` 为抓取目标即可。HAOS 用户可通过社区 Add-on 商店寻找对应的 Exporter 插件。

### 日志中心化 (Loki 方案)
HA 的核心日志通常位于 `/config/home-assistant.log`。可使用 Promtail 或更现代的 Grafana Alloy 将日志推送到 Loki。
以 Promtail 的配置文件 `promtail-config.yml` 为例：
```yaml
clients:
  - url: http://<Loki_IP>:3100/loki/api/v1/push
scrape_configs:
  - job_name: home-assistant
    static_configs:
      - targets: [localhost]
        labels:
          job: home-assistant-logs
          __path__: /config/home-assistant.log
```
*提示：Grafana 官方目前主推 Grafana Alloy 作为大一统的采集探针。若你的集群较新，建议直接使用 Alloy 替代单独的 Promtail。*

---

## 五、 智能家居网络 VLAN 安全隔离架构

为了防止安全性较弱的物联网设备（如智能摄像头、廉价网关）成为黑客入侵家庭核心网络的跳板，建议采用双网卡（有线+无线）及 VLAN 隔离方案。

**架构设计目标：**
*   **安全局域网 (VLAN 60)**：存放个人 PC、NAS，管理 HA。
*   **物联网局域网 (VLAN 50)**：专供智能设备联网，无法主动访问 VLAN 60。
*   **Home Assistant (树莓派)**：同时接入两个网络，作为指令下发的唯一合法网关。

### 路由端配置 (以企业级路由器为例)
1.  建立 VLAN 50 (例如网段 `192.168.50.x`) 与 VLAN 60 (例如网段 `192.168.60.x`)。
2.  将连接树莓派有线网口 (`eth0`) 的交换机端口划为 VLAN 60 (Access/Untagged)。
3.  将智能家居的专用 WiFi AP 划入 VLAN 50。
4.  **安全验证**：在路由器的防火墙或 ACL 策略中，确保拦截由 VLAN 50 发往 VLAN 60 的所有主动通讯请求。

### 树莓派底层网络配置
识别网卡：`eth0`（有线，连核心网）、`wlan0`（无线，连 IoT 网）。
配置双网卡静态 IP（编辑 `/etc/dhcpcd.conf` 或使用 NetworkManager）：

```conf
# --- 安全管理网 (VLAN 60) ---
interface eth0
static ip_address=192.168.60.10/24   
static routers=192.168.60.1          # 默认网关必须设在此处，确保设备正常上网
static domain_name_servers=192.168.60.1 223.5.5.5

# --- IoT 隔离网 (VLAN 50) ---
interface wlan0
static ip_address=192.168.50.10/24   
# 警告：绝对不要在这里设置 static routers，防止出现双网关冲突或流量侧漏
```
配置 IoT WiFi 连接密码（`/etc/wpa_supplicant/wpa_supplicant.conf`）：
```conf
network={
    ssid="<IoT_网络名称>"
    psk="<IoT_网络密码>"
}
```

### 核心安全锁：彻底禁用 IP 转发
> **⚠️ 危险警告**：如果树莓派同时连接两个网络且开启了 IP 转发功能，它自身就会变成一台路由器，彻底击穿之前在主路由上做的 VLAN 隔离，导致 IoT 病毒可以直接通过树莓派跃迁到安全网段。

必须编辑 `/etc/sysctl.conf`，强制关闭转发功能：
```conf
net.ipv4.ip_forward=0
net.ipv6.conf.all.forwarding=0
net.ipv6.conf.default.forwarding=0
```
保存后执行 `sudo sysctl -p` 生效。可通过 `cat /proc/sys/net/ipv4/ip_forward` 确认输出值为 `0`。

在此架构下，你在安全网段的 PC 可以毫无阻碍地通过 `192.168.60.10:8123` 管理 Home Assistant，而 HA 也能通过 `192.168.50.10` 顺畅地向隔离区的小米设备发送控制指令，兼顾了极致的安全性与便利性。