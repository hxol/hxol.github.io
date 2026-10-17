---
title: Linux alias 命令
date: 2026-10-01 15:30:07
tags: [笔记, Linux, alias]
---

# Linux alias 命令

# 语法
```bash
alias [别名]='[指令名称]'
```
参数说明：若不加任何参数，则列出目前所有的别名设置。

## 实例

### 创建别名
```bash
alias ll='ls -alF'
```
此命令创建一个名为 ll 的别名，用于显示当前目录下所有文件和目录的详细列表。

### 查看所有别名
```bash
alias
```
此命令将显示当前系统上所有的别名及其相应的命令。

### 删除别名
```bash
unalias ll
```
此命令将删除名为 ll 的别名。


以 root 权限执行命令：
```bash
alias sudo='sudo '
```
此命令将创建一个名为 sudo 的别名，以便您可以通过在命令前加上 sudo 来以 root 权限执行命令。

注意：在命令末尾有一个空格，这是因为如果没有空格，则无法正确识别以root权限执行的命令。

### 永久生效

bash     
添加进 `~/.bashrc`

zsh      
添加进 `~/.zshrc`


alias 实际上是文本替换，不是真正的函数。

### 例子

alias
```bash
echo "alias gx='sudo apt update && sudo apt full-upgrade -y'" >> ~/.bashrc
```

函数
```bash
echo '
gx() {
    sudo apt update &&
    sudo apt full-upgrade -y &&
    sudo apt autoremove -y &&
    sudo apt autoclean
}
' >> ~/.bashrc
```

```bash
source ~/.bashrc
```