# Daytona 部署断点记录

**记录时间**: 2026-02-09
**服务器 IP**: 47.236.111.154
**部署状态**: 已完成基础配置，待测试 SSH 和 WebSocket

---

## ✅ 已完成的工作

### 1. HTTPS 反向代理配置
- ✅ Nginx 配置完成 (`docker/nginx/nginx.conf`)
- ✅ SSL 自签名证书已生成 (`docker/nginx/ssl/`)
- ✅ HTTP 自动重定向到 HTTPS
- ✅ 端口 80/443 已开放

### 2. 认证配置 (Dex)
- ✅ Dex issuer 配置为 `https://47.236.111.154/dex`
- ✅ 回调地址已添加到 redirectURIs
- ✅ 默认账户: `dev@daytona.io` / 密码: `password`
- ⚠️ **安全提醒**: 建议修改默认密码

### 3. API 服务配置
- ✅ 所有 localhost 已替换为实际 IP
- ✅ PUBLIC_OIDC_DOMAIN 配置为 HTTPS 地址
- ✅ DASHBOARD_URL 配置正确
- ✅ PostHog 分析已禁用（完全本地化）

### 4. API Key 创建问题修复
- ✅ 问题: 创建 API key 报 401 错误
- ✅ 原因: Nginx 未转发 Authorization/Cookie headers
- ✅ 解决: 添加 header 转发和 CORS 配置
- ✅ 状态: **已解决，API key 创建成功**

### 5. Region 配置
- ✅ 默认 region "us" 已设置
- ✅ Sandbox 创建成功
- ✅ Sandbox ID: `d4689404-349d-46bd-99e7-1f4517eed9dd`
- ✅ Sandbox 名称: `my-sandbox`

### 6. SSH 连接问题修复
- ✅ 问题: SSH 连接立即断开
- ✅ 原因: Runner domain 配置为 localhost
- ✅ 解决方案:
  - 修改 `docker-compose.yaml`: `RUNNER_DOMAIN=runner`
  - 更新数据库: `UPDATE runner SET domain = 'runner'`
  - 重启服务
- ⏳ 状态: **待测试**

### 7. WebSocket 连接问题修复
- ✅ 问题: WebSocket 连接失败
- ✅ 解决方案:
  - 添加 connection upgrade mapping
  - 优化 Connection header 处理
  - 重启 nginx
- ⏳ 状态: **待测试**

---

## 🔧 当前配置

### 服务访问地址
| 服务 | 地址 | 状态 |
|------|------|------|
| Dashboard | `https://47.236.111.154/dashboard` | ✅ 可访问 |
| API | `https://47.236.111.154/api` | ✅ 可访问 |
| Dex 认证 | `https://47.236.111.154/dex` | ✅ 可访问 |
| SSH Gateway | `47.236.111.154:2222` | ⏳ 待测试 |
| Proxy | `http://47.236.111.154:4000` | ✅ 运行中 |

### 管理界面（可选）
| 服务 | 地址 | 端口 |
|------|------|------|
| PgAdmin | `http://47.236.111.154:5050` | 5050 |
| MinIO | `http://47.236.111.154:9001` | 9001 |
| Registry UI | `http://47.236.111.154:5100` | 5100 |
| MailDev | `http://47.236.111.154:1080` | 1080 |

### 环境变量配置

```bash
# API 访问配置
export DAYTONA_API_KEY="你的API密钥"
export DAYTONA_API_URL="https://47.236.111.154/api"

# 测试 API
curl -k -X GET "$DAYTONA_API_URL/config" \
  -H "Authorization: Bearer $DAYTONA_API_KEY"
```

### 默认登录凭据

```
Dashboard 登录:
- 邮箱: dev@daytona.io
- 密码: password

PgAdmin:
- 邮箱: dev@daytona.io
- 密码: pgadmin

MinIO:
- 用户名: minioadmin
- 密码: minioadmin
```

---

## ⏳ 待测试项目

### 1. SSH 连接测试

```bash
# 获取 SSH token（在 Dashboard 中）
# 然后测试连接
ssh -p 2222 <TOKEN>@47.236.111.154

# 预期结果: 成功连接到 sandbox
```

**最新 token**: `i73YA5uQjkjQgvnD9FDU18jQvqV85LlQ`
**对应 sandbox**: `my-sandbox` (d4689404-349d-46bd-99e7-1f4517eed9dd)

### 2. 在线终端测试

1. 访问 `https://47.236.111.154/dashboard`
2. 进入 sandbox: `my-sandbox`
3. 点击在线终端
4. **清除浏览器缓存后测试**

**预期结果**: 终端能正常打开并响应命令

### 3. WebSocket 连接验证

打开浏览器开发者工具（F12）：
- Network 标签 → WS 过滤器
- 应该看到 `wss://47.236.111.154/api/socket.io/...` 连接成功

---

## 🐛 已知问题和解决方案

### 问题 1: 无法访问 Dashboard (502/CORS 错误)
**已解决** ✅
- 原因: localhost 配置 + 缺少 HTTPS + 跨域问题
- 解决: Nginx 反向代理 + HTTPS + CORS 配置

### 问题 2: 创建 API Key 401 错误
**已解决** ✅
- 原因: Nginx 未转发认证 headers
- 解决: 添加 Authorization 和 Cookie header 转发

### 问题 3: SSH 连接立即断开
**已修复，待测试** ⏳
- 原因: Runner domain = localhost，ssh-gateway 无法连接
- 解决:
  ```bash
  # 数据库修复
  docker compose exec db psql -U user -d daytona -c \
    "UPDATE runner SET domain = 'runner' WHERE id = 'b0b11d31-c78c-4229-aa7c-f70321baa94d';"

  # 重启服务
  docker compose restart ssh-gateway
  ```

### 问题 4: WebSocket 连接失败
**已修复，待测试** ⏳
- 原因: Nginx WebSocket 配置不完善
- 解决: 添加 connection upgrade mapping

---

## 📋 重要文件清单

### 配置文件
- `docker/docker-compose.yaml` - Docker Compose 主配置 ✅
- `docker/dex/config.yaml` - Dex 认证配置 ✅
- `docker/nginx/nginx.conf` - Nginx 反向代理配置 ✅
- `docker/nginx/ssl/daytona.crt` - SSL 证书 ✅
- `docker/nginx/ssl/daytona.key` - SSL 私钥 ✅

### 文档文件
- `DEPLOYMENT_GUIDE.md` - 完整部署指南 ✅
- `CHANGES_SUMMARY.md` - 配置变更记录 ✅
- `CHECKPOINT.md` - 本断点文件 ✅

---

## 🔍 排查命令

### 检查服务状态
```bash
# 查看所有服务
docker compose ps

# 查看特定服务日志
docker compose logs -f api
docker compose logs -f dex
docker compose logs -f nginx
docker compose logs -f ssh-gateway
docker compose logs -f runner

# 检查服务健康状态
docker compose ps | grep -E "healthy|unhealthy"
```

### 测试 API 端点
```bash
# 健康检查
curl -k https://47.236.111.154/api/health

# 获取配置
curl -k https://47.236.111.154/api/config

# 列出 sandbox
curl -k -X GET "https://47.236.111.154/api/sandbox" \
  -H "Authorization: Bearer $DAYTONA_API_KEY"

# 查看特定 sandbox
curl -k -X GET "https://47.236.111.154/api/sandbox/my-sandbox" \
  -H "Authorization: Bearer $DAYTONA_API_KEY"
```

### 数据库查询
```bash
# 查看 runner 配置
docker compose exec db psql -U user -d daytona -c \
  "SELECT id, name, domain FROM runner;"

# 查看 region 配置
docker compose exec db psql -U user -d daytona -c \
  "SELECT id, name FROM region;"

# 查看 sandbox 列表
docker compose exec db psql -U user -d daytona -c \
  "SELECT id, name, state FROM sandbox LIMIT 5;"
```

### 网络调试
```bash
# 测试端口连通性
nc -zv 47.236.111.154 443
nc -zv 47.236.111.154 2222

# 查看容器网络
docker compose exec ssh-gateway ping -c 2 runner
docker compose exec ssh-gateway nc -zv runner 2220

# 检查端口监听
docker compose exec runner netstat -tlnp | grep 2220
```

---

## 🚀 重启服务命令

### 重启所有服务
```bash
cd /root/daytona/docker
docker compose restart
```

### 重启特定服务
```bash
# 重启 API
docker compose restart api

# 重启认证服务
docker compose restart dex

# 重启反向代理
docker compose restart nginx

# 重启 SSH 相关服务
docker compose restart ssh-gateway runner
```

### 完全重新部署
```bash
# 停止所有服务
docker compose down

# 启动所有服务
docker compose up -d

# 查看启动日志
docker compose logs -f
```

---

## 📊 当前 Sandbox 信息

