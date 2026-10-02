---
title: Proxmox VE (PVE) 运维与高级配置指南
date: 2026-10-01 15:30:36
tags: [笔记, PVE]
---

# Proxmox VE (PVE) 运维与高级配置指南

本文档涵盖了 Proxmox VE (PVE) 的初始配置、网络优化、存储管理、虚拟机（VM）与容器（LXC）的高级使用技巧。

## 1. 系统初始配置与优化

### 1.1 替换为非订阅源 (PVE 8 Bookworm)
PVE 默认使用企业订阅源，如果没有购买订阅，更新时会报错。可以通过添加非订阅源来解决。

```bash
# 创建或编辑非订阅源配置文件
nano /etc/apt/sources.list.d/pve-no-subscription.list
```
在文件中加入以下内容（针对 PVE 8.x）：
```text
deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```
*注：配置完成后，建议进入 PVE Web 界面 -> 节点 -> Updates -> Repositories，禁用带有 `enterprise` 字样的源。*

### 1.2 去除登录“无订阅”弹窗提示
每次登录 Web 界面都会弹出版权提示，可通过以下单行命令去除：
```bash
sed -i -e "s/Proxmox.Utils.checked_command(Ext.emptyFn);//" /usr/share/pve-manager/js/pvemanagerlib.js
```
*注：PVE 系统升级后可能需要重新执行此命令，修改后需按 `Ctrl+F5` 强制刷新浏览器缓存。*

### 1.3 CPU 节能与风扇降噪优化
PVE 默认倾向于性能模式，可能导致 Mini PC 等设备的风扇噪音过大。

**步骤一：安装管理工具**
```bash
apt update && apt install linux-cpupower -y
```

**步骤二：切换为节能模式 (powersave)**
```bash
cpupower frequency-set -g powersave
# 查看当前状态：cpupower frequency-info
```

**步骤三：设置开机自启**
```bash
crontab -e
```
在末尾添加：
```cron
@reboot /usr/bin/cpupower frequency-set -g powersave
```

> **⚠️ 进阶降噪（关闭睿频）**：
> 如果您使用的是 Intel N100/N5105 或 8 代以上酷睿（`intel_pstate` 驱动），`powersave` 模式依然会自动睿频导致风扇狂转。最有效的降噪方法是直接关闭睿频：
> `echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo`
> 可将其加入 crontab 实现开机自启：
> `@reboot echo 1 > /sys/devices/system/cpu/intel_pstate/no_turbo`

---

## 2. 网络配置与优化

### 2.1 命令行修改 PVE IP 地址
当 PVE 无法联网或更换路由器网段时，需在后台手动修改网络配置。

**1. 修改网络接口配置**
```bash
nano /etc/network/interfaces
```
```text
auto vmbr0
iface vmbr0 inet static
        address 192.168.x.200    # 修改为新 IP
        netmask 255.255.255.0
        gateway 192.168.x.1      # 修改为新网关
        bridge_ports eth0        # 确认物理网卡名称是否与 ip a 命令显示的一致
        bridge_stp off
        bridge_fd 0
```

**2. 修改 Hosts 映射**
```bash
nano /etc/hosts
```
```text
127.0.0.1 localhost.localdomain localhost
192.168.x.200 pve.lan pve      # 必须与上面的 IP 保持一致
```

**3. 修改 DNS**
```bash
nano /etc/resolv.conf
```
```text
nameserver 114.114.114.114
nameserver 8.8.8.8
```
完成后执行 `reboot` 重启生效。

### 2.2 修复网卡断流问题 (关闭 TSO/GSO)
Intel I225/I226 等网卡在特定情况下可能出现断流，可通过关闭网卡硬件卸载（TSO）解决。

