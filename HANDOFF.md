# 设备交接清单

**最后更新**: 2026-04-02 20:30  
**更新设备**: 家里电脑  
**更新者**: Qoder（基于 Claude Code 提交）

---

## 🔥 立即关注

### 当前任务
- [x] 修复 commit 59f0549 误删的数据文件（已完成）
- [x] 市场搜索静态文件优化（已部署）
- [x] 虫洞查询功能开发（已完成并部署）
- [x] 虫洞页面优化（侧边栏/样式/中文化/天体/KM）
- [x] 虫洞行星类型解析优化 + 搜索错误处理
- [x] Claude Code 全面 Review + Bug修复（Token缓存/通知性能/安全优化）
- [ ] 验证市场搜索功能正常（待确认）
- [ ] 修复通知接口 ESI universe/names 调用失败（日志 ERROR）

### 服务器状态
| 项目 | 状态 |
|------|------|
| 最后部署 | 2026-04-02 |
| 最后 commit | `3e6a96a` (fix: 全面修复Token缓存/通知性能/星空效果/安全优化) |
| 待部署变更 | 无 |
| Docker 容器 | eve-esi-app / eve-esi-nginx / eve-esi-redis 全部运行中 |
| GitHub 同步 | ✅ 已同步至 `3e6a96a` |

### ⚠️ 重要注意事项
1. **PHP 静态变量缓存**：修改 PHP 代码后需要 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，现已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 文件不能删除，git status 时注意检查
4. **虫洞行星类型**：改用 `/universe/types/{type_id}/?language=zh` 获取中文名称，已缓存30天
5. **Token缓存闭包**：所有API控制器的 `Cache::remember` 闭包必须在内部获取最新Token，不可在外部获取后传入
6. **ESI datasource**：国服ESI所有调用必须带 `datasource=serenity`
7. **多AI协作**：Claude Code 已加入开发（CLAUDE.md），代码推送走 GitHub，部署由 Qoder 执行

---

## 📋 上下文摘要

### 最近完成（7天内）
| 日期 | 内容 |
|------|------|
| 4-02 | Claude Code Review + 全面修复：Token缓存闭包/通知summary端点/星空动画/CSRF/datasource |
| 4-02 | 部署 `3e6a96a` 到服务器（git pull + 缓存清理） |
| 4-01 | GitHub 同步：提交4个虫洞优化文件到主仓库 |
| 3-28 | 虫洞功能优化 - 行星类型解析(type_id→/universe/types)/搜索404→友好错误页/KM限5条 |
| ~3-26 | 虫洞页面优化 - 侧边栏登录状态/下拉框样式/效果中文化/天体信息/KM时间范围/URL参数 |
| ~3-25 | 虫洞查询功能完整上线 - 2604星系/90虫洞类型/6效果/完整数据 |

### 下一步计划
1. 修复通知接口 `ESI request failed for universe/names` 错误
2. 验证 Claude Code 修复后各功能正常运行
3. 验证市场搜索功能
4. 测试后台管理缓存功能

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
