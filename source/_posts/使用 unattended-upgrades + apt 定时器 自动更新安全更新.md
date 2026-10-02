---
title: 使用 unattended-upgrades + apt 定时器 自动更新安全更新
date: 2026-10-01 15:30:19
tags: [笔记, Linux, Debian, unattended-upgrades, apt 定时器, 安全更新]
---

# 使用 unattended-upgrades + apt 定时器 自动更新安全更新

## 一、安装必要组件

```bash
sudo apt update && sudo apt install unattended-upgrades apt-listchanges -y
```

---

## 二、开启自动更新机制

执行：

```bash
sudo dpkg-reconfigure -plow unattended-upgrades
```
会弹出选择界面
```
Applying updates on a frequent basis is an important part of keeping systems secure. By default, updates need to  be applied manually using package management tools. Alternatively, you can choose to have this system  automatically download and install important updates.     

Automatically download and install stable updates? 
```
选择 **Yes**。
这一步会创建基础配置文件。


---

## 三、修改配置文件

编辑：

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

找到类似下面的内容：

### 1. 设置自动重启（最重要）
找到 `Automatic-Reboot`。
*   默认是 `false`，改为 `true`。
*   **作用**：内核更新后如果不重启是不生效的，所以这一步很关键。

检查结果
```bash
grep --color 'Unattended-Upgrade::Automatic-Reboot' /etc/apt/apt.conf.d/50unattended-upgrades
```
得到
```conf
Unattended-Upgrade::Automatic-Reboot "true";
```

### 2. 设置重启时间
找到 `Automatic-Reboot-Time`。
*   设置为你服务器流量最低的时候（比如凌晨 4 点）。

检查结果
```bash
grep --color 'Unattended-Upgrade::Automatic-Reboot-Time' /etc/apt/apt.conf.d/50unattended-upgrades
```
得到
```conf
Unattended-Upgrade::Automatic-Reboot-Time "04:00";
```

### 3. 自动清理旧内核（防硬盘塞满）
找到 `Remove-Unused-Kernel-Packages`。
*   改为 `true`。
*   **作用**：VPS 空间通常不大，不开启这个，久而久之 `/boot` 分区满了会导致系统无法启动。

检查结果
```bash
grep --color 'Unattended-Upgrade::Remove-Unused-Kernel-Packages' /etc/apt/apt.conf.d/50unattended-upgrades
```
得到
```conf
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
```

> **关于更新源（Allowed-Origins）：**
> 文件最开头的 `Unattended-Upgrade::Allowed-Origins` 部分，**保持默认不要动**。默认配置通常只包含 `${distro-id}:${distro-codename}-security`，这意味着系统只会自动安装**安全更新**，而不会升级软件大版本（比如不会把 PHP 7.4 自动升到 8.0），这对 VPS 稳定性至关重要。

保存退出（Ctrl+O, 回车, Ctrl+X）。

---

## 四、启用每日检查

修改完配置如果不启用也是白搭。确保每日运行的开关是打开的。

打开周期配置文件：
```bash
sudo nano /etc/apt/apt.conf.d/20auto-upgrades
```

**把里面的内容全删了，直接复制下面这两行进去：**

```conf
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```
*   `"1"` 代表“每 1 天执行一次”。

---

## 五、启用并确认 systemd 定时器状态

启动并设置开机自启升级定时器
```bash
sudo systemctl enable --now apt-daily-upgrade.timer
```

启动更新列表的定时器（apt-daily.timer 负责 apt update，必须它先运行，升级定时器才能拿到最新的软件列表）
```bash
sudo systemctl enable --now apt-daily.timer
```

```bash
systemctl list-timers | grep apt
```

你应该看到：

* `apt-daily.timer`
* `apt-daily-upgrade.timer`

这两个就是自动更新的核心。

---

## 六、手动测试一次（非常建议）

先试运行看看有没有问题：

```bash
sudo unattended-upgrade -d
```

日志查看：

```bash
cat /var/log/unattended-upgrades/unattended-upgrades.log
```

运行下面的命令进行“空跑”测试：

```bash
sudo unattended-upgrades --dry-run --debug
```

*   如果不报错，最后显示类似 `All upgrades installed` 或者 `No packages found that can be upgraded unattended`，那就说明配置成功了。

## 其他

