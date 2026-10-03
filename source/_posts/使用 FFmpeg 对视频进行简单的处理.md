---
title: 使用 FFmpeg 对视频进行简单的处理
date: 2026-10-02 13:01:40
tags: [笔记, FFmpeg]
---


# 使用 FFmpeg 对视频进行简单的处理
## 核心概念与画质控制

控制视频编码质量的核心在于平衡**体积**与**画质**。主流编码器通常采用“恒定质量”或“平均码率”模式。

*   **恒定速率因子 (CRF / RF)**：适用于纯 CPU 编码（如 `libx264`, `libx265`, `SVT-AV1`）。
    *   **原则**：数值越**小**，质量越**高**，文件越大。
    *   **推荐值 (x264/x265)**：480p (18-22)；720p (19-23)；1080p (20-24)；4K (22-28)。
    *   **提示**：对于动画片（动漫、卡通），由于画面纯色较多，可适当调低数值以保留线条细节。对于包含大量噪点、雪花的老旧片源，极低的 CRF 会保留“噪点”从而导致体积暴增。
*   **恒定质量模式 (CQ / `-q:v`)**：适用于部分硬件加速编码（如苹果的 VideoToolbox）。
    *   **原则**：与 CRF 完全相反。数值越**大**，质量越**高**，文件越大。

---

## 画面裁切 (Crop) 与预览

去除视频黑边或截取画面特定区域。

**获取裁切参数**
利用 `cropdetect` 滤镜自动检测黑边边界。截取第 8 分钟处的 10 帧进行分析：
```bash
ffmpeg -ss 00:08:00 -i input_video.mkv -vframes 10 -vf cropdetect -f null -
```
*日志输出示例将包含 `crop=1024:704:86:10`，代表：`宽度:高度:左侧偏移量:顶部偏移量`。*

**实时预览裁切效果**
无需编码，直接使用 `ffplay` 播放器查看裁切是否准确：
```bash
ffplay -vf crop=1024:704:86:10 input_video.mp4
```

**应用裁切并重新编码**
确认无误后，结合视频编码器输出去黑边版本：
```bash
ffmpeg -i input_video.mkv -vf "crop=1024:704:86:10" -c:v libx265 -crf 24 -c:a copy output_no_borders.mkv
```

---

## macOS 硬件加速 (VideoToolbox)

在搭载 Apple Silicon (M1/M2/M3 等) 的 Mac 上，调用硬件编解码器可实现极速、零发热的转码，但同等画质下文件体积会比纯 CPU 软件编码大 10%~20%。

**标准硬件加速转码命令**
```bash
ffmpeg -hwaccel videotoolbox -i input_video.mkv -c:v hevc_videotoolbox -q:v 55 -tag:v hvc1 -c:a copy output_video.mp4
```

**关键参数解析**
*   `-hwaccel videotoolbox`：放在输入文件前，启用硬件解码，进一步降低 CPU 占用。
*   `-c:v hevc_videotoolbox`：调用苹果硬件级 H.265 (HEVC) 编码器。
*   `-q:v 55`：硬件编码的质量控制参数（取值 1-100）。日常压制推荐 45~55；接近无损推荐 60~65。
*   `-tag:v hvc1`：**必加参数**。为 HEVC 写入标准的 FourCC 标签，否则在 macOS/iOS 原生播放器中可能只有声音没有画面。
*   `-b:v 5000k`：（替代方案）如果需要严格控制体积，可去掉 `-q:v` 并使用此参数限制平均码率。

**硬件 CQ (`-q:v`) 与 软件 CRF 对标经验**
*   `CRF 21` ≈ `-q:v 65`（高画质/肉眼无损，适合蓝光压制）
*   `CRF 22` ≈ `-q:v 60`（较高画质，较好的平衡点）
*   `CRF 24` ≈ `-q:v 55`（标准画质，适合日常观看）
*   `CRF 25` ≈ `-q:v 50`（网络流媒体级别）

---

## 常用编辑操作

**截取视频片段**
精确截取指定时间段，流复制不重新编码（速度极快）：
```bash
ffmpeg -ss 00:01:54 -to 00:06:53 -i input.mp4 -c copy output_clip.mp4
```

