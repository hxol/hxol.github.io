---
title: SSH 进阶指南：代理转发与端口转发（隧道）
date: 2026-10-01 15:30:37
tags: [笔记, SSH, Linux, 安全]
---

# SSH 进阶指南：代理转发与端口转发（隧道）

## 第一部分：SSH 代理转发（Agent Forwarding）

`ssh-agent` 是一个帮助我们安全管理私钥的工具。

### 1. 工作原理简述

在 `ssh-agent` 中，私钥是解密后保存在内存中的。当 SSH 客户端需要与服务端进行认证时，服务端会发送一段挑战数据，客户端将其交给本机的 `ssh-agent`，使用私钥对数据进行签名处理，再发送回服务端进行验证。**整个过程中，私钥始终保留在本地，不会在网络中传输。**

### 2. 经典痛点场景

假设我们有三台服务器：
* **Server A**（本地管理机 / 你的电脑）：`<ServerA_IP>`
* **Server B**（中间机 / 跳板机）：`<ServerB_IP>`
* **Server C**（目标机）：`<ServerC_IP>`

目前，Server B 上有一个文件 `/path/to/test`，我们需要将其拷贝到 Server C。但你当前登录在 Server A 上。你可能会在 Server A 执行以下命令：

```bash
scp <user>@<ServerB_IP>:/path/to/test <user>@<ServerC_IP>:/path/to/
```

**问题在于：**
即使 Server A 已经把公钥分别推送到 Server B 和 Server C，实现了 A 免密登录 B 和 C。但是执行上述跨机 `scp` 命令时，**仍然会提示输入 Server C 的密码**。

**原因分析：**
上述 `scp` 命令的认证过程是：Server A 告诉 Server B 去连接 Server C。此时，是 **Server B 向 Server C 发起认证请求**。由于 Server B 上并没有相应的私钥，B 与 C 之间也没有配置免密认证，因此只能要求手动输入密码。

为了避免在 Server B 上放置私钥（增加泄露风险），我们引入了**代理转发**。

### 3. 代理转发的解决方案

**代理转发（Agent Forwarding）** 的原理是：让 Server B 充当一个“中介”。当 Server C 对 Server B 提出认证要求时，Server B 会将这个要求**转发**回 Server A。Server A 的 `ssh-agent` 处理完成后，将签名结果原路返回给 C。这样，无需在 B 上存放任何私钥，即可完成认证。

```text
[Server A 的 ssh-agent] <--(转发)--> [Server B (代理中介)] <--(认证请求)--> [Server C]
```

#### 配置步骤

**1. 在 Server A（客户端）启用代理转发：**
修改客户端配置文件 `~/.ssh/config` 或 `/etc/ssh/ssh_config`：
```text
Host *
    ForwardAgent yes
```
同时确保 Server A 的代理已启动并加载了私钥：
```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_rsa
```

**2. 在 Server B（服务端）允许代理转发：**
确保 Server B 的 `/etc/ssh/sshd_config` 中包含：
```text
AllowAgentForwarding yes  # OpenSSH 默认通常为 yes
```

配置完成后，再次执行之前的 `scp` 命令，即可完全免密。同样，你可以通过 `ssh -A <user>@<ServerB_IP>` 登录 B，然后在 B 上直接免密 `ssh` 登录 C。

> **⚠️ 现代安全补充（ProxyJump 推荐）：**
> 虽然代理转发可以解决跳板机问题，但如果 Server B 被恶意控制，超级管理员（root）有可能会劫持你的 `ssh-agent` socket，从而冒充你登录其他机器。
> **现代更推荐的做法是使用 ProxyJump（`-J` 参数）**，它相当于在 A 和 C 之间建立了一条通过 B 的透明加密隧道，B 无法触碰你的认证过程：
> `ssh -J <user>@<ServerB_IP> <user>@<ServerC_IP>`


---


