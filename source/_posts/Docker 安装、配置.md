---
title: Docker 安装、配置
date: 2026-10-01 15:30:33
tags: [笔记, Docker, docker compose, 防火墙]
---

# Docker 安装、配置与运维全栈指南

## 一、 Docker 安装与卸载

### 1. 准备工作：卸载旧版本
在安装新版本之前，建议清理系统中可能存在的旧版本 Docker 组件。

**卸载软件本体:**
```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```
> **注意**：如果 `apt-get` 报告未安装这些软件包，可以直接忽略。

**清理残留数据（可选）:**
卸载软件不会自动删除 `/var/lib/docker/` 中的内容（包括镜像、容器、卷和网络）。如需彻底清除，请执行：
```bash
sudo rm -rf /var/lib/docker/
sudo rm -rf /var/lib/containerd/
```

### 2. Debian 系统安装 Docker
#### 方法一：使用官方脚本快速安装
适用于测试或开发环境的快速部署。
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

#### 方法二：通过官方存储库安装（生产环境推荐）
针对 Debian 13 (Trixie) 及主流版本：

1. **安装依赖并添加 GPG 密钥:**
```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg2 lsb-release -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
2. **添加存储库源:**
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
> **提示**：若国内网络下载缓慢，可将 `https://download.docker.com` 替换为 `https://mirrors.aliyun.com/docker-ce`。

