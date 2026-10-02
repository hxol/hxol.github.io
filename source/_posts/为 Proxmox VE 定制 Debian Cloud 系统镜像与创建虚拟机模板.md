---
title: 为 Proxmox VE 定制 Debian Cloud 系统镜像与创建虚拟机模板
date: 2026-10-01 15:30:38
tags: [笔记, Proxmox VE, PVE, Debian Cloud, 系统镜像]
---


# 为 Proxmox VE 定制 Debian Cloud 系统镜像与创建虚拟机模板

## 定制 Debian Cloud 系统镜像

### 安装工具
```bash
sudo apt-get update && sudo apt-get install libguestfs-tools qemu-utils -y
```

### 下载 Debian Cloud 系统镜像
```bash
curl -fsSL https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2 -o debian-13-genericcloud-amd64-xxx-src.qcow2
```

### 环境变量
```bash
set +H
```

```bash
export LIBGUESTFS_APPLIANCE_PREFER_UNCOMPRESSED=1
```

### 脚本

```bash
#! /bin/bash
virt-customize -a debian-13-genericcloud-amd64-xxx-src.qcow2 \
  --smp 2 --verbose \
  --timezone "Asia/Hong_Kong" \
  --append-line "/etc/default/grub:# disables OS prober to avoid loopback detection which breaks booting" \
  --append-line "/etc/default/grub:GRUB_DISABLE_OS_PROBER=true" \
  --run-command "update-grub" \
  --run-command "systemctl enable serial-getty@ttyS1.service" \
  --run-command "export DEBIAN_FRONTEND=noninteractive" \
  \
  # --append-line "/etc/hosts:192.168.60.30 nexus.example.com" \ # 若不使用 nexus3 则注释掉本行 ，注意把 example.com 换成真实域名。
  # \ # 若不使用 nexus3 则注释掉本行 
  # --run-command "mv /etc/apt/sources.list.d/debian.sources /etc/apt/sources.list.d/debian.sources.disabled || true" \ # 若不使用 nexus3 则注释掉本行 
  --run-command "cat <<'NEXUS_EOF' > /etc/apt/sources.list.d/nexus.sources.back # 若不使用 nexus3 则把本行的 nexus.sources 加上 .back 
# /etc/apt/sources.list.d/nexus.sources
Types: deb
URIs: https://nexus.example.com/repository/debian-trixie-proxy/
Suites: trixie trixie-updates trixie-backports
Components: main contrib non-free-firmware non-free
Enabled: yes
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://nexus.example.com/repository/debian-trixie-security-proxy/
Suites: trixie-security
Components: main contrib non-free-firmware non-free
Enabled: yes
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
NEXUS_EOF" \
  \
  --run-command "sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen" \
  --run-command "locale-gen" \
  --run-command "echo 'LANG=en_US.UTF-8' > /etc/default/locale" \
  --run-command "echo 'LC_ALL=en_US.UTF-8' >> /etc/default/locale" \
  --run-command "cat <<'LOCALE_SH_EOF' > /etc/profile.d/set_locale.sh
#!/bin/sh
export LANG=\"en_US.UTF-8\"
export LC_ALL=\"en_US.UTF-8\"
LOCALE_SH_EOF" \
  --run-command "chmod +x /etc/profile.d/set_locale.sh" \
  \
  --run-command "cat <<'FSTAB_TMP_SHM_EOF' >> /etc/fstab
tmpfs /tmp tmpfs defaults,rw,nosuid,nodev,noexec,relatime 0 0
tmpfs /dev/shm tmpfs defaults,noexec,nosuid,nodev 0 0
FSTAB_TMP_SHM_EOF" \
  \
  --run-command "sed -i 's/^GRUB_CMDLINE_LINUX=\"\(.*\)\"/GRUB_CMDLINE_LINUX=\"\1 apparmor=1 security=apparmor audit=1 audit_backlog_limit=8192\"/' /etc/default/grub" \
  --run-command "update-grub" \
  --run-command "chown root:root /boot/grub/grub.cfg && chmod og-rwx /boot/grub/grub.cfg" \
  \
  --run-command "echo 'Authorized uses only. All activity may be monitored and reported.' > /etc/issue.net" \
  --run-command "echo 'Authorized uses only. All activity may be monitored and reported.' > /etc/motd" \
  --run-command "rm /etc/motd.d/* || true" \
  \
  --run-command "apt purge -y ntp chrony rsync rpcbind || true" \
  --run-command "sed -i 's/^#\\?NTP=.*/NTP=time.apple.com time.windows.com/' /etc/systemd/timesyncd.conf" \
  --run-command "sed -i 's/^#\\?FallbackNTP=.*/FallbackNTP=/' /etc/systemd/timesyncd.conf" \
  --run-command "echo 'RootDistanceMax=1' >> /etc/systemd/timesyncd.conf" \
  --run-command "systemctl enable systemd-timesyncd.service" \
  \
  --run-command "echo 'Storage=persistent' >> /etc/systemd/journald.conf" \
  \
  --run-command "cat <<'MODPROBE_EOF' > /etc/modprobe.d/hardening.conf
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
install dccp /bin/true
install sctp /bin/true
install rds /bin/true
install tipc /bin/true
MODPROBE_EOF" \
  \
  --run-command "chown root:root /etc/crontab || true && chmod og-rwx /etc/crontab || true" \
  --run-command "chown root:root /etc/cron.hourly/ || true && chmod og-rwx /etc/cron.hourly/ || true" \
  --run-command "chown root:root /etc/cron.daily/ || true && chmod og-rwx /etc/cron.daily/ || true" \
  --run-command "chown root:root /etc/cron.weekly/ || true && chmod og-rwx /etc/cron.weekly/ || true" \
  --run-command "chown root:root /etc/cron.monthly/ || true && chmod og-rwx /etc/cron.monthly/ || true" \
  --run-command "chown root:root /etc/cron.d/ || true && chmod og-rwx /etc/cron.d/ || true" \
  --run-command "rm /etc/cron.deny || true && touch /etc/cron.allow && chmod g-wx,o-rwx /etc/cron.allow && chown root:root /etc/cron.allow" \
  --run-command "rm /etc/at.deny || true && touch /etc/at.allow && chmod g-wx,o-rwx /etc/at.allow && chown root:root /etc/at.allow" \
  \
  --run-command "chown root:root /etc/ssh/sshd_config && chmod og-rwx /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?X11Forwarding/c\X11Forwarding no' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?MaxAuthTries/c\MaxAuthTries 4' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?PermitRootLogin/c\PermitRootLogin no' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?ClientAliveInterval/c\ClientAliveInterval 300' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?ClientAliveCountMax/c\ClientAliveCountMax 3' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?LoginGraceTime/c\LoginGraceTime 60' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?Banner/c\Banner /etc/issue.net' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?AllowTcpForwarding/c\AllowTcpForwarding no' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^#\?MaxStartups/c\MaxStartups 10:30:100' /etc/ssh/sshd_config" \
  --run-command "sed -i '/^MACs /d' /etc/ssh/sshd_config && echo 'MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com,hmac-sha2-512,hmac-sha2-256' >> /etc/ssh/sshd_config" \
  --run-command "systemctl restart ssh.service || true" \
  \
  --update --install "sudo,qemu-guest-agent,cloud-guest-utils,spice-vdagent,bash-completion,unzip,wget,curl,net-tools,iputils-ping,iputils-arping,iputils-tracepath,nano,most,screen,less,vim,bzip2,lldpd,mtr-tiny,htop,dnsutils,zstd,p7zip-full,build-essential,tmux,glances,iftop,iotop,bmon,dstat,git,gnupg2,gnupg-agent,locales-all,libpam-pwquality,rsyslog,aide,aide-common,logrotate,jq,cron" \
  \
  --run-command "sed -i 's/^\s*create.*/create 0640 root utmp/' /etc/logrotate.conf" \
  \
  --run-command "sed -i '/^#\s*minlen/s/^#\s*minlen.*/minlen = 14/' /etc/security/pwquality.conf || echo 'minlen = 14' >> /etc/security/pwquality.conf" \
  --run-command "sed -i '/^#\s*minclass/s/^#\s*minclass.*/minclass = 4/' /etc/security/pwquality.conf || echo 'minclass = 4' >> /etc/security/pwquality.conf" \
  --run-command "sed -i '/^#\s*PASS_MAX_DAYS/c\PASS_MAX_DAYS 365' /etc/login.defs" \
  --run-command "sed -i '/^#\s*PASS_MIN_DAYS/c\PASS_MIN_DAYS 1' /etc/login.defs" \
  --run-command "sed -i '/^#\s*PASS_WARN_AGE/c\PASS_WARN_AGE 7' /etc/login.defs" \
  --run-command "sed -i '/^#INACTIVE/c\INACTIVE=30' /etc/default/useradd || echo 'INACTIVE=30' >> /etc/default/useradd" \
  --run-command "useradd -D -f 30" \
  \
  --run-command "sed -i '/^password\s*\[success=1 default=ignore\]\s*pam_unix.so/s/$/ sha512/' /etc/pam.d/common-password" \
  --run-command "sed -i '/^password\s*requisite\s*pam_pwquality.so/s/$/ retry=3/' /etc/pam.d/common-password" \
  \
  --run-command "sed -i '1iauth\trequired\t\tpam_faillock.so preauth silent audit deny=5 unlock_time=900' /etc/pam.d/common-auth" \
  --run-command "sed -i '/^auth\s*\[success=1 default=ignore\]\s*pam_unix.so/a\auth\t[default=die]\t\tpam_faillock.so authfail' /etc/pam.d/common-auth" \
  --run-command "sed -i '/^account\s*requisite\s*pam_deny.so/a\account\trequired\t\tpam_faillock.so' /etc/pam.d/common-account" \
  \
  --run-command "groupadd sugroup || true" \
  --run-command "sed -i '/^#auth\s*required\s*pam_wheel.so use_uid/s/^#auth/auth/' /etc/pam.d/su || \
                 echo 'auth required pam_wheel.so use_uid group=sugroup' >> /etc/pam.d/su" \
  \
  --run-command "apt-get -y autoremove --purge && apt-get -y clean" \
  --delete "/var/log/*.log" \
  --delete "/var/lib/apt/lists/*" \
  --delete "/var/cache/apt/*" \
  --truncate "/etc/machine-id"
```

