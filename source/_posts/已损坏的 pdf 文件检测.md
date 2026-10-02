---
title: 已损坏的 pdf 文件检测
date: 2026-10-02 13:02:01
tags: [笔记, pdf, python]
---


# 已损坏的 pdf 文件检测

## 环境与依赖

### 第一步：确保系统安装了创建虚拟环境的工具
Debian/Ubuntu 系统默认可能没有安装 `venv` 模块，你需要先用系统包管理器装一下（如果已经安装过，系统会提示已是最新版）：
```bash
sudo apt update  && sudo apt install python3-venv && mkdir test && cd test
```

### 第二步：创建一个虚拟环境
运行以下命令来创建一个名为 `.venv` 的虚拟环境（前面加个点是为了让它成为隐藏文件夹，不干扰你的目录视线）：
```bash
python3 -m venv .venv
```
*执行完后，当前目录下会多出一个名为 `.venv` 的隐藏文件夹，这里面包含了独立的 Python 解释器和 pip。*

### 第三步：激活（进入）虚拟环境
创建完之后，必须“激活”它，让当前的终端切换到这个独立环境中：
```bash
source .venv/bin/activate
```
*激活成功后，你会看到你的命令行提示符最前面多了一个 `(.venv)`，变成类似 `(.venv) hx@m93p:~$` 的样子。这就说明你已经在一个完全独立且安全的环境里了！*

### 第四步：在虚拟环境中安装依赖并运行脚本

```bash
pip install pypdf
```

```bash
python check_pdfs.py
```

#### Python 脚本

```python
import os
import shutil
from pathlib import Path

try:
    from pypdf import PdfReader
except ImportError:
    print("缺少必要的依赖库。请先在终端运行：pip install pypdf")
    exit(1)

def is_pdf_corrupted(file_path):
    """
    检测 PDF 文件是否损坏（针对大型电子书进行优化，极力避免误判）。
    """
    try:
        size = os.path.getsize(file_path)
        # 1. 极小文件检测：正常的 PDF 哪怕全白也有几百字节。小于 16 字节绝对是无效/损坏文件
        if size < 16:
            return True
        
        with open(file_path, 'rb') as f:
            # 2. 文件头检测：读取前 1024 字节，如果没有 %PDF- 标志，说明根本不是 PDF 格式或头部已完全损坏
            header = f.read(1024)
            if b'%PDF-' not in header:
                return True
            
            # 3. 尝试用 pypdf 解析 (strict=False 启用宽容模式，忽略轻微不规范)
            f.seek(0)
            try:
                reader = PdfReader(f, strict=False)
                _ = len(reader.pages)
                return False  # 正常解析，未损坏
            except Exception:
                # 走到这里，说明 pypdf 无法解析。但为了防止对数 GB 大文件的误判，我们进行兜底检测。
                pass 
            
            # 4. 兜底特征检测 (针对多GB大型 PDF)：
            # 如果文件过大或结构不标准导致 pypdf 崩溃，但只要文件尾部存在 %%EOF 结束符，
            # 绝大多数现代 PDF 阅读器都能自动重建索引并正常打开。我们假定它未损坏。
            # 读取文件末尾最多 1MB 的数据寻找 EOF 标志（处理末尾可能存在冗余数据的情况）
            seek_offset = min(size, 1024 * 1024)
            f.seek(-seek_offset, os.SEEK_END)
            tail_data = f.read()
            
            if b'%%EOF' in tail_data:
                # 存在结束符，极有可能是可以被阅读器自动修复的大文件，为了安全起见，不视为损坏
                return False
            else:
                # 既无法解析，又没有结束符，说明文件大概率下载中断或被严重截断，判定为损坏
                return True
                
    except Exception as e:
        # 文件由于权限、I/O错误等根本无法读取，视作损坏或异常
        print(f"无法读取文件 {file_path.name}: {e}")
        return True

def main():
    current_dir = Path.cwd()
    corrupted_dir = current_dir / "已损坏pdf文件"
    
    corrupted_files = []
    
    print("正在扫描并检测 PDF 文件（已启用大型文件防误判机制）...")
    
    # 1. 搜寻并检测所有的 PDF 文件
    for pdf_path in current_dir.rglob("*.pdf"):
        # 跳过我们用来存放损坏文件的文件夹
        if corrupted_dir in pdf_path.parents:
            continue
            
        if is_pdf_corrupted(pdf_path):
            corrupted_files.append(pdf_path)
            
    # 2. 列出结果
    if not corrupted_files:
        print("\n太棒了！扫描完成，没有发现任何损坏的 PDF 文件。")
        return

    print(f"\n扫描完成，共发现 {len(corrupted_files)} 个损坏（或截断）的 PDF 文件：")
    for file_path in corrupted_files:
        print(f"- {file_path.relative_to(current_dir)}")

    # 3. 按原目录结构移动文件
    print(f"\n正在将损坏的文件移动到: {corrupted_dir}")
    
    for file_path in corrupted_files:
        # 获取相对于当前工作目录的路径 (例如: 文史/[北宋] 司马光/资治通鉴/1.pdf)
        rel_path = file_path.relative_to(current_dir)
        
        # 拼接出新的目标完整路径
        dest_path = corrupted_dir / rel_path
        
        # 核心：自动创建包含新文件的父级目录树 (例如自动创建 文史/[北宋] 司马光/资治通鉴/)
        dest_path.parent.mkdir(parents=True, exist_ok=True)
        
        # 处理极小概率的文件名冲突
        counter = 1
        original_dest = dest_path
        while dest_path.exists():
            dest_path = original_dest.parent / f"{original_dest.stem}_{counter}{original_dest.suffix}"
            counter += 1
            
        try:
            shutil.move(str(file_path), str(dest_path))
            print(f"已移动并保持目录结构: {rel_path}")
        except Exception as e:
            print(f"移动文件 {rel_path} 时出错: {e}")
            
    print("\n所有损坏的文件已处理完毕！")
    print("提示：如果发现误判，你可以直接进入'已损坏pdf文件'文件夹，将里面的文件夹直接拖回原处覆盖即可恢复。")

if __name__ == "__main__":
    main()
```


