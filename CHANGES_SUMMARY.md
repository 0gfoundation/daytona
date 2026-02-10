# Daytona 服务器部署 - 配置修改总结

## 📊 修改统计

```
修改文件数: 2
新增文件数: 3
总计修改: 28 行新增，11 行删除
```

---

## 📝 修改清单

### 1. docker/docker-compose.yaml

#### ✅ 新增 Nginx 服务 (16行)

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - 80:80
      - 443:443
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

**说明：** 添加 Nginx 作为反向代理和 SSL 终止点

---

#### ✅ 修改 API 服务 (7处修改)

**1. 移除直接端口暴露**
```diff
api:
- ports:
-   - 3000:3000
```
**原因：** 通过 Nginx 统一入口，API 不再直接对外

**2. 修改 OIDC 公网域名**
```diff
- PUBLIC_OIDC_DOMAIN=http://localhost:5556/dex
+ PUBLIC_OIDC_DOMAIN=https://47.236.111.154/dex
```
**原因：** 浏览器需要通过 HTTPS 访问 Dex

**3. 修改 Dashboard URL**
```diff
- DASHBOARD_URL=http://localhost:3000/dashboard
+ DASHBOARD_URL=https://47.236.111.154/dashboard
```
**原因：** 前端需要正确的 HTTPS 地址

**4. 修改 Dashboard API URL**
```diff
- DASHBOARD_BASE_API_URL=http://localhost:3000
+ DASHBOARD_BASE_API_URL=https://47.236.111.154
```
**原因：** API 基础 URL 需要指向 HTTPS

**5. 修改 Proxy 域名**
```diff
- PROXY_DOMAIN=proxy.localhost:4000
+ PROXY_DOMAIN=47.236.111.154:4000
```
**原因：** Proxy 服务需要使用实际 IP

**6. 修改 Proxy 模板 URL**
```diff
- PROXY_TEMPLATE_URL=http://{{PORT}}-{{sandboxId}}.proxy.localhost:4000
+ PROXY_TEMPLATE_URL=http://{{PORT}}-{{sandboxId}}.47.236.111.154.nip.io:4000
```
**原因：** 沙盒访问需要使用通配符 DNS (nip.io)

**7. 保持内网访问不变**
```yaml
OIDC_ISSUER_BASE_URL=http://dex:5556/dex  # 不变，内网访问
```
**重要：** API 访问 Dex 使用内网地址，避免 SSL 证书问题

---

#### ✅ 修改 Proxy 服务 (1处)

```diff
proxy:
  environment:
-   OIDC_PUBLIC_DOMAIN=http://localhost:5556/dex
+   OIDC_PUBLIC_DOMAIN=https://47.236.111.154/dex
```
**原因：** Proxy 服务的公网 OIDC 地址也需要 HTTPS

---

### 2. docker/dex/config.yaml

#### ✅ 修改 Dex Issuer (1处)

```diff
- issuer: http://localhost:5556/dex
+ issuer: https://47.236.111.154/dex
```
**原因：** OIDC issuer 必须与浏览器访问地址一致

#### ✅ 修改 CORS 配置 (1处)

```diff
web:
  http: 0.0.0.0:5556
  allowedOrigins: ['*']
- allowedHeaders: ['x-requested-with']
+ allowedHeaders: ['*']
```
**原因：** 允许所有请求头，避免 CORS 问题

#### ✅ 添加 HTTPS 回调地址 (4处)

```diff
staticClients:
  - id: daytona
    redirectURIs:
      - 'http://localhost:3000'
      - 'http://localhost:3000/api/oauth2-redirect.html'
      - 'http://localhost:3009/callback'
      - 'http://proxy.localhost:4000/callback'
+     - 'https://47.236.111.154'
+     - 'https://47.236.111.154/api/oauth2-redirect.html'
+     - 'https://47.236.111.154/callback'
+     - 'http://47.236.111.154:4000/callback'
```
**原因：** OIDC 回调需要包含实际的 HTTPS 地址

---

### 3. 新增文件

#### ✅ docker/nginx/nginx.conf (72行)

**核心配置：**
```nginx
# HTTP → HTTPS 重定向
server {
    listen 80;
    return 301 https://$host$request_uri;
}

# HTTPS 服务
server {
    listen 443 ssl;
    
    # SSL 证书
    ssl_certificate /etc/nginx/ssl/daytona.crt;
    ssl_certificate_key /etc/nginx/ssl/daytona.key;
    
    # 路由规则
    location /dex/  { proxy_pass http://dex/dex/; }
    location /api/  { proxy_pass http://api; }
    location /      { proxy_pass http://api; }
}
```

**作用：**
1. 统一入口 (80/443 端口)
2. SSL 终止
3. 反向代理 (API + Dex)
4. 自动 HTTP → HTTPS 重定向

---

#### ✅ docker/nginx/ssl/daytona.crt

**生成命令：**
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout docker/nginx/ssl/daytona.key \
  -out docker/nginx/ssl/daytona.crt \
  -subj "/C=CN/ST=State/L=City/O=Daytona/OU=Dev/CN=47.236.111.154"
