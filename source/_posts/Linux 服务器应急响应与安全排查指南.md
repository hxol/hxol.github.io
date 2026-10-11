---
title: Linux 服务器应急响应与安全排查指南
date: 2026-10-09 17:30:23
tags: [笔记, Linux, 安全排查]
---





# Linux 服务器应急响应与安全排查指南

当服务器出现 CPU 飙升、带宽满载或收到安全告警时，如何快速定位恶意程序并清除后门？

## 一、 异常进程与端口排查

黑客入侵后通常会运行挖矿程序或远控木马，第一步需要揪出这些异常进程。

### 1. 查找资源占用异常的进程
*   **按 CPU 排序**：运行 `top` 命令后，按大写字母 `P`。
*   **按内存排序**：运行 `top` 命令后，按大写字母 `M`。
*   **快捷命令行查看内存 Top 10**：
    ```bash
    ps aux | sort -k4nr | head -n 10
    ```

### 2. 溯源异常进程
找到可疑进程的 `$PID` 后，深入挖掘其对应的文件和行为：
*   **查看进程可执行文件路径**：`ls -la /proc/$PID/exe`
*   **查看进程完整运行环境**：`ls -la /proc/$PID`
*   **查看进程打开的文件**：`lsof -p $PID`
*   **动态跟踪进程行为（高级）**：`strace -tt -T -e trace=all -p $PID`

### 3. 查看网络端口连接
*   **查看所有网络连接（含进程信息）**：
    ```bash
    netstat -anltp 
    # 或使用更高效的 ss 命令
    ss -antpl
    ```
*   **查看指定进程的端口**：`netstat -anltp | grep $PID`

### 4. 处置恶意进程
> **⚠️ 提示：** 建议先挂起进程进行取证分析，再彻底清除。
*   **挂起进程（取证用）**：`kill -STOP $PID`
*   **追踪进程树并强制结束**：
    ```bash
    ps -elf | grep $PID
    kill -9 $PID
    ```

---

## 二、 账号安全与登录日志审计

黑客往往会留下提权账号或开启远程登录权限，排查账号是重中之重。

### 1. 揪出异常账号
*   **查找 UID 为 0 的特权用户**（通常只有 root）：
    ```bash
    awk -F ":" '$3==0{print $1}' /etc/passwd
    ```
*   **查找具有远程登录权限的账号**：
    ```bash
    awk '/\$1|\$6/{print $1}' /etc/shadow
    ```
*   **查找具有 sudo 权限的账号**：
    ```bash
    cat /etc/sudoers | grep -v "^#\|^$" | grep "ALL=(ALL)"
    ```

### 2. 分析异常登录行为
*   `w`：查看当前在线用户及其执行的命令。
*   `last`：查看所有用户的登录、注销及系统重启记录。
*   `lastlog`：查看所有用户最后一次登录的时间。
*   `lastb`：查看**登录失败**的记录（识别暴力破解）。

### 3. 日志深度挖掘 (`/var/log/secure`)
*   **查看登录成功的 IP 和用户名**：
    ```bash
    grep "Accepted " /var/log/secure* | awk '{print $1,$2,$3,$9,$11}'
    ```
*   **统计试图暴力破解本机的恶意 IP 排行**：
    ```bash
    grep "Failed password" /var/log/secure* | grep -E -o "([0-9]{1,3}\.){3}[0-9]{1,3}" | sort | uniq -c | sort -nr | head -n 10
    ```
*   **查看爆破 root 账号的 IP**：
    ```bash
    grep "Failed password for root" /var/log/secure | awk '{print $11}' | sort | uniq -c | sort -nr
    ```
*   **查看黑客常用的爆破用户名字典**：
    ```bash
    grep "Failed password" /var/log/secure | awk '{print $9}' | sort | uniq -c | sort -nr
    ```

---

## 三、 检查定时任务与开机启动项

这是木马实现“权限维持”最常用的手段。

### 1. 排查定时任务 (Cron)
除了常规的 `crontab -l`，还需要检查以下所有系统级定时任务目录：
```bash
cat /etc/crontab
ls -al /var/spool/cron/*
ls -al /etc/cron.d/*
ls -al /etc/cron.daily/*
ls -al /etc/cron.hourly/*
ls -al /etc/cron.monthly/*
ls -al /etc/cron.weekly/
```
*   **查看异步定时任务**：`cat /etc/anacrontab` 和 `ls -al /var/spool/anacron/*`

