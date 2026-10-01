---
title: Linux AWK 命令
date: 2026-10-01 15:30:00
tags: [笔记, Linux, awk]
---

# Linux AWK 命令

**简介**：AWK 是一种强大的文本处理工具和编程语言，也是 Linux/Unix 下的“三剑客”之一（grep, sed, awk）。它特别擅长处理结构化数据（如日志、CSV），生成报表和格式化输出。

> **版本说明**：本指南基于 Linux 系统中最常用的 **GNU awk (gawk)**。

---

## 1. 基础语法与工作原理

### 1.1 基本结构
AWK 是逐行处理文本的。
```bash
awk [选项] '模式 {动作}' 文件名
```
*   **模式 (Pattern)**：条件。如果当前行满足条件，则执行动作。如果不写模式，默认处理所有行。
*   **动作 (Action)**：具体操作（如打印、计算）。如果不写动作，默认打印整行。

### 1.2 核心概念
*   **记录 (Record)**：默认一行就是一个记录。
*   **字段 (Field)**：一行文本被分隔符（默认是空格）切开后的每一段。
*   **$0**：代表整行文本。
*   **$1, $2...**：代表第一列、第二列，以此类推。
*   **$NF**：代表最后一列（NF 是字段数量，`$NF` 取对应内容）。

---

## 2. 常用命令快速查阅 (Cheatsheet)

以下是日常最高频使用的场景：

| 场景 | 命令示例 | 说明 |
| :--- | :--- | :--- |
| **打印特定列** | `awk '{print $1, $3}' file.txt` | 打印第1和第3列 |
| **打印整行** | `awk '{print $0}' file.txt` | 相当于 `cat` |
| **指定分隔符** | `awk -F: '{print $1}' /etc/passwd` | 以冒号分隔，打印用户名 |
| **打印最后两列** | `awk '{print $(NF-1), $NF}' file.txt` | 使用算术运算定位倒数第二列 |
| **行号过滤** | `awk 'NR==5' file.txt` | 只处理第5行 |
| **正则匹配** | `awk '/error/ {print $0}' log.txt` | 打印包含 "error" 的行 |
| **条件过滤** | `awk '$3 > 500' file.txt` | 如果第3列数值大于500则打印 |
| **添加表头** | `awk 'BEGIN{print "Name\tID"} {print $1,$2}' file` | 处理前先打印表头 |
| **统计行数** | `awk 'END{print NR}' file.txt` | 处理结束后打印总行号 |

---

## 3. 字段与分隔符

AWK 处理文本的核心在于如何“切分”文本。

### 3.1 输入分隔符 (FS)
默认情况下，AWK 将连续的空格或制表符视为一个分隔符。
*   **使用 `-F` 选项**：
    ```bash
    awk -F"#" '{print $1}' file.txt  # 指定 # 为分隔符
    ```
*   **使用 `FS` 变量**：
    ```bash
    awk -v FS="#" '{print $1}' file.txt
    # 或者在 BEGIN 中定义
    awk 'BEGIN{FS="#"}{print $1}' file.txt
    ```

### 3.2 输出分隔符 (OFS)
`print` 语句中，用逗号 `,` 分隔字段时，输出默认用空格连接。可以通过 `OFS` 修改。
```bash
# 将输出分隔符修改为 "---"
awk -v OFS="---" '{print $1, $2}' file.txt
# 输出结果示例: column1---column2
```

### 3.3 记录分隔符 (RS 与 ORS)
*   **RS (Input Record Separator)**：输入换行符，默认为 `\n`（回车）。如果改为 ` `（空格），awk 会认为遇到空格就是新的一行。
*   **ORS (Output Record Separator)**：输出换行符，默认为 `\n`。如果改为 `+`，print 输出时不换行，而是连成一串用 `+` 隔开。

---

## 4. 变量详解

### 4.1 内置变量
AWK 预定义了一组变量来描述当前处理的状态：

