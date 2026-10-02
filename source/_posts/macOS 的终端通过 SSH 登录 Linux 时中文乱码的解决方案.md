---
title: macOS 的终端通过 SSH 登录 Linux 时中文乱码的解决方案
date: 2026-10-01 15:30:15
tags: [笔记, Linux, macOS, SSH, 乱码]
---

# macOS 的终端通过 SSH 登录 Linux 时中文乱码的解决方案

## 一、阻止 macOS SSH 客户端发送不必要的 LC_ALL 变量

修改 macOS 的 SSH 客户端配置，阻止发送 LC_ALL 或所有 LC_* 变量。
编辑你的 macOS 上的 SSH 配置文件 ~/.ssh/config
```zsh
nano ~/.ssh/config
```
添加内容
```
Host *
    SendEnv LANG
    SendEnv LC_CTYPE
```

## 二、Debian 服务器端正确配置或生成所需的 locale

安装 locales-all 包
```bash
sudo apt-get update
sudo apt-get install locales-all
```

设置 locale
```bash
sudo dpkg-reconfigure locales
```

## 三、在 Debian 服务器上显式设置 LC_ALL (推荐的永久方案)
```bash
nano ~/.bashrc
```
在文件末尾添加以下行：

```ini
export LANG="en_US.UTF-8"
export LC_ALL="en_US.UTF-8" # 或者直接 export LC_ALL=$LANG
# 或者如果你主要使用中文环境
# export LANG="zh_CN.UTF-8"
# export LC_ALL="zh_CN.UTF-8"
```

也可以更方便的用命令行直接追加
```bash
echo 'export LANG="en_US.UTF-8"' >> ~/.bashrc
```
```
echo 'export LC_ALL="en_US.UTF-8"' >> ~/.bashrc
```

使配置生效
```bash
source ~/.bashrc
```

## 一键脚本

```bash
#!/bin/bash
set -e # 如果任何命令失败，立即退出

# --- 配置 ---
# 你希望设为系统默认的区域设置
DEFAULT_LOCALE="en_US.UTF-8"

# 你希望额外生成并启用的区域设置列表 (空格分隔)
# 确保这些是有效的区域设置名称，例如 "zh_CN.UTF-8"
# 在 /etc/locale.gen 中的格式通常是 "zh_CN.UTF-8 UTF-8"
ADDITIONAL_LOCALES_TO_GENERATE=("zh_CN.UTF-8" "fr_FR.UTF-8")
# --- 结束配置 ---

if [ "$(id -u)" -ne 0 ]; then
  echo "此脚本必须以 root 权限运行。请使用 sudo。" >&2
  exit 1
fi

echo "--- 开始区域设置配置 ---"

# 1. 更新 /etc/locale.gen
echo "[1/4] 更新 /etc/locale.gen..."
# 备份原始文件
BACKUP_FILE="/etc/locale.gen.bak.$(date +%F_%T)"
cp /etc/locale.gen "$BACKUP_FILE"
echo "已备份 /etc/locale.gen 到 $BACKUP_FILE"

# 合并默认和额外的区域设置，确保唯一性
ALL_LOCALES_TO_ENABLE=("$DEFAULT_LOCALE")
for loc in "${ADDITIONAL_LOCALES_TO_GENERATE[@]}"; do
    # 检查是否已在数组中，避免重复
    is_present=false
    for existing_loc in "${ALL_LOCALES_TO_ENABLE[@]}"; do
        if [[ "$existing_loc" == "$loc" ]]; then
            is_present=true
            break
        fi
    done
    if ! $is_present; then
        ALL_LOCALES_TO_ENABLE+=("$loc")
    fi
done


for LOCALE_NAME_WITH_CODESET in "${ALL_LOCALES_TO_ENABLE[@]}"; do
    # /etc/locale.gen 中的条目格式通常是 "en_US.UTF-8 UTF-8"
    # 我们从区域设置名称中提取字符集 (例如，从 en_US.UTF-8 中提取 UTF-8)
    CODESET="${LOCALE_NAME_WITH_CODESET##*.}" # 获取点号之后的部分
    LOCALE_GEN_ENTRY="${LOCALE_NAME_WITH_CODESET} ${CODESET}"

    # 为 sed 转义点号: . 变成 \.
    ESCAPED_LOCALE_GEN_ENTRY="${LOCALE_GEN_ENTRY//./\\.}" # 全局替换点号

    # 检查条目是否存在并且被注释掉了，如果是，则取消注释
    if grep -qP "^\s*#\s*${ESCAPED_LOCALE_GEN_ENTRY}" /etc/locale.gen; then
        echo "在 /etc/locale.gen 中启用 ${LOCALE_GEN_ENTRY}"
        sed -i -E "s/^\s*#\s*(${ESCAPED_LOCALE_GEN_ENTRY})/\1/" /etc/locale.gen
    # 检查条目是否已存在且未被注释
    elif grep -qP "^\s*${ESCAPED_LOCALE_GEN_ENTRY}" /etc/locale.gen; then
        echo "${LOCALE_GEN_ENTRY} 已在 /etc/locale.gen 中启用或配置。"
    # 如果条目完全不存在（既没注释也没取消注释），则添加它
    # 对于标准区域设置这不太常见，但增加了脚本的健壮性
    else
        echo "将 ${LOCALE_GEN_ENTRY} 添加到 /etc/locale.gen"
        echo "${LOCALE_GEN_ENTRY}" >> /etc/locale.gen
    fi
done

# 2. 生成区域设置
echo "[2/4] 使用 locale-gen 生成区域设置..."
locale-gen
echo "locale-gen 完成。"

# 3. 设置系统默认区域设置
echo "[3/4] 使用 update-locale 将系统默认区域设置为 ${DEFAULT_LOCALE}..."
# update-locale 会处理 /etc/default/locale
# 通常设置 LANG 就足够了。如果需要特定的覆盖，可以设置 LC_ALL。
# LANGUAGE 用于消息翻译的优先顺序。
LANGUAGE_VAR="${DEFAULT_LOCALE%%.*}" # 例如，从 en_US.UTF-8 中得到 en_US
update-locale LANG="${DEFAULT_LOCALE}" LC_ALL="${DEFAULT_LOCALE}" LANGUAGE="${LANGUAGE_VAR}"
echo "默认区域设置已设定。"

# 4. 非交互式地重新配置 locales (可选但推荐)
# 这一步确保系统完全认可新的设置，特别是 /etc/default/locale 中的设置。
echo "[4/4] 运行 dpkg-reconfigure --frontend=noninteractive locales..."
dpkg-reconfigure --frontend=noninteractive locales
echo "dpkg-reconfigure locales 完成。"

echo "--- 区域设置配置完成 ---"
echo "当前系统区域设置 (来自 /etc/default/locale):"
cat /etc/default/locale
echo ""
echo "要将更改应用到当前的 shell 会话，你可能需要重新加载你的配置文件 (例如 source ~/.bashrc) 或注销后重新登录。"
echo "对于新的会话和系统服务，更改将自动生效。"

sudo apt-get update
sudo apt-get install locales-all -y

echo 'export LANG="en_US.UTF-8"' >> ~/.bashrc

echo 'export LC_ALL="en_US.UTF-8"' >> ~/.bashrc

```