**创建自启服务：**
```bash
nano /etc/systemd/system/off_tso.service
```
写入以下内容（请将 `<网卡名>` 替换为实际名称，如 `enp1s0`, `enp2s0` 等）：
```ini
[Unit]
Description=Turn off TSO for NIC
After=network.target network-online.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ethtool -K <网卡名1> tso off
ExecStart=/usr/sbin/ethtool -K <网卡名2> tso off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
**启用服务：**
```bash
systemctl daemon-reload
systemctl enable --now off_tso.service
```

---

## 3. 存储与磁盘管理

### 3.1 常用目录路径
*   **ISO 镜像存放目录**: `/var/lib/vz/template/iso/`
*   **虚拟机备份存放目录**: `/var/lib/vz/dump/`

### 3.2 合并 local-lvm 和 local 分区
PVE 默认将硬盘分为 `local` 和 `local-lvm`，对于小容量硬盘，合并可以最大化利用空间。

1. 删除 local-lvm：`lvremove pve/data`
2. 扩展 local (root) 空间：`lvextend -l +100%FREE -r pve/root`
3. Web 页面清理：
   - 进入 `数据中心 (Datacenter)` -> `存储 (Storage)`，删除 `local-lvm`。
   - 编辑 `local`，在“内容”下拉菜单中选中所有可用选项（如 磁盘映像, 容器等）。

### 3.3 修改 Swap (交换分区) 策略
减少 Swap 写入，延长 SSD 寿命并提升性能。
```bash
echo "vm.swappiness = 1" >> /etc/sysctl.conf
sysctl -p
```

### 3.4 宿主机系统备份 (Timeshift)
用于保护 PVE 宿主机系统本身配置（非虚拟机数据）。
需先将备份盘格式化为 `ext4`。编辑 `/etc/timeshift/timeshift.json`，**务必配置以下排除项**以防止备份体积过大：
```json
"exclude" : [
    "/var/lib/ceph/**",
    "/root/**",
    "/var/lib/vz/**",              // 排除虚拟机镜像和备份
    "/etc/pve/qemu-server/**",     // 排除VM配置，防恢复时被覆盖
    "/mnt/pve/**",                 // 排除挂载的外部存储
    "/mnt/**"                      // 排除所有挂载的 NAS 等
]
```

---

## 4. 虚拟机 (VM) 管理与操作

### 4.1 虚拟机常规操作
*   **强制关闭虚拟机** (Web界面无响应时): `qm stop <VM_ID>` (例: `qm stop 100`)
*   **定时重启 (Crontab)**: `34 2 * * * /usr/sbin/reboot` (每天凌晨2:34重启宿主机)

### 4.2 虚拟机硬盘扩容
**步骤一：PVE Web端扩容**
选中虚拟机 -> 硬件 -> 硬盘 -> 磁盘操作 (Disk Action) -> 调整大小 (Resize) -> 输入**增量**大小。

**步骤二：虚拟机内部扩容 (以普通 Linux Ext4 为例)**
```bash
apt install cloud-guest-utils
growpart /dev/sda 1      # 扩展分区表
resize2fs /dev/sda1      # 扩展文件系统 (若是 xfs 则用 xfs_growfs /)
```

**步骤三：虚拟机内部扩容 (以 OpenWrt 为例)**
1. 进入 OpenWrt：`opkg update && opkg install block-mount e2fsprogs fdisk blkid`
2. 使用 `fdisk /dev/sda` 创建新分区 (`sda3`)。
3. 格式化并迁移根目录：
```bash
mkfs.ext4 /dev/sda3
mkdir /mnt/sda3 && mount /dev/sda3 /mnt/sda3
mkdir -p /tmp/cproot && mount --bind / /tmp/cproot
tar -C /tmp/cproot -cvf - . | tar -C /mnt/sda3 -xf -
umount /tmp/cproot /mnt/sda3
```
4. 在 OpenWrt 挂载点设置中，将 `/` 挂载到新的 `sda3` 分区并重启。

### 4.3 Windows 11 虚拟机完美安装指南
Windows 11 需要满足 TPM 2.0 和 Secure Boot 要求，且需加载 VirtIO 驱动。

**前期准备：**
1. 下载 Windows 11 ISO。
2. 下载 [VirtIO 驱动 ISO (stable)](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso)。
3. 将两个 ISO 上传至 PVE 的 `local` 存储。

**创建 VM 核心参数：**
*   **操作系统 (OS)**: 客户机类型选 `Microsoft Windows`，版本 `11/2022`。
*   **系统 (System)**:
    *   显卡: `Default`
    *   机型 (Machine): `q35`
    *   BIOS: `OVMF (UEFI)` (必须)
    *   EFI 存储: 选本地存储，勾选 `预注册密钥 (Pre-Enroll Keys)`。
    *   添加 `TPM 2.0` 存储。
*   **硬盘 (Disks)**: 总线选 `VirtIO Block`，缓存建议 `Write back`，勾选 `SSD 仿真` 和 `Discard`。
*   **CPU**: 类别选 `host` (性能最好)。
*   **网络 (Network)**: 模型选 `VirtIO (半虚拟化)`。

**安装步骤与加载驱动：**
1. 创建完成后，在 VM 硬件中 **再添加一个 CD/DVD 驱动器**，挂载 `virtio-win.iso`。
2. 开机启动，在选择安装磁盘界面，会提示找不到硬盘。
3. 点击 **加载驱动程序** -> 浏览 -> 找到 VirtIO CDROM -> `viostor` -> `w11` -> `amd64`。
4. 加载成功后即可看到磁盘并继续安装。如果跳过联网界面，可按 `Shift+F10` 调出 CMD，输入 `oobe\bypassnro` 重启即可跳过。
5. 进入桌面后，打开 VirtIO 光盘，运行 `virtio-win-gt-x64.msi` 安装全套驱动。
6. 进入 `guest-agent` 文件夹，安装 `qemu-ga-x86_64.msi`，并在 PVE 的 VM 选项中开启 `QEMU Guest Agent`，重启 VM 即可显示 IP 等详细状态。

---

## 5. 容器 (LXC) 进阶操作

### 5.1 宿主机核显直通给 LXC (用于媒体服务器)
将核显直通给非特权容器（如 Docker 中的 Jellyfin/Emby）用于硬件解码。

**1. 确认宿主机核显状态**
在 PVE 宿主机执行：
```bash
ls -l /dev/dri
```
*正常应输出 `renderD128` (负责编解码) 和 `card0` (负责显示输出)。*

**2. PVE 网页端图形化直通**
对于 PVE 8+，可直接通过 UI 添加，无需再手写复杂的 `udev` 规则：
1. 选中容器 (`<LXC_ID>`) -> **资源** -> **添加** -> **设备直通 (Device Passthrough)**。
2. 添加核心解码设备：
   * **设备路径**: `/dev/dri/renderD128`
   * **访问模式/权限**: 填写 `0666` (⚠️重要：让非特权容器内的服务拥有直接读写权限)
3. 再次添加备用/输出设备：
   * **设备路径**: `/dev/dri/card0`
   * **访问模式/权限**: 填写 `0666`

启动 LXC 容器，进入控制台执行 `ls -l /dev/dri`，若看到设备且权限为 `crw-rw-rw-`，则说明直通成功。

### 5.2 将 NAS 存储挂载到 LXC (非特权容器)
**挂载逻辑**：NAS 提供 NFS 服务 -> PVE 宿主机挂载 NFS -> 通过 PVE 配置文件映射到 LXC。
> *直接在非特权 LXC 内部挂载 NFS 会因权限不足报错，必须通过宿主机中转。*

**1. NAS 端准备**
开启目标共享文件夹的 NFS 权限，并在防火墙放行 PVE 宿主机 IP。

**2. PVE 宿主机挂载 (使用 systemd 避免开机卡死)**
```bash
apt update && apt install nfs-common -y
mkdir -p /mnt/NAS_Media
nano /etc/fstab
```
加入挂载参数（替换 `<NAS_IP>` 和路径）：
```text
<NAS_IP>:/volume1/Media  /mnt/NAS_Media  nfs  noauto,x-systemd.automount,x-systemd.idle-timeout=60,x-systemd.mount-timeout=15s,soft,timeo=50,retrans=2,ro,nfsvers=4.1  0  0
```
刷新并验证挂载：
```bash
systemctl daemon-reload
systemctl restart remote-fs.target
ls -la /mnt/NAS_Media   # 查看文件是否已出现
```

**3. 将挂载点映射到 LXC**
关闭 LXC 容器 (`pct stop <LXC_ID>`)。
编辑 LXC 配置文件：
```bash
nano /etc/pve/lxc/<LXC_ID>.conf
```
在文件末尾添加映射规则 (`mp0`, `mp1` 依次递增)：
```text
mp0: /mnt/NAS_Media,mp=/mnt/video/Media
# 语法：mp序号: 宿主机绝对路径,mp=容器内部绝对路径
```
保存后启动 LXC，NAS 文件即会出现在容器内的 `/mnt/video/Media` 目录中。