运行脚本
```bash
sudo bash build-debian-image.sh
```

### 减少镜像体积      
```bash
virt-sparsify --compress debian-13-genericcloud-amd64-xxx-src.qcow2 debian-13-genericcloud-amd64-xxx.qcow2
```

## 创建 PVE 虚拟机模板

### 在 Proxmox VE 中导入镜像并创建虚拟机模板

```bash
qm create 900000 \
  --machine q35 \
  --cpu cputype=host \
  --name "debian-13-cloud-template" \
  --scsi2 "local-lvm:cloudinit" \
  --serial0 socket \
  --vga std \
  --scsihw virtio-scsi-single \
  --net0 virtio,bridge=vmbr0 \
  --agent 1 \
  --ostype l26 \
  --memory 1024
```

### 将镜像导入到 PVE 的存储中、并 attach 到刚刚创建的虚拟机中、并设置为该虚拟机的启动盘
```bash
qm importdisk 900000 "/root/debian-13-genericcloud-amd64-xxx.qcow2" local-lvm -format qcow2
```

```bash
qm set 900000 --scsi0 "local-lvm:vm-900000-disk-0,discard=on,ssd=1"
```

```bash
qm set 900000 --boot order=scsi0
```

### 为模板的 Cloud-init 中设置默认通过 DHCP 获取 IP 地址
```bash
qm set 900000 --ipconfig0 ip=dhcp
```

### 验证一下储存在 PVE 中的 Cloud-init 配置
```bash
qm cloudinit dump 900000 user
```
```bash
qm cloudinit dump 900000 network
```

### 将这个虚拟机转换为模板
```bash
qm template 900000
```