```

**说明：** 自签名 SSL 证书（365天有效期）

---

#### ✅ docker/nginx/ssl/daytona.key

**说明：** SSL 私钥文件

---

## 🔑 关键配置对比

### 内网 vs 外网访问

| 配置项 | 内网访问 (API → Dex) | 外网访问 (浏览器 → Dex) |
|--------|---------------------|------------------------|
| 地址 | `http://dex:5556/dex` | `https://47.236.111.154/dex` |
| 协议 | HTTP | HTTPS |
| 用途 | API 内部通信 | 浏览器访问 |
| 配置 | `OIDC_ISSUER_BASE_URL` | `PUBLIC_OIDC_DOMAIN` |

**重要：** 这是解决 SSL 证书问题的关键！

---

### localhost vs 实际地址对比

| 配置项 | 原始值 (localhost) | 修改后 (服务器) |
|--------|-------------------|----------------|
| Dex Issuer | `http://localhost:5556/dex` | `https://47.236.111.154/dex` |
| Dashboard URL | `http://localhost:3000/dashboard` | `https://47.236.111.154/dashboard` |
| Dashboard API | `http://localhost:3000` | `https://47.236.111.154` |
| Public OIDC | `http://localhost:5556/dex` | `https://47.236.111.154/dex` |
| Proxy Domain | `proxy.localhost:4000` | `47.236.111.154:4000` |

---

## 📦 端口变化

### 修改前
```
对外暴露:
  - 3000 (API + 前端)
  - 5556 (Dex) ← 浏览器无法访问

问题: 跨域、502 错误
```

### 修改后
```
对外暴露:
  - 80 (HTTP，重定向到 443)
  - 443 (HTTPS，统一入口)

内网端口:
  - 3000 (API)
  - 5556 (Dex)

优势: 
  ✅ 统一入口
  ✅ 无跨域问题
  ✅ 符合 HTTPS 要求
  ✅ 标准生产架构
```

---

## 🎯 修改要点总结

### 必须修改的地方

1. **所有 `localhost` → 实际 IP/域名**
   - `PUBLIC_OIDC_DOMAIN`
   - `DASHBOARD_URL`
   - `DASHBOARD_BASE_API_URL`
   - `PROXY_DOMAIN`
   - Dex `issuer`
   - Dex `redirectURIs`

2. **添加 Nginx 反向代理**
   - 新增 Nginx 服务
   - 配置 SSL
   - 配置路由规则

3. **内外网分离**
   - `OIDC_ISSUER_BASE_URL`: 保持内网 `http://dex:5556/dex`
   - `PUBLIC_OIDC_DOMAIN`: 使用外网 `https://IP/dex`

### 不需要修改的地方

1. **内网服务配置**
   - 数据库连接 (`DB_HOST=db`)
   - Redis 连接 (`REDIS_HOST=redis`)
   - MinIO 连接 (`S3_ENDPOINT=http://minio:9000`)

2. **API 内部访问 Dex**
   - `OIDC_ISSUER_BASE_URL=http://dex:5556/dex` ← 保持不变

3. **其他内网服务**
   - Runner, Registry, SMTP 等配置保持不变

---

## 📋 快速检查清单

部署前检查：
```
✅ 已生成 SSL 证书
✅ 已创建 nginx.conf
✅ 已修改 docker-compose.yaml (所有 localhost)
✅ 已修改 dex/config.yaml (issuer + redirectURIs)
✅ 安全组已开放 80 和 443 端口
```

部署后验证：
```
✅ docker compose ps (所有服务 Up)
✅ curl -k https://IP/api/health (返回 ok)
✅ curl -k https://IP/dex/.well-known/openid-configuration (返回 JSON)
✅ 浏览器访问 https://IP (看到登录页面)
✅ 能够成功登录
```

---

## 🔍 Git Diff 总结

```bash
# 查看修改统计
git diff --stat
# docker/dex/config.yaml     |  8 ++++++--
# docker/docker-compose.yaml | 31 ++++++++++++++++++++++---------
# 2 files changed, 28 insertions(+), 11 deletions(-)

# 查看详细修改
git diff docker/docker-compose.yaml
git diff docker/dex/config.yaml

# 未跟踪的新文件
git status
# docker/nginx/nginx.conf
# docker/nginx/ssl/daytona.crt
# docker/nginx/ssl/daytona.key
```

---

## 💡 经验总结

1. **开源项目默认配置往往是为本地开发设计的**
   - 需要根据部署环境调整
   - 关注所有 localhost 配置

2. **反向代理是生产环境标准做法**
   - 统一入口管理
   - 简化安全配置
   - 易于扩展和维护

3. **内外网分离很重要**
   - 内网用 HTTP（快速、无证书问题）
   - 外网用 HTTPS（安全、符合标准）

4. **浏览器安全策略越来越严格**
   - 加密 API 需要 HTTPS
   - 跨域限制更加严格
   - 证书验证更加严格

---

**文档版本:** 1.0  
**最后更新:** 2026-02-07  
**对应版本:** Daytona v0.139.0
