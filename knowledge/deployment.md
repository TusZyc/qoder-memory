# 部署与运维手册

**最后更新**: 2026-03-24

---

## 服务器信息

| 项目 | 值 |
|------|-----|
| **云服务商** | 阿里云 ECS |
| **IP 地址** | 47.116.125.182 |
| **操作系统** | Ubuntu |
| **SSH 用户** | root |
| **SSH 密钥** | `d:\Qoder-work\.qoder\openclaw.pem` |
| **项目目录** | /opt/eve-esi |

---

## SSH 连接

```bash
# 家里电脑
ssh -i "d:\Qoder-work\.qoder\openclaw.pem" root@47.116.125.182

# 公司电脑
ssh -i "d:\Qoder-work\.qoder\qoder_server.pem" root@47.116.125.182
```

---

## Docker 容器

| 容器名 | 服务 | 说明 |
|--------|------|------|
| eve-esi-app | PHP-FPM | Laravel 应用 |
| eve-esi-nginx | Nginx | Web 服务器 |
| eve-esi-redis | Redis | 缓存服务 |

### 常用 Docker 命令
```bash
# 查看容器状态
docker ps

# 查看日志
docker compose logs -f app
docker compose logs -f nginx

# 重启容器
docker restart eve-esi-app
docker restart eve-esi-nginx

# 进入容器
docker compose exec app bash
```

---

## 部署流程

### 标准部署流程

1. **在服务器上修改代码**（推荐）
   ```bash
   cd /opt/eve-esi
   # 编辑文件
   vim app/Services/MarketService.php
   ```

2. **或者从本地上传文件**
   ```bash
   # 本地执行 - 上传到服务器 /tmp
   scp -i "d:\Qoder-work\.qoder\openclaw.pem" app/Services/MarketService.php root@47.116.125.182:/tmp/
   
   # 服务器执行 - 复制到项目目录
   cp /tmp/MarketService.php /opt/eve-esi/app/Services/
   ```

3. **清理缓存**
   ```bash
   docker compose exec -T app php artisan cache:clear
   docker compose exec -T app php artisan config:clear
   docker compose exec -T app php artisan view:clear
   docker compose exec -T app php artisan route:clear
   ```

4. **重启应用（如果修改了 PHP 代码）**
   ```bash
   docker restart eve-esi-app
   ```

5. **提交代码到 GitHub**
   ```bash
   cd /opt/eve-esi
   git add -A
   git commit -m "feat: xxx"
   git push origin main
   ```

### ⚠️ 重要注意事项

1. **代码以服务器为准**：所有代码修改都应该在服务器上完成或上传到服务器
2. **推送从服务器执行**：git push 应该在服务器上执行，而不是本地
3. **tarball 部署问题**：tarball 方式曾导致 PHP `$` 变量符号丢失，使用 SCP 直传
4. **git push 不会自动部署**：推送后需要手动 SCP + 清理缓存

---

## 缓存管理

### 清理缓存
```bash
# 清理所有缓存
docker compose exec -T app php artisan cache:clear

# 清理配置缓存
docker compose exec -T app php artisan config:clear

# 清理视图缓存
docker compose exec -T app php artisan view:clear

# 清理路由缓存
docker compose exec -T app php artisan route:clear
```

### 预热缓存
```bash
# 市场分组缓存（避免首次加载超时）
docker compose exec -T app php artisan market:cache-groups
```

### Redis 操作
```bash
# 进入 Redis
docker exec -it eve-esi-redis redis-cli

# 查看所有键
KEYS *

# 删除指定键
DEL key_name

# 清空所有
FLUSHALL
```

---

## 数据更新

### EVE 静态数据更新
```bash
# 手动更新（从 ceve-market.org 下载 evedata.xlsx）
docker compose exec -T app php artisan eve:update-data
```

### 自动更新
- **时间**：每周一凌晨 2:00
- **配置**：host 级别 crontab
- **命令**：`eve:update-data`

---

## HTTPS 证书

### 证书信息
- **工具**：acme.sh v3.1.3
- **证书目录**：/etc/nginx/ssl/
- **有效期**：90 天
- **验证方式**：DNS-01（阿里云 DNS API）

### 自动续期
- **Cron**：`1 14 * * *` 每天 14:01 检查续期
- **续期后**：自动执行 `docker restart eve-esi-nginx`

### 手动续期
```bash
# 强制续期
~/.acme.sh/acme.sh --renew -d 51-eve.online --force

# 安装证书
~/.acme.sh/acme.sh --install-cert -d 51-eve.online \
  --key-file /etc/nginx/ssl/51-eve.online.key \
  --fullchain-file /etc/nginx/ssl/51-eve.online.crt \
  --reloadcmd "docker restart eve-esi-nginx"
```

---

## 故障排查

### 常见问题

#### 1. 页面显示英文
**原因**：数据文件丢失或 PHP 静态变量缓存  
**解决**：
```bash
# 检查数据文件
ls -la /opt/eve-esi/data/

# 重启应用清除 PHP 缓存
docker restart eve-esi-app
```

#### 2. 市场搜索 500 错误
**原因**：Redis 缓存丢失，ESI API 超时  
**解决**：
```bash
# 预热市场分组缓存
docker compose exec -T app php artisan market:cache-groups
```

#### 3. 登录后跳转循环
**原因**：Session 有效但 Token 过期  
**解决**：用户需要重新授权

#### 4. 文件修改不生效
**原因**：PHP 静态变量缓存  
**解决**：
```bash
docker restart eve-esi-app
```

### 日志查看
```bash
# Laravel 日志
docker compose exec app tail -f storage/logs/laravel.log

# Nginx 访问日志
docker compose logs -f nginx

# 查看最近错误
docker compose exec app grep -i error storage/logs/laravel.log | tail -20
```

---

## 备份策略

### 数据库备份
```bash
# 导出 SQLite 数据库
cp /opt/eve-esi/database/database.sqlite /opt/backup/database_$(date +%Y%m%d).sqlite
```

### 数据文件备份
```bash
# 备份静态数据
tar -czf /opt/backup/data_$(date +%Y%m%d).tar.gz /opt/eve-esi/data/
```

---

## 监控

### 健康检查
```bash
# 检查服务状态
docker ps

# 检查网站访问
curl -I https://51-eve.online

# 检查 Redis
docker exec eve-esi-redis redis-cli ping
```

### 磁盘空间
```bash
df -h
du -sh /opt/eve-esi/storage/logs/
```