3. **更新索引并安装:**
```bash
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

#### 方法三：离线手动安装 (.deb)
适用于无法联网的内网机器。
1. 访问下载地址: [https://download.docker.com/linux/debian/dists/](https://download.docker.com/linux/debian/dists/)
2. 进入对应 Debian 版本及 `pool/stable/<架构>` 目录（如 amd64）。
3. 下载 `containerd.io`, `docker-ce`, `docker-ce-cli`, `docker-buildx-plugin`, `docker-compose-plugin` 这 5 个 `.deb` 文件。
4. **执行安装并验证:**
```bash
sudo dpkg -i ./*.deb
sudo systemctl start docker
sudo docker run hello-world
```

### 3. 完全卸载 Docker
若需彻底移除 Docker：
```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras
sudo rm -rf /var/lib/docker
sudo rm -rf /var/lib/containerd
```

---

## 二、 Docker 核心配置 (Daemon)

日常使用中，我们需要经常修改 Docker 的底层行为。配置统一在 `/etc/docker/daemon.json` 中进行。

### 1. 基础权限与安全配置
**配置非 Root 用户运行：**
避免每次执行 docker 命令都需要加 `sudo`。
```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker # 立即生效
```
**启用 Docker Content Trust (DCT)：**
对镜像进行签名验证，增强安全性。
```bash
echo "export DOCKER_CONTENT_TRUST=1" >> ~/.bashrc
source ~/.bashrc
```

### 2. 统一配置 daemon.json 模板
以下包含了**日志轮转限制**、**Live Restore（热重载）**、**DNS 设定**以及**数据目录迁移**和**IPv6 基础支持**的最佳实践。

编辑配置文件：
```bash
sudo nano /etc/docker/daemon.json
```
写入以下内容（按需删减）：
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "dns": ["8.8.8.8", "8.8.4.4"],
  "data-root": "/data/docker",
  "ipv6": true,
  "fixed-cidr-v6": "fd00:1::/64"
}
```
**配置详解：**
*   **log-opts**：限制容器日志大小，防止 `/var/lib/docker/containers` 塞满硬盘。
*   **live-restore**：当包管理器更新或重启 Docker Daemon 时，运行中的容器不会停止，防止瞬间产生海量 I/O 与 CPU 峰值击垮宿主机。
*   **data-root**：默认存储在 `/var/lib/docker`，若系统盘较小，建议迁移至数据盘（如 `/data/docker`）。
*   **ipv6 / fixed-cidr-v6**：在默认 Bridge 网络启用 IPv6，并分配私网 ULA 地址。

重启服务生效：
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

*附：一键清空当前已存在的巨大历史日志*
```bash
sudo find /var/lib/docker/containers/ -type f -name "*-json.log" -exec truncate -s 0 {} \;
```

---

## 三、 网络与安全防火墙 (UFW)

UFW 是 Ubuntu/Debian 上流行的防火墙前端，但 **Docker 的端口映射会直接修改 iptables 规则并绕过 UFW**。这意味着 `ufw deny 8080` 无法阻止外部访问 Docker 的 8080 端口。

### 1. 解决 UFW 与 Docker 冲突 (IPv4)

修改 UFW 配置文件：
```bash
sudo nano /etc/ufw/after.rules
```
在文件末尾追加：
```text
# BEGIN UFW AND DOCKER
*filter
:ufw-user-forward - [0:0]
:ufw-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw-user-forward

-A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j RETURN
-A DOCKER-USER -m conntrack --ctstate INVALID -j DROP
-A DOCKER-USER -i docker0 -o docker0 -j ACCEPT

-A DOCKER-USER -j RETURN -s 10.0.0.0/8
-A DOCKER-USER -j RETURN -s 172.16.0.0/12
-A DOCKER-USER -j RETURN -s 192.168.0.0/16

-A DOCKER-USER -j ufw-docker-logging-deny -m conntrack --ctstate NEW -d 10.0.0.0/8
-A DOCKER-USER -j ufw-docker-logging-deny -m conntrack --ctstate NEW -d 172.16.0.0/12
-A DOCKER-USER -j ufw-docker-logging-deny -m conntrack --ctstate NEW -d 192.168.0.0/16

-A DOCKER-USER -j RETURN

-A ufw-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER BLOCK] "
-A ufw-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER
```

### 2. 解决 UFW 与 Docker 冲突 (IPv6)

```bash
sudo nano /etc/ufw/after6.rules
```
在文件最底部的 `COMMIT` 之后添加：
```text
# BEGIN UFW AND DOCKER IPV6
*filter
:ufw6-user-forward - [0:0]
:ufw6-docker-logging-deny - [0:0]
:DOCKER-USER - [0:0]
-A DOCKER-USER -j ufw6-user-forward

# 允许本地局域网或Docker自定义的 ULA IPv6 网段 (比如 fd00::/8)
-A DOCKER-USER -j RETURN -s fd00::/8
-A DOCKER-USER -j RETURN -s fe80::/10

# 允许 DNS 查询
-A DOCKER-USER -p udp -m udp --sport 53 --dport 1024:65535 -j RETURN

# 阻断外部试图建立到 Docker 私网的 TCP/UDP 连接并记录日志
-A DOCKER-USER -j ufw6-docker-logging-deny -p tcp -m tcp --tcp-flags FIN,SYN,RST,ACK SYN -d fd00::/8
-A DOCKER-USER -j ufw6-docker-logging-deny -p udp -m udp --dport 0:32767 -d fd00::/8

-A DOCKER-USER -j RETURN

-A ufw6-docker-logging-deny -m limit --limit 3/min --limit-burst 10 -j LOG --log-prefix "[UFW DOCKER IPV6 BLOCK] "
-A ufw6-docker-logging-deny -j DROP

COMMIT
# END UFW AND DOCKER IPV6
```
应用规则：
```bash
sudo systemctl restart ufw
```
*(注：有时需重启服务器方可彻底生效)*

### 3. UFW 下如何主动暴露容器端口？
拦截生效后，如需对外开放特定容器的端口，请使用 UFW 的 `route` 转发规则，**且需指定容器内部的真实端口**（非宿主机映射端口）：

*   开放所有容器的 80 端口: `sudo ufw route allow proto tcp from any to any port 80`
*   仅对特定 IP 开放特定容器端口: `sudo ufw route allow proto tcp from 192.168.1.1 to 172.17.0.2 port 3000`

---

## 四、 Docker Compose 进阶配置

### 1. 限制容器 CPU 与内存用量
在 `docker-compose.yml` 中防止恶意/泄漏程序耗尽宿主机资源：
```yaml
services:
  app:
    image: redis:7.4-alpine
    deploy:
      resources:
        limits:
          cpus: '0.5'     # 硬上限：最多使用 0.5 个核心
          memory: '1G'    # 硬上限：超过 1G 将触发 OOM 杀掉容器
        reservations:     # 软预留：Docker 确保容器至少能获得这些资源
          cpus: '0.25'
          memory: '512M'
```
**验证是否生效：**
```bash
docker inspect --format '{{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' <容器名>
# 内存单位为字节 (1G ≈ 1073741824)，CPU 0.5 为 500000000
```

### 2. 将 tmp 目录挂载到内存 (tmpfs)
为了提升 I/O 性能或保护固态硬盘寿命，可将频繁读写的缓存目录映射到内存中。
```yaml
services:
  app:
    image: my_app:latest
    volumes:
      - /path/to/host/data:/app/data
      # ▼ 下面是将 /tmp 映射到内存的配置 ▼
      - type: tmpfs
        target: /tmp
        tmpfs:
          mode: 01777      # 赋予标准 tmp 权限，解决自定义 UID 读写问题
          size: "512M"     # (强烈建议) 限制最大使用的内存大小，防止 OOM
      # ▲ 新增结束 ▲
```
*如需赋予可执行 (exec) 权限的简写语法：*
```yaml
    tmpfs:
      - /tmp:mode=1777,size=1024m,exec
```

### 3. 在 Compose 中启用 IPv6（自定义网络）
这是官方推荐的 IPv6 玩法，不影响全局：
```yaml
networks:
  ip6net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: fd00:1234::/64  # 使用 ULA 私网地址
```
测试运行：
```bash
docker run --rm --network ip6net -p 80:80 traefik/whoami
curl http://[::1]:80
```

---

## 五、 日常高频命令速查

### 1. 容器状态与资源监控
*   **按内存使用量排序：**
    `docker stats --no-stream | sort -k4 -h -r`
*   **按 CPU 使用率排序：**
    `docker stats --no-stream | sort -k3 -h -r`
*   **基础查看：**
    `docker ps -a` (列出所有容器)
    `docker logs -f <容器名>` (跟踪日志)

### 2. 容器执行与文件交互
*   **进入容器内部：**
    `docker exec -it <容器名> /bin/bash` (或 `/bin/sh`)
*   **宿主机与容器互传文件：**
    `docker cp /本地/路径 <容器名>:/容器/路径`
    `docker cp <容器名>:/容器/路径 /本地/路径`

**实战：MySQL 数据库无缝备份与还原**
*   备份导出：
    `docker exec <db_container> mysqldump -uroot -p<PASSWORD> test_db > /opt/test_db.sql`
*   还原导入（注意用 `-i` 而不是 `-it`）：
    `docker exec -i <db_container> mysql -uroot -p<PASSWORD> test_db < /opt/test_db.sql`

### 3. Compose 项目管理
*   启动/后台运行：`docker compose up -d`
*   重启特定服务：`docker compose restart <服务名>`
*   **彻底清理重置（危险）：**
    `docker compose down --volumes --rmi all --remove-orphans`
    *(此操作会停止容器并删除所有挂载卷、镜像及孤儿容器)*

### 4. 系统垃圾清理 (减肥)
随着使用时间增加，定期清理可释放大量空间。
*   安全清理（清理无标签的悬空镜像）：`docker image prune`
*   卷清理（清理未挂载的匿名卷）：`docker volume prune`
*   **核弹级清理（慎用）：** `docker system prune -a` (清理所有停止的容器、未使用的镜像和网络)。

---

## 六、 附录：Dockerfile 与 Podman

### 1. Dockerfile 常用指令速查

| 指令 | 说明 | 示例 |
| :--- | :--- | :--- |
| **FROM** | 指定基础镜像 (必须在首行) | `FROM ubuntu:20.04` |
| **WORKDIR** | 设置工作目录 | `WORKDIR /app` |
| **COPY** | 复制本地文件到镜像内 | `COPY . /app` |
| **RUN** | 构建时执行的 Shell 命令 | `RUN apt-get update && apt-get install -y vim`|
| **CMD** | 容器启动时默认命令 (可被覆盖) | `CMD ["python3", "app.py"]` |
| **ENTRYPOINT**| 容器启动入口 (不可被直接覆盖) | `ENTRYPOINT ["nginx", "-g", "daemon off;"]`|
| **ENV** | 设置环境变量 | `ENV PORT=8080` |
| **EXPOSE** | 声明开放端口 (仅作文档作用) | `EXPOSE 8080` |
| **VOLUME** | 定义匿名数据卷挂载点 | `VOLUME ["/data"]` |

### 2. Podman 与 Docker 的差异
Podman 与 Docker 命令高度兼容（可直接通过 `alias docker=podman` 替代使用），但有以下核心区别：
1.  **无守护进程 (Daemonless)**：Podman 没有常驻后台的 `dockerd`，直接由当前用户触发，安全性更高。
2.  **网络模式限制**：因为无守护进程权限隔离，Podman 经常使用 `--network=host` 模式启动。
3.  **Pod 原生支持**：提供了 `podman pod ls`，可像 Kubernetes 一样管理 Pods。