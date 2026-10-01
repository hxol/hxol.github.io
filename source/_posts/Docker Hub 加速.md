---
title: Docker Hub 加速
date: 2026-09-30 15:30:00
tags: [笔记, Docker Hub]
---


> **转载说明**：本文为转载文章。原作者为 **Dirge**，原文链接：[https://www.iszy.cc/posts/nginx-docker-hub/](https://www.iszy.cc/posts/nginx-docker-hub/)。文章所有内容除特别声明外，均采用 BY-NC-SA 许可协议。转载请务必保留此声明及文末的出处信息。

# Docker Hub 加速

众所周知，目前在国内往往无法顺畅拉取 Docker 镜像。常见的解决方式要么是部署私有仓库，要么是使用国内的公共镜像源。然而，国内公共镜像源的版本同步往往不够及时，且近期许多公共镜像源因各种原因相继失效；而部署私有仓库会在本地缓存大量镜像包，对于只需加速拉取的需求来说过于笨重。

因此，我最终决定通过 Nginx 反向代理 Docker Hub 官方 Registry 地址来实现加速。如果你手头正好有一台能够流畅访问官方 Docker 地址的海外服务器，不妨尝试以下方案。

---

## 一、Nginx 反代方案

### 1. 准备工作与环境搭建

首先，在你的服务器上安装 Nginx 以及申请 SSL 证书所需的 Certbot 工具（本文以配合 Cloudflare DNS 验证为例）：

```bash
sudo apt update && sudo apt install nginx certbot python3-certbot-dns-cloudflare -y
```

创建用于存放 Cloudflare API 凭证的目录并赋予对应权限：

```bash
sudo mkdir -p /etc/cloudflare && sudo chmod 700 /etc/cloudflare
```

写入 Cloudflare API Token（**注意：请将下方 `YOUR_CLOUDFLARE_API_TOKEN` 替换为你自己在 CF 后台生成的真实 Token**）：

```bash
sudo tee /etc/cloudflare/cloudflare.ini > /dev/null << 'EOF'
dns_cloudflare_api_token = YOUR_CLOUDFLARE_API_TOKEN
EOF
```

修改凭证文件的权限以确保安全：

```bash
sudo chmod 600 /etc/cloudflare/cloudflare.ini
```

使用 Certbot 申请泛域名泛证书（**注意：请将 `example.com` 替换为你的实际域名**）：

```bash
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini \
  --dns-cloudflare-propagation-seconds 40 \
  -d example.com -d '*.example.com' \
  --register-unsafely-without-email \
  --deploy-hook "systemctl reload nginx"
```

### 2. 配置 Nginx 代理

创建一个新的 Nginx 站点配置文件：

```bash
sudo nano /etc/nginx/sites-available/docker-proxy.conf
```

将以下配置粘贴进去。此基础配置代理了官方仓库地址 (`registry-1.docker.io`)、JWT 授权地址 (`auth.docker.io`) 以及 API 地址 (`index.docker.io`)。
*(注：单纯使用 Nginx 会受到 Docker Hub 单 IP 请求次数的限制，后续的“整合方案”将解决此问题)*

```nginx
# 使用 map 来匹配和替换 upstream 头部中的 auth.docker.io
map $upstream_http_www_authenticate $m_www_authenticate_replaced {
    "~auth\.docker\.io(.*)" "$1";
    default "";
}

map $m_www_authenticate_replaced $m_final_replaced {
    "~(.*)" 'Bearer realm=\"$scheme://$host$1';
    default "";
}

server {
    listen 443 ssl http2;
    
    # 替换为你要使用的加速域名
    server_name docker.example.com;

    # SSL 证书配置 (请替换为你的实际域名路径)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 24h;

    # TLS 版本与加密套件控制
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;      
    ssl_ciphers TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:EECDH+CHACHA20:EECDH+AESGCM:EECDH+AES;

    proxy_ssl_server_name on;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 修改 JWT 授权地址
    proxy_hide_header www-authenticate;
    add_header www-authenticate "$m_final_replaced" always;

    # 关闭缓存
    proxy_buffering off;
    
    # 转发认证相关 Header
    proxy_set_header Authorization $http_authorization;
    proxy_pass_header Authorization;

    # 对 upstream 状态码检查，实现 error_page 错误重定向
    proxy_intercept_errors on;
    recursive_error_pages on;
    # 根据状态码执行对应操作 (301、302、307 触发重定向处理)
    error_page 301 302 307 = @handle_redirect;

    # v1 api
    location /v1 {
        proxy_pass https://index.docker.io;
        proxy_set_header Host index.docker.io;
    }

    # v2 api
    location /v2 {
        proxy_pass https://index.docker.io;
        proxy_set_header Host index.docker.io;
    }

    # JWT 授权地址
    location /token {
        proxy_pass https://auth.docker.io;
        proxy_set_header Host auth.docker.io;
    }

    # Docker Hub 官方镜像仓库
    location / {
        proxy_pass https://registry-1.docker.io;
        proxy_set_header Host registry-1.docker.io;
    }
    
    # 处理重定向
    location @handle_redirect {
        resolver 1.1.1.1;
        set $saved_redirect_location '$upstream_http_location';
        proxy_pass $saved_redirect_location;
    }
}
```

启用配置并重启 Nginx：

```bash
sudo ln -s /etc/nginx/sites-available/docker-proxy.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 二、Cloudflare Worker 方案

Cloudflare Worker 在国内的访问速度可能受网络波动影响，但胜在完全免费，且速度总体优于直接拉取官方镜像，非常适合作为备用方案。

**部署步骤：**
1. 登录 Cloudflare 面板，在左侧菜单找到 **“Workers 和 Pages”**。
2. 点击 **“创建应用程序”** -> **“创建 Worker”**。
3. 修改一个你喜欢的 Worker 名称，点击 **“部署”**。
4. 部署完成后，点击 **“编辑代码”**。
5. 在 `worker.js` 里粘贴以下内容（**注意修改 `workers_url` 变量为你的实际 Worker 地址或绑定的自定义域名**），点击保存并部署即可。

```javascript
'use strict'

const hub_host = 'registry-1.docker.io'
const auth_url = 'https://auth.docker.io'
// 请将其替换为实际的 Worker 地址，或者绑定的自定义域名
const workers_url = 'https://docker.your-domain.workers.dev' 

/** @type {RequestInit} */
const PREFLIGHT_INIT = {
    status: 204,
    headers: new Headers({
        'access-control-allow-origin': '*',
        'access-control-allow-methods': 'GET,POST,PUT,PATCH,TRACE,DELETE,HEAD,OPTIONS',
        'access-control-max-age': '1728000',
    }),
}

function makeRes(body, status = 200, headers = {}) {
    headers['access-control-allow-origin'] = '*'
    return new Response(body, {status, headers})
}

function newUrl(urlStr) {
    try {
        return new URL(urlStr)
    } catch (err) {
        return null
    }
}

addEventListener('fetch', e => {
    const ret = fetchHandler(e)
        .catch(err => makeRes('cfworker error:\n' + err.stack, 502))
    e.respondWith(ret)
})

async function fetchHandler(e) {
  const getReqHeader = (key) => e.request.headers.get(key);

  let url = new URL(e.request.url);

  if (url.pathname === '/token') {
      let token_parameter = {
        headers: {
        'Host': 'auth.docker.io',
        'User-Agent': getReqHeader("User-Agent"),
        'Accept': getReqHeader("Accept"),
        'Accept-Language': getReqHeader("Accept-Language"),
        'Accept-Encoding': getReqHeader("Accept-Encoding"),
        'Connection': 'keep-alive',
        'Cache-Control': 'max-age=0'
        }
      };
      let token_url = auth_url + url.pathname + url.search
      return fetch(new Request(token_url, e.request), token_parameter)
  }

  url.hostname = hub_host;
  
  let parameter = {
    headers: {
      'Host': hub_host,
      'User-Agent': getReqHeader("User-Agent"),
      'Accept': getReqHeader("Accept"),
      'Accept-Language': getReqHeader("Accept-Language"),
      'Accept-Encoding': getReqHeader("Accept-Encoding"),
      'Connection': 'keep-alive',
      'Cache-Control': 'max-age=0'
    },
    cacheTtl: 3600
  };

  if (e.request.headers.has("Authorization")) {
    parameter.headers.Authorization = getReqHeader("Authorization");
  }

  let original_response = await fetch(new Request(url, e.request), parameter)
  let original_response_clone = original_response.clone();
  let original_text = original_response_clone.body;
  let response_headers = original_response.headers;
  let new_response_headers = new Headers(response_headers);
  let status = original_response.status;

  if (new_response_headers.get("WWW-Authenticate")) {
    let re = new RegExp(auth_url, 'g');
    new_response_headers.set("WWW-Authenticate", response_headers.get("WWW-Authenticate").replace(re, workers_url));
  }

  if (new_response_headers.get("Location")) {
    return httpHandler(e.request, new_response_headers.get("Location"))
  }

  let response = new Response(original_text, {
            status,
            headers: new_response_headers
        })
  return response;
}

function httpHandler(req, pathname) {
    const reqHdrRaw = req.headers
    if (req.method === 'OPTIONS' && reqHdrRaw.has('access-control-request-headers')) {
        return new Response(null, PREFLIGHT_INIT)
    }
    const reqHdrNew = new Headers(reqHdrRaw)
    let urlStr = pathname
    const urlObj = newUrl(urlStr)

    const reqInit = {
        method: req.method,
        headers: reqHdrNew,
        redirect: 'follow',
        body: req.body
    }
    return proxy(urlObj, reqInit, '', 0)
}

async function proxy(urlObj, reqInit, rawLen) {
    const res = await fetch(urlObj.href, reqInit)
    const resHdrOld = res.headers
    const resHdrNew = new Headers(resHdrOld)

    if (rawLen) {
        const newLen = resHdrOld.get('content-length') || ''
        const badLen = (rawLen !== newLen)
        if (badLen) {
            return makeRes(res.body, 400, {
                '--error': `bad len: ${newLen}, except: ${rawLen}`,
                'access-control-expose-headers': '--error',
            })
        }
    }
    const status = res.status
    resHdrNew.set('access-control-expose-headers', '*')
    resHdrNew.set('access-control-allow-origin', '*')
    resHdrNew.set('Cache-Control', 'max-age=1500')
    
    resHdrNew.delete('content-security-policy')
    resHdrNew.delete('content-security-policy-report-only')
    resHdrNew.delete('clear-site-data')

    return new Response(res.body, {
        status,
        headers: resHdrNew
    })
}
```

---

## 三、终极整合方案（推荐）

直接访问 Docker Hub 容易触发官方的 429 (Too Many Requests) 速率限制。完美的解决方案是：**以 Nginx 代理为主，当超出请求数量限制返回 429 错误时，将后端请求无缝转发给 Cloudflare Worker 处理。**

### 1. 修改 Nginx 配置

在原有 Nginx 配置中增加对 429 状态码的拦截与转发：

```nginx
# 使用 map 来匹配和替换 upstream 头部中的 auth.docker.io
map $upstream_http_www_authenticate $m_www_authenticate_replaced {
    "~auth\.docker\.io(.*)" "$1";
    default "";
}

map $m_www_authenticate_replaced $m_final_replaced {
    "~(.*)" 'Bearer realm=\"$scheme://$host$1';
    default "";
}

server {
    listen 443 ssl http2;
    server_name docker.example.com; # 改为你的 Nginx 代理域名

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ssl_session_timeout 24h;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;      
    ssl_ciphers TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:EECDH+CHACHA20:EECDH+AESGCM:EECDH+AES;

    proxy_ssl_server_name on;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_hide_header www-authenticate;
    add_header www-authenticate "$m_final_replaced" always;

    proxy_buffering off;
    proxy_set_header Authorization $http_authorization;
    proxy_pass_header  Authorization;

    proxy_intercept_errors on;
    recursive_error_pages on;
    
    # 301、302、307 触发重定向
    error_page 301 302 307 = @handle_redirect;
    # 重点：429 错误触发 CF Worker 转发
    error_page 429 = @handle_too_many_requests;

    location /v1 {
        proxy_pass https://index.docker.io;
        proxy_set_header Host index.docker.io;
    }

    location /v2 {
        proxy_pass https://index.docker.io;
        proxy_set_header Host index.docker.io;
    }

    location /token {
        proxy_pass https://auth.docker.io;
        proxy_set_header Host auth.docker.io;
    }

    location / {
        proxy_pass https://registry-1.docker.io;
        proxy_set_header Host registry-1.docker.io;
    }
    
    location @handle_redirect {
        resolver 1.1.1.1;
        set $saved_redirect_location '$upstream_http_location';
        proxy_pass $saved_redirect_location;
    }

    # 处理 429 错误，回源到 CF Worker
    location @handle_too_many_requests {
        # 替换为你刚才创建的 CF Worker 域名
        proxy_set_header Host docker.your-domain.workers.dev;  
        proxy_pass https://docker.your-domain.workers.dev;
    }
}
```

### 2. 修改 Worker.js 内容

为了让授权流程闭环，当 Nginx 将请求转发给 Worker 时，Worker 需要告诉客户端返回 Nginx 获取 token，因此需要将 `worker.js` 顶部的 `workers_url` 改为你的 **Nginx 代理域名**。

```javascript
// 在 CF Worker 代码顶部修改此行
const workers_url = 'https://docker.example.com' // 改为你的 Nginx 代理域名
```

修改完成后，重新部署 Worker，并重载 Nginx（`sudo systemctl reload nginx`）即可生效。

---

## 四、如何使用自建的加速镜像

配置完成后，在你需要拉取镜像的机器上（如国内的服务器、NAS 或本地电脑），修改 Docker 配置文件 `/etc/docker/daemon.json`（如果文件不存在则新建）：

```json
{
  "registry-mirrors": [
    "https://docker.example.com"
  ]
}
```
*注：请将 `https://docker.example.com` 替换为您最终配置好的 Nginx 代理域名。*

修改保存后，重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

现在，你可以享受高速且稳定的 Docker 镜像拉取体验了！

***

**版权声明：**
- **本文作者：** Dirge
- **本文链接：** [https://www.iszy.cc/posts/nginx-docker-hub/](https://www.iszy.cc/posts/nginx-docker-hub/)
- **版权声明：** 本博客所有文章除特别声明外，均采用 [BY-NC-SA 许可协议](https://creativecommons.org/licenses/by-nc-sa/4.0/)。转载请注明出处！