## 第二部分：SSH 端口转发（SSH 隧道）

SSH 代理转发针对的是**身份认证**的转发，而 SSH 端口转发（俗称 SSH 隧道）则是针对**网络通信数据**的转发。

### 1. 经典场景：加密不安全的明文流量
假设主机 A 上有 MySQL 客户端，主机 B 上有 MySQL 服务端（监听 3306 端口）。MySQL 默认明文传输数据，直接通过公网连接非常不安全。

利用 SSH，我们可以搭建一条“加密隧道”。MySQL 客户端不再直接连接目标服务端，而是连接到本机的某个端口，SSH 会将数据加密后传输到服务端机器，再解密交给 MySQL 服务。

---

### 2. 本地转发（Local Port Forwarding）

**概念：** 将访问本地机器某个端口的流量，通过 SSH 隧道转发到远端主机的指定端口。

#### 使用场景
你处于 Server A，想要安全地访问 Server B 上的 MySQL 服务。

#### 命令语法
```bash
# 在 Server A 上执行
ssh -N -f -L 9906:127.0.0.1:3306 <user>@<ServerB_IP>
```

#### 参数解析：
* `-L 9906:127.0.0.1:3306`：核心参数。意思是监听本地的 `9906` 端口，将数据转发到目标端（这里是 Server B 自己）的 `127.0.0.1:3306`。
* `-N`：不执行远程命令（仅建立隧道，不打开 Shell）。
* `-f`：后台运行。建立连接后将 SSH 放入后台，即使关闭终端也不会断开。

此时，在 Server A 上执行 `mysql -h 127.0.0.1 -P 9906`，就相当于跨越了公网安全地访问了 Server B 的 MySQL。

> **扩展：让其他机器也能通过 A 访问隧道**
> 默认情况下，`-L` 只监听本地的环回地址（127.0.0.1）。如果希望局域网内其他机器通过 Server A 的 9906 端口访问该隧道，可以加上 `-g` 选项（开启网关模式），或者显式指定绑定 IP：
> `ssh -N -f -L 0.0.0.0:9906:127.0.0.1:3306 <user>@<ServerB_IP>`

---

### 3. 远程转发（Remote Port Forwarding）

**概念：** 也被称为“内网穿透”或“反向隧道”。将远端机器某个端口的流量，通过 SSH 隧道反向转发回本地机器（或本地机器能访问的其他内网机器）。

#### 使用场景
Server B 位于公司内网，没有公网 IP，但运行着 MySQL（3306）。Server A 是一台拥有公网 IP 的云服务器。你在家里想要访问公司内网的 Server B。
由于没有公网 IP，A 无法主动连接 B。但 B 可以主动连接外网的 A。

#### 命令语法
```bash
# 在内网 Server B 上执行（主动外呼发起连接）
ssh -N -f -R 9906:127.0.0.1:3306 <user>@<ServerA_IP>
```
**含义：** Server B 主动连接 Server A，并在 Server A 上开启 `9906` 端口监听。任何人访问 Server A 的 `9906` 端口，数据都会顺着隧道回传给 Server B，最终交由 Server B 的 `3306` 端口处理。

> **💡 重要补充：为什么远程转发时指定 IP 无效？**
> 在实际操作中，你可能会发现即使执行了 `-R 0.0.0.0:9906...`，Server A 上依然只监听了 `127.0.0.1`，导致别人无法通过 A 的公网 IP 访问。
> **原因：** 出于安全考虑，OpenSSH 服务端默认禁止远程绑定非环回地址。
> **解决方案：** 需要修改拥有公网 IP 的那台机器（Server A）的 `/etc/ssh/sshd_config` 文件，将 `GatewayPorts` 的值设为 `yes` 或 `clientspecified`，然后重启 sshd 服务即可。

---

### 4. 远程转发与本地转发的区别总结

为了方便记忆，请记住以下口诀：**谁作为 SSH 客户端发起连接，谁就是“本地”。**