| 变量 | 含义 | 备注 |
| :--- | :--- | :--- |
| **FS** | 输入字段分隔符 | 默认为空格 |
| **OFS** | 输出字段分隔符 | 默认为空格 |
| **NF** | (Number of Fields) 字段数量 | 当前行有多少列 |
| **NR** | (Number of Records) 行号 | 当前处理到第几行（全局计数） |
| **FNR** | 文件内行号 | 处理多个文件时，每个文件单独计数 |
| **RS** | 输入记录分隔符 | 默认为换行符 |
| **ORS** | 输出记录分隔符 | 默认为换行符 |
| **FILENAME** | 当前文件名 | 正在处理的文件名 |
| **ARGC** | 命令行参数个数 | 包括 awk 命令本身 |
| **ARGV** | 命令行参数数组 | 存储参数的具体内容 |

**FNR 与 NR 的区别示例**：
假设 `file1` 有2行，`file2` 有2行：
```bash
awk '{print "NR:"NR " FNR:"FNR " Content:"$0}' file1 file2
```
*输出：*
> NR:1 FNR:1 Content:file1_line1
> NR:2 FNR:2 Content:file1_line2
> NR:3 FNR:1 Content:file2_line1  <-- 注意这里 FNR 重置为1，NR 继续增加
> NR:4 FNR:2 Content:file2_line2

### 4.2 自定义变量
*   **使用 `-v` 参数**：`awk -v myvar="hello" '{print myvar, $1}' file`
*   **在脚本中定义**：`awk '{a=1; b="test"; print a, b}' file`

---

## 5. 格式化输出 (printf)

当 `print` 无法满足对齐、精度需求时，使用 `printf`。其用法与 C 语言完全一致。

**语法**：`printf "格式字符串", 参数1, 参数2...`
*注意：printf 不会自动换行，需手动添加 `\n`。*

**常用格式符**：
*   `%s`：字符串
*   `%d`：十进制整数
*   `%f`：浮点数
*   `%10s`：占用10个字符宽度，右对齐
*   `%-10s`：占用10个字符宽度，左对齐
*   `%.2f`：保留两位小数

**示例**：格式化输出 /etc/passwd 的用户名和 ID
```bash
awk -F: '{printf "User: %-15s ID: %8d\n", $1, $3}' /etc/passwd
```

---

## 6. 模式 (Pattern) - 筛选条件

模式决定了是否对当前行执行 `{动作}`。

### 6.1 特殊模式
*   **BEGIN**：在读取文件**之前**执行一次。常用于初始化变量、打印表头、修改 FS。
*   **END**：在处理完所有行**之后**执行一次。常用于打印汇总结果。

### 6.2 关系运算模式
支持 `> < >= <= == !=`。
```bash
awk '$3 >= 500 {print $1}' /etc/passwd  # 打印 UID 大于等于 500 的用户
```

### 6.3 正则模式
支持扩展正则表达式。
*   **匹配**：`/正则/` 或 `~ /正则/`
*   **不匹配**：`!~ /正则/`

```bash
awk '/^root/ {print $0}' /etc/passwd     # 打印以 root 开头的行
awk '$0 ~ /error/ {print $0}' log.txt    # 打印包含 error 的行
awk '$2 !~ /^\d+$/ {print $0}' data.txt  # 打印第二列不是数字的行
```

### 6.4 行范围模式
从匹配 start 的行开始，到匹配 end 的行结束。
```bash
awk '/StartPattern/, /EndPattern/ {print $0}' file.txt
```
*注意：如果在同一行中同时匹配到 start 和 end，范围也会生效。*

---

## 7. 流程控制

AWK 拥有完整的编程控制结构。

### 7.1 If 判断
```awk
{
    if ($3 < 500) {
        print "System User: " $1
    } else if ($3 >= 1000) {
        print "Common User: " $1
    } else {
        print "Other: " $1
    }
}
```

### 7.2 循环 (Loop)
*   **For 循环** (C语言风格)：
    ```awk
    BEGIN {
        for (i=1; i<=5; i++) print i
    }
    ```
*   **While 循环**：
    ```awk
    {
        i=1
        while (i <= NF) {
            print $i
            i++
        }
    }
    ```

### 7.3 控制语句
*   `break`：跳出整个循环。
*   `continue`：跳出本次循环。
*   `next`：**跳过当前行**，直接开始处理下一行（类似于循环中的 continue，但是针对的是行读取）。
*   `exit`：退出 awk 执行（如果存在 END 块，会跳转到 END 执行）。

---

## 8. 数组 (Array)

AWK 的数组是**关联数组**，下标可以是数字或字符串。这使得统计功能非常强大。

### 8.1 基本操作
*   **赋值**：`arr["name"] = "zhangsan"`
*   **删除**：`delete arr["name"]`
*   **检查存在**：`if ("name" in arr)`

### 8.2 遍历数组
由于是关联数组，遍历顺序通常是无序的。
```awk
for (key in arr) {
    print key, arr[key]
}
```

### 8.3 经典案例：统计日志中 IP 出现次数
假设日志文件 `access.log` 第一列是 IP：
```bash
awk '{count[$1]++} END{for(ip in count) print ip, count[ip]}' access.log
```
**原理**：
1. `count[$1]++`：遇到一个 IP，就将该 IP 作为下标，对应的值加 1（利用了未定义变量参与运算默认为0的特性）。
2. `END`：处理完所有行后，遍历 `count` 数组输出结果。

---

## 9. 常用内置函数

### 9.1 字符串函数
*   **length([s])**：返回字符串长度。如果不指定参数，默认是 `$0`。
*   **index(s, t)**：返回子串 t 在 s 中出现的位置。
*   **match(s, r)**：测试 s 是否包含匹配正则 r 的字符串。
*   **split(s, a, sep)**：将字符串 s 用 sep 分隔符切分，存入数组 a。
    ```awk
    str="A:B:C"; split(str, arr, ":"); # arr[1]="A", arr[2]="B"...
    ```
*   **sub(r, t, s)**：将字符串 s 中**第一次**匹配正则 r 的部分替换为 t。
*   **gsub(r, t, s)**：**全局**替换。
*   **substr(s, i, [n])**：截取字符串 s，从第 i 位开始，截取 n 个字符。
*   **tolower(s) / toupper(s)**：大小写转换。

### 9.2 算术函数
*   **int(x)**：取整。
*   **rand()**：生成 0-1 之间的随机数。
*   **srand()**：设置随机数种子（通常配合 `rand` 使用，否则每次运行随机数一样）。
    ```awk
    BEGIN{srand(); print int(rand()*100)} # 生成100以内随机整数
    ```

### 9.3 排序函数 (gawk 特有)
*   **asort(arr)**：根据数组的值进行排序，下标会被重置为 1, 2, 3...
*   **asorti(arr)**：根据数组的下标进行排序。

---

## 10. 进阶技巧与拾遗

### 10.1 三元运算
语法：`条件 ? 结果1 : 结果2`
```bash
# 判断用户类型
awk -F: '{type = ($3<500) ? "Sys" : "User"; print $1, type}' /etc/passwd
```

### 10.2 奇偶行打印技巧
利用 AWK 的运行机制：
1. 模式的结果为“真”（非0或非空）时，默认动作为打印整行。
2. 变量 `i` 未初始化时默认为空（假）。

**打印奇数行**：
```bash
awk 'i=!i' file.txt
```
*解析*：
*   第1行：i (空) -> !i (真) -> 赋值给 i。模式为真，打印。
*   第2行：i (真) -> !i (假) -> 赋值给 i。模式为假，不打印。
*   以此类推。

**打印偶数行**：
```bash
awk '!(i=!i)' file.txt
```

### 10.3 外部命令执行
*   **system()**：执行 shell 命令。
    ```awk
    BEGIN { system("ls -l") }
    ```
*   **cmd | getline**：获取 shell 命令的输出结果到变量。
    ```awk
    "date" | getline current_time; print current_time
    ```

---

**总结**：AWK 是一门精深的数据处理语言。掌握 `Pattern {Action}` 结构、数组统计以及 `printf` 格式化，即可解决 Linux 运维与数据清洗中 90% 的问题。