---
title: 使用 Docker 部署 Baikal 日历通讯录服务及 Mac 客户端配置指南
date: 2026-09-30 15:39:00
tags: [笔记, Baikal, 自托管]
---

# 使用 Docker 部署 Baikal 日历通讯录服务及 Mac 客户端配置指南

## 项目信息简介

Baikal 是一款轻量级的 CalDAV 和 CardDAV 服务器，可以帮助你轻松搭建属于自己的日历和通讯录同步服务，摆脱对第三方云服务的依赖。

*   **Baikal 官方仓库**: [sabre-io/Baikal](https://github.com/sabre-io/Baikal)
*   **Docker 镜像参考**: [ckulka-baikal-docker](https://github.com/ckulka/baikal-docker)

---

## 1. 镜像准备

> **说明**：Baikal 官方目前没有提供 Docker 镜像。你可以参考 [ckulka/baikal-docker](https://github.com/ckulka/baikal-docker) 编写 Dockerfile 来制作属于自己的镜像，或者直接拉取社区已有的镜像（如 `ckulka/baikal:0.12.1`）。
> *本文以下步骤假设你已准备好镜像，或直接使用社区版镜像。*

---

## 2. 安装与部署

### 2.1 创建必要目录并进入工作区
我们需要为 Baikal 创建数据持久化目录。
```bash
sudo mkdir -p /opt/baikal/{config,Specific/db} 
cd /opt/baikal
```

### 2.2 修改目录权限
容器内部默认使用 `www-data`（UID: 33）用户运行服务，因此需要将挂载目录的所有权赋予该用户，以避免读写报错。
```bash
sudo chown -R 33:33 config Specific 
sudo chmod -R 755 config Specific
```

### 2.3 编写编排文件
创建 `docker-compose.yml` 文件：
```bash
sudo nano docker-compose.yml
```
填入以下内容（请根据注释修改为你自己的配置）：

```yaml
services:
  baikal:
    # 如果你自行打包了镜像，请替换为你的本地镜像名（例如 local/baikal:0.12.1）
    # 否则可以直接使用社区镜像：ckulka/baikal:0.12.1
    image: ckulka/baikal:0.12.1 
    container_name: baikal
    restart: always
    ports:
      - 1072:80
    environment:
      # 设置时区，对日历时间的准确同步非常重要
      - TZ=Asia/Shanghai
      # 设置为你规划的域名，减少 Apache 日志报错；如果没有域名则填 localhost
      - BAIKAL_SERVERNAME=baikal.example.com
      # 开启后，容器启动时会自动修复挂载目录的权限（建议保持 0/开启 状态）
      - BAIKAL_SKIP_CHOWN=0
    volumes:
      # 数据持久化映射
      - ./config:/var/www/baikal/config
      - ./Specific:/var/www/baikal/Specific
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: "300M"
```

### 2.4 启动服务
确认在 `/opt/baikal` 目录下，执行以下命令启动容器：
```bash
sudo docker compose up -d
```
> **提示**：启动后，由于对外暴露的是 `1072` 端口，建议你通过 Nginx、Nginx Proxy Manager 或 Caddy 等工具**配置反向代理并申请 SSL 证书**。苹果设备的 CalDAV/CardDAV 协议对 HTTPS 有着严格要求。

---

## 3. Web 后台初始化与用户创建

在配置 Mac 客户端之前，你需要先在 Baikal 管理后台创建用户账号：
1. 浏览器访问你的域名（或 `http://IP:1072`）。
2. 首次进入会进行初始化设置，设置管理员密码（Admin Password）。
3. 初始化完成后，进入 **Users and groups** 菜单。
4. 点击 **+ Add User**，创建一个新用户（例如用户名为 `<你的用户名>`），并设置用户密码。**（此账号密码将用于 Mac 客户端登录）**。

---

## 4. 客户端配置指南

将自托管的 Baikal 添加到 Mac 自带的“日历”和“通讯录”App 中非常简单，但 **Mac 对服务器路径的填写有特定的要求**。

通常情况下，Mac 不需要你提供具体某个日历/通讯录的冗长链接（如 `.../calendars/<你的用户名>/default/`），而是需要**主账号路径（Principal URL）**，Mac 会自动发现该账号下的所有资源。

### 4.1 在 Mac 的「日历」App 添加 Baikal 源

**推荐方法：使用“高级”设置添加**

1. 打开 Mac 上的 **日历** App。
2. 在屏幕左上角的菜单栏中，点击 **日历** > **设置...**（较老版本的 macOS 叫“偏好设置”）。
3. 切换到 **账户** 标签页。
4. 点击左下角的 **`+`** 号添加新账户。
5. 选择 **其他 CalDAV 账户...**，然后点击“继续”。
6. 在弹出的窗口中，将 **账户类型** 下拉菜单改为 **高级**。
7. **按照以下格式准确填写（非常重要）**：
   * **用户名**：`<你的用户名>` （在 Baikal Web 后台创建的用户名）
   * **密码**：你为该用户设置的密码
   * **服务器地址**：`baikal.example.com` （你的实际域名）
   * **服务器路径**：`/dav.php/principals/<你的用户名>/`  *(⚠️注意：这里填的是 principals 路径，不要填 calendars 的长链接)*
   * **端口**：`443`
   * **使用 SSL**：勾选 *(前提是你已配置好了 HTTPS 反向代理)*
8. 点击 **登录**。

设置成功后，Mac 日历列表左侧就会出现你的 Baikal 账户，名为 `default` 的日历会自动同步过来，你可以右键对其进行重命名。

### 4.2 在 Mac 的「通讯录」App 添加 Baikal 源

添加通讯录（CardDAV）的原理与添加日历完全一致，同样只需要填写 **主账号路径（Principal URL）**。

**推荐方法：使用“高级”设置添加**

1. 打开 Mac 上的 **通讯录** App。
2. 在屏幕左上角的菜单栏中，点击 **通讯录** > **设置...**。
3. 切换到 **账户** 标签页。
4. 点击左下角的 **`+`** 号添加新账户。
5. 选择 **其他通讯录账户...**，然后点击“继续”。
6. 在弹出的窗口中，将 **账户类型** 的下拉菜单改为 **高级**（如果是较新系统，直接在“账户类型”选“手动”即可，界面类似）。
7. **按照以下格式准确填写**：
   * **用户名**：`<你的用户名>`
   * **密码**：对应的用户密码
   * **服务器地址**：`baikal.example.com`
   * **服务器路径**：`/dav.php/principals/<你的用户名>/` *(⚠️注意：和日历一样填 principals，不要填 addressbooks 链接)*
   * **端口**：`443`
   * **使用 SSL**：勾选
8. 点击 **登录**。

---

## 5. 升级与备份操作指南

数据无价，在升级 Baikal 版本或迁移服务器前，请务必按照以下步骤进行备份。

1. **停止当前运行的旧容器**：
   ```bash
   cd /opt/baikal
   sudo docker compose down
   ```

2. **打包备份数据目录**：
   将 `Specific`（包含数据库文件）和 `config`（包含配置文件）打包为 `baikal_backup.tar.gz`。
   ```bash
   sudo tar -czvf baikal_backup.tar.gz ./Specific ./config
   ```

3. **确认备份文件已生成**：
   ```bash
   ls -lh baikal_backup.tar.gz
   ```
   > 建议将打包好的 `.tar.gz` 文件下载到本地电脑或其他安全的地方妥善保管。

4. **恢复数据（如果需要）**：
   只需将压缩包解压覆盖回原目录，并重新执行修改权限命令（`chown -R 33:33 ...`），然后 `docker compose up -d` 即可无损恢复。
