---
title: 基于 Docker 部署 qBittorrent 并挂载 OneDrive 教程
date: 2026-09-30 15:30:00
tags: [笔记, qBittorrent, Rclone, 自托管]
---

# 基于 Docker 部署 qBittorrent 并挂载 OneDrive 教程

## 核心逻辑说明
1. **本地获取配置**：OneDrive 的 OAuth2 授权需要浏览器跳转，VPS 通常没有图形化界面，难以直接完成。因此，先在本地电脑（Windows/Mac）配置好 Rclone，获取授权后的 `rclone.conf` 配置文件。
2. **Docker 部署 Rclone**：以容器方式在 VPS 上运行 Rclone，将 OneDrive 挂载到 VPS 的宿主机目录。
3. **Docker 部署 qBittorrent**：将上一步挂载好的宿主机目录映射给 qBittorrent 容器，实现下载文件直接写入 OneDrive。

## 涉及开源项目
* [rclone/rclone](https://github.com/rclone/rclone)
* [linuxserver/docker-qbittorrent](https://github.com/linuxserver/docker-qbittorrent)

---

### 第一步：在本地电脑获取 Rclone 配置

1. 在你的本地电脑下载并解压 [Rclone](https://github.com/rclone/rclone/releases)。
2. 在 Rclone 所在文件夹打开命令行（Windows 建议使用 PowerShell 或 CMD，Mac 使用终端），输入 `rclone config`（或 `./rclone.exe config`）。
3. 按照以下交互提示进行操作：
   * `No remotes found, make a new one?` -> 输入 **`n`** (New remote)
   * `Enter name for new remote.` -> 输入 **`onedrive`** （注意：此名称需与后续步骤保持一致）
   * `Choose a number from below, or type in your own value.` -> 选择 **`Microsoft OneDrive`** (通常是数字 42 左右，请根据实际列表选择)
   * `client_id` / `client_secret` -> **直接回车留空**
   * `Choose national cloud region for OneDrive.` -> 选择 **`Microsoft Cloud Global`** (通常为 1)
   * `ID of the service principal's tenant.` -> **直接回车留空**
   * `Edit advanced config?` -> 输入 **`n`**
   * `Use web browser to automatically authenticate rclone with remote?` -> 输入 **`y`**。此时会自动弹出浏览器，登录你的 Microsoft 账号并授权。
   * 授权成功后回到命令行 `Type of connection:` -> 选择 **`OneDrive Personal or Business`** (通常为 1)
   * `config_driveid` -> 选择提示的 **`OneDrive (personal)`** 或对应盘符
   * 最后确认信息无误，保存并退出配置。
4. **获取配置文件内容**：
   * 在命令行输入 `rclone config file` 查看配置文件所在的完整路径。
   * 用记事本打开该 `rclone.conf` 文件，内容类似如下：
```ini
[onedrive]
type = onedrive
token = {"access_token":"xxxxx...","expiry":"202x-xx-xxTxx:xx:xx"}
drive_id = 你的专属Drive_ID # 这里会是一串字母和数字
drive_type = personal
```
   * **请将该文件保留，稍后在 VPS 上会用到其中的全部内容。**

---

### 第二步：VPS 环境准备工作

1. 通过 SSH 连接到你的 VPS。
2. 安装 Docker 和 Docker Compose（如已安装可跳过）。
3. **关键步骤：安装 FUSE 支持**（Rclone 挂载网盘必须依赖该组件）：
```bash
sudo apt update && sudo apt install -y fuse3
```
4. 创建项目所需的目录结构：
```bash
sudo mkdir -p /opt/qb-drive/{config,qb_config,onedrive} && cd /opt/qb-drive
```
5. 创建并写入 Rclone 配置文件：
```bash
sudo nano /opt/qb-drive/config/rclone.conf
```
将你在**第一步**获取到的 `rclone.conf` 完整内容粘贴进去，保存并退出（在 nano 中按 `Ctrl+O` 回车保存，按 `Ctrl+X` 退出）。

6. 设置目录权限（防止容器内应用没有写入权限）：
```bash
sudo chown -R 1000:1000 /opt/qb-drive
```

---

### 第三步：编写 Docker Compose 编排文件

在 `/opt/qb-drive` 目录下创建 `docker-compose.yml` 文件：
```bash
cd /opt/qb-drive && sudo nano docker-compose.yml
```

粘贴以下内容（**请仔细阅读注释并根据需要调整**）：

```yaml
services:
  rclone:
    image: rclone/rclone:latest
    container_name: rclone
    restart: unless-stopped
    environment:
      - TZ=Asia/Shanghai
      - PUID=1000   # 根据你的系统用户 UID 调整
      - PGID=1000   # 根据你的系统用户 GID 调整
    volumes:
      - ./config:/config/rclone
      - ./onedrive:/data:shared       # :shared 允许挂载传播给宿主机和其他容器，极重要！
      - /etc/passwd:/etc/passwd:ro
      - /etc/group:/etc/group:ro
    devices:
      - /dev/fuse:/dev/fuse
    cap_add:
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    # 下方的 onedrive:/ 必须与你在第一步中创建的 remote 名称完全一致
    command: mount onedrive:/ /data --allow-other --allow-non-empty --vfs-cache-mode full --vfs-cache-max-size 10G --vfs-cache-max-age 24h --buffer-size 32M

  qbittorrent:
    image: ghcr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    depends_on:
      - rclone
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - ./qb_config:/config           # qB的配置文件目录
      - ./onedrive:/downloads         # 直接映射 Rclone 挂载后的宿主机目录
    ports:
      - 8080:8080
      - 51413:6881
      - 51413:6881/udp
```

#### 📌 Rclone 参数重点解释：
* **`--vfs-cache-mode full`**：**极其重要**。OneDrive 原生不支持随机读写（而 qBittorrent 下载正是碎片化的随机写入）。开启此参数后，下载的文件会先缓存在 VPS 的本地磁盘中，等单个文件块完成后再异步上传到 OneDrive。如果不加此参数，qBittorrent 会直接报 I/O 错误。
* **`--vfs-cache-max-size 10G`**：本地缓存的磁盘上限。请务必根据你 VPS 的实际磁盘剩余空间进行调整。
* **`devices` & `cap_add`**：赋予该容器在宿主机上挂载文件系统的底层权限。

---

### 第四步：启动与基础设置

1. 在 `/opt/qb-drive` 目录下启动容器组：
```bash
cd /opt/qb-drive && sudo docker compose up -d
```
2. 验证挂载是否成功：
```bash
ls -l /opt/qb-drive/onedrive
```
如果你能看到 OneDrive 网盘里的原有文件，说明挂载已成功。

3. **访问 qBittorrent WebUI**：
   * 浏览器打开 `http://你的VPS_IP:8080`。
   * 默认账号：`admin`
   * 默认密码：新版 qBittorrent 会在日志中生成随机临时密码，可通过以下命令获取：
     ```bash
     sudo docker logs qbittorrent | grep password
     ```
   *(注：登录后请务必第一时间在设置中修改为您自己的密码。)*

4. **修改下载保存路径**：
   * 进入 qB 的 `Tools` (工具) -> `Options` (选项) -> `Downloads` (下载)。
   * 将 `Default Save Path` (默认保存路径) 修改为 **/downloads**。

---

## 进阶设置与排障

### 1. 关闭严格的主机头验证 (防止 WebUI 无法登录)
新版 qBittorrent 为了防范 DNS Rebinding 攻击，默认开启了主机头验证，有时会导致通过 IP 访问 WebUI 报错。

**步骤 1：必须先停止 qB 容器**（否则修改无效，会被运行中的进程覆盖）
```bash
sudo docker compose stop qbittorrent
```
**步骤 2：修改配置文件**
```bash
sudo nano ./qb_config/qBittorrent/qBittorrent.conf
```
找到 `[Preferences]` 标签段落，在其下方添加（或修改）以下两行参数：
```ini
WebUI\HostHeaderValidation=false
WebUI\CSRFProtection=false
```
*(保存退出：`Ctrl+O` 回车，`Ctrl+X`)*
**步骤 3：重启容器**
```bash
sudo docker compose start qbittorrent
```
**步骤 4**：请**开启浏览器的无痕模式 (Ctrl+Shift+N)** 或清除缓存后，再次尝试访问 WebUI。

### 2. 配置防火墙
如果你的 VPS 开启了 UFW 防火墙，请放行 qBittorrent 所需的端口：
```bash
# 放行 WebUI 端口
sudo ufw allow 8080/tcp

# 放行 BT 通讯端口 (对应 docker-compose 映射的外部端口)
sudo ufw allow 51413/tcp
sudo ufw allow 51413/udp
```

---

### ⚠️ 重要避坑指南（必读）

1. **API 频率限制**：OneDrive 存在 API 调用频率限制。如果你同时启动几十上百个种子任务，极易触发微软的 API 风控，导致网盘被暂时熔断封禁。建议控制并发下载数量。
2. **本地磁盘空间预留**：虽然最终文件归档在 OneDrive，但因为开启了 `--vfs-cache-mode full`，**下载过程中必须占用 VPS 本地磁盘空间作为中转缓存**。下载完成并成功上传后，缓存会自动释放。**切勿一次性下载体积超过你 VPS 本地剩余硬盘容量的文件！**
3. **上传延迟**：下载完成后，Rclone 需要在后台静默上传至网盘。在 qBittorrent 中看到进度达 100% 时，并不意味着网盘端立即可见，请耐心等待 Rclone 跑完上传队列。
4. **读写权限报错**：如果在 qB 中发现状态为“错误”且日志提示无法写入，可尝试在宿主机执行授权：`sudo chmod -R 777 /opt/qb-drive/onedrive`。
5. **挂载稳定性**：网络波动偶尔会导致 Rclone 挂载掉线。如果发现 `/opt/qb-drive/onedrive` 目录变为空白，直接执行 `sudo docker compose restart rclone` 即可恢复。

> **💡 最佳实践建议**：本方案非常适合日常轻量级的追剧观影下载。但如果你是重度 PT 玩家（保种狂魔），频繁的 I/O 会给 Rclone 带来巨大压力。针对重度需求，更推荐的方案是：**先将文件纯粹下载到 VPS 本地磁盘，再利用 Rclone 的 `move` 配合 Cron 定时脚本，在闲时周期性搬运到网盘**。
