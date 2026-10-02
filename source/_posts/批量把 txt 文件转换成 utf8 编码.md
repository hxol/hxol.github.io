---
title: 批量把 txt 文件转换成 utf8 编码
date: 2026-10-02 13:02:01
tags: [笔记, utf8, txt, python]
---

# 批量把 txt 文件转换成 utf8 编码

## 目标

自动识别本目录内每个txt文件现在的编码，然后正确的把编码转换成utf-8，最后检查所有转码后的txt的健康状况，有问题则统计并在最后显示。

## 环境配置

### 第一步：确保系统安装了创建虚拟环境的工具
Debian/Ubuntu 系统默认可能没有安装 `venv` 模块，你需要先用系统包管理器装一下（如果已经安装过，系统会提示已是最新版）：
```bash
sudo apt update
sudo apt install python3-venv
```

### 第二步：在当前目录创建一个虚拟环境
在存放 txt 文件的目录下，运行以下命令来创建一个名为 `.venv` 的虚拟环境（前面加个点是为了让它成为隐藏文件夹，不干扰你的目录视线）：
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
现在，可以放心地使用 `pip` 安装库了，这只会安装到刚才那个 `.venv` 文件夹里，绝不会影响你的系统：
```bash
pip install chardet
```

安装完成后，直接运行脚本(内容见后面)：
```bash
python convert_and_check.py
```
*(注意：在虚拟环境里，直接打 `python` 就会默认调用 `python3`)*

### 第五步：退出虚拟环境（收尾）
当脚本运行完毕，你的 txt 转换任务结束后，你可以通过以下命令退出虚拟环境，恢复到正常的系统状态：
```bash
deactivate
```
*执行后，命令行的 `(.venv)` 前缀就会消失。*

---

### Python 脚本内容

```python
import os
import chardet
from pathlib import Path

def process_txt_files():
    current_dir = Path.cwd()
    # 需求2：使用 rglob 递归查找当前目录及所有子目录下的 .txt 文件
    txt_files = list(current_dir.rglob('*.txt'))

    if not txt_files:
        print("⚠️ 当前目录及所有子目录下没有找到任何 .txt 文件。")
        return

    print(f"🔍 找到 {len(txt_files)} 个 .txt 文件，开始安全检测并转码...\n")

    success_count = 0
    skipped_count = 0  # 新增：记录已经是 UTF-8 而跳过的文件数
    problem_files = []

    for file_path in txt_files:
        # 使用相对路径，方便在多层目录时看清是哪个文件
        rel_path = file_path.relative_to(current_dir)
        temp_file_path = file_path.with_suffix('.txt.tmp')
        
        try:
            with open(file_path, 'rb') as f:
                raw_data = f.read()

            if not raw_data:
                problem_files.append((str(rel_path), "文件为空（无需转换）"))
                continue

            # 需求1 & 4(英文)：最快检测是否已是 UTF-8（或纯英文ASCII）
            # 直接尝试用 utf-8 解码，如果没报错，说明它已经是标准 UTF-8，直接跳过
            try:
                raw_data.decode('utf-8')
                skipped_count += 1
                continue  # 进入下一个文件，不修改原文件
            except UnicodeDecodeError:
                pass # 报错说明不是 UTF-8，继续往下走检测逻辑

            # 1. 自动识别文件原编码（走到这里说明是非UTF-8编码，如繁体、日文等）
            result = chardet.detect(raw_data)
            encoding = result['encoding']
            
            if not encoding:
                problem_files.append((str(rel_path), "无法识别原文件编码"))
                continue

            # 需求4：优化各语言兼容性
            encoding_lower = encoding.lower()
            if encoding_lower in ['gb2312', 'gbk']:
                # 简体中文兼容扩展
                encoding = 'gb18030'
            elif encoding_lower == 'big5':
                # 繁体中文兼容扩展 (香港增补字符集，包含更多繁体字)
                encoding = 'big5hkscs'
            # 日文通常会被识别为 shift_jis 或 euc-jp，直接用该编码解码即可

            # 2. 内存中解码
            try:
                text_data = raw_data.decode(encoding)
            except UnicodeDecodeError as e:
                problem_files.append((str(rel_path), f"以识别的编码 [{encoding}] 解码失败: {str(e)}"))
                continue

            # 3. 安全写入：先写入同目录的临时文件，绝不触碰原文件
            with open(temp_file_path, 'w', encoding='utf-8') as f:
                f.write(text_data)

            # 4. 健康检查：读取临时文件进行严格验证
            with open(temp_file_path, 'r', encoding='utf-8', errors='strict') as f:
                f.read()

            # 5. 安全替换：需求3 (原地替换，由于 file_path 包含原目录路径，文件位置绝不会变)
            temp_file_path.replace(file_path)
            
            success_count += 1

        except UnicodeError:
            problem_files.append((str(rel_path), "转码后健康检查失败：生成的 UTF-8 数据存在异常"))
            if temp_file_path.exists():
                temp_file_path.unlink()
                
        except Exception as e:
            problem_files.append((str(rel_path), f"处理过程中发生未知错误：{str(e)}"))
            if temp_file_path.exists():
                temp_file_path.unlink()

    # ==============================
    # 统计与最终结果展示
    # ==============================
    print("-" * 50)
    print("📊 安全转换与健康检查任务完成！统计报告如下：")
    print(f"总计找到文件: {len(txt_files)} 个")
    print(f"⏭️ 无需修改（已是 UTF-8 或 纯英文）: {skipped_count} 个")
    print(f"✅ 成功转码为 UTF-8 的文件: {success_count} 个")
    
    if problem_files:
        print(f"❌ 存在问题（已跳过且未损坏源文件）的文件: {len(problem_files)} 个")
        print("\n🚨 发现问题的文件列表及原因统计：")
        for name, reason in problem_files:
            print(f"  -> 【{name}】 : {reason}")
    else:
        print(f"❌ 存在问题的文件: 0 个")
        print("\n🎉 太棒了！所有文件均处理完毕，未发现任何错误！")

if __name__ == "__main__":
    process_txt_files()
```