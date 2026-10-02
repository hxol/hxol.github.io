---
title: Git 简单配置与用法
date: 2026-10-02 13:01:59
tags: [笔记, Git]
---

# Git 简单配置与用法

Git 是一个分布式的版本控制系统，核心价值在于**记录历史**、**安全回溯**以及**多人协作**。本指南采用模块化结构，方便随时查阅与扩充。

### **核心基石：区域与状态**

理解 Git 的运作机制，本质上是理解文件在以下四个区域之间的流转：

*   **工作区 (Working Directory)**：你在电脑上能看到的项目文件夹，是直接进行代码编写、文件增删改查的地方。
*   **暂存区 (Staging Area / Index)**：一个临时缓冲区域，用于存放你“打算放入下一个版本”的修改。
*   **本地仓库 (Local Repository)**：存放项目完整历史快照的地方。每次提交（Commit）都会在这里生成一个永久的版本记录。
*   **远程仓库 (Remote Repository)**：托管在云端（如 GitHub、GitLab、Gitea）的仓库，用于备份和团队协作。

---

### **身份配置与多账号管理**

在开始任何操作前，Git 需要知道“你是谁”，以便在提交记录中署名。推荐**放弃全局配置，采用仓库级配置**，这是解决“个人开源代码”与“公司私有代码”身份混淆的最佳实践。

**仓库级身份绑定**
在进入任何一个具体的项目文件夹后，执行以下命令绑定当前项目的身份：
```bash
# 个人项目使用
git config --local user.name "你的网名/开源ID"
git config --local user.email "your_personal_email@example.com"

# 公司项目使用
git config --local user.name "你的真实姓名/工号"
git config --local user.email "your_work_email@company.com"
```
> **💡 扩展提示**：`--local` 配置会保存在当前项目根目录的 `.git/config` 文件中，优先级高于全局配置。

---

### **本地核心工作流**

#### 仓库初始化
*   **全新本地项目**：进入项目文件夹后执行 `git init`，会生成一个隐藏的 `.git` 目录。
*   **克隆已有项目**：执行 `git clone <仓库的URL地址>`，会自动下载并初始化。

#### 暂存与提交 (Add & Commit)
这是日常高频操作的循环：

*   **查看状态**：`git status`（红色代表未暂存，绿色代表已暂存等待提交）。
*   **添加到暂存区**：
    *   暂存所有更改（最常用）：`git add .`
    *   暂存指定文件：`git add <文件名>`
*   **生成版本快照**：
    *   执行提交并附带说明：`git commit -m "清晰的修改描述，如：新增用户登录接口"`

#### 历史查看与安全回退
*   **查看日志**：
    *   详细历史：`git log`
    *   单行简明历史（推荐）：`git log --oneline`
*   **撤销与回退**：
    *   **丢弃工作区的修改**（未 add）：`git restore <文件名>` （*Git 2.23+ 新特性，替代旧的 checkout 操作*）
    *   **彻底回退到历史版本**：`git reset --hard <哈希值>`
    > **⚠️ 危险警告**：`--hard` 参数会无情抹除当前未提交的所有代码，操作前务必确认或通过 `git stash` 暂存。

---

### **分支并行开发**

分支允许你从主线（通常是 `main` 或 `master`）分离出来，安全地开发新功能，互不干扰。

*   **查看分支**：`git branch`（带 `*` 号的为当前所在分支）
*   **创建与切换（拥抱新命令 switch）**：
    *   切换到已有分支：`git switch <分支名>`
    *   创建并立即切换到新分支：`git switch -c <新分支名>`
*   **合并分支**：
    假设 `feature-x` 开发完毕，需要合并回主线：
    ```bash
    git switch main           # 1. 切回主分支
    git merge feature-x       # 2. 将目标分支的代码合并过来
    ```

---

### **远程协作与 SSH 免密体系**

推荐使用 SSH 协议替代 HTTPS 协议，一次配置密钥，永久免密推送拉取，且更安全。以下是**多平台（如 GitHub + 公司 Gitea）并存**的终极配置方案。

#### 生成现代化的 SSH 密钥对
采用更现代、更安全的 `ed25519` 算法，为不同平台生成专属密钥：
```bash
# 生成个人 GitHub 密钥
ssh-keygen -t ed25519 -C "your_personal_email@example.com" -f ~/.ssh/github_id

# 生成公司平台密钥
ssh-keygen -t ed25519 -C "your_work_email@company.com" -f ~/.ssh/gitea_company_id
```

