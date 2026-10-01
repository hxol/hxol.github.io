---
title: Arch Linux 安装与配置全指南
date: 2023-10-01 15:30:17
tags: [笔记, Linux, Arch Linux]
---

# Arch Linux 安装与配置全指南

## 一、 准备工作

### 1. 获取与制作
*   **下载镜像**：访问 [Arch Linux 官网](https://archlinux.org/download/) 下载最新的 ISO 镜像。
*   **制作启动盘**：
    *   **Windows**: 推荐使用 [Rufus](https://rufus.ie/) (选择 GPT 分区方案，UEFI 目标系统) 或 [BalenaEtcher](https://www.balena.io/etcher/)。
    *   **Linux**: 使用 `dd` 命令：
        ```bash
        # /dev/sdx 请替换为你的U盘设备名，切勿填错
        sudo dd bs=4M if=archlinux-x86_64.iso of=/dev/sdx status=progress oflag=sync
        ```

### 2. BIOS 设置
*   关闭 **Secure Boot** (安全启动)。
*   设置启动模式为 **UEFI** (部分主板需关闭 CSM/Legacy)。
*   调整启动顺序，将 U 盘设为第一启动项。

---

## 二、 基础系统安装 (Live 环境)

启动进入 Arch ISO 后，你将面对一个 root 权限的 Zsh 终端。

### 1. 验证启动模式
```bash
ls /sys/firmware/efi/efivars
```
如果有输出文件列表，说明是 **UEFI** 模式；若报错，请检查 BIOS 设置。

### 2. 连接网络

#### 无线连接 (Wi-Fi)
使用 `iwctl` 工具：
```bash
iwctl                           # 进入交互模式
device list                     # 列出网卡 (如 wlan0)
station wlan0 scan              # 扫描网络
station wlan0 get-networks      # 查看可用网络
station wlan0 connect SSID      # 连接 (SSID替换为你的WiFi名)
# 输入密码后回车
exit                            # 退出
```

#### 有线连接 & 测试
插上网线通常会自动连接。测试连通性：
```bash
ping -c 3 archlinux.org
```

### 3. 更新系统时间
```bash
timedatectl set-ntp true
```

### 4. 优化镜像源 (重要)
为提高下载速度，建议禁用自动更新镜像服务 `reflector` (国内环境通常较慢) 并手动指定源。
```bash
systemctl stop reflector.service
vim /etc/pacman.d/mirrorlist
```
*操作提示*：使用 `dd` 剪切行，`gg` 回到顶部，`P` 粘贴。将国内源（清华、中科大、网易等）移至文件最上方。
```text
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.ustc.edu.cn/archlinux/$repo/os/$arch
```

### 5. 磁盘分区 (UEFI + GPT)
使用 `cfdisk` 进行可视化分区（比 fdisk 更直观）：
```bash
cfdisk /dev/sda  # sda 为你的目标硬盘
```
**推荐分区方案**：

| 分区类型 | 大小 | 挂载点 | 说明 |
| :--- | :--- | :--- | :--- |
| **EFI System** | 512M - 1G | `/boot` 或 `/efi` | 必须 FAT32 格式 |
| **Linux filesystem** | 剩余空间 | `/` | 根目录 (ext4/btrfs) |
| *Swap* | *可选* | - | 本教程推荐使用 Swap 文件代替分区 |

*操作*：选中 `Write` 输入 `yes` 写入，然后 `Quit`。

### 6. 格式化与挂载
假设 EFI 分区为 `sda1`，根分区为 `sda2`：

```bash
# 格式化
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/sda2

# 挂载 (注意顺序)
mount /dev/sda2 /mnt
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

### 7. 安装基础系统
安装内核、固件及必要工具：
```bash
pacstrap /mnt base base-devel linux linux-headers linux-firmware vim nano git networkmanager sudo intel-ucode (或 amd-ucode)
```
*注：如果你需要 LTS (长期支持) 内核，将 `linux` 替换为 `linux-lts`，`linux-headers` 替换为 `linux-lts-headers`。*

### 8. 生成 Fstab
```bash
genfstab -U /mnt >> /mnt/etc/fstab
cat /mnt/etc/fstab # 检查内容是否正确
```

---

## 三、 系统配置 (Chroot 环境)

进入新安装的系统进行配置：
```bash
arch-chroot /mnt
```

### 1. 时区与本地化
```bash
# 设置时区
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc

# 生成 Locale
vim /etc/locale.gen
# 去掉以下行的注释(#)：
# en_US.UTF-8 UTF-8
# zh_CN.UTF-8 UTF-8

locale-gen

# 设置默认语言 (建议使用英文，避免 TTY 乱码)
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### 2. 网络配置
```bash
# 设置主机名
echo "ArchPC" > /etc/hostname

# 编辑 hosts
vim /etc/hosts
# 添加：
# 127.0.0.1   localhost
# ::1         localhost
# 127.0.1.1   ArchPC.localdomain ArchPC
```

### 3. 设置 Root 密码
```bash
passwd root
```

### 4. 安装引导程序 (GRUB)
```bash
pacman -S grub efibootmgr os-prober

# 安装 GRUB 到 EFI 分区
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ArchLinux

# 如果是双系统，需启用 os-prober
vim /etc/default/grub
# 取消注释最后一行：GRUB_DISABLE_OS_PROBER=false

# 生成配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

### 5. 创建用户
创建一个日常使用的普通用户（不要一直使用 root）：
```bash
useradd -m -G wheel -s /bin/bash myuser  # myuser 换成你的名字
passwd myuser

# 配置 sudo 权限
EDITOR=vim visudo
# 取消注释这一行： %wheel ALL=(ALL:ALL) ALL
```

### 6. 配置 Swap 文件
```bash
dd if=/dev/zero of=/swapfile bs=1M count=4096 status=progress # 4G大小
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# 写入 fstab
echo "/swapfile none swap defaults 0 0" >> /etc/fstab
```

### 7. 退出并重启
```bash
exit
umount -R /mnt
reboot
```
*拔掉 U 盘，你现在应该能看到 GRUB 引导界面了。*

---

## 四、 桌面环境与驱动配置

重启后使用新创建的普通用户登录，确连网：
```bash
sudo systemctl start NetworkManager
sudo systemctl enable NetworkManager
# 若无网络，使用 nmtui 连接 WiFi
```

### 1. 显卡驱动
**查询显卡**：`lspci -k | grep -A 2 -E "(VGA|3D)"`

*   **Intel 核显**:
    ```bash
    sudo pacman -S mesa vulkan-intel
    # xf86-video-intel 现通常不建议安装，使用 modesetting 即可
    ```
*   **AMD 显卡**:
    ```bash
    sudo pacman -S mesa xf86-video-amdgpu vulkan-radeon
    ```
*   **NVIDIA 独显**:
    ```bash
    sudo pacman -S nvidia nvidia-utils # 若使用 lts 内核，请装 nvidia-lts
    ```

### 2. 声音服务
现代 Arch 推荐使用 `Pipewire` 替代 PulseAudio，但如果你习惯 PulseAudio：
```bash
sudo pacman -S pulseaudio pulseaudio-alsa pavucontrol
```

### 3. 安装桌面环境 (二选一)

#### 方案 A: KDE Plasma (推荐，现代、功能丰富)
```bash
# 安装基础包
sudo pacman -S plasma-meta konsole dolphin

# 安装显示管理器 (登录界面)
sudo pacman -S sddm
sudo systemctl enable sddm
```

#### 方案 B: XFCE (轻量、稳定)
```bash
sudo pacman -S xfce4 xfce4-goodies

# 安装显示管理器
sudo pacman -S lightdm lightdm-gtk-greeter
sudo systemctl enable lightdm
```

### 4. 中文字体
防止方块字：
```bash
sudo pacman -S noto-fonts noto-fonts-cjk noto-fonts-emoji wqy-microhei wqy-zenhei
```

### 5. 中文输入法 (Fcitx5)
```bash
sudo pacman -S fcitx5-im fcitx5-chinese-addons fcitx5-material-color
```
配置环境变量：`vim ~/.pam_environment`
```text
GTK_IM_MODULE DEFAULT=fcitx5
QT_IM_MODULE  DEFAULT=fcitx5
XMODIFIERS    DEFAULT=\@im=fcitx5
```
*注：重启后在系统设置或 Fcitx 配置工具中添加 "Pinyin" 即可。*

---

## 五、 常用软件与进阶配置

### 1. 安装 AUR 助手 (Yay)
AUR 是 Arch 的精髓，包含大量官方仓库没有的软件。
```bash
sudo pacman -S --needed git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
```

### 2. 常用软件清单
```bash
# 浏览器
sudo pacman -S firefox chromium

# 工具
sudo pacman -S unarchiver unzip p7zip tree neofetch

# 开发
sudo pacman -S code git

# 办公与媒体
sudo pacman -S libreoffice-fresh vlc obs-studio
```

### 3. 优化 Pacman 配置
开启彩色输出和并行下载：
`sudo vim /etc/pacman.conf`
*   取消注释 `Color`
*   取消注释 `ParallelDownloads = 5`
*   取消注释 `[multilib]` 及其下一行 (用于运行 32 位程序，如 Steam)
最后运行 `sudo pacman -Syu` 更新。

### 4. 解决 Windows 双系统时间不一致
Linux 使用 UTC，Windows 使用 LocalTime。建议修改 Linux 设置以兼容 Windows：
```bash
sudo timedatectl set-local-rtc 1 --adjust-system-clock
```

---

## 六、 开发者与极客工具 (选配)

### 1. 虚拟化 (KVM/QEMU)
```bash
sudo pacman -S qemu-full virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat
sudo systemctl enable --now libvirtd
# 将用户加入 libvirt 组
sudo usermod -aG libvirt $USER
```

### 2. Android 调试 (ADB)
```bash
sudo pacman -S android-tools
# 常用命令
adb devices
adb shell
```

### 3. Wireshark (网络抓包)
```bash
sudo pacman -S wireshark-qt
sudo usermod -aG wireshark $USER
# 需注销重新登录后生效
```

### 4. 蓝牙
```bash
sudo pacman -S bluez bluez-utils
sudo systemctl enable --now bluetooth
```