**合并视频文件**
先创建一个文本文件 `list.txt`，内容格式如下：
```text
file 'video_part1.mkv'
file 'video_part2.mkv'
```
然后执行无损拼接命令：
```bash
ffmpeg -f concat -safe 0 -i list.txt -c copy output_merged.mkv
```

脚本实现
```bash
#!/bin/bash

# 设置输出文件名
output="merged_output.mp4"

# 创建一个临时文件清单
list_file="videos_to_merge.txt"
> "$list_file"

# 查找当前目录下的视频文件，按文件名排序
for f in $(find . -maxdepth 1 -type f \( -iname "*.mp4" -o -iname "*.mov" -o -iname "*.mkv" \) | sort); do
    echo "file '$f'" >> "$list_file"
done

# 检查是否有文件
if [ ! -s "$list_file" ]; then
    echo "没有找到要合并的视频文件。"
    exit 1
fi

# 使用 ffmpeg 合并
ffmpeg -f concat -safe 0 -i "$list_file" -c copy "$output"

# 清理临时文件
rm "$list_file"

echo "视频已成功合并为 $output"
```

**为视频添加或替换音轨**
将独立的音频文件混流进视频，需注意指定 `-c copy` 以避免视频被重新编码：
```bash
ffmpeg -i video_input.mp4 -i audio_input.aac -c:v copy -c:a aac -map 0:v:0 -map 1:a:0 output_combined.mp4
```

**转码前后画质对比（同帧截图）**
利用 `-ss` 定位到特定时间点，分别从原视频和转码后的视频抽取一帧 PNG 图像进行肉眼比对：
```bash
ffmpeg -ss 00:01:23 -i source.mp4 -frames:v 1 before.png
ffmpeg -ss 00:01:23 -i transcoded.mp4 -frames:v 1 after.png
```

---

## 自动化与批量处理脚本

### 文件夹内视频批量转码 (macOS Bash)
将当前目录下的常用视频格式批量调用苹果硬件加速转换为 H.265 MP4。
```bash
#!/bin/bash

if ! command -v ffmpeg &> /dev/null; then
    echo "错误: 未找到 ffmpeg，请先使用 'brew install ffmpeg' 安装。"
    exit 1
fi

OUTPUT_DIR="hevc_output"
mkdir -p "$OUTPUT_DIR"
echo "开始批量转换视频... 输出文件夹: $OUTPUT_DIR"

shopt -s nocaseglob nullglob
for file in *.{mp4,mov,mkv,avi,m4v,flv,wmv,ts}; do
    filename=$(basename "$file")
    name="${filename%.*}"
    output_file="$OUTPUT_DIR/${name}.mp4"
    
    echo "▶ 正在处理: $file"
    ffmpeg -hwaccel videotoolbox -i "$file" \
           -c:v hevc_videotoolbox -q:v 60 -tag:v hvc1 \
           -c:a copy -y "$output_file"
done
echo "✔ 全部处理完成！"
```

### 批量无损更换封装格式
适用于只需将 `.ts` / `.mkv` 等格式秒转为 `.mp4`，不改变视频原始画质的情况。

*   **Linux / macOS (Bash)**:
    ```bash
    for f in *.ts; do ffmpeg -i "$f" -c copy "${f%.ts}.mp4"; done
    ```
*   **Windows (Command Prompt)**:
    ```cmd
    for %f in (*.ts) do ffmpeg -i "%f" -c copy "%~nf.mp4"
    ```
*   **Windows (PowerShell) - 剥离多余字幕流版**:
    只提取第一个视频流和音频流，忽略不兼容的字幕流。
    ```powershell
    Get-ChildItem *.ts | ForEach-Object { 
        ffmpeg -i $_.Name -map 0:v:0 -map 0:a:0 -c copy ($_.BaseName + ".mp4") 
    }
    ```

### 批量转换音频格式
*   **提取或转换音频 (Bash - WMA 转 MP3)**:
    ```bash
    for f in *.wma; do ffmpeg -i "$f" -q:a 2 "${f%.wma}.mp3"; done
    ```