配置翻译参考
```
// Unattended-Upgrade::Origins-Pattern 控制哪些软件包会被自动升级。
//
// 下面的行格式为 "关键词=值,..."。
// 一个软件包只有当它的元数据与某一行中提供的所有关键字全部匹配时，才会被升级。
// （换句话说，省略的关键字相当于通配符。）
// 这些关键字来源于 Release 文件，但也接受一些别名。
// 接受的关键字包括：
// a,archive,suite (例如 "stable")
// c,component (例如 "main", "contrib", "non-free")
// l,label (例如 "Debian", "Debian-Security")
// o,origin (例如 "Debian", "Unofficial Multimedia Packages")
// n,codename (例如 "jessie", "jessie-updates")
// site (例如 "http.debian.net")
// 系统上可用的值可以通过命令 "apt-cache policy" 查看，也可以通过运行 "unattended-upgrades -d" 并查看日志文件来进行调试。
//
// 在配置行中，unattended-upgrades 允许使用两个宏，其值来源于 /etc/debian_version：
// ${distro_id}    已安装的 origin（发行版ID）
// ${distro_codename} 已安装的 codename（例如 "buster"）
Unattended-Upgrade::Origins-Pattern {
        // 基于 codename 的匹配方式：
        // 这种方式会跟随一个发行版在不同归档之间的迁移
        // （例如从 testing → stable → 之后的 oldstable）
        // 软件会保持所指定发行版中最新的版本，
        // 但 Debian 发行版本身不会被自动升级。
// "origin=Debian,codename=${distro_codename}-updates";
// "origin=Debian,codename=${distro_codename}-proposed-updates";
        "origin=Debian,codename=${distro_codename},label=Debian";
        "origin=Debian,codename=${distro_codename},label=Debian-Security";
        "origin=Debian,codename=${distro_codename}-security,label=Debian-Security";
// "o=Debian Backports,n=${distro_codename}-backports,l=Debian Backports";
        // 基于 archive 或 suite 的匹配方式：
        // 注意：这种方式在发行版迁移后会悄无声息地匹配到不同的发行版
        // （例如 testing 变成新的 stable）
// "o=Debian,a=stable";
// "o=Debian,a=stable-updates";
// "o=Debian,a=proposed-updates";
// "o=Debian Backports,a=stable-backports,l=Debian Backports";
};
// 使用 Python 正则表达式，匹配要从升级中排除的软件包
Unattended-Upgrade::Package-Blacklist {
    // 以下匹配所有以 linux- 开头的软件包
// "linux-";
    // 使用 $ 来明确指定包名的结尾。如果不加 $，则 "libc6" 会匹配所有相关包
// "libc6$";
// "libc6-dev$";
// "libc6-i686$";
    // 特殊字符需要转义
// "libstdc\+\+6$";
    // 以下匹配类似 xen-system-amd64、xen-utils-4.1、xenstore-utils 和 libxenstore3.0 的包
// "(lib)?xen(store)?";
    // 关于 Python 正则表达式的更多信息，请参阅
    // https://docs.python.org/3/howto/regex.html
};
// 此选项控制当 dpkg 非正常退出时，是否让 unattended-upgrades 自动运行
// dpkg --force-confold --configure -a
// 默认值为 true，以确保更新能够继续安装
//Unattended-Upgrade::AutoFixInterruptedDpkg "true";
// 将升级拆分成尽可能小的块，这样就可以用 SIGTERM 中断升级。
// 这会让升级稍微慢一些，但好处是升级运行中也可以关机（会有少量延迟）
//Unattended-Upgrade::MinimalSteps "true";
// 在系统关机时安装所有更新，而不是在后台运行时安装。
// 这显然会让关机过程变慢。
// Unattended-upgrades 会将 logind 的 InhibitDelayMaxSec 提高到 30 秒。
// 这给 unattended-upgrades 更多时间优雅关闭，或者在 InstallOnShutdown 模式下甚至安装少量软件包，
// 但相比之前允许的 30 分钟仍然是很大的退步。
// 建议启用 InstallOnShutdown 模式的用户进一步将 InhibitDelayMaxSec 调高，可能到 30 分钟。
//Unattended-Upgrade::InstallOnShutdown "false";
// 出现问题或软件包升级时发送邮件到这个地址
// 如果为空或未设置，则不发送邮件。请确保系统有正常工作的邮件配置。
// 必须安装提供 'mailx' 的软件包。例如 "user@example.com"
//Unattended-Upgrade::Mail "";
// 可设置为以下值之一：
// "always", "only-on-error" 或 "on-change"
// 如果未设置，则使用旧版的 MailOnlyOnError（布尔值）来决定是 "only-on-error" 还是 "on-change"
//Unattended-Upgrade::MailReport "on-change";
// 自动移除不再使用的、自动安装的内核相关软件包
// （包括内核映像、内核头文件和版本锁定的工具）
//Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
// 升级后自动移除新产生的不再需要的依赖包
//Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
// 升级后自动移除不再使用的软件包
// （相当于 apt-get autoremove）
//Unattended-Upgrade::Remove-Unused-Dependencies "false";
// 如果升级后发现 /var/run/reboot-required 文件存在，
// 则*不经确认*自动重启
//Unattended-Upgrade::Automatic-Reboot "false";
// 当 Unattended-Upgrade::Automatic-Reboot 为 true 时，
// 即使当前有用户登录也自动重启
//Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
// 如果启用了自动重启且需要重启，则在指定时间而不是立即重启
// 默认值："now"
//Unattended-Upgrade::Automatic-Reboot-Time "02:00";
// 使用 apt 的带宽限制功能，此例将下载速度限制为 70kb/sec
//Acquire::http::Dl-Limit "70";
// 启用 syslog 日志记录。默认值为 False
// Unattended-Upgrade::SyslogEnable "false";
// 指定 syslog facility。默认值为 daemon
// Unattended-Upgrade::SyslogFacility "daemon";
// 仅在使用交流电源（非电池）时下载和安装升级
// （即电池供电时跳过或优雅停止更新）
// Unattended-Upgrade::OnlyOnACPower "true";
// 仅在非计费网络连接上下载和安装升级
// （即在计费连接上跳过或优雅停止更新）
// Unattended-Upgrade::Skip-Updates-On-Metered-Connections "true";
// 详细日志
// Unattended-Upgrade::Verbose "false";
// 在 unattended-upgrades 和 unattended-upgrade-shutdown 中同时打印调试信息
// Unattended-Upgrade::Debug "false";
// 允许当 Pin-Priority > 1000 时进行软件包降级
// Unattended-Upgrade::Allow-downgrade "false";
// 当 APT 无法标记某个软件包进行升级或安装时，尝试调整相关软件包的候选版本，
// 以帮助 APT 的解析器找到可行方案。
// 这是个临时解决办法，直到 APT 的解析器被修复能总是找到解法（参见 Debian bug #711128）。
// 除 Debian sid 外默认启用此回退机制，因为 sid 经常出现不可安装的包。
// 禁用此回退可以加快有不可安装包时的 unattended-upgrades 速度，
// 但代价是偶尔会保留本可以升级或安装的软件包。
// Unattended-Upgrade::Allow-APT-Mark-Fallback "true";
```