---
title: DjVu 批量转换为 PDF 的简单方法
date: 2026-10-02 13:02:05
tags: [笔记, DjVu, pdf, 电子书]
---


# DjVu 批量转换为 PDF 的简单方法

DjVu 格式作为存档比较好用，但是很多电子书阅读管理软件不认DjVu，只能转换为兼容性更好的 PDF 格式。以下脚本利用 Bash 脚本结合多线程技术，提供了高效、稳定的批量转换及校验方案。

### 环境依赖
在运行脚本前，请确保系统中已安装必要的依赖工具。脚本核心依赖于 `djvulibre`（提供转换与信息读取功能）、`poppler-utils`（提供 PDF 信息读取）以及多线程并发工具。

*   **Debian / Ubuntu:** 
    `sudo apt install djvulibre-bin poppler-utils parallel`
*   **macOS (Homebrew):** 
    `brew install djvulibre poppler parallel`

---

### 方案 A：彩色文档批量转换 (基于 xargs)

此脚本适用于包含图片或彩色内容的 DjVu 文件。利用 `xargs` 开启多进程并行处理，并通过动态获取系统核心数来最大化硬件利用率。

```bash
#!/bin/bash

# 定义核心转换函数
convert_djvu() {
    local f="$1"
    local out="${f%.djvu}.pdf"

    if [ -f "$out" ]; then
        echo "⏭️  跳过已存在: $out"
        return 0 
    fi

    echo "🔄 正在转换: $f"
    
    # -format=pdf 指定输出格式
    # -quality=85 控制输出 PDF 的图片质量与体积平衡
    if ddjvu -format=pdf -quality=85 "$f" "$out"; then
        echo "✅ 成功: $out"
    else
        echo "❌ 失败: $f"
        # 发生错误时，清理生成的半成品文件避免后续误判
        rm -f "$out"
    fi
}

# 导出函数，使其在 xargs 的子进程中可用
export -f convert_djvu

# 获取当前系统的 CPU 核心数，用于动态分配线程
CPU_CORES=$(nproc 2>/dev/null || echo 4)

# 执行批量查找与转换
# -print0 和 -0 搭配，完美处理带有空格或特殊字符的文件名
find . -type f -name "*.djvu" -print0 | xargs -0 -n 1 -P "$CPU_CORES" bash -c 'convert_djvu "$@"' _
```

---

### 方案 B：黑白文档批量转换 (基于 GNU Parallel)

对于纯文字内容的扫描件，使用黑白模式可以极大缩小生成的 PDF 体积。这里展示了另一种多线程工具 `parallel` 的用法，它的语法更为简洁直观。

```bash
#!/bin/bash

# 定义核心转换函数
convert_djvu_bw() {
    local f="$1"
    local out="${f%.djvu}.pdf"

    if [ -f "$out" ]; then
        echo "⏭️  跳过已存在: $out"
        return 0
    fi

    echo "🔄 正在转换 (黑白模式): $f"
    
    # -mode=black 强制黑白二值化输出，适合纯文字文档
    if ddjvu -format=pdf -mode=black "$f" "$out"; then
        echo "✅ 成功: $out"
    else
        echo "❌ 失败: $f"
        rm -f "$out"
    fi
}

export -f convert_djvu_bw

CPU_CORES=$(nproc 2>/dev/null || echo 4)

# 使用 parallel 进行并行处理，-j 指定并发任务数
find . -type f -name "*.djvu" -print0 | parallel -0 -j "$CPU_CORES" convert_djvu_bw
```

---

### 自动化校验与查漏补缺

由于转换过程可能会因为源文件损坏或内存不足中断，批量处理后必须进行完整性核对。该脚本通过精准比对转换前后的“页面总数”来判断文件是否完好。

```bash
#!/bin/bash

echo "🔍 开始校验转换结果..."

find . -type f -name "*.djvu" -print0 | while IFS= read -r -d '' f; do
    pdf_file="${f%.djvu}.pdf"

    # 检查目标文件是否生成
    if [ ! -f "$pdf_file" ]; then
        echo "⚠️ 丢失预警 (未转换或失败): $f"
        continue
    fi

    # 提取源文件与目标文件的总页数
    djvu_pages=$(djvused -e 'n' "$f" 2>/dev/null)
    pdf_pages=$(pdfinfo "$pdf_file" 2>/dev/null | grep -a "^Pages:" | awk '{print $2}')

    # 数据异常容错处理
    if [ -z "$djvu_pages" ] || [ -z "$pdf_pages" ]; then
        echo "❓ 读取异常 (无法获取页数信息): $f"
        continue
    fi

    # 逻辑比对
    if [ "$djvu_pages" != "$pdf_pages" ]; then
        echo "❌ 页数不匹配! DjVu: $djvu_pages 页 | PDF: $pdf_pages 页 -> $f"
    else
        # 校验通过
        echo "✅ 校验通过 ($djvu_pages 页): $f"
    fi
done

echo "🎉 校验任务完成。"
```

---

### 自定义与扩展指南

这套脚本设计为高内聚、低耦合的模式。未来若需调整功能，只需修改 `ddjvu` 命令的参数即可：

*   **调整输出画质：** 修改 `convert_djvu` 中的 `-quality=85`。数值越低体积越小，但画面会有损失；数值最高可设为 100。
*   **灰度模式：** 如果文档既不是全彩，也不适合强制黑白（例如带有大量老照片的报纸），可将参数改为 `-mode=foreground` 或考虑保留默认彩色模式。
*   **指定处理目录：** 将脚本底部的 `find .` 替换为 `find /你的/目标/路径` 即可实现跨目录操作。
*   **限制 CPU 占用：** 默认配置会跑满所有 CPU 核心。如果希望在后台转换时不影响电脑日常使用，可手动将 `CPU_CORES=$(nproc)` 修改为 `CPU_CORES=2`（根据实际情况指定保守的线程数）。