### Sandbox: my-sandbox
```json
{
  "id": "d4689404-349d-46bd-99e7-1f4517eed9dd",
  "organizationId": "1bbc5c77-25cb-42af-a861-d1843a910e29",
  "name": "my-sandbox",
  "target": "us",
  "snapshot": "daytonaio/sandbox:0.5.0-slim",
  "state": "creating/running",
  "cpu": 1,
  "memory": 1,
  "disk": 3,
  "runnerId": "b0b11d31-c78c-4229-aa7c-f70321baa94d"
}
```

### 访问方式
```bash
# SSH 访问
ssh -p 2222 <TOKEN>@47.236.111.154

# API 访问
curl -k https://47.236.111.154/api/sandbox/my-sandbox \
  -H "Authorization: Bearer $DAYTONA_API_KEY"

# Web 访问（如果 sandbox 运行了 web 服务）
http://3000-d4689404-349d-46bd-99e7-1f4517eed9dd.47.236.111.154.nip.io:4000
```

---

## 🔐 安全建议

### 立即执行
1. ⚠️ **修改默认密码**
   ```bash
   # 生成新密码 hash
   echo "your-new-password" | htpasswd -BinC 10 admin | cut -d: -f2

   # 更新 docker/dex/config.yaml
   # 重启 dex: docker compose restart dex
   ```

2. ⚠️ **配置防火墙规则**
   - 只允许必要的 IP 访问
   - 或使用 VPN

### 建议执行
3. 📜 **使用正式 SSL 证书**（如果有域名）
   ```bash
   certbot certonly --standalone -d 你的域名
   ```

4. 🔒 **启用 API 速率限制**
   - 在 nginx 中配置 rate limiting

5. 📝 **配置日志监控**
   - 定期检查 nginx、api、dex 日志
   - 设置异常告警

---

## 📞 问题排查流程

### 如果 SSH 连接失败
1. 检查 ssh-gateway 日志: `docker compose logs --tail=50 ssh-gateway`
2. 检查 runner 状态: `docker compose ps runner`
3. 验证 runner domain: `docker compose exec db psql -U user -d daytona -c "SELECT domain FROM runner;"`
4. 测试网络连通性: `docker compose exec ssh-gateway nc -zv runner 2220`

### 如果 WebSocket 连接失败
1. 清除浏览器缓存（强制刷新）
2. 检查 nginx 日志: `docker compose logs --tail=50 nginx`
3. 检查 api 日志: `docker compose logs --tail=50 api | grep -i websocket`
4. 使用无痕模式测试

### 如果 API 请求失败
1. 检查 API key 是否正确
2. 验证 API 服务状态: `docker compose ps api`
3. 查看 API 日志: `docker compose logs --tail=100 api`
4. 测试基础端点: `curl -k https://47.236.111.154/api/health`

---

## 📝 下一步计划

### 测试验证
- [ ] 测试 SSH 连接是否成功
- [ ] 测试在线终端是否可用
- [ ] 验证 WebSocket 连接正常
- [ ] 测试 sandbox 内运行 web 服务并访问

### 功能完善
- [ ] 修改默认密码
- [ ] 创建额外的用户账户
- [ ] 配置备份策略
- [ ] 设置监控告警

### 文档完善
- [ ] 更新 DEPLOYMENT_GUIDE.md（如有新问题）
- [ ] 记录最佳实践
- [ ] 整理常见问题 FAQ

---

## 💡 有用的技巧

### 快速查看服务端口
```bash
docker compose ps --format json | jq -r '.[] | "\(.Service): \(.Publishers)"'
```

### 查看资源使用
```bash
docker stats --no-stream
```

### 备份重要数据
```bash
# 备份数据库
docker compose exec db pg_dump -U user daytona > backup.sql

# 备份配置文件
tar -czf daytona-config-backup.tar.gz docker/
```

### 清理 Docker 资源
```bash
# 清理未使用的镜像
docker image prune -a

# 清理未使用的卷
docker volume prune

# 查看磁盘使用
docker system df
```

---

**记录完成时间**: 2026-02-09 16:06 CST
**下次连接提醒**: 测试 SSH 和 WebSocket 功能

---

## 🎯 快速恢复命令

```bash
# 1. 进入项目目录
cd /root/daytona/docker

# 2. 检查服务状态
docker compose ps

# 3. 如果需要重启
docker compose restart

# 4. 设置环境变量
export DAYTONA_API_KEY="你的API密钥"
export DAYTONA_API_URL="https://47.236.111.154/api"

# 5. 测试 SSH
ssh -p 2222 i73YA5uQjkjQgvnD9FDU18jQvqV85LlQ@47.236.111.154

# 6. 查看文档
cat /root/daytona/CHECKPOINT.md
cat /root/daytona/DEPLOYMENT_GUIDE.md
```
