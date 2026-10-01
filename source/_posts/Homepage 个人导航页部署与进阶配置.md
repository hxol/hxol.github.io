---
title: Homepage 个人导航页部署与进阶配置
date: 2026-09-30 15:30:00
tags: [笔记, Homepage, 自托管]
---

# Homepage 个人导航页部署与进阶配置

## 1. 项目信息

- **官方仓库**: [gethomepage/homepage](https://github.com/gethomepage/homepage)
- **官方文档**: [gethomepage.dev](https://gethomepage.dev/)

---

## 2. 基础环境部署 (Linux)

### 2.1 创建挂载目录

将配置文件、图标和图片目录映射到宿主机，方便后续备份和修改：

```bash
sudo mkdir -p /opt/homepage/{config,icons,images}
cd /opt/homepage
```

### 2.2 配置权限 (UID 与 GID)

为了避免 Docker 挂载数据卷时出现权限问题，建议先确认当前用户的 `uid` 与 `gid`：

```bash
id
```
*(假设输出为 `uid=1000(user) gid=1000(user)`，请将下方命令和后续 `.env` 中的数字替换为你实际的 ID。这里以 1000 和 1001 为例。)*

赋予目录相应的归属和读写权限：

```bash
sudo chown -R 1000:1001 /opt/homepage 
sudo chmod -R 755 /opt/homepage
```

### 2.3 配置环境变量 (.env)

使用 `.env` 文件来管理变量，便于维护和保障安全：

```bash
sudo nano /opt/homepage/.env
```
填入以下内容：

```env
# 镜像版本
HOMEPAGE_IMAGE=ghcr.io/gethomepage/homepage:latest

# 运行权限设置 (请与前面 id 命令获取的数值保持一致)
APP_USER_ID=1000
APP_GROUP_ID=1001

# 允许访问的白名单 (增强安全性，防止 DNS 重新绑定攻击)
# 请填写你的宿主机IP:端口 或 实际绑定的域名，多个用逗号隔开
HOMEPAGE_ALLOWED_HOSTS=192.168.60.99:1026,homepage.lab.io
```

### 2.4 编写 Docker Compose 文件

```bash
sudo nano /opt/homepage/docker-compose.yaml
```
填入以下内容：

```yaml
services:
  homepage:
    image: ${HOMEPAGE_IMAGE}
    container_name: homepage
    ports:
      - 1026:3000
    volumes:
      - ./config:/app/config
      - ./icons:/app/public/icons
      - ./images:/app/public/images
      # - /var/run/docker.sock:/var/run/docker.sock:ro # (可选) 开启后可通过 Docker 引擎自动发现本地服务
    environment:
      HOMEPAGE_ALLOWED_HOSTS: ${HOMEPAGE_ALLOWED_HOSTS}
      PUID: ${APP_USER_ID}
      PGID: ${APP_GROUP_ID}
      TZ: Asia/Shanghai # 设置时区
    restart: unless-stopped
```

### 2.5 启动与状态管理

**启动服务**：
```bash
cd /opt/homepage && docker compose up -d
```

**查看运行状态与日志**：
```bash
# 查看容器状态
cd /opt/homepage && docker compose ps

# 实时查看运行日志
cd /opt/homepage && docker compose logs -f homepage
```

---

## 3. 基础配置指南

Homepage 的所有配置都在 `/opt/homepage/config` 目录下的 `.yaml` 文件中。
> 💡 **提示**：Homepage 支持**热重载 (Hot Reload)**。当你修改并保存这些 `.yaml` 文件后，只需刷新浏览器页面即可看到变化，**通常不需要**重启 Docker 容器。

- **书签** (`bookmarks.yaml`): 用于分类存放常用的外链书签。
- **全局设置** (`settings.yaml`): 页面主题、布局、搜索引擎、标题等基础设置。
- **服务监控** (`services.yaml`): 核心文件，用于添加服务卡片和配置 API 数据获取（如 PVE 状态）。
- **信息小部件** (`widgets.yaml`): 顶部显示的小组件（如天气、系统资源、时间等）。

如果你修改了 `.env` 或遇到了热重载失效的情况，可以手动重启容器以应用配置：
```bash
cd /opt/homepage && docker compose restart homepage
```

---

## 4. 进阶：服务数据集成 (API 配置)

### 4.1 集成 Proxmox VE (PVE) 状态

为了安全起见，我们需要在 PVE 中为 Homepage 创建一个专属的**只读 (Read-Only)** API 密钥。

#### A. 在 PVE 中生成专用密钥

1. 登录 PVE Web 界面，点击左侧菜单的 **Datacenter (数据中心)**。
2. 展开 **Permissions (权限)** 菜单，点击 **Groups (群组)**。
3. 点击 **Create (创建)** 按钮，命名为一个易于识别的名称，例如 `api-ro-users`。
4. 回到 **Permissions (权限)** 主界面，点击 **Add (添加) -> Group Permission (群组权限)**。
   - **Path (路径)**: `/`
   - **Group (群组)**: 选择第 3 步创建的 `api-ro-users`
   - **Role (角色)**: 选择 `PVEAuditor` (审计员/只读)
   - **Propagate (传播/继承)**: 勾选
5. 展开 **Permissions (权限)** 下的 **Users (用户)**，点击 **Add (添加)** 按钮。
   - **User name (用户名)**: 填写便于识别的名称，如 `api`
   - **Realm (域)**: `Linux PAM standard authentication`
   - **Group (群组)**: 选择 `api-ro-users`
6. 展开 **Permissions (权限)** 下的 **API Tokens (API 令牌)**，点击 **Add (添加)** 按钮。
   - **User (用户)**: 选择刚才创建的 `api@pam`
   - **Token ID (令牌 ID)**: 填写用途标识，如 `homepage`
   - **Privilege Separation (权限分离)**: 取消勾选（或保留默认，因用户本身已是只读）
   - *注意：此时屏幕会弹出 Token Secret (密钥)，请立即复制保存，关闭后将无法再次查看！*
7. **(可选/双保险)** 回到 **Permissions (权限)**，点击 **Add -> API Token Permission (API 令牌权限)**，将刚才的 Token 显式绑定 `/` 路径下的 `PVEAuditor` 角色并勾选继承。

#### B. 在 Homepage 中配置 PVE

编辑 `services.yaml` 文件，填入以下内容。
注意：用户名的格式必须为 `用户名@域!TokenID`（例如：`api@pam!homepage`），密码为刚才保存的 Secret。

```yaml
- 基础设施:
    - Proxmox VE:
        icon: proxmox.png
        href: https://<你的PVE_IP>:8006
        description: 虚拟机宿主机管理
        widget:
          type: proxmox
          url: https://<你的PVE_IP>:8006
          username: api@pam!homepage
          password: <你的_Token_Secret>
          # 允许显示的字段
          fields: ["vms", "lxc", "resources.cpu", "resources.mem"]
          # node: pve1 # (可选) 默认显示整个集群平均值，填入节点名称可单独显示某节点数据
```

---

### 4.2 集成 Technitium DNS Server 状态

同样的，为了安全，我们为 Technitium DNS 创建一个仅有 Dashboard 查看权限的低权限账户。

#### A. 创建低权限用户与获取 Token

1. 登录 Technitium 管理后台，进入 **Settings -> Administration -> Groups -> Add Group**。
   - **Name**: `View`
   - **Description**: `专为 homepage 创建的低权限组`
   - 点击保存。
2. 导航到 **Settings -> Administration -> Permissions**，找到 `Dashboard`，点击右侧的三个竖点选择 **Edit Permissions**。
   - 点击 **Add Group**。
   - 选择刚才创建的 `View` 组，在 Group Permissions 中赋予 `View` 权限。
   - 点击保存。
3. 导航到 **Settings -> Administration -> Users -> Add User**。
   - **Display Name**: `homepage-stats`
   - **Username**: `homepage-stats`
   - 填写并确认密码。
4. 将新建的用户 `homepage-stats` 加入到 `View` 组中。
5. **注销当前管理员账号**，使用 `homepage-stats` 登录后台。
6. 点击右上角的 **用户名 -> Create API Token**，生成并复制该 Token。

#### B. 在 Homepage 中配置 Technitium DNS

编辑 `services.yaml` 文件，将刚才获取的 API Token 填入：

```yaml
- 网络服务:
    - Technitium DNS:
        icon: technitium.png
        href: http://<你的Technitium_IP>:5380
        description: 本地 DNS 服务器与广告拦截
        widget:
          type: technitium
          url: http://<你的Technitium_IP>:5380
          key: <你的_API_Token>
          range: LastDay
          fields:
            - totalQueries
            - totalBlocked
            - totalCached
            - totalClients
```
