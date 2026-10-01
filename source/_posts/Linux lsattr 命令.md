---
title: Linux lsattr 命令
date: 2026-10-01 15:30:09
tags: [笔记, Linux, lsattr]
---

chattr 和 lsattr 这两个命令是用来查看和改变文件、目录属性的，与 chmod 这个命令相比，chmod 只是改变文件的读写、执行权限，而更底层的属性控制是由 chattr 来改变的。

# Linux lsattr 命令
这个命令就是很纯粹的用来查看文件/目录属性的，一般使用时可不加任何参数，直接接文件路径即可查看其属性，当然也有一些可选的参数。
```bash
lsattr [-RVadlpv] [files...]
```
具体参数的含义如下：
```
-R ：递归列出目录及其内容的属性。
-V ：显示程序版本。
-a ：列出目录中的所有文件，包括以“.”开头的文件。
-d ：列出与其他文件类似的目录，而不是列出其内容。
-l ：使用长名称而不是单个字符缩写打印选项。
-p ：列出文件的项目编号。
-v ：列出文件的版本号。
```

# chattr 命令
这个命令被用来修改文件/目录的属性，比较常用的就是前面提到的 i 属性：
```bash
chattr [ -RVf ] [ -v version ] [ -p project ] [ mode ] files...
```
其中最关键的是 [mode] 部分，这部分是由 `-+=aAcCdDeijPsStTu` 这些字符组合形成的，是实际控制文件属性的参数。每个字符的具体含义如下：
```
- ：在原有参数设定基础上，移除参数。
+ ：在原有参数设定基础上，追加参数。
= ：更新为指定参数设定。
a ：append only，设定该参数后，只能向文件中添加数据，而不能删除。
A ：no atime updates，atime (access time)不可被修改(modified)。
c ：compresse，设定文件是否经压缩后再存储。读取时需要经过自动解压操作。
C ：no copy on write
d ：no dump，设定文件不能成为dump程序的备份目标。
D ：synchronous  directory  updates，同步目录更新
e ：extent format
i ：immutable，设定文件不能被删除、改名、设定链接关系，同时不能写入或新增内容。
j ：journal，设定此参数使得通过mount参数：data=ordered 或者 data=writeback 挂载的文件系统中，文件在写入时会先被记录(在journal中)。如果filesystem被设定参数为 data=journal，则该参数自动失效。
P ：project hierarchy
s ：secure deletion，保密性地删除文件或目录，即硬盘空间被全部收回。
S ：synchronous updates，硬盘I/O同步选项，功能类似sync。
t ：no tail-merging
T ：top of directory hierarchy
u ：undeletable，与s相反，当设定为u时，数据内容其实还存在磁盘中。
```
