---
title: 如何用 Python 将视频转换为PDF
date: 2026-10-11 15:30:23
tags: [笔记, Python, 脚本, pdf, 图片]
---





# 如何用 Python 将视频转换为PDF


## 准备工作

**1. 安装 Python 环境**
如果你还没有安装 Python，请前往 [Python官网](https://www.python.org/) 下载最新版本。
> ⚠️ **注意**：Windows 用户在安装时，请务必勾选底部的 **"Add Python to PATH"**。

**2. 安装必要的依赖库**
打开终端（或命令提示符 CMD），运行以下命令安装所需的第三方库：
```bash
pip install opencv-python Pillow imagehash
```
*(注：`opencv` 用于读取视频，`Pillow` 用于处理图片和生成PDF，`imagehash` 用于计算图片相似度以实现智能去重。)*

---

## 第一步：智能提取非重复视频帧

这个脚本会自动遍历视频，通过计算图像哈希值（Hash）来对比前后画面的差异。只有当画面发生明显变化时（例如翻页了），才会截取保存。

新建一个 Python 文件，命名为 `extract_frames.py`，粘贴以下代码：

```python
import cv2
import os
from PIL import Image
import imagehash

# === 配置参数 ===
video_path = "input.mkv"         # 输入视频的文件名（请确保与脚本在同级目录）
output_dir = "output_frames"     # 提取出的图片保存目录
threshold = 5                    # 画面变化敏感度（数值越小越敏感，推荐范围 3-8）
jpeg_quality = 90                # 截图保存质量（1-100）

# 创建输出目录
os.makedirs(output_dir, exist_ok=True)

# 初始化视频捕获
cap = cv2.VideoCapture(video_path)
if not cap.isOpened():
    print("❌ 无法打开视频文件，请检查文件名和路径是否正确。")
    exit()

# 读取并保存第一帧
success, prev_frame = cap.read()
if not success:
    print("❌ 无法读取视频内容。")
    exit()

# 处理首帧
prev_hash = imagehash.average_hash(Image.fromarray(cv2.cvtColor(prev_frame, cv2.COLOR_BGR2RGB)))
cv2.imwrite(os.path.join(output_dir, "frame_0000.jpg"), prev_frame, [cv2.IMWRITE_JPEG_QUALITY, jpeg_quality])
count = 1

print("⏳ 正在提取视频帧，请稍候...")

# 处理后续帧
while True:
    success, frame = cap.read()
    if not success:
        break
    
    # 计算当前帧的哈希值
    current_hash = imagehash.average_hash(Image.fromarray(cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)))
    
    # 对比哈希差异（超过阈值则认为是新画面）
    if (prev_hash - current_hash) > threshold:
        filename = os.path.join(output_dir, f"frame_{count:04d}.jpg")
        cv2.imwrite(filename, frame, [cv2.IMWRITE_JPEG_QUALITY, jpeg_quality])
        prev_hash = current_hash
        count += 1

cap.release()
print(f"✅ 提取完成！共提取 {count} 个非重复帧，已保存至 [{output_dir}] 目录。")
```

**如何运行：**
1. 准备一个测试视频，重命名为 `input.mkv`（或修改代码中的 `video_path` 以匹配你的文件名），放在脚本同一目录下。
2. 在终端运行：`python extract_frames.py`。

---

## 第二步：手动微调（可选）

视频提取完成后，打开生成的 `output_frames` 文件夹。
浏览一下提取出来的图片，如果有讲师的过渡画面、不重要的动画帧或者不需要的废片，**直接手动删除即可**。剩下的图片将全部进入最后的PDF。

---

## 第三步：将图片合成高清 PDF

整理好图片后，我们将它们统一缩放并打包成一个便于在平板或电脑上阅读的 PDF 文件。

新建一个 Python 文件，命名为 `create_pdf.py`，粘贴以下代码：

```python
from PIL import Image
import os

# === 配置参数 ===
image_folder = "output_frames"   # 第一步生成的图片目录
output_pdf = "output_notes.pdf"  # 最终输出的 PDF 文件名
target_width = 1200              # 图片统一宽度（推荐 800-1200，适合平板阅读）
dpi = 150                        # 输出分辨率 DPI（推荐 150-300）

# 获取并过滤图片列表
images = []
for filename in sorted(os.listdir(image_folder)):
    if filename.lower().endswith(('.png', '.jpg', '.jpeg')):
        img_path = os.path.join(image_folder, filename)
        try:
            img = Image.open(img_path)
            # 统一转换为 RGB 模式，避免 PDF 报错
            if img.mode != 'RGB':
                img = img.convert('RGB')
            images.append(img)
        except Exception as e:
            print(f"⚠️ 跳过无效图片 {filename}: {str(e)}")

# 调整图片尺寸以统一排版
resized_images = []
print("⏳ 正在处理图片尺寸...")
for img in images:
    # 按比例缩放高度
    w_percent = target_width / float(img.size[0])
    h_size = int(float(img.size[1]) * float(w_percent))
    
    # 调整大小 (使用 LANCZOS 算法保证清晰度)
    resized_img = img.resize((target_width, h_size), Image.Resampling.LANCZOS)
    resized_images.append(resized_img)

# 生成 PDF
if resized_images:
    print("⏳ 正在生成 PDF...")
    resized_images[0].save(
        output_pdf,
        "PDF",
        resolution=dpi,
        save_all=True,
        append_images=resized_images[1:]
    )
    print(f"✅ PDF 已成功生成：{output_pdf}")
else:
    print("❌ 没有找到有效图片，请检查图片目录。")
```

**如何运行：**
在终端运行：`python create_pdf.py`。等待几秒钟，你的当前目录下就会多出一个 `output_notes.pdf` 文件！

---

## 进阶指南：如何调出最完美的效果？

由于不同视频的画质和内容不同，你可能需要微调代码顶部的**参数**来达到最佳效果：

**1. 提取脚本 (`extract_frames.py`)：**
*   `threshold`（差异阈值）：**核心参数！** 默认值是 5。
    *   如果你发现生成的图片**漏掉了**很多幻灯片，说明不够敏感，请调低（如改为 `3` 或 `4`）。
    *   如果你发现生成的图片**重复内容太多**（比如讲师鼠标动一下就被截图），说明太敏感了，请调高（如改为 `7` 或 `8`）。
*   `jpeg_quality`（图片质量）：推荐 85-95。太高会导致最终 PDF 体积极大，90 是一个极佳的平衡点。

**2. PDF生成脚本 (`create_pdf.py`)：**
*   `target_width`（PDF页面宽度）：
    *   `800`：适合在手机或小尺寸阅读器上查看。
    *   `1200`：适合主流 10-11 寸平板（iPad等），**强烈推荐**。
    *   `1600`：适合电脑大屏，但 PDF 文件体积会变大。
*   `dpi`：如果只是屏幕阅读，`150` 足够清晰；如果后续需要打印出来，建议修改为 `300`。
