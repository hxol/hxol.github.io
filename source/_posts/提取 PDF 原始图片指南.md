---
title: 批量重建与清理 PDF 文件元数据指南
date: 2026-10-02 13:02:03
tags: [笔记, pdf, bash 脚本]
---

# 提取 PDF 原始图片指南

利用 `poppler-utils` 工具包中的 `pdfimages` 命令，我们可以直接从 PDF 文件中“无损剥离”原始图片。这种方式不是对 PDF 进行截图渲染，而是直接提取内嵌的图像文件，因此速度极快且不会损失画质。

## 环境准备

在使用之前，需要确保系统中安装了 `poppler-utils` 工具包。您可以根据当前使用的操作系统选择对应的安装命令：

**Debian / Ubuntu 系列**
```bash
sudo apt update && sudo apt install poppler-utils
```

**macOS (借助 Homebrew)**
```bash
brew install poppler
```

**CentOS / RHEL 系列**
```bash
sudo yum install poppler-utils
```

## 基础用法：单文件提取

`pdfimages` 的核心语法非常简单，基本格式为：`pdfimages [参数] 输入文件.pdf 输出前缀`。

强烈建议始终携带 `-all` 参数。加上此参数后，程序会尽可能以图片原本的格式（如 JPEG、PNG、TIFF 等）输出；如果不加，旧版本的工具可能会默认输出体积巨大的 `.ppm` 或 `.pbm` 原始位图文件。

**提取示例**
假设我们有一个名为 `sample-document.pdf` 的文件，需要将其中的图片提取到 `output_images` 文件夹中，并以 `pic` 作为文件名前缀：

```bash
# 提取命令演示
pdfimages -all 'sample-document.pdf' ./output_images/pic
```
执行后，目标文件夹中会自动生成类似 `pic-000.jpg`, `pic-001.png` 的图片文件。

## 进阶用法：批量递归提取脚本

当面对大量的 PDF 文件，或者 PDF 散落在多个不同的子文件夹中时，手动提取会非常繁琐。

以下是一个自动化的 Bash 脚本。它的作用是递归查找当前目录及其子目录下的所有 PDF 文件，将提取出的图片以原 PDF 的文件名作为前缀，统一存放到一个专属的输出目录中，并且**完美保持原有的多级目录结构**。

**使用方法：**
新建一个名为 `extract_pdf_images.sh` 的文件，赋予执行权限（`chmod +x extract_pdf_images.sh`），然后将以下代码粘贴进去并运行。

```bash
#!/bin/bash

# 配置项：专门存放提取图片的根目录名称（可随时修改）
OUTPUT_ROOT="extracted_images"

echo "开始执行... 提取的图片将保存到: ./$OUTPUT_ROOT/"

# 使用 find 查找当前目录及所有子目录下的 PDF 文件
# 技巧说明：
# -path "./$OUTPUT_ROOT" -prune 用于排除我们的输出目录，避免递归死循环冲突
# -type f -iname "*.pdf" 忽略大小写查找文件
# -print0 配合 read -d '' 能够完美处理文件名中包含空格或特殊字符的情况
find . -path "./$OUTPUT_ROOT" -prune -o -type f -iname "*.pdf" -print0 | while IFS= read -r -d '' pdf_file; do
    
    # 获取 PDF 所在的相对目录路径 (例如 "./doc_folder/sub_folder")
    dir_name=$(dirname "$pdf_file")
    
    # 获取 PDF 的文件名，并剥离文件后缀 (例如 "报告.pdf" 变为 "报告")
    base_name=$(basename "$pdf_file")
    file_prefix="${base_name%.*}"
    
    # 构造对应的镜像输出目录路径
    target_dir="$OUTPUT_ROOT/$dir_name"
    
    # 按需创建多级父目录，目录已存在也不会报错
    mkdir -p "$target_dir"
    
    # 执行提取命令，输出前缀设置为：目标目录 + PDF文件名
    echo "正在处理: $pdf_file"
    pdfimages -all "$pdf_file" "$target_dir/$file_prefix"
    
done

echo "🎉 全部处理完成！所有图片均已存放在 ./$OUTPUT_ROOT/ 目录下，并保持了原有的目录结构。"
```

## 扩展备忘录

> 💡 **维护提示**：这里预留了扩展空间。未来如果您发现了新的参数、遇到了特殊 PDF（如加密 PDF），或者想补充 Python 脚本（如 `PyMuPDF`），可以直接在此处追加记录。

*   **处理加密文件**：如果 PDF 设置了用户密码，可以使用 `-upw <密码>` 参数进行解锁提取。
*   **提取特定页码**：如果只需要提取某几页的图片，可以配合 `-f` (first page) 和 `-l` (last page) 参数。例如只提取第 5 到第 10 页：`pdfimages -all -f 5 -l 10 input.pdf output_prefix`。