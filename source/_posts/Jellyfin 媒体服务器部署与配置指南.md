---
title: Jellyfin 媒体服务器部署与配置指南
date: 2026-09-30 15:32:00
tags: [笔记, Jellyfin, 自托管]
---

# Jellyfin 媒体服务器部署与配置指南

**项目信息**：本指南基于 [linuxserver/jellyfin Docker 镜像](https://github.com/linuxserver/docker-jellyfin/pkgs/container/jellyfin)。该镜像自带易用的权限管理（PUID/PGID）及丰富的第三方 MOD 支持。

---

## 1. 环境准备与安装

### 1.1 创建目录结构

建议将配置文件与媒体数据分离，使用相对路径或统一挂载点。以下以 `/opt/jellyfin` 作为配置目录为例：

```bash
mkdir -p /opt/jellyfin/config
mkdir -p /opt/jellyfin/cache
mkdir -p /opt/jellyfin/fonts
cd /opt/jellyfin
```

### 1.2 配置权限 (PUID & PGID)

为了避免 Docker 产生的文件出现权限问题，建议使用当前非 root 用户的 UID 和 GID。

**查看当前用户的 uid 与 gid：**
```bash
# 获取 uid (例如输出: 1000)
id -u

# 获取特定用户组的 gid (如果使用默认用户，直接输入 id -g 即可)
# 假设有一个专门的媒体用户组 media-group：
grep '^media-group:' /etc/group | cut -d: -f3
```

**修改目录权限：**
（假设查询到的 uid 为 `1000`，gid 为 `1000`）
```bash
sudo chown -R 1000:1000 /opt/jellyfin
sudo chmod -R 755 /opt/jellyfin
```

### 1.3 创建自定义 Docker 网络（可选）

为便于容器间通信或接入反向代理，可创建自定义网络：
```bash
# 创建公共网络 (用于对外服务/反代等)
sudo docker network create public-net

# 创建私有网络 (用于数据库等后端服务通信)
sudo docker network create privacy-net
```

> **注意**：请确保系统防火墙放行了这些 Docker 网络的网段及 Jellyfin 的端口（默认 `8096`）。

### 1.4 配置环境变量 (.env)

在项目根目录创建 `.env` 文件，方便集中管理变量：

```bash
vi /opt/jellyfin/.env
```
写入以下内容：
```env
JELLYFIN_IMAGE=ghcr.io/linuxserver/jellyfin:latest
PUID=1000
PGID=1000
TZ=Asia/Shanghai
URL=https://jellyfin.example.com  # 替换为你的实际域名
```

### 1.5 编写 Docker Compose 文件

```bash
vi /opt/jellyfin/docker-compose.yaml
```
写入以下内容（请根据实际情况修改 `/data/media` 为你硬盘上的媒体路径）：

```yaml
services:
  jellyfin:
    image: ${JELLYFIN_IMAGE}
    container_name: jellyfin
    shm_size: "10g" # 挂载 10G 内存盘用作视频转码缓存，减少硬盘损耗
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - JELLYFIN_PublishedServerUrl=${URL}
      # 解决中文字体变方块的问题（推荐开启）
      # - DOCKER_MODS=linuxserver/mods:universal-package-install
      # - INSTALL_PACKAGES=fonts-noto-cjk-extra
    volumes:
      - ./config:/config
      # 如果使用 /dev/shm 作为转码缓存，建议注释掉下一行的本地 cache 挂载
      # - ./cache:/cache
      - ./fonts:/usr/local/share/fonts/custom:ro
      # --- 媒体库路径挂载 ---
      - /data/media/Movies:/Movies:ro
      - /data/media/TV-series:/TV-series:ro
      - /data/media/Cartoons:/Cartoons:ro
      - /data/media/Documentaries:/Documentaries:ro
      - /data/media/Variety:/Variety:ro
      - /data/media/Others:/Others:ro
    ports:
      - 1051:8096 # 主机端口 1051 映射到容器 8096
      # - 8920:8920 # HTTPS 端口 (如果有需求)
      # - 1900:1900/udp # DLNA 发现端口
    # devices: 
      # --- 硬件解码挂载 (补充) ---
      # - /dev/dri:/dev/dri # Intel/AMD 核显直通
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: "2"      # 根据机器性能调整
          memory: "20G"  # 限制最大内存占用
    networks:
      - public-net

networks:
  public-net:
    external: true
```

### 1.6 启动与维护

**后台启动容器：**
```bash
sudo docker compose up -d
```

**查看运行状态与日志：**
```bash
sudo docker compose ps
sudo docker compose logs -f jellyfin
```

---

## 2. 系统设置与优化

### 2.1 视频转码设置

使用内存（RAM Disk）作为转码缓存，可以大幅提升转码响应速度并保护硬盘寿命。

1. 进入 Jellyfin 网页端：**我的 -> 控制台 -> 播放 -> 转码**。
2. **转码路径**：填入 `/dev/shm`。
3. **防止内存耗尽设置**：
   * 勾选 ✅ **限制转码速度**（Throttle Transcoding）
   * 勾选 ✅ **分段删除**（分段保留时间建议设置小一些）
4. 保存设置。

> **补充说明**：若要实现真正的硬件加速（如 Intel QuickSync），需要在 `docker-compose.yaml` 中取消 `devices: - /dev/dri:/dev/dri` 的注释，并在 Jellyfin 的“硬件加速”选项中选择对应的硬件解码器。

### 2.2 插件推荐与安装

由于默认刮削器对中文环境支持有限，建议按需安装以下第三方插件。**请勿盲目全部安装**，多余的插件反而可能导致刮削到大量无关或外语内容。官方的 TMDb 插件已内置且支持中文，通常作为主力使用。

在 **控制台 -> 插件 -> 存储库** 中添加以下链接：

*   **Jellyfin 官方稳定版库**:
    `https://repo.jellyfin.org/releases/plugin/manifest-stable.json`
*   **MetaShark (推荐)**: 
    主要从豆瓣获取影视信息，配合 TMDB 补全数据，对动画命名格式兼容极好。
    `https://github.com/cxfksword/jellyfin-plugin-metashark/releases/download/manifest/manifest.json`
*   **MeiamSubtitles (中文字幕)**: 
    支持自动从迅雷影音、射手网等源精确匹配并下载中文字幕。
    `https://github.com/91270/MeiamSubtitles.Release/raw/main/Plugin/manifest-stable.json`
*   **Bangumi (二次元番剧)**: 
    专为番剧爱好者设计，从国内/外中文源抓取番剧信息和封面。
    `https://jellyfin-plugin-bangumi.pages.dev/repository.json`
*   **Metatube (特殊影片)**: 
    提供从不同来源抓取元数据的后端服务。
    `https://raw.githubusercontent.com/metatube-community/jellyfin-plugin-metatube/dist/manifest.json`

### 2.3 开启 IPv6 支持

Jellyfin 的 IPv6 监听需要手动开启：
1. 进入 **控制台 -> 高级 -> 联网 -> IP协议**。
2. 选择允许 IPv4 和 IPv6，然后重启容器。

验证容器内 IPv6 连通性：
```bash
docker exec -it jellyfin /bin/bash
curl 6.ipw.cn
```

### 2.4 解决中文字体显示方块 (□□) 问题

在海报生成封面或内嵌字幕渲染时，若系统缺少中文字体，会出现方块（豆腐块）。

**方法一：使用环境变量自动安装（推荐）**
对于 `linuxserver.io` 版本的镜像，最优雅的方式是在 `docker-compose.yaml` 中添加以下环境变量（本文 1.5 章节已默认添加）：
```yaml
- DOCKER_MODS=linuxserver/mods:universal-package-install
- INSTALL_PACKAGES=fonts-noto-cjk-extra
```
*此方法在容器每次更新或重建后都会自动生效，详见 [Issue #161](https://github.com/linuxserver/docker-jellyfin/issues/161)。*

**方法二：手动进入容器安装（不推荐，重建容器后失效）**
```bash
docker exec -it jellyfin /bin/bash
apt update && apt install fonts-noto-cjk-extra fontconfig -y
fc-cache -fv
fc-list :lang=zh | head # 检查系统是否已识别 Noto Sans CJK 字体
```

---

## 3. 常见问题排查

### 3.1 媒体库扫描缓慢 / UI 界面卡顿
Jellyfin 在面对海量文件首次扫描时，存在数据库锁和 I/O 争用问题，可能导致扫描耗时数天甚至在此期间 UI 假死。
*   **现状**：官方团队已在 GitHub 上追踪该性能瓶颈（详见 [Issue #2600](https://github.com/jellyfin/jellyfin/issues/2600) 和 [Issue #6906](https://github.com/jellyfin/jellyfin/issues/6906)），但在最新版本中完全解决仍需时间。
*   **建议**：
    *   将大型媒体库分批次添加。
    *   在高级设置中关闭“提取章节图像”和“提取视频预览图”（这两项极度消耗性能）。
    *   耐心等待首次扫描完成，后续增量扫描速度会恢复正常。

### 3.2 无法实时监控文件变动 (inotify 失效)
如果你的媒体是通过 NFS/SMB 网络挂载的，或者文件数量过多，会导致 Linux 内核的 `inotify` 监控失效，Jellyfin 无法自动发现新文件。

**修复方法：增加宿主机的 inotify 监听上限**
在宿主机（非容器内）执行：
```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
*注：网络挂载（如 Rclone/NFS 默认不支持 inotify 通知）通常需要结合第三方脚本（如 `autoscan`）或改用定时扫描机制。*

---

## 4. 媒体库文件命名规范

遵守严格的目录与命名结构，是 100% 准确刮削元数据的前提。

为了提高识别准确率，强烈建议在文件夹名称后加上**年份**，或直接加上**媒体数据库的 ID**。
*例如：* `电影名称 (年份) [tmdbid-XXXXXX]`。

### 4.1 标准电影目录结构
电影必须位于媒体库根目录，或为其单独建立子文件夹（推荐，方便存放字幕和海报）。
```text
电影根目录/
├── 金玫瑰洞 (1991).mp4
├── 伊豆舞女 (1954).mp4
├── 低俗小说 (1994) [tmdbid-680]/
│   └── 低俗小说.mkv
└── 白头神探 (1988)/
    ├── 白头神探-cd1.avi
    └── 白头神探-cd2.avi
```

### 4.2 电影的多版本共存
如需将同一部电影的不同版本（如 1080P、4K、导演剪辑版）合并显示在一个页面，前缀必须完全一致。
**规则**：`电影名称 (年份) - 标签.扩展名` （注意连字符 `-` 前后各有一个空格）。

```text
电影根目录/
└── 朗读者 (2008) [imdbid-tt0976051]/
    ├── 朗读者 (2008) [imdbid-tt0976051] - 1080p.mp4
    ├── 朗读者 (2008) [imdbid-tt0976051] - 2160p.mp4
    └── 朗读者 (2008) [imdbid-tt0976051] - 导演剪辑版.mp4
```

### 4.3 切片电影的合并 (Split files)
如果一部电影被分割为多个文件，可以通过特定的后缀让系统自动无缝播放。
**支持的标签**：`cd`, `dvd`, `part`, `pt`, `disc`, `disk`。
**分隔符**：空格、`.`、`-`、`_`。

```text
电影名称 (2010)/
├── 电影名称-part1.mkv
└── 电影名称-part2.mkv
```

### 4.4 额外花絮与预告片 (Extras)
Jellyfin 支持将幕后花絮、预告片、删减片段等归类到专属目录下，或使用特定文件名后缀。

**方式一：使用特定子目录**
支持的文件夹名称包括：`behind the scenes` (幕后), `deleted scenes` (删减), `interviews` (访谈), `scenes`, `samples`, `shorts`, `featurettes`, `clips`, `extras`, `trailers`。

```text
金玫瑰洞 (2019)/
├── 金玫瑰洞 (2019).mp4
├── behind the scenes/
│   └── 拍摄花絮.mp4
└── interviews/
    └── 导演访谈.mp4
```

**方式二：使用特定后缀**
在主文件名后加上连字符及花絮类型，如 `-trailer`, `-sample`, `-behindthescenes`, `-deleted`, `-interview`。

```text
Best_Movie_Ever (2019)/
├── Best_Movie_Ever (2019).mp4
├── Release Trailer-trailer.mp4
├── Teaser-sample.mp4
└── Making of The Movie-behindthescenes.mp4
```
此外，单独命名为 `trailer.mp4`（预告）或 `theme.mp3`（主题曲音频）也可被自动识别。

### 4.5 3D 电影的识别
在文件名中加入 `.3D.` 标签以及格式后缀，Jellyfin 即可自动识别 3D 文件。
*   `hsbs` / `fsbs` = 左右半宽 / 左右全宽
*   `htab` / `ftab` = 上下半高 / 上下全高
*   `mvc` = 多视图视频编码

```text
Awesome 3D Movie (2022).3D.FTAB.mp4
Awesome 3D Movie (2022)-3d-hsbs.mp4
```

> **参考来源**：更多详细命名规则请查阅 [Jellyfin 官方文档](https://jellyfin.org/docs/general/server/media/movies/)。