*   **视频内置音频转码 (PowerShell - 修复不兼容音频)**:
    遍历视频，仅将音频轨转为 FLAC，视频保持无损拷贝（适用于将古老的 pcm_dvd 音频更新为通用格式）。
    ```powershell
    $extensions = '.mpg','.mkv','.mp4','.ts','.vob'
    Get-ChildItem -File | Where-Object { $_.Extension -in $extensions } | ForEach-Object {
        $output = [IO.Path]::ChangeExtension($_.FullName, "_flac.mkv")
        if (-not (Test-Path $output)) {
            Write-Host "正在处理: $($_.Name)" -ForegroundColor Green
            ffmpeg -fflags +genpts -i "$($_.FullName)" -map 0 -c:v copy -c:a flac -c:s copy -async 1 -y "$output"
        }
    }
    ```

---

## 进阶工具箱

### Python: 视频噪点/冗余码率检测脚本
通过计算视频的 BPPF（每像素每帧比特数）与该编码器的理论标准值进行对比，判断视频中是否包含大量难以压缩的噪点或雪花。若溢出比过高，建议在转码时加入降噪滤镜（如 `hqdn3d`）。

保存为 `check_noise.py` 并通过 `python3 check_noise.py input.mp4` 运行。

```python
import subprocess
import json
import sys

# 各编码器达到“透明画质”所需的标准经验 BPPF
CODEC_BPPF_MAP = {
    'mpeg2video': 0.18, 
    'mpeg4': 0.12,       
    'h264': 0.075,       
    'vp8': 0.08,
    'hevc': 0.045,       
    'vp9': 0.045,        
    'av1': 0.035,        
}
DEFAULT_BPPF = 0.08
NOISE_THRESHOLD = 1.5  # 判定阈值

def get_video_info(file_path):
    cmd = [
        'ffprobe', '-v', 'quiet', '-print_format', 'json',
        '-show_format', '-show_streams', '-select_streams', 'v:0', file_path
    ]
    try:
        result = subprocess.run(cmd, stdout=subprocess.PIPE, text=True, encoding='utf-8')
        return json.loads(result.stdout)
    except Exception as e:
        print(f"读取文件失败: {e}")
        sys.exit(1)

def parse_fraction(fraction_str):
    if '/' in fraction_str:
        num, den = fraction_str.split('/')
        return float(num) / float(den) if float(den) != 0 else 0
    return float(fraction_str)

def main():
    if len(sys.argv) < 2:
        print("用法: python check_noise.py <视频文件路径>")
        sys.exit(1)

    file_path = sys.argv[1]
    info = get_video_info(file_path)

    if not info.get('streams'):
        print("未检测到视频流！")
        sys.exit(1)

    v_stream = info['streams'][0]
    format_info = info.get('format', {})

    width = int(v_stream.get('width', 0))
    height = int(v_stream.get('height', 0))
    fps = parse_fraction(v_stream.get('r_frame_rate', '25/1'))
    codec_name = v_stream.get('codec_name', 'unknown').lower()

    actual_bitrate = v_stream.get('bit_rate') or format_info.get('bit_rate')
    if not actual_bitrate:
        print("无法提取码率数据。")
        sys.exit(1)
    
    actual_bitrate = float(actual_bitrate)
    target_bppf = CODEC_BPPF_MAP.get(codec_name, DEFAULT_BPPF)
    theoretical_bitrate = width * height * fps * target_bppf
    ratio = actual_bitrate / theoretical_bitrate

    print("="*45)
    print(f"📁 文件: {file_path}")
    print(f"🎞️ 编码: {codec_name.upper()} (基准 BPPF: {target_bppf})")
    print(f"📺 规格: {width}x{height} @ {fps:.2f} fps")
    print("-" * 45)
    print(f"📉 理论合理码率: {theoretical_bitrate / 1000:.0f} kbps")
    print(f"📈 实际视频码率: {actual_bitrate / 1000:.0f} kbps")
    print(f"⚖️ 码率溢出倍数: {ratio:.2f} 倍")
    print("-" * 45)

    if ratio >= NOISE_THRESHOLD:
        print("⚠️ 结论: 强烈建议开启【降噪/滤镜】！")
        print(f"   原因: 实际码率达到了理论值的 {ratio:.2f} 倍，画面极大概率包含大量难以压缩的噪点或胶片颗粒。")
    else:
        print("✅ 结论: 无需开启降噪滤镜。")
        print("   原因: 视频体积符合正常数据密度，片源画面较为干净。")
    print("="*45)

if __name__ == "__main__":
    main()
```