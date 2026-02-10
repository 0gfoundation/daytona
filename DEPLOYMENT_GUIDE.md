# Daytona 服务器部署指南

## 📋 目录
1. [问题分析](#问题分析)
2. [解决方案](#解决方案)
3. [必要修改清单](#必要修改清单)
4. [部署步骤](#部署步骤)
5. [验证测试](#验证测试)
6. [常见问题](#常见问题)

---

## 问题分析

### 初始现象
访问 `http://服务器IP:3000/` 时，浏览器显示错误：
- **错误 1:** `Failed to fetch` - 无法加载数据
- **错误 2:** `502 Bad Gateway` - 访问 Dex 认证服务失败
- **错误 3:** `CORS policy` - 跨域错误
- **错误 4:** `Crypto.subtle is available only in secure contexts` - 浏览器安全策略限制

### 根本原因

#### 1. 配置问题（核心）
Daytona 的默认 `docker-compose.yaml` 是为**本地开发**设计的：
- 大量使用 `localhost` 配置
- 假设所有服务在同一台机器上通过内网访问
- 不适合远程服务器部署

**本地开发 vs 服务器部署对比：**

| 场景 | 访问方式 | 问题 |
|------|---------|------|
| 本地开发 | 浏览器和服务都在 localhost | ✅ 无问题 |
| 服务器部署 | 浏览器在本地，服务在远程 | ❌ localhost 无法访问 |

#### 2. 跨域问题
- 前端在 `3000` 端口
- Dex 在 `5556` 端口
- 浏览器认为跨端口是跨域（即使同一 IP）
- 云服务商可能对非标准端口有限制

#### 3. 浏览器安全策略
OIDC 认证使用 `Crypto.subtle` API，只能在安全上下文中使用：
- ✅ `https://` (HTTPS)
- ✅ `http://localhost`
- ❌ `http://IP地址` ← **你的情况**

---

## 解决方案

### 架构设计

**改造前（有问题）：**
```
浏览器
  ├─> http://IP:3000 → 前端 + API ✅
  └─> http://IP:5556 → Dex ❌ 被阻拦/不安全
```

**改造后（正常）：**
```
浏览器
  └─> https://IP (单一入口)
        ↓
      Nginx (反向代理 + SSL)
        ├─> /      → API (前端)
        ├─> /api/  → API (后端)
        └─> /dex/  → Dex (认证)

内部：
  API ←→ Dex (内网通信，HTTP)
```

### 核心改动
1. **添加 Nginx 反向代理** - 统一入口，解决跨域
2. **配置 HTTPS** - 满足浏览器安全要求
3. **修改配置** - 将 localhost 改为实际访问地址
4. **内外网分离** - API 内部用 HTTP，外部用 HTTPS

---

## 必要修改清单

### 1. docker-compose.yaml

#### ✅ 添加 Nginx 服务
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - 80:80      # HTTP (自动重定向到 HTTPS)
      - 443:443    # HTTPS
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
      - dex
    networks:
      - daytona-network
    restart: always
```

#### ✅ 修改 API 服务配置
```yaml
api:
  # 删除直接端口暴露（通过 Nginx 访问）
  # ports:
  #   - 3000:3000  # ❌ 删除这行

  environment:
    # 浏览器访问的公网地址（HTTPS）
    - PUBLIC_OIDC_DOMAIN=https://你的IP或域名/dex
    - DASHBOARD_URL=https://你的IP或域名/dashboard
    - DASHBOARD_BASE_API_URL=https://你的IP或域名

    # API 内部访问 Dex（内网 HTTP）
    - OIDC_ISSUER_BASE_URL=http://dex:5556/dex  # 保持内网地址

    # ⚠️ Proxy 配置（重要：使用 HTTPS，无端口号）
    - PROXY_DOMAIN=你的IP或域名.nip.io
    - PROXY_PROTOCOL=https
    - PROXY_TEMPLATE_URL=https://{{PORT}}-{{sandboxId}}.你的IP或域名.nip.io
    - OIDC_PUBLIC_DOMAIN=https://你的IP或域名/dex
```

#### ✅ 修改 Proxy 服务配置
```yaml
proxy:
  environment:
    # 使用 HTTPS 协议
    - PROXY_PROTOCOL=https
    # 支持子域名 cookie
    - COOKIE_DOMAIN=.你的IP或域名.nip.io
```

### 2. dex/config.yaml

```yaml
# 修改 issuer 为公网 HTTPS 地址
issuer: https://你的IP或域名/dex

web:
  http: 0.0.0.0:5556
  allowedOrigins: ['*']
  allowedHeaders: ['*']  # 允许所有请求头

staticClients:
  - id: daytona
    redirectURIs:
      # 保留原有的 localhost（开发用）
      - 'http://localhost:3000'
      - 'http://localhost:3000/api/oauth2-redirect.html'
      
      # 添加服务器的 HTTPS 回调地址
      - 'https://你的IP或域名'
      - 'https://你的IP或域名/api/oauth2-redirect.html'
      - 'https://你的IP或域名/callback'
      - 'http://你的IP或域名:4000/callback'
```

### 3. 新增文件

#### docker/nginx/nginx.conf
```nginx
events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # WebSocket 连接升级映射
    map $http_upgrade $connection_upgrade {
        default upgrade;
        '' close;
    }

    upstream api {
        server api:3000;
    }

    upstream dex {
        server dex:5556;
    }

    # ⚠️ 重要：Daytona Proxy upstream（用于 Web Terminal & VNC）
    upstream daytona_proxy {
        server proxy:4000;
    }

    # HTTP → HTTPS 重定向
    server {
        listen 80;
        server_name _;
        return 301 https://$host$request_uri;
    }

    # HTTPS 服务 - 主域名
    server {
        listen 443 ssl;
        server_name _;
        client_max_body_size 100M;

        # SSL 证书
        ssl_certificate /etc/nginx/ssl/daytona.crt;
        ssl_certificate_key /etc/nginx/ssl/daytona.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # Dex 认证服务
        location /dex/ {
            proxy_pass http://dex/dex/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_buffering off;
        }

        # API 路由
        location /api/ {
            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header Authorization $http_authorization;
            proxy_set_header Cookie $http_cookie;
            proxy_cache_bypass $http_upgrade;
            proxy_read_timeout 300s;
            proxy_connect_timeout 75s;

            # CORS 支持
            add_header 'Access-Control-Allow-Origin' $http_origin always;
            add_header 'Access-Control-Allow-Credentials' 'true' always;
            add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS, PATCH' always;
            add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin, X-Requested-With' always;

            # 处理 OPTIONS 预检请求
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' $http_origin always;
                add_header 'Access-Control-Allow-Credentials' 'true' always;
                add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS, PATCH' always;
                add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin, X-Requested-With' always;
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain; charset=utf-8';
                add_header 'Content-Length' 0;
                return 204;
            }
        }

        # 前端静态文件
        location / {
            proxy_pass http://api;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
            proxy_set_header Authorization $http_authorization;
            proxy_set_header Cookie $http_cookie;
        }
    }

    # ⚠️ 重要：Daytona Proxy - 处理子域名端口转发（Web Terminal & VNC）
    # 匹配格式: 22222-xxx.你的IP或域名.nip.io
    server {
        listen 443 ssl;
        server_name ~^[0-9]+-[a-z0-9]+\.你的IP或域名\.nip\.io$;

        # SSL 证书配置
        ssl_certificate /etc/nginx/ssl/daytona.crt;
        ssl_certificate_key /etc/nginx/ssl/daytona.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
            proxy_pass http://daytona_proxy;
            proxy_http_version 1.1;

            # WebSocket 支持
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            # 保留原始 Host 头（重要！proxy 需要它来路由）
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 超时设置
            proxy_read_timeout 300s;
            proxy_connect_timeout 75s;
            proxy_send_timeout 300s;

            # 禁用缓冲（对于实时终端很重要）
            proxy_buffering off;
        }
    }
}
```

---

## 部署步骤

### 前置条件
- 已安装 Docker 和 Docker Compose
- 已克隆 Daytona 仓库
- 有服务器公网 IP 或域名
- 云服务器安全组已开放 80 和 443 端口

### 步骤 1：生成 SSL 证书

```bash
# 创建 SSL 目录
mkdir -p docker/nginx/ssl

# 生成自签名证书（有效期 365 天）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout docker/nginx/ssl/daytona.key \
  -out docker/nginx/ssl/daytona.crt \
  -subj "/C=CN/ST=State/L=City/O=Daytona/OU=Dev/CN=你的IP或域名"
```

**注意：** 如果有域名，建议使用 Let's Encrypt 免费证书：
```bash
# 使用 certbot 申请证书
certbot certonly --standalone -d 你的域名
```

### 步骤 2：创建 Nginx 配置

```bash
# 创建配置目录
mkdir -p docker/nginx

# 将上面的 nginx.conf 内容保存到文件
vim docker/nginx/nginx.conf
```

### 步骤 3：修改配置文件

**3.1 修改 docker-compose.yaml**
- 添加 Nginx 服务
- 移除 API 的 `ports: - 3000:3000`
- 修改所有 `localhost` 为你的 IP/域名
- 修改 `PUBLIC_OIDC_DOMAIN`, `DASHBOARD_URL` 等为 HTTPS 地址

**3.2 修改 docker/dex/config.yaml**
- 修改 `issuer` 为 HTTPS 地址
- 添加 HTTPS 回调地址到 `redirectURIs`

### 步骤 4：启动服务

```bash
# 进入 docker 目录
cd docker

# 停止旧服务（如果有）
docker compose down

# 启动所有服务
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志（如果有问题）
docker compose logs -f api
docker compose logs -f dex
docker compose logs -f nginx
```

### 步骤 5：验证部署

```bash
# 1. 检查服务健康状态
docker compose ps

# 2. 测试 HTTPS 端点
curl -k https://你的IP/api/health
# 预期输出: {"status":"ok"}

# 3. 测试 Dex OIDC
curl -k https://你的IP/dex/.well-known/openid-configuration
# 预期输出: JSON 配置信息

# 4. 测试 HTTP 重定向
curl -I http://你的IP/
# 预期输出: HTTP/1.1 301 Moved Permanently
#          Location: https://...
```

### 步骤 6：浏览器访问

1. 打开浏览器，访问 `https://你的IP`
2. 会显示证书警告（自签名证书）
3. 点击 "高级" → "继续访问"
4. 应该看到 Daytona 登录页面

**默认登录凭据：**
- 邮箱: `dev@daytona.io`
- 密码: `password`

---

## 验证测试

### 服务状态检查

```bash
# 所有服务应该是 Up 和 healthy 状态
docker compose ps

# 关键服务状态
SERVICE   STATUS
api       Up XX seconds (healthy)
dex       Up XX seconds (healthy)
nginx     Up XX seconds
```

### 端点测试

```bash
# 1. API 健康检查
curl -k https://你的IP/api/health
# ✅ 返回: {"status":"ok"}

# 2. API 配置
curl -k https://你的IP/api/config
# ✅ 返回: JSON 配置（包含 OIDC issuer 等）

# 3. Dex OIDC 配置
curl -k https://你的IP/dex/.well-known/openid-configuration
# ✅ 返回: OIDC 发现文档

# 4. 前端页面
curl -k https://你的IP/
# ✅ 返回: HTML 页面
```

### 浏览器测试

1. **访问测试**
   - 地址: `https://你的IP`
   - 预期: 看到登录页面

2. **控制台检查**（F12 打开开发者工具）
   - Network 标签: 所有请求应该是 200 状态
   - Console 标签: 无红色错误信息

3. **登录测试**
   - 输入默认凭据
   - 预期: 成功进入 Dashboard

---

## 常见问题

### Q1: 浏览器显示 "连接不安全" 或证书错误

**原因：** 使用自签名证书，浏览器不信任

**解决：**
1. 点击 "高级"
2. 点击 "继续访问" 或 "接受风险并继续"
3. 或者：使用 Let's Encrypt 正式证书

### Q2: 服务启动后访问不了

**检查步骤：**
```bash
# 1. 检查容器状态
docker compose ps
# 如果有容器不是 Up 状态，查看日志：
docker compose logs <服务名>

# 2. 检查端口是否开放
curl https://你的IP
# 如果超时，检查防火墙和安全组

# 3. 检查 Nginx 日志
docker compose logs nginx
```

### Q3: API 一直重启

**可能原因：**
- 数据库连接失败
- Redis 连接失败
- Dex 连接失败（SSL 证书问题）

**检查日志：**
```bash
docker compose logs api | tail -100
```

**常见错误：**
- `Failed to fetch OpenID configuration: self-signed certificate`
  - 确保 `OIDC_ISSUER_BASE_URL` 使用内网地址 `http://dex:5556/dex`

### Q4: 登录后跳转失败

**检查配置：**
```bash
# 1. 确认 Dex redirectURIs 包含正确的地址
cat docker/dex/config.yaml | grep -A 10 redirectURIs

# 2. 确认 DASHBOARD_URL 正确
docker compose exec api printenv | grep DASHBOARD
```

### Q5: 如何使用正式的 SSL 证书？

**使用 Let's Encrypt（需要域名）：**

```bash
# 1. 安装 certbot
apt-get install certbot

# 2. 申请证书
certbot certonly --standalone -d 你的域名

# 3. 修改 nginx.conf
ssl_certificate /etc/letsencrypt/live/你的域名/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/你的域名/privkey.pem;

# 4. 在 docker-compose.yaml 中挂载证书
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:ro
```

### Q6: 如何从 HTTP 迁移到 HTTPS？

如果你之前配置了 HTTP 访问，迁移步骤：

```bash
# 1. 备份当前配置
cp docker/docker-compose.yaml docker/docker-compose.yaml.backup

# 2. 按照本文档进行 HTTPS 配置

# 3. 停止旧服务
docker compose down

# 4. 启动新服务
docker compose up -d

# 5. 清除浏览器缓存（重要！）
# 或使用无痕模式访问
```

### Q7: 创建 API Key 报 401 Unauthorized 错误

**现象：**
- Dashboard 可以正常访问和登录
- 但创建 API key 时报错：`POST https://IP/api/api-keys 401 (Unauthorized)`
- 浏览器控制台显示认证失败

**根本原因：**
Nginx 没有正确转发认证相关的 HTTP headers（`Authorization` 和 `Cookie`），导致后端 API 无法验证用户身份。

**解决方案：**

修改 `docker/nginx/nginx.conf`，在 `/api/` location 块中添加以下配置：

```nginx
location /api/ {
    proxy_pass http://api;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    # ✅ 添加这两行 - 转发认证信息
    proxy_set_header Authorization $http_authorization;
    proxy_set_header Cookie $http_cookie;

    proxy_cache_bypass $http_upgrade;
    proxy_read_timeout 300s;
    proxy_connect_timeout 75s;

    # ✅ 添加 CORS 支持
    add_header 'Access-Control-Allow-Origin' $http_origin always;
    add_header 'Access-Control-Allow-Credentials' 'true' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS, PATCH' always;
    add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin, X-Requested-With' always;

    # ✅ 处理 OPTIONS 预检请求
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin' $http_origin always;
        add_header 'Access-Control-Allow-Credentials' 'true' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS, PATCH' always;
        add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type, Accept, Origin, X-Requested-With' always;
        add_header 'Access-Control-Max-Age' 1728000;
        add_header 'Content-Type' 'text/plain; charset=utf-8';
        add_header 'Content-Length' 0;
        return 204;
    }
}
```

同样，在 `location /` 块中也添加认证 headers：

```nginx
location / {
    proxy_pass http://api;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;

    # ✅ 添加认证信息转发
    proxy_set_header Authorization $http_authorization;
    proxy_set_header Cookie $http_cookie;
}
```

**应用修改：**

```bash
# 重启 Nginx 服务
docker compose restart nginx

# 检查服务状态
docker compose ps nginx
```

**验证修复：**

1. 访问 `https://你的IP/dashboard`
2. 登录系统
3. 进入 API Keys 或 Settings 页面
4. 创建新的 API key
5. ✅ 应该能成功创建

**技术说明：**

- `proxy_set_header Authorization $http_authorization;` - 转发 Bearer token
- `proxy_set_header Cookie $http_cookie;` - 转发会话 cookie
- CORS headers - 允许跨域请求携带凭证
- OPTIONS 处理 - 处理浏览器的预检请求

这个问题通常在使用反向代理时出现，因为默认情况下 Nginx 不会自动转发所有 headers。

### Q8: SSH 连接立即断开，无法进入 Sandbox

**现象：**
```bash
ssh -p 2222 <TOKEN>@47.236.111.154
Connection to 47.236.111.154 closed.
```

SSH 连接建立后立即断开，无法进入 sandbox 终端。

**根本原因：**

**容器使用了旧的环境变量配置** ⚠️

**问题**: docker-compose.yaml 中已配置 `RUNNER_DOMAIN=runner`，但容器是用旧配置（`localhost`）创建的

**诊断日志**:
```
Failed to connect to runner: dial tcp [::1]:2220: connect: connection refused
```

ssh-gateway 尝试连接到 `[::1]:2220`（localhost），而不是 `runner:2220`。

**原因**:
- 初始部署时使用默认配置（`RUNNER_DOMAIN=localhost`）
- 后来修改了 docker-compose.yaml 为 `RUNNER_DOMAIN=runner`
- **但只执行了 `docker compose restart`，这不会应用环境变量的修改！**

**关键认知**:
- 环境变量在**容器创建时**注入，不会在重启时更新
- `docker compose restart` 只是停止和启动进程，不重新创建容器
- 修改环境变量后必须用 `--force-recreate` 重建容器

**正确解决方案**:
```bash
cd /root/daytona/docker

# ❌ 错误做法（不会应用环境变量修改）
docker compose restart runner

# ✅ 正确做法（重新创建容器）
docker compose up -d --force-recreate runner

# 重启 ssh-gateway 让它重新读取配置
docker compose restart ssh-gateway

# 验证环境变量已生效
docker compose exec runner env | grep RUNNER_DOMAIN
# 应该输出: RUNNER_DOMAIN=runner
```

**附加检查**（非根本原因，但可能影响连接）:

如果上述修复后仍然失败，检查：

1. **Sandbox 容器状态**
   ```bash
   # 检查 sandbox 是否真的在运行
   docker compose exec runner docker ps -a | grep <sandbox-id>

   # 如果显示 "Exited"，启动它
   docker compose exec runner docker start <sandbox-id>
   ```

2. **网络连通性**
   ```bash
   # 测试 ssh-gateway 到 runner 的连接
   docker compose exec ssh-gateway nc -zv runner 2220
   # 应该输出: runner (IP:2220) open
   ```

**完整解决流程**:

```bash
# 步骤 1: 验证配置文件
grep "RUNNER_DOMAIN" docker/docker-compose.yaml
# 应该看到: RUNNER_DOMAIN=runner

# 步骤 2: 重新创建 runner 容器（关键！）
docker compose up -d --force-recreate runner

# 步骤 3: 等待 runner 启动并注册
sleep 10

# 步骤 4: 验证数据库配置（应该自动更新）
docker compose exec db psql -U user -d daytona -c \
  "SELECT domain FROM runner;"
# 应该输出: runner

# 步骤 5: 确保 sandbox 运行中
SANDBOX_ID=$(docker compose exec db psql -U user -d daytona -t -c \
  "SELECT id FROM sandbox LIMIT 1;" | xargs)
docker compose exec runner docker start $SANDBOX_ID

# 步骤 6: 重启 ssh-gateway
docker compose restart ssh-gateway

# 步骤 7: 测试 SSH 连接
ssh -p 2222 <TOKEN>@47.236.111.154
```

**验证清单**:

| 检查项 | 命令 | 期望结果 |
|--------|------|---------|
| 配置文件 | `grep RUNNER_DOMAIN docker-compose.yaml` | `runner` |
| 容器环境变量 | `docker compose exec runner env \| grep RUNNER_DOMAIN` | `runner` |
| 数据库配置 | `docker compose exec db psql -U user -d daytona -c "SELECT domain FROM runner;"` | `runner` |
| 网络连通 | `docker compose exec ssh-gateway nc -zv runner 2220` | `open` |
| Sandbox 状态 | `docker compose exec runner docker ps` | `Up` |

**常见错误**:

❌ **错误 1**: 只修改数据库，不修改配置文件
- Runner 服务会用环境变量覆盖数据库配置

❌ **错误 2**: 修改配置文件后只执行 `restart`
- 必须用 `--force-recreate` 重建容器

❌ **错误 3**: 假设数据库状态就是实际状态
- Sandbox 可能在数据库中显示 `started` 但容器实际已退出

**技术说明**:

配置流向：
```
docker-compose.yaml (源头)
    ↓
容器环境变量 (创建时注入)
    ↓
Runner 服务 (读取环境变量)
    ↓
数据库 (Runner 自动注册)
    ↓
SSH-Gateway (从数据库读取)
```

不要手动修改数据库，应该从源头（docker-compose.yaml）修复，让配置自然流动。

---

## 架构对比总结

### 本地开发（原始配置）
```
localhost:3000 (前端 + API)
localhost:5556 (Dex)
```
✅ 适合：本地开发
❌ 不适合：服务器部署

### 服务器部署（修改后）
```
https://IP (统一入口)
  ├─ Nginx (反向代理 + SSL)
  ├─ API (内网)
  └─ Dex (内网)
```
✅ 适合：生产环境
✅ 解决：跨域、安全、端口管理

---

## 端口说明

### 对外开放端口
- **80** - HTTP (自动重定向到 HTTPS)
- **443** - HTTPS (统一入口)

### 内网端口（不需要开放）
- 3000 - API
- 5556 - Dex
- 6379 - Redis
- 5432 - PostgreSQL

### 可选管理端口
- 5050 - PgAdmin (数据库管理)
- 9001 - MinIO (对象存储管理)
- 5100 - Registry UI (镜像仓库管理)

---

## 安全建议

1. **生产环境使用正式证书**
   - 推荐 Let's Encrypt 免费证书
   - 或购买商业证书

2. **配置防火墙规则**
   ```bash
   # 只开放必要端口
   - 入站: 80, 443
   - 出站: 全部
   ```

3. **定期更新证书**
   ```bash
   # Let's Encrypt 证书每 90 天过期
   # 建议设置自动续期
   certbot renew
   ```

4. **修改默认密码**
   - 登录后立即修改 admin 用户密码
   - 修改 Dex 配置中的默认用户

5. **配置 Nginx 访问日志**
   ```nginx
   access_log /var/log/nginx/access.log;
   error_log /var/log/nginx/error.log;
   ```

---

## 总结

### 核心修改
1. ✅ 添加 Nginx 反向代理
2. ✅ 配置 HTTPS（自签名或正式证书）
3. ✅ 修改所有 `localhost` 为实际地址
4. ✅ 区分内网和外网访问地址

### 关键配置点
- `OIDC_ISSUER_BASE_URL`: 内网 HTTP 地址
- `PUBLIC_OIDC_DOMAIN`: 外网 HTTPS 地址
- Dex `issuer`: 外网 HTTPS 地址
- Dex `redirectURIs`: 包含所有回调地址

### 部署原则
- **统一入口**: 所有流量通过 Nginx
- **内外分离**: 内网用 HTTP，外网用 HTTPS
- **最小暴露**: 只开放 80/443 端口
- **标准架构**: 反向代理是生产环境标准做法

---

## 参考资源

- [Daytona 官方文档](https://www.daytona.io/docs)
- [Nginx 反向代理配置](https://nginx.org/en/docs/)
- [Let's Encrypt 证书](https://letsencrypt.org/)
- [Docker Compose 文档](https://docs.docker.com/compose/)

---

### Q9: Web Terminal 和 VNC 外部访问 502 错误（阿里云端口限制）

**现象：**
- Dashboard 内点击 ">_" Web Terminal 按钮后跳转到类似 `http://22222-xxx.47.236.111.154.nip.io:4000`
- 浏览器返回 `HTTP ERROR 502`
- VNC Computer Use 同样返回 502
- **但从服务器内部访问 `localhost:4000` 是正常的**

**根本原因：**

**阿里云默认屏蔽非标准端口的外部访问** ⚠️

即使在安全组中开放了 4000 端口，阿里云的 DDoS 防护或防火墙可能仍然阻止外部流量访问非标准端口（80/443 以外）。

**诊断流程：**
```bash
# 1. 从服务器内部测试（正常）
curl http://localhost:4000
# 返回: 404 (正常，因为 proxy 需要正确的 Host header)

# 2. 从外部访问（失败）
curl http://47.236.111.154:4000
# 返回: 502 Bad Gateway 或超时

# 3. 检查 Daytona proxy 日志（有 404 错误表示服务正常）
docker compose logs proxy | grep "not found"
# 看到: level=error msg="API ERROR" URI=/ error="not found: not found" method=GET status=404
# 说明 proxy 服务本身正常，只是外部无法访问
```

**解决方案：使用 Nginx 反向代理绕过端口限制**

通过让所有流量都走 443 端口，用子域名区分不同的服务：

**步骤 1: 修改 docker-compose.yaml 环境变量**

```yaml
api:
  environment:
    # 修改 PROXY 配置使用 HTTPS 和无端口号的域名
    - PROXY_DOMAIN=47.236.111.154.nip.io
    - PROXY_PROTOCOL=https
    - PROXY_TEMPLATE_URL=https://{{PORT}}-{{sandboxId}}.47.236.111.154.nip.io

proxy:
  environment:
    # 修改 PROXY_PROTOCOL 为 HTTPS
    - PROXY_PROTOCOL=https
    # 添加 COOKIE_DOMAIN 支持子域名
    - COOKIE_DOMAIN=.47.236.111.154.nip.io
```

**步骤 2: 修改 nginx.conf 添加子域名代理**

在 `docker/nginx/nginx.conf` 中添加：

```nginx
http {
    # ... 现有配置 ...

    # 添加 daytona_proxy upstream
    upstream daytona_proxy {
        server proxy:4000;
    }

    # 在现有 server 块之后添加新的 server 块
    # Daytona Proxy - 处理子域名端口转发 (Terminal & VNC)
    # 匹配格式: 22222-xxx.47.236.111.154.nip.io
    server {
        listen 443 ssl;
        server_name ~^[0-9]+-[a-z0-9]+\.47\.236\.111\.154\.nip\.io$;

        # SSL 证书配置
        ssl_certificate /etc/nginx/ssl/daytona.crt;
        ssl_certificate_key /etc/nginx/ssl/daytona.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        location / {
            proxy_pass http://daytona_proxy;
            proxy_http_version 1.1;

            # WebSocket 支持
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            # 保留原始 Host 头（重要！proxy 需要它来路由）
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # 超时设置
            proxy_read_timeout 300s;
            proxy_connect_timeout 75s;
            proxy_send_timeout 300s;

            # 禁用缓冲（对于实时终端很重要）
            proxy_buffering off;
        }
    }
}
```

**步骤 3: 应用配置**

```bash
cd /root/daytona/docker

# 1. 重新创建 API 和 Proxy 容器以应用环境变量
docker compose up -d --force-recreate api proxy

# ⚠️ 重要：重启 nginx 刷新 DNS 缓存
# 因为 API 容器被重建后 IP 地址会变化
docker compose restart nginx

# 2. 验证配置
docker compose ps
docker compose logs api | grep PROXY
docker compose logs nginx | tail -20
```

**步骤 4: 验证修复**

1. 打开 Dashboard: `https://47.236.111.154`
2. 进入 Sandbox，点击 ">_" Web Terminal 按钮
3. 应该跳转到 `https://22222-xxx.47.236.111.154.nip.io` (无端口号)
4. ✅ 终端正常打开
5. VNC Computer Use 也应该正常工作

**关键配置说明：**

| 配置项 | 修改前 | 修改后 | 说明 |
|--------|--------|--------|------|
| PROXY_DOMAIN | `47.236.111.154.nip.io:4000` | `47.236.111.154.nip.io` | 移除端口号 |
| PROXY_PROTOCOL | `http` | `https` | 使用 HTTPS |
| PROXY_TEMPLATE_URL | `http://{{PORT}}-{{sandboxId}}.47.236.111.154.nip.io:4000` | `https://{{PORT}}-{{sandboxId}}.47.236.111.154.nip.io` | HTTPS + 无端口 |
| nginx server_name | - | `~^[0-9]+-[a-z0-9]+\.47\.236\.111\.154\.nip\.io$` | 正则匹配子域名 |

**工作原理：**

```
浏览器访问 https://22222-xxx.47.236.111.154.nip.io
    ↓
Nginx 监听 443 端口
    ↓
匹配 server_name 正则规则
    ↓
proxy_pass 到内部 proxy:4000
    ↓
Daytona Proxy 根据 Host header 路由到对应 sandbox
    ↓
终端/VNC 服务
```

**常见错误：**

❌ **错误 1**: 修改配置后只执行 `docker compose restart`
- 环境变量修改必须用 `--force-recreate`

❌ **错误 2**: 忘记重启 nginx
- API 容器重建后 IP 变化，nginx 缓存旧 IP 会导致 502

❌ **错误 3**: server_name 正则写错
- 必须完整匹配子域名格式，包括域名后缀

**云服务商端口限制对比：**

| 云服务商 | 标准端口 (80/443) | 非标准端口 (4000) | 解决方案 |
|----------|-------------------|-------------------|----------|
| 阿里云 | ✅ 开放 | ❌ 可能被 DDoS 防护屏蔽 | Nginx 反向代理 |
| AWS | ✅ 开放 | ✅ 安全组配置即可 | 可选 |
| 腾讯云 | ✅ 开放 | ⚠️ 部分地区限制 | 建议使用反向代理 |

**技术说明：**

此方案不仅解决了端口限制问题，还带来其他优势：
1. ✅ 统一使用 HTTPS，更安全
2. ✅ 只需开放 80/443 端口，简化防火墙配置
3. ✅ 子域名自动获得 SSL 证书（使用通配符证书）
4. ✅ 符合生产环境最佳实践

---

**文档版本:** 1.3
**更新日期:** 2026-02-10
**适用版本:** Daytona v0.139.0
**更新内容:**
- v1.1: 添加 API Key 创建 401 错误解决方案 (Q7)
- v1.2: 添加 SSH 连接问题排查和解决方案 (Q8)
- v1.3: 添加阿里云端口限制问题和 Nginx 反向代理解决方案 (Q9)
