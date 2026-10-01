---
title: Linux grep 命令详解与实战指南
date: 2026-10-01 15:30:02
tags: [笔记, Linux, grep]
---

# Linux grep 命令详解与实战指南

## 1. 简介
**grep** (Global search Regular Expression and Print out the line) 是 Linux 中最强大的文本搜索工具之一，与 `sed`、`awk` 并称为 Linux 文本处理的“三剑客”。

它的主要功能是：**在文件中查找符合条件的字符串（支持正则表达式），并将包含该字符串的行打印出来。**

## 2. 基础语法

```bash
grep [选项] "搜索模式" [文件...]
```
*   **搜索模式**：可以是纯字符串，也可以是正则表达式。建议使用双引号包裹。
*   **文件**：指定要搜索的文件，支持通配符（如 `*.log`）。

---

## 3. 核心常用选项 (必知必会)

这些是日常使用频率最高的选项，建议优先掌握。

### 3.1 搜索控制
*   **`-i` (Ignore Case)**：忽略大小写。
    *   示例：`grep -i "error" server.log` (能匹配 Error, ERROR, error)
*   **`-v` (Invert Match)**：反向匹配，显示**不包含**匹配文本的行。
    *   示例：`grep -v "ok" result.txt` (只显示不包含 ok 的行)
*   **`-r` 或 `-R` (Recursive)**：**[补充]** 递归搜索，查找指定目录及其子目录下所有文件。
    *   示例：`grep -r "db_host" /etc/` (在 /etc 下所有文件中查找 db_host)
*   **`-w` (Word)**：精确匹配整个单词。
    *   示例：`grep -w "is" text.txt` (匹配 "is"，但不匹配 "this" 或 "island")

### 3.2 输出控制
*   **`-n` (Line Number)**：显示匹配行在文件中的行号。
*   **`-c` (Count)**：只统计匹配的**总行数**，不显示具体内容。
*   **`-o` (Only Matching)**：只输出匹配到的**部分内容**，而不是整行。
    *   注意：如果一行有多个匹配项，`-o` 会分多行显示。
*   **`-l` (File with matches)**：**[补充]** 只列出包含匹配项的**文件名**，不显示具体行内容。常配合 `-r` 使用。
    *   示例：`grep -rl "main" ./src/` (列出 src 目录下包含 main 的所有文件)
*   **`-q` (Quiet)**：静默模式，不输出任何信息。
    *   用途：通常用于 Shell 脚本中，通过 `$?` 判断是否找到。找到返回 0，未找到返回 1。

---

## 4. 上下文控制 (Context)

当你需要查看搜索结果附近的行（例如报错信息的前后文）时，这些选项非常有用。

*   **`-A <n>` (After)**：显示匹配行 **及其后 n 行**。
*   **`-B <n>` (Before)**：显示匹配行 **及其前 n 行**。
*   **`-C <n>` (Context)**：显示匹配行 **及其前后各 n 行**。

**实战场景：**
假设 `info.txt` 内容如下：
```text
姓名：张三
年龄：18
职业：学生
```
要查找年龄是18岁的人是谁：
```bash
grep -B 1 "年龄：18" info.txt
# 输出：
# 姓名：张三
# 年龄：18
```

---

## 5. 高级匹配模式

*   **`-e` (Expression)**：指定多个搜索模式（逻辑或）。
    *   示例：`grep -e "error" -e "warning" app.log` (同时搜索 error 和 warning)
*   **`-E` (Extended Regex)**：支持**扩展正则表达式**。
    *   等同于命令 `egrep`。
    *   用途：使用 `|` (或)、`+` (一次或多次)、`?` (零次或一次) 等正则符号时必须加此选项。
    *   示例：`grep -E "error|warning" app.log`
*   **`-F` (Fixed string)**：**[补充]** 将搜索模式视为**固定字符串**，而非正则表达式。
    *   等同于命令 `fgrep`。
    *   用途：当搜索包含 `.`、`*` 等特殊字符的字符串时使用，且速度极快。
*   **`-P` (Perl Regex)**：支持 Perl 格式的正则表达式（功能最强大的正则引擎）。
    *   示例：`grep -P "\d{3}-\d{8}" phone.txt`

---

## 6. 实战演示

假设有一个测试文件 `test.txt`，内容如下：
```text
Hello World
hello linux
test line 1
test line 2
This is a test.
abc
123
```

### 场景 1：忽略大小写并显示行号
```bash
grep -in "hello" test.txt
```
**输出结果：**
```text
1:Hello World
2:hello linux
```

### 场景 2：统计包含 "test" 的行数
```bash
grep -c "test" test.txt
```
**输出结果：**
```text
3
```

### 场景 3：反向查找（查找不包含 test 的行）
```bash
grep -v "test" test.txt
```
**输出结果：**
```text
Hello World
hello linux
abc
123
```

### 场景 4：提取文件中的所有数字（配合正则表达式）
```bash
grep -o "[0-9]\+" test.txt
# 或者使用 -P
grep -P -o "\d+" test.txt
```
**输出结果：**
```text
1
2
123
```
*(注：这里分别提取了第3行的1，第4行的2，和第7行的123)*

---

## 7. 总结速查表

| 选项 | 含义 | 助记 |
| :--- | :--- | :--- |
| **-i** | 忽略大小写 | **I**gnore case |
| **-v** | 反向匹配（排除） | In**v**ert match |
| **-n** | 显示行号 | Line **N**umber |
| **-r / -R** | 递归搜索子目录 | **R**ecursive |
| **-w** | 精确匹配整词 | **W**ord |
| **-o** | 只输出匹配部分 | **O**nly matching |
| **-c** | 统计匹配行数 | **C**ount |
| **-l** | 只显示匹配的文件名 | Fi**l**e with matches |
| **-A n** | 显示后 n 行 | **A**fter |
| **-B n** | 显示前 n 行 | **B**efore |
| **-C n** | 显示前后 n 行 | **C**ontext |
| **-E** | 使用扩展正则表达式 | **E**xtended (相当于 egrep) |
| **-F** | 固定字符串搜索 (快) | **F**ixed (相当于 fgrep) |
| **--color** | 高亮显示匹配项 | Color |

**三剑客对比小贴士：**
*   **grep**：擅长单纯的**查找**和匹配。
*   **sed**：擅长对文本进行**编辑**（替换、删除、插入）。
*   **awk**：擅长对文本进行**格式化处理**和**数据计算**。