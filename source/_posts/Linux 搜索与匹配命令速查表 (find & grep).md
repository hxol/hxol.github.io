---
title: Linux 搜索与匹配命令速查表 (find & grep)
date: 2026-10-01 15:30:10
tags: [笔记, Linux, find, grep]
---

# Linux 搜索与匹配命令速查表 (find & grep)

## 1. find (文件搜索)
**find** 用于在目录树中查找文件，支持按名称、类型、时间、大小等多种条件，并可对查找结果执行操作。

- **基本语法**：`find [路径] [选项] [操作]`
- **核心技巧**：路径可以用 `.` 代表当前目录，`/` 代表根目录。

### 1.1 常用查找条件
| 选项 | 描述 | 示例 |
| :--- | :--- | :--- |
| `-name` | 按文件名查找 (区分大小写) | `find . -name "demo.txt"` |
| `-iname` | 按文件名查找 (**忽略**大小写) | `find . -iname "Demo.txt"` |
| `-type` | 按类型查找 (`f`=文件, `d`=目录, `l`=链接) | `find . -type d` (只找目录) |
| `-maxdepth`| 指定搜索深度 (避免遍历太深) | `find . -maxdepth 2 -name "*.c"` |
| `-perm` | 按权限查找 | `find . -perm 755` |
| `-user` | 按属主查找 | `find . -user root` |

### 1.2 按时间和大小查找
**时间单位**：`-n` (n天以内/新于n天)，`+n` (n天以前/旧于n天)，`n` (正好n天)。
*注：`-mtime` (修改内容时间), `-atime` (访问时间), `-ctime` (权限/属性变更时间)*

```bash
# 查找 5天以内 修改过的文件
find /var/log -mtime -5

# 查找 3天以前 修改过的文件
find /var/log -mtime +3

# 查找大小大于 100MB 的文件 (k, M, G)
find . -size +100M

# 查找大小为 0 的空文件
find . -size 0
```

### 1.3 查找并执行操作 (exec 与 xargs)
找到文件后，通常需要删除、移动或查看。

**方式一：使用 -exec (标准方式)**
格式：`-exec 命令 {} \;` (`{}`代表文件名，`\;`是结束符)
```bash
# 查找所有 .txt 文件并查看详细信息
find . -name "*.txt" -exec ls -l {} \;

# 查找 5天前的文件并删除 (慎用)
find . -mtime +5 -exec rm -f {} \;

# 安全删除模式 (删除前询问)
find . -name "*.log" -ok rm {} \;
```

**方式二：使用 | xargs (高效方式，推荐)**
`xargs` 可以将结果分批传递给命令，避免参数过长报错，且性能更好。
```bash
# 删除 3天前的文件
find . -mtime +3 | xargs rm -rf

# 查找所有 .c 文件，并在其中搜索 "main" 字符串
find . -name "*.c" | xargs grep "main"

# 查找并备份文件
find . -name "*.conf" | xargs -I {} cp {} {}.bak
```

### 1.4 高级过滤 (Prune)
在搜索时排除特定目录（如 `.git` 或 `node_modules`）。
```bash
# 在当前目录查找文件，但排除 /bin 目录
find . -path "./bin" -prune -o -print
```

---

## 2. grep (文本搜索)
**grep** 用于在文件中搜索包含指定模式（字符串或正则表达式）的行。

- **基本语法**：`grep [选项] "模式" 文件名`

### 2.1 最常用选项
| 选项 | 含义 | 示例 |
| :--- | :--- | :--- |
| `-i` | **忽略大小写** (Ignore case) | `grep -i "error" log.txt` |
| `-v` | **反向查找** (Invert match，显示不匹配的行) | `grep -v "abc" log.txt` |
| `-r` | **递归查找** (Recursive，查目录必用) | `grep -r "main" ./src/` |
| `-n` | 显示行号 (Line number) | `grep -n "TODO" code.c` |
| `-c` | 统计匹配行数 (Count) | `grep -c "Error" log.txt` |
| `-w` | 精确匹配整词 (Word) | `grep -w "is" file` (不匹配 "this") |
| `-l` | 只列出匹配的文件名 (List) | `grep -l "main" *.c` |
| `-E` | 支持扩展正则表达式 (等同于 `egrep`) | `grep -E "a|b" file` |

### 2.2 常见组合场景
```bash
# 在当前目录及子目录下，查找包含 "config" 的行，并显示行号
grep -rn "config" .

# 查找非空行 (排除空行)
grep -v "^$" file.txt

# 查找不包含注释 (#开头) 的行
grep -v "^#" config.conf

# 配合 ps 查找进程
ps -ef | grep nginx | grep -v grep

# 统计文件中 "Error" 出现的行数
grep -c "Error" server.log
```

### 2.3 正则表达式速查
grep 支持正则表达式，配合 `-E` 选项更强大。

**基本元字符**：
*   `^` : 行首锚定 (如 `^abc` 匹配以 abc 开头的行)
*   `$` : 行尾锚定 (如 `abc$` 匹配以 abc 结尾的行)
*   `.` : 匹配任意单个字符
*   `*` : 匹配前一个字符 0 次或多次
*   `[]`: 匹配集合内的字符 (如 `[0-9]` 匹配数字)
*   `[^]`: 匹配集合外的字符 (如 `[^a-z]` 匹配非小写字母)

**扩展元字符 (建议配合 `grep -E`)**：
*   `+` : 匹配前一个字符 1 次或多次
*   `?` : 匹配前一个字符 0 次或 1 次
*   `|` : 或者 (如 `grep -E "error|warn" log.txt`)
*   `()`: 分组

### 2.4 单词锚定
*   `\b` 或 `\< \>` : 锁定单词边界。
    *   `grep "\bgrep\b" file` ：只匹配 "grep"，不匹配 "greps"。