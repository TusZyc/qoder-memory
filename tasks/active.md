# 进行中的任务

**最后更新**: 2026-04-02 20:30

---

## 记忆系统重构

**状态**: ✅ 已完成  
**优先级**: 高

### 完成情况
- [x] 创建目录结构
- [x] 创建 HANDOFF.md、knowledge/*.md、tasks/*.md
- [x] 推送到 GitHub
- [x] 验证完整性

---

## Claude Code 全面 Review 与 Bug 修复

**状态**: ✅ 已完成并部署  
**优先级**: 高  
**提交**: `3e6a96a` (Co-Authored-By: Claude Opus 4.6)

### 修复内容
- [x] 所有API控制器 `Cache::remember` 闭包内Token过期问题（闭包内获取最新Token，失败不缓存）
- [x] TokenRefreshService 添加 Redis 锁防止并发刷新
- [x] 新增通知轻量级 summary 端点（下拉框秒开）
- [x] 首页视频加载前星空流星 CSS 动画
- [x] 补全 ESI 调用缺失的 `datasource=serenity`
- [x] 三个布局文件添加 CSRF meta 标签
- [x] 合同接口改为5分钟缓存（移除错误的 `Cache::forget`）
- [x] 移除技能页面60秒自动刷新
- [x] 删除无用的 `CharacterController::refresh()` 空方法及路由
- [x] 添加 CLAUDE.md 项目开发指南

### 涉及文件（20个）
```
CLAUDE.md (+)
app/Http/Controllers/Api/BookmarkDataController.php
app/Http/Controllers/Api/CharacterKillmailDataController.php
app/Http/Controllers/Api/CharacterOnlineController.php
app/Http/Controllers/Api/ContactDataController.php
app/Http/Controllers/Api/ContractDataController.php
app/Http/Controllers/Api/FittingDataController.php
app/Http/Controllers/Api/MailDataController.php
app/Http/Controllers/Api/NotificationDataController.php
app/Http/Controllers/Api/SkillDataController.php
app/Http/Controllers/Api/StandingDataController.php
app/Http/Controllers/Api/WalletDataController.php
app/Http/Controllers/CharacterController.php
app/Services/TokenRefreshService.php
resources/views/layouts/admin.blade.php
resources/views/layouts/app.blade.php
resources/views/layouts/guest.blade.php
resources/views/skills/index.blade.php
resources/views/welcome.blade.php
routes/web.php
```

---

## 待办队列

### 高优先级
- [ ] 修复通知接口 `ESI request failed for universe/names` 错误
- [ ] 验证市场搜索功能

### 中优先级
- [ ] 测试后台管理缓存功能
- [ ] Redis 缓存预热优化

### 低优先级
- [ ] CacheKeyService 采用率提升（当前 53%）
- [ ] 缓存键命名风格统一

---

## 历史任务归档

### 2026-04-02 完成
- [x] Claude Code 全面 Review + Bug 修复（20个文件，+622/-182）
- [x] 部署 `3e6a96a` 到服务器

### 2026-04-01 完成
- [x] GitHub 同步（4个虫洞优化文件）

### 2026-03-25 ~ 3-28 完成
- [x] 虫洞查询功能完整开发（2604星系/90虫洞类型/6效果）
- [x] 虫洞页面多轮优化（侧边栏/样式/KM/行星类型解析）

### 2026-03-24 完成
- [x] 修复 commit 59f0549 误删的数据文件
- [x] 修复市场搜索静态文件加载

### 2026-03-23 完成
- [x] 斥候工具（Scout）开发

### 2026-03-22 完成
- [x] LP Store 价格计算模式
- [x] Guide 使用指南页面
