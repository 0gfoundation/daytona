# Daytona 快速参考卡片

## 🌐 访问地址
- **Dashboard**: https://47.236.111.154/dashboard
- **API**: https://47.236.111.154/api
- **SSH**: 47.236.111.154:2222

## 🔑 登录凭据
- **用户**: dev@daytona.io
- **密码**: password

## 🚀 快速启动
```bash
cd /root/daytona/docker
docker compose up -d
```

## 🔍 状态检查
```bash
docker compose ps
docker compose logs -f api
```

## 📡 API 使用
```bash
export DAYTONA_API_KEY="你的密钥"
export DAYTONA_API_URL="https://47.236.111.154/api"

# 测试
curl -k -X GET "$DAYTONA_API_URL/sandbox" \
  -H "Authorization: Bearer $DAYTONA_API_KEY"
```

## 🔧 SSH 测试
```bash
ssh -p 2222 <TOKEN>@47.236.111.154
```

## ⏳ 待测试
- [ ] SSH 连接
- [ ] 在线终端
- [ ] WebSocket 连接

## 📚 详细文档
- `CHECKPOINT.md` - 完整断点记录
- `DEPLOYMENT_GUIDE.md` - 部署指南
- `CHANGES_SUMMARY.md` - 变更记录

## 🆘 问题排查
```bash
# SSH 问题
docker compose logs --tail=50 ssh-gateway

# WebSocket 问题
# 清除浏览器缓存，F12 查看 Network → WS

# API 问题
docker compose logs --tail=100 api
```
