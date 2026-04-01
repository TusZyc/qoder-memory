# 设备交接清单

**最后更新**: 2026-04-01 11:00  
**更新设备**: 家里电脑  
**更新者**: Qoder

---

## 🔥 立即关注

### 当前任务
- [x] 修复 commit 59f0549 误删的数据文件（已完成）
- [x] 市场搜索静态文件优化（已部署）
- [x] 虫洞查询功能开发（已完成并部署）
- [x] 虫洞页面优化（侧边栏/样式/中文化/天体/KM）
- [x] 虫洞行星类型解析优化 + 搜索错误处理
- [ ] 验证市场搜索功能正常（待确认）

### 服务器状态
| 项目 | 状态 |
|------|------|
| 最后部署 | 2026-04-01 |
| 最后 commit | `0067fe4` (fix: 虫洞功能优化 - 行星类型解析/搜索错误处理/星系半径计算/KM数量限制) |
| 待部署变更 | 无 |
| Docker 容器 | eve-esi-app / eve-esi-nginx / eve-esi-redis 全部运行中 |
| GitHub 同步 | ✅ 已同步至 `0067fe4` |

### ⚠️ 重要注意事项
1. **PHP 静态变量缓存**：修改 PHP 代码后需要 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，现已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 文件不能删除，git status 时注意检查
4. **虫洞行星类型**：改用 `/universe/types/{type_id}/?language=zh` 获取中文名称，已缓存30天

---

## 📋 上下文摘要

### 最近完成（7天内）
| 日期 | 内容 |
|------|------|
| 4-01 | GitHub 同步：提交4个虫洞优化文件到主仓库 |
| 3-28 | 虫洞功能优化 - 行星类型解析(type_id→/universe/types)/搜索404→友好错误页/KM限5条 |
| ~3-26 | 虫洞页面优化 - 侧边栏登录状态/下拉框样式/效果中文化/天体信息/KM时间范围/URL参数 |
| ~3-25 | 删除废弃 navbar.blade.php |
| ~3-25 | 虫洞查询功能完整上线 - 2604星系/90虫洞类型/6效果/完整数据 |
| 3-24 | 修复数据文件误删 + 市场搜索静态文件优化 |
| 3-23 | 斥候工具（Scout）开发完成 - 扫描数据解析+分享链接 |

### 下一步计划
1. 验证虫洞功能正常运行（行星类型显示、搜索错误处理）
2. 验证市场搜索功能
3. 测试后台管理缓存功能

---

## 🔗 快速链接

| 文件 | 用途 |
|------|------|
| `knowledge/pitfalls.md` | 踩坑记录与调试经验 |
| `knowledge/project-overview.md` | 项目功能概览 |
| `knowledge/deployment.md` | 部署与运维手册 |
| `tasks/active.md` | 当前任务详情 |
| `ideas/admin-panel-design.md` | 后台管理设计方案 |

---

## 🖥️ 环境速查

### SSH 连接
```bash
ssh -i "d:\Qoder-work\.qoder\openclaw.pem" root@47.116.125.182
```

### 常用命令
```bash
# 进入项目目录
cd /opt/eve-esi

# 清理缓存
docker compose exec -T app php artisan cache:clear

# 重启应用（清除 PHP 静态变量缓存）
docker restart eve-esi-app

# 查看日志
docker compose logs -f app
```

### 网站地址
- 正式：https://51-eve.online
- 后台：https://51-eve.online/admin