| 对比维度 | 本地转发 (`-L`) | 远程转发 (`-R`) |
| :--- | :--- | :--- |
| **发起方** | 需求方（客户端 A）主动发起连接 | 被访问方（内网端 B）主动外呼发起连接 |
| **端口监听位置** | 监听在本地机器（发起方 A）上 | 监听在远端机器（接收方 A）上 |
| **流量方向** | 访问 A 的端口 $\rightarrow$ 数据流向 B | 访问 A 的端口 $\rightarrow$ 数据流向 B |
| **适用场景** | 突破防火墙访问外部服务，加密明文通信 | 内网穿透（暴露内网服务给公网机器） |

---

### 5. 进阶场景拓展

**场景：访问第三方机器**
Server A 想访问 Server C（没有公网 IP，仅限内网访问），但是 Server A 可以访问有公网 IP 的 Server B，且 B 与 C 在同一内网下。
利用**本地转发**，将 B 作为跳板建立隧道：
```bash
# 在 Server A 执行
ssh -N -f -L 9906:<ServerC_IP>:3306 <user>@<ServerB_IP>
```
此时，访问 Server A 的 9906 端口，流量会发往 Server B，Server B 再将其转发给 Server C。
*(注意：此时 A 到 B 是加密的，但 B 到 C 在内网中是明文传输的。)*

---

## 第三部分：关键配置与稳定性保障

### 1. 开启端口转发权限
要让 SSH 端口转发正常工作，需要确保 SSH 服务端（被连接的那台机器）的 `/etc/ssh/sshd_config` 中配置了：
```text
AllowTcpForwarding yes  # 默认通常已开启
```

### 2. 保持隧道稳定存活 (Keep-Alive)
长时间没有数据传输时，防火墙或 NAT 路由器可能会切断闲置的 TCP 连接，导致隧道失效（尤其在远程转发场景下极为常见）。

**配置方法：**
在 SSH **客户端**的 `~/.ssh/config` 中添加：
```text
Host *
    ServerAliveInterval 60  # 每 60 秒向服务端发送一次心跳包
    ServerAliveCountMax 3   # 连续 3 次无响应则断开
```
或者在 SSH **服务端**的 `/etc/ssh/sshd_config` 中添加：
```text
ClientAliveInterval 60
ClientAliveCountMax 3
```

> **补充：** 对于极不稳定的网络环境，极其推荐使用 `autossh` 替代原生 `ssh` 命令建立隧道。`autossh` 会监控隧道的健康状态，一旦断开会自动重连，是生产环境中维持内网穿透的首选工具。

### 3. SSH 转发终极形态：动态转发（-D）
如果您不满足于单个端口的映射，而是希望代理所有的网络请求（例如将 SSH 服务器直接当作全局 SOCKS5 代理使用），可以使用动态转发：
```bash
ssh -N -f -D 1080 <user>@<Server_IP>
```
此时本地 1080 端口就是一个标准 SOCKS5 代理，在浏览器或系统中配置此代理后，所有流量都会通过目标机器转发。

---

## 最终总结

SSH 是运维和开发工作中不可或缺的利器，熟练掌握其核心参数能极大提升工作效率与安全性：

* **`-A` (Agent Forwarding)**：代理转发，用于跨越多台跳板机时免除重复认证。（推荐使用更安全的 `-J` 替代）。
* **`-L` (Local Forwarding)**：本地转发，建立本地到远端的隧道，用于加密通信或突破网络限制。
* **`-R` (Remote Forwarding)**：远程转发，建立远端回本机的反向隧道，用于内网穿透。
* **`-D` (Dynamic Forwarding)**：动态转发，建立 SOCKS5 代理。
* **辅助参数**：
  * `-f`：后台运行。
  * `-N`：不执行远程命令。
  * `-g`：开启网关模式，允许局域网其他机器使用你的本地隧道。