---
title: BentoPDF 部署与本地安装完全
date: 2026-09-30 15:30:00
tags: [笔记, BentoPDF, GitHub Pages, 自托管]
---

# BentoPDF 部署与本地安装完全

## 📌 项目信息

- **GitHub 仓库**: [alam00000/bentopdf](https://github.com/alam00000/bentopdf)
- **官方文档**: [BentoPDF 部署文档](https://www.bentopdf.com/docs/self-hosting/)

> **💡 部署建议**：
> 对于 BentoPDF 这种纯前端项目，**强烈推荐使用 GitHub Pages 免费部署**。除非您需要企业内网访问或自定义 Nginx 配置，否则使用 Docker 部署并没有太多额外收益。

---

## 🚀 方式一：部署到 GitHub Pages（推荐）

### 第一步：Fork 项目仓库
1. 打开 BentoPDF 的 GitHub 仓库：https://github.com/alam00000/bentopdf
2. 点击页面右上角的 **Fork** 按钮，将项目复制到您自己的账号下。
3. 完成后，您的仓库地址将变为类似：`https://github.com/您的用户名/bentopdf`（例如 `hxol/bentopdf`）。

### 第二步：关闭 SEO Audit（⚠️ 重要，防部署失败）
为了防止在 GitHub Actions 构建时，因为 SEO 审查不通过而导致整个部署流程失败，我们需要修改代码直接关闭这一步。
1. 在您 Fork 后的仓库主页，找到并点击打开 `package.json` 文件。
2. 点击右上角的 ✏️ (**Edit this file**) 图标进入编辑模式。
3. 找到代码中的 `"scripts"` 区域里的 `"build"` 这一行（原代码很长）：
   ```json
   "build": "node scripts/generate-blog.mjs && node scripts/generate-static-tool-links.mjs && tsc && vite build && node scripts/enhance-seo-pages.mjs && NODE_OPTIONS='--max-old-space-size=3072' node scripts/generate-i18n-pages.mjs && node scripts/generate-sitemap.mjs && node scripts/generate-security-headers.mjs && node scripts/seo-audit.mjs",
   ```
4. **删除**该行末尾的 `&& node scripts/seo-audit.mjs`，修改后变为：
   ```json
   "build": "node scripts/generate-blog.mjs && node scripts/generate-static-tool-links.mjs && tsc && vite build && node scripts/enhance-seo-pages.mjs && NODE_OPTIONS='--max-old-space-size=3072' node scripts/generate-i18n-pages.mjs && node scripts/generate-sitemap.mjs && node scripts/generate-security-headers.mjs",
   ```
   > **🛑 注意**：只删除 `"build"` 这一行里的这小段，**千万不要删除**它下一行的 `"seo-audit": "node scripts/seo-audit.mjs"`！
5. 修改完成后，点击右上角的绿底按钮 **Commit changes...** 保存提交。

### 第三步：启用 GitHub Pages
1. 在您的仓库首页，依次点击顶部菜单栏的 **Settings** -> 左侧侧边栏的 **Pages**。
2. 在 `Build and deployment` 区域，找到 `Source` 选项。
3. 将其更改为 **GitHub Actions** 并保存。

### 第四步：创建 BASE_URL 环境变量
由于项目部署在您的子路径下，需要配置基础路径变量。
1. 在仓库中依次点击 **Settings** -> 左侧的 **Secrets and variables** -> **Actions** -> 顶部切换到 **Variables** 标签页。
2. 点击绿色的 **New repository variable** 按钮。
3. 填写变量信息：
   - **Name**: `BASE_URL`
   - **Value**: `/bentopdf`
   > **⚠️ 注意**：如果您的仓库名称修改为了其他名字（例如 `my-pdf`），那么此处的 Value 应该对应填写为 `/my-pdf`。

### 第五步：开启 GitHub Actions 权限
1. 点击仓库顶部菜单栏的 **Actions**。
2. 如果是第一次使用，系统会提示确认，点击 **I understand my workflows, go ahead and enable them** 启用工作流。

### 第六步：运行部署流程
1. 在 **Actions** 页面，看左侧的 Workflows 列表。
2. 找到并点击 **Deploy static content to Pages**。
3. 点击右侧的 **Run workflow** 下拉菜单，再次点击绿色的 **Run workflow** 按钮开始运行。

### 第七步：等待构建
1. 构建过程大约需要 **1~3分钟**。
2. 当看到前面出现绿色的勾选图标 `✅ Success` 时，说明网站已经部署成功。

### 第八步：访问网站
部署成功后，您的网站地址通常为：
👉 `https://您的用户名.github.io/bentopdf/`
*(如果您修改了仓库名，例如 `my-pdf`，则访问 `https://您的用户名.github.io/my-pdf/`)*

---

### 🔧 GitHub Pages 维护与进阶

**1. 以后怎么更新？**
如果官方更新了 BentoPDF，您只需要：
- 进入您的 Fork 仓库首页。
- 点击 **Sync fork** 或 **Fetch upstream** 同步最新代码。
- 同步完成后，记得检查一下 `package.json` 中的 `"build"` 是否又被官方加回了 SEO Audit（如有，请按照第二步重新删除一次）。
- 确认无误后，GitHub Actions 会**自动**重新构建并发布，您无需再手动点击部署。

**2. 想绑定自己的域名？**
- 确保网站已成功部署。
- 进入 **Settings** -> **Pages**，找到 **Custom domain**。
- 填写您的自有域名（如 `pdf.example.com`）并保存。
- 前往您的域名服务商控制台，添加对应的 DNS 记录（CNAME 指向您的 `用户名.github.io`）即可。

---

## 🐳 方式二：本地安装（Docker 部署）

如果您需要在本地服务器或 NAS 上运行，可以参考以下 Docker 部署方案。

### 1. 创建工作目录
创建存放 BentoPDF 配置和数据的目录，并进入该目录：
```bash
sudo mkdir -p /opt/bentopdf && cd /opt/bentopdf
```

### 2. 配置目录权限
为避免容器运行时的权限问题，需修改目录所属用户（假设映射到 1000:1001）。
您可以先通过 `id` 命令查看当前用户的 uid 与 gid：
```bash
id
```
然后执行赋权操作：
```bash
sudo chown -R 1000:1001 /opt/bentopdf
sudo chmod -R 755 /opt/bentopdf
```

### 3. 创建编排文件
使用 nano 编辑器创建并编辑 `docker-compose.yml` 文件：
```bash
sudo nano /opt/bentopdf/docker-compose.yml
```
填入以下内容并保存（`Ctrl+O` 回车保存，`Ctrl+X` 退出）：
```yaml
services:
  bentopdf:
    image: ghcr.io/alam00000/bentopdf-simple:latest
    container_name: bentopdf
    restart: unless-stopped
    ports:
      - '1090:8080'  # 宿主机端口 1090 映射到容器端口 8080
    # environment:
      # - DISABLE_IPV6=true # 如果是纯 IPv4 环境，可取消此行注释
    mem_limit: "4096m"  # 限制容器最大内存为 4GB
```

### 4. 运行服务
在 `docker-compose.yml` 所在目录执行启动命令：
```bash
cd /opt/bentopdf
sudo docker compose up -d
```

### 5. 查看状态与日志
启动后，您可以通过以下命令检查容器运行状态和实时日志：
```bash
# 查看容器运行状态
sudo docker compose ps

# 查看实时运行日志（按 Ctrl+C 退出）
sudo docker compose logs -f bentopdf
```
服务启动完成后，即可通过浏览器访问 `http://服务器IP:1090` 使用 BentoPDF。