### 2. 排查开机启动项
*   **检查 rc.local**：
    ```bash
    cat /etc/rc.local
    cat /etc/init.d/rc.local
    ```
*   **Systemd 服务排查（现代 Linux）**：排查有无可疑的开机自启服务
    ```bash
    systemctl list-unit-files --type=service | grep enabled
    ```
*   **SysV 服务排查（老旧系统）**：
    ```bash
    chkconfig --list
    ```

---

## 四、 追踪近期被篡改的异常文件

通过时间维度找出系统被入侵的确切时间点留下的恶意文件。

*   **查找最近 3 天内被修改过的系统核心文件**：
    ```bash
    find /etc/ /usr/bin/ /usr/sbin/ /bin/ /usr/local/bin/ -type f -mtime -3 | xargs ls -la
    ```
*   **查找 10 分钟内被修改的 PHP（或 JSP/HTML）文件（查找 Webshell）**：
    ```bash
    find ./ -cmin -10 -name "*.php"
    ```
*   **按时间倒序查看目录下文件**：
    ```bash
    ls -alt | head -n 10
    ```
*   **校验系统命令是否被替换（RedHat/CentOS系）**：
    ```bash
    rpm -Va > rpm.log  # 检查二进制文件完整性，若命令被替换会提示
    ```

---

## 五、 深度隐藏后门排查

### 1. 恶意 Alias（别名劫持）
黑客可能会劫持 `ls`、`ps` 等命令来隐藏自身。检查以下文件是否有异常配置：
```bash
cat ~/.bashrc
cat ~/.bash_profile
cat /etc/profile
cat /etc/bashrc
```

### 2. 恶意 SSH 公钥（免密登录后门）
排查是否被写入了未知的 SSH 公钥：
```bash
cat /root/.ssh/authorized_keys
cat /home/*/.ssh/authorized_keys
```

### 3. 系统命令被替换（Rootkit 劫持）
如果发现 `ls`、`ps`、`top` 无法看到恶意进程，很可能是动态链接库 (`ld.so.preload`) 被劫持。此时系统原生命令已不可信！
**解决办法：使用静态编译的 Busybox**
```bash
# 检查是否被加载了恶意动态库
cat /etc/ld.so.preload 

# 使用 busybox 替代系统命令进行排查
busybox ps -ef | grep watchdogs
busybox ps -ef | grep ksoftirqds
busybox top
```

---

## 六、 借助第三方工具查杀

单靠手工排查可能有所遗漏，建议配合专业工具扫描：

1.  **Rootkit 查杀**：
    *   **chkrootkit**：[http://www.chkrootkit.org](http://www.chkrootkit.org)
    *   **rkhunter**：[http://rkhunter.sourceforge.net](http://rkhunter.sourceforge.net)
2.  **病毒与木马查杀**：
    *   **ClamAV** (免费开源杀毒)：[http://www.clamav.net](http://www.clamav.net)
3.  **Webshell 查杀**：
    *   **牧云 (CloudWalker)**：[https://github.com/chaitin/cloudwalker](https://github.com/chaitin/cloudwalker)

---

## 七、 防患未然：强化命令审计

为了在下次发生安全事件时能够快速溯源，建议现在就为系统配置详细的历史命令记录，包含**执行时间**和**登录 IP**。

**操作步骤：**
在 `/etc/profile` 文件的尾部追加以下配置：

```bash
# 1. 扩大历史记录保存条数
HISTSIZE=10000

# 2. 获取登录用户的IP地址
USER_IP=`who -u am i 2>/dev/null | awk '{print $NF}' | sed -e 's/[()]//g'`
if [ "$USER_IP" = "" ]; then
    USER_IP=`hostname`
fi

# 3. 设置历史命令的时间格式和附带信息
export HISTTIMEFORMAT="%F %T $USER_IP `whoami` "

# 4. 实时追加历史记录，防止多终端覆盖
shopt -s histappend
export PROMPT_COMMAND="history -a"
```
保存后执行 `source /etc/profile` 生效。之后运行 `history` 命令，你将看到清晰的时间戳、IP 和执行人信息。