#### 配置 SSH 路由规则 (`~/.ssh/config`)
编辑（或创建）`~/.ssh/config` 文件，告诉系统连接哪个平台时使用哪把“钥匙”：

```text
# 个人 GitHub 配置
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/github_id
    IdentitiesOnly yes

# 公司内部平台配置 (Host 为自定义别名，HostName 为真实域名/IP)
Host git.company.com
    HostName git.company.com
    User git
    Port 22      # 如果公司更改了 SSH 端口，在此修改
    IdentityFile ~/.ssh/gitea_company_id
    IdentitiesOnly yes
```

#### 部署公钥与连接测试
将生成的 `.pub` 文件（如 `github_id.pub`）内容复制，粘贴到对应平台（GitHub/Gitea）的 `Settings -> SSH Keys` 中。随后在终端测试：
```bash
ssh -T git@github.com
ssh -T git@git.company.com
```

#### 远程交互命令
*   **关联远程仓库**：`git remote add origin <远程仓库的SSH地址>`
*   **查看当前关联**：`git remote -v`
*   **推送代码**：`git push -u origin main`（首次推送需加 `-u` 绑定上下游，后续直接 `git push`）
*   **拉取更新**：`git pull origin main`

---

### **环境集成与工程化规范**

#### 纯净仓库守则：跨平台忽略规范

在涉及 macOS、Windows、Linux 混合开发或使用 NAS 时，系统会自动生成诸多元数据文件（如 `.DS_Store`）。必须做好防范，隔离系统垃圾。

**防线一：设置全局忽略（斩草除根）**
建立全局忽略名单，避免每次都在不同项目中写一遍：
```bash
# 1. 创建全局忽略文件
touch ~/.gitignore_global

# 2. 写入常见系统垃圾配置
echo ".DS_Store" >> ~/.gitignore_global
echo "._*" >> ~/.gitignore_global
echo "Thumbs.db" >> ~/.gitignore_global

# 3. 告知 Git 生效
git config --global core.excludesfile ~/.gitignore_global
```

**防线二：项目级 `.gitignore` 模板**
在项目根目录创建 `.gitignore`，针对具体技术栈进行屏蔽：

```text
# ==========================
# 操作系统 (OS) 生成的文件
# ==========================
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Windows
Thumbs.db
ehthumbs.db
Desktop.ini
$RECYCLE.BIN/

# Linux
*~

# Synology (群晖)
@eaDir/
*@SynoResource
*@SynoEAStream

# ==========================
# 编辑器 & IDE 配置文件
# ==========================
# VS Code (通常不需要上传本地配置，除非团队强制规范)
.vscode/*
!.vscode/settings.json
!.vscode/tasks.json
!.vscode/launch.json
!.vscode/extensions.json
*.code-workspace

# JetBrains (WebStorm, IDEA, PyCharm)
.idea/
*.iml

# Vim / Sublime / Emacs
*.swp
*.swo
*.sublime-workspace

# ==========================
# 常见博客/Web 依赖 (根据你的技术栈选择)
# ==========================
# Node.js (Hexo, Gatsby, VuePress, Next.js 等)
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
pnpm-debug.log*
.npm/

# Python (MkDocs, Pelican 等)
__pycache__/
*.py[cod]
*$py.class
venv/
.venv/
env/

# Ruby (Jekyll)
.sass-cache/
.jekyll-metadata
Gemfile.lock

# ==========================
# 构建输出 & 临时文件 (非常重要)
# ==========================
# 静态博客生成的最终网页 (Hexo/Hugo/Jekyll 通常生成在这些目录)
public/
dist/
build/
_site/

# Hexo 特有
.deploy_git/
db.json

# Hugo 特有
resources/_gen/

# 临时文件夹
tmp/
temp/

# ==========================
# 安全与日志
# ==========================
# 环境变量 (包含密钥，绝对不能上传)
.env
.env.local
.env.*.local

# 各种日志文件
*.log
```

**急救措施：如果垃圾文件已经提交进仓库怎么办？**
不要慌，执行以下命令从 Git 的追踪索引中剔除它们（不会删除本地硬盘上的实体文件），然后重新提交即可：
```bash
git rm -r --cached .DS_Store
git commit -m "chore: clean macOS junk files"
```