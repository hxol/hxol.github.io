---
title: Jellyfin 媒体服务器部署与配置指南
date: 2026-09-30 15:32:00
tags: [笔记, Jellyfin, 自托管]
---

# Jellyfin 媒体服务器部署与配置指南

---

## 通过 Docker 镜像安装
**项目信息**：本指南的 docker 镜像基于 [linuxserver/jellyfin Docker 镜像](https://github.com/linuxserver/docker-jellyfin/pkgs/container/jellyfin)。该镜像自带易用的权限管理（PUID/PGID）及丰富的第三方 MOD 支持。

### 创建目录结构

建议将配置文件与媒体数据分离，使用相对路径或统一挂载点。以下以 `/opt/jellyfin` 作为配置目录为例：

```bash
mkdir -p /opt/jellyfin/config
mkdir -p /opt/jellyfin/cache
mkdir -p /opt/jellyfin/fonts
cd /opt/jellyfin
```

### 配置权限 (PUID & PGID)

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

### 创建自定义 Docker 网络（可选）

为便于容器间通信或接入反向代理，可创建自定义网络：
```bash
# 创建公共网络 (用于对外服务/反代等)
sudo docker network create public-net

# 创建私有网络 (用于数据库等后端服务通信)
sudo docker network create privacy-net
```

> **注意**：请确保系统防火墙放行了这些 Docker 网络的网段及 Jellyfin 的端口（默认 `8096`）。

### 配置环境变量 (.env)

在项目根目录创建 `.env` 文件，方便集中管理变量：

```bash
sudo nano /opt/jellyfin/.env
```
写入以下内容：
```env
JELLYFIN_IMAGE=ghcr.io/linuxserver/jellyfin:latest
PUID=1000
PGID=1000
TZ=Asia/Shanghai
URL=https://jellyfin.example.com  # 替换为你的实际域名
```

### 编写 Docker Compose 文件

```bash
sudo nano /opt/jellyfin/docker-compose.yaml
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
      # --- 硬件解码挂载---
      # - /dev/dri:/dev/dri # Intel/AMD 核显直通到本机以后才可以
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

### 启动与维护

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

## 使用官方脚本安装 jellyfin 二进制文件

[官方相关文档](https://jellyfin.org/docs/general/installation/linux/#debuntu-debian-ubuntu-and-derivatives-using-apt)

### 下载并验证官方安装脚本

```bash
curl -s https://repo.jellyfin.org/install-debuntu.sh -O && \
curl -s https://repo.jellyfin.org/install-debuntu.sh.sha256sum -O && \
sha256sum -c install-debuntu.sh.sha256sum
```

### 使用脚本安装二进制文件

```bash
sudo bash install-debuntu.sh
```

---

## 系统设置与优化

### 视频转码设置

使用内存（RAM Disk）作为转码缓存，可以大幅提升转码响应速度并保护硬盘寿命。

1. 进入 Jellyfin 网页端：**我的 -> 控制台 -> 播放 -> 转码**。
2. **转码路径**：填入 `/dev/shm`。
3. **防止内存耗尽设置**：
   * 勾选 ✅ **限制转码速度**（Throttle Transcoding）
   * 勾选 ✅ **分段删除**（分段保留时间建议设置小一些）
4. 保存设置。

> **补充说明**：若要实现真正的硬件加速（如 Intel QuickSync），需要在 `docker-compose.yaml` 中取消 `devices: - /dev/dri:/dev/dri` 的注释，并在 Jellyfin 的“硬件加速”选项中选择对应的硬件解码器。

### 插件推荐与安装

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

### 开启 IPv6 支持

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

## 常见问题排查


### 无法实时监控文件变动 (inotify 失效)
如果你的媒体是通过 NFS/SMB 网络挂载的，或者文件数量过多，会导致 Linux 内核的 `inotify` 监控失效，Jellyfin 无法自动发现新文件。

**修复方法：增加宿主机的 inotify 监听上限**
在宿主机（非容器内）执行：
```bash
echo "fs.inotify.max_user_watches=524288" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```
*注：网络挂载（如 Rclone/NFS 默认不支持 inotify 通知）通常需要结合第三方脚本（如 `autoscan`）或改用定时扫描机制。*

---

## 媒体库文件命名规范

### 电视节目 (TV Shows)

可以通过在 Jellyfin 中创建“电视节目” (Shows) 类型的媒体库来添加剧集。

Jellyfin 支持大多数常见的视频格式，如 `mp4` 和 `mkv`。此外，电影和音乐视频还支持 `VIDEO_TS` 和 `BDMV` 文件夹格式，但请注意，它们不支持多版本、多部分合并或外挂字幕/音轨功能。
*注意：不支持 `.iso` 及其他光盘镜像格式（虽然有时能播，但不保证正常工作）。建议将光盘镜像重新封装为 mkv 格式，或提取为 `VIDEO_TS` / `BDMV` 文件夹。*

文件名应尽可能与元数据提供商（如 TMDB、TVDB 等）列出的名称保持一致。切勿在文件名中使用 Jellyfin 的保留字符，否则**会**引发错误。已知会引发问题的字符包括：`<, >, :, ", /, \, |, ?, *`

#### 组织结构
电视节目应首先放入“剧集文件夹”，然后在每个剧集下建立“季文件夹”。

```text
Shows
├── 剧集名称 A (2010)
│   ├── Season 00
│   │   ├── 某特别篇.mkv
│   │   ├── 剧集名称 A S00E01.mkv
│   │   └── 剧集名称 A S00E02.mkv
│   ├── Season 01
│   │   ├── 剧集名称 A S01E01-E02.mkv
│   │   ├── 剧集名称 A S01E03.mkv
│   │   └── 剧集名称 A S01E04.mkv
│   └── Season 02
│       ├── 剧集名称 A S02E01.mkv
│       ├── 剧集名称 A S02E02.mkv
│       ├── 剧集名称 A S02E03 Part 1.mkv
│       └── 剧集名称 A S02E03 Part 2.mkv
└── 剧集名称 B (2018)
    ├── Season 01
    |   ├── 剧集名称 B S01E01.mkv
    |   └── 剧集名称 B S01E02.mkv
    └── Season 02
        ├── 剧集名称 B S02E01-E02.mkv
        └── 剧集名称 B S02E03.mkv
```

单个视频文件中可以包含多个独立集数，但这在系统中只会显示为一个包含了多集元数据的单一词条。建议使用 MKVToolNix 等工具将此类视频拆分为单集文件。

#### 命名规则
剧集文件夹应采用以下格式命名：
`剧集名称 (年份) [元数据提供商ID]`
年份和提供商ID是可选的，但添加它们能大幅提高识别准确率。

* 仅名称：`Jellyfin Documentary.mkv`
* 含年份：`Jellyfin Documentary (2030)`
* 含提供商ID：`Jellyfin Documentary [imdbid-tt00000000]`
* 完整格式：`Jellyfin Documentary (2030) [imdbid-tt00000000]`

季文件夹应命名为 `Season *`（* 为任意数字）。请勿将其缩写为 `S01` 或 `SE01`。为了获得最佳效果，请用 `0` 补齐前面的位数（例如：Season 5 -> Season 05）。另外，不要将季文件夹与视频文件混放在 Shows 的根目录中。

季和集数文件同样也可以添加元数据提供商 ID 以提升匹配精度。

#### 外挂字幕和音轨
可以通过给文件添加后缀名来引入外挂字幕和音轨。后缀通常包含标题或特殊标志。多个标志之间使用 `.` 分隔。

| 类型 | 标志 (Flag) |
| --- | --- |
| 默认 (Default) | `default` |
| 强制/外语片段 (Forced) | `forced`, `foreign` |
| 听障/无障碍 (Hearing Impaired) | `sdh`, `cc`, `hi` |

*注意：`hi` 标志与印地语（Hindi）的缩写冲突。单独写 `hi` 会被识别为印地语；如果与其他语言代码连用（如 `title.en.hi.srt`），则会识别为该语言的听障字幕。*

如果容器本身包含多个流，则标志会被忽略。任何无法解析为语言或标志的任意文本都会被合并，并作为该流的标题显示（例如 `Film.English Commentary.en.mp3` 会被识别为带有“English Commentary”标题的英语音轨）。

#### 附加内容 (Extras)
附加内容可以放置在电影、剧集、季以及音乐视频中，有以下几种添加方式：

**1. 附加内容文件夹**
这是最整洁的方式，将附加视频放入特定名称的子目录中。支持的子文件夹名称包括：
* `behind the scenes` (幕后)
* `deleted scenes` (删减片段)
* `interviews` (访谈)
* `scenes` (片段)
* `samples` (示例)
* `shorts` (短片)
* `featurettes` (花絮)
* `clips` (剪辑)
* `other` 或 `extras` (未知类型的通用分类)
* `trailers` (预告片)
* `theme-music` (主题音乐)
* `backdrops` (背景视频)

**2. 特定文件名**
如果只有单一附加文件，将其命名为特定名称并放在同级目录即可。支持：`trailer`，`sample`。

**3. 文件后缀**
如果你希望所有文件都在同一个文件夹中，可以在文件名后加上特定后缀（注意，除特殊说明外，这些后缀中**不能包含空格**）：
`-trailer`, `.trailer`, `_trailer`, ` trailer` (前面带空格)
`-sample`, `.sample`, `_sample`, ` sample` (前面带空格)
`-scene`, `-clip`, `-interview`, `-behindthescenes`, `-deleted`, `-deletedscene`, `-featurette`, `-short`, `-other`, `-extra`

#### 剧集特别篇 (Show Specials)
特别篇应放在 `Season 00` 文件夹中。可以通过元数据信息将其插入到正常的剧情季中观看（在仪表盘 -> 媒体库 -> 显示中开启对应选项）：
* **Airs before season / episode**：将特别篇插在某季或某集之前播放。
* **Airs after series**：在指定的整个季结束后播放。
如果多个特别篇的位置设置相同，它们将按在特别篇文件夹中的存储顺序播放。

#### 主题媒体 (Theme Media)
通过在背景播放主题音乐或视频，让媒体浏览更具视听体验。
* **歌曲**：`theme.ext` 或放在 `theme-music/` 文件夹中。
* **视频**：放在 `backdrops/` 文件夹中。
*注意：用户需要在客户端设置中开启此功能。优先播放视频。建议使用 WebM(VP9/Opus) 等标准格式，以免转码导致延迟。*

#### 多版本 (Multiple Versions)
Jellyfin 支持同一剧集的不同版本（如 1080p、720p、加长版等）。只要它们在同一个季文件夹中，且按标准命名方案被识别为同一集即可。
UI 中的版本列表按字母顺序排列；但分辨率后缀（以 `p` 或 `i` 结尾）会按分辨率从高到低排列。

#### 3D 视频 (3D Videos)
通过文件名标签识别 3D 视频。首个标签必须是 `3D`，然后结合以下任意格式标签：
`hsbs` (半宽左右格式), `fsbs` (全宽左右格式), `htab` (半高上下格式), `ftab` (全高上下格式), `mvc`。
*支持空格、连字符、点或下划线分隔，且不区分大小写。*

#### 多部分/分盘 (Multiple Parts)
如果视频被分成了多段，支持的标识符包括：`cd`, `dvd`, `part`, `pt`, `disc`, `disk`（配合数字使用）。分隔符支持空格、点、连字符或下划线。
当合并多段文件和多版本文件时，带有多段的文件版本会在下拉列表中优先显示。

#### 元数据图像 (Metadata Images)
图片既可以内嵌在媒体文件中，也可以作为外部文件放置在视频旁（外部图片优先级更高）。

| 类型 | 文件名/用途 |
| --- | --- |
| Primary (主海报/封面) | `cover.jpg`, `folder.jpg` |
| Backdrop (背景图) | `backdrop.webp` (支持加数字实现轮换，如 `backdrop-1.jpg`) |
| Banner (横幅) | `banner.jpg` (用于横幅浏览模式) |
| Logo (标志) | `logo.png` (显示在媒体顶部) |
| Thumb (缩略图) | `thumb.jpg` (用于首页和缩略图模式) |

#### 占位符 (Placeholders)
允许你向库中添加当前没有实际视频文件的条目。只需创建一个与正常媒体同名的空文件，并以 `.disc` 为后缀即可。
支持指定媒介类型，将类型加在 `.disc` 前面即可，例如：`Blade Runner (1982).dvd.disc` 或 `Blade Runner (1982).bluray.disc`。

---

### 音乐 (Music)

音乐专辑必须以文件夹组织，每个文件夹包含且仅包含一张专辑。Jellyfin 不关心专辑目录层级，只要遵循单文件夹单专辑原则即可。主要通过提取音频文件内嵌的元数据进行识别。如果未找到元数据，Jellyfin 才会使用文件名作为曲目标题。

请尽量避免文件名中出现特殊字符：`<, >, :, ", /, \, |, ?, *`

#### 多盘专辑 (Discs)
多盘专辑主要通过元数据标签中的“光盘编号”(disc number) 和“总盘数”(total discs) 来识别。所有光盘的曲目可以全放在同一个文件夹下；也可以分文件夹存放（内嵌元数据优先级始终最高）。

#### 歌词 (Lyrics)
歌词文件必须保存在同一个文件夹中，且文件名必须与对应的音频文件一致，后缀名支持 `.lrc`、`.elrc` 或 `.txt`。
* **同步歌词**：包含时间戳，用户可以在 UI 中点击每一行跳转到该时间点（可使用如 MiniLyrics 等软件生成）。
* **非同步歌词**：仅包含纯文本。

#### 元数据图片
音乐文件夹根目录放置的图片将被作为专辑封面或艺术家头像。如果没有提供外部图片，Jellyfin 将从第一个带有内嵌封面的音频曲目中提取。

#### 文件扩展名与容器 (File Extensions / Containers)
* 仅包含音频的 `MP4` 文件不会被识别为音乐，请将其重命名为 `.m4a`。
* 仅包含音频的 `MKV / WebM` 文件不会被识别为音乐，请重命名为 `.mka`。
* 不支持 `.weba`，请重命名为 `.mka`。
* 带有 WebP 图片或 ID3 标签的 `.flac` 在某些基于 Chromium 的浏览器中可能无法播放。如遇此问题，请在客户端开启“始终重新混流(Remux) FLAC 音频”选项。
* ID3v1 标签对大多数字段有 30 字节的长度限制。建议升级为 ID3v2.4。

如果遇到文件问题，可通过 FFmpeg 重新封装为 `.mka` 容器：
`ffmpeg -i <Input File> -c:a copy <Output File>.mka`

---

### 音乐视频 (Music Videos)

音乐视频的组织方式与电影大致相同，但有几个关键区别：默认不提供外部元数据抓取，并且**支持多层文件夹嵌套**。

#### 组织与命名
因为没有元数据抓取器介入，你可以随心所欲地命名文件夹和视频文件。文件夹和文件的名称会直接作为条目标题在 Jellyfin 中显示。

#### 功能支持
音乐视频同样完美支持 **外挂字幕和音轨**、**多版本**、**3D 视频**、**多部分**、**附加内容** 和 **元数据图像**，设置方式与“电视节目”和“电影”部分所述完全一致。

---

### 电影 (Movies)

电影库的规则与上述“电视节目”基本互通。Jellyfin 支持常见的视频和光盘文件夹格式（同样不建议使用 `.iso`）。

#### 组织与命名
电影应该组织在独立的文件夹中，并且建议按照 `电影名称 (年份) [元数据提供商ID]` 格式命名该文件夹。
视频文件应与所在的文件夹名称保持一致。

```text
Movies
├── Best_Movie_Ever (2019)
│   ├── Best_Movie_Ever (2019).mp4
│   ├── Best_Movie_Ever (2019).nfo
│   ├── Best_Movie_Ever (2019).en_us.srt
│   └── cover.png
└── Movie (2021) [imdbid-tt12801262]
    └── backdrop.jpg
```

#### 多版本识别 (Multiple Versions)
对于电影来说，同一电影文件夹内的视频通过 **前缀匹配** 以及 **后缀区分** 来被识别为多版本。
**每个文件名必须完全以其父文件夹的基础名称作为开头**（包括年份和 ID），连一个字符都不能差，然后再加上版本标签。版本标签需用 `-`、`.`、`_` 或 `[]` 与基础名称隔开。

```text
Movie (2021) [imdbid-tt12801262]
├── Movie (2021) [imdbid-tt12801262] - 2160p.mp4
├── Movie (2021) [imdbid-tt12801262] - 1080p.mp4
└── Movie (2021) [imdbid-tt12801262] - Directors Cut.mp4
```


### 切片电影的合并 (Split files)
如果一部电影被分割为多个文件，可以通过特定的后缀让系统自动无缝播放。
**支持的标签**：`cd`, `dvd`, `part`, `pt`, `disc`, `disk`。
**分隔符**：空格、`.`、`-`、`_`。

```text
电影名称 (2010)/
├── 电影名称-part1.mkv
└── 电影名称-part2.mkv
```

### 额外花絮与预告片 (Extras)
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

### 3D 电影的识别
在文件名中加入 `.3D.` 标签以及格式后缀，Jellyfin 即可自动识别 3D 文件。
*   `hsbs` / `fsbs` = 左右半宽 / 左右全宽
*   `htab` / `ftab` = 上下半高 / 上下全高
*   `mvc` = 多视图视频编码

```text
Awesome 3D Movie (2022).3D.FTAB.mp4
Awesome 3D Movie (2022)-3d-hsbs.mp4
```
---

### 混合电影和电视节目 (Mixed Movies and Shows)

混合媒体库应该遵循对应的电影和电视节目命名指南。但是需要注意的是：在混合库中，某些特定功能可能无法像独立媒体库那样正常运作。为了获得最佳体验，**强烈建议为电影和剧集使用专用的、分开的媒体库**。

---

### 排除文件和目录 (Excluding Files and Directories)

可以通过在媒体库目录中放置 `.ignore` 文件，将特定目录和文件从 Jellyfin 的媒体库扫描中排除。（添加或更新 `.ignore` 后需要重新扫描媒体库生效）。

#### 排除整个目录
若要排除整个目录，只需在其中放置一个**空**的 `.ignore` 文件即可。该目录及其包含的所有文件和子目录都将被忽略。

#### 排除特定文件或子目录
如果 `.ignore` 文件**不为空**，系统会将其视作排除列表，其格式和规则与 Git 的 `.gitignore` 文件完全相同。
*(注意：这是 Jellyfin 10.11.x 及更高版本的新特性；旧版本不关心该文件的内容。旧版非空的 .ignore 文件必须手动更正为空，否则功能可能出现异常)*

示例 `.ignore` 内容：
```text
*.avi
sample.mkv
specials/
```
这将导致 Jellyfin 忽略所有 `.avi` 后缀文件、名为 `sample.mkv` 的文件，以及整个 `specials` 子目录。

---

### 书籍 (Books)

书籍应该首先按照类型分类（有声读物 Audiobooks、书籍 Books、漫画 Comics），然后（可选地）按作者分类。每本书应该放置在自己独立的文件夹中。

* **有声读物支持格式**：常见的音频扩展名。不支持带有阅读追踪的同步有声读物。
* **其他书籍支持格式**：`azw`, `azw3`, `cb7`, `cbr`, `cbt`, `cbz`, `epub`, `mobi`, `pdf`, `zip`, `rar`, `7z`。

#### 元数据与封面
书籍可以使用内置的在线元数据提供商（如 GoogleBooks, ComicVine, OpenLibrary）进行信息抓取，也可从支持的文件中读取内嵌元数据。
* **音频与 epub**：支持直接读取内嵌标签或内嵌元数据。（如果是含有 WebP 图片的 FLAC 播放报错，请在设置中开启“始终重新混流 FLAC”）。
* **其他格式**：可以在文件夹中提供外部的 `content.opf`, `metadata.opf` 或 `ComicInfo.xml` 文件（兼容 ComicRack 等软件）。
* **文件名**：可以在文件名中添加年份和期数信息。对于跨越多年的连载漫画或杂志，应使用第一期发行的年份。

外部封面图片与视频库类似，支持 `folder`, `poster`, `cover`, `banner`, `logo` 和 `thumb` 等命名方式。

> **参考来源**：更多详细命名规则请查阅 [Jellyfin 官方文档](https://jellyfin.org/docs/general/server/media/movies/)。
