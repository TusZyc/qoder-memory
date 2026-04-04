# 进行中的任务

**最后更新**: 2026-04-04
**更新者**: [Claude Code]

---

## 🔴 服务器 OOM 恢复

**状态**: 🔴 待处理
**优先级**: 最高
**负责**: [Qoder]

### 背景
2026-04-04 重建 Docker 镜像时（编译 PHP GD FreeType 扩展），服务器内存耗尽导致 OOM kill。
VNC 截图显示内核 OOM killer 杀掉了 php-fpm, systemd, AliYunDunMonito 等进程。

### 恢复步骤
- [ ] VNC 登录，确认系统是否可正常访问
- [ ] 添加 2GB swap：`fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile`
- [ ] `cd /opt/eve-esi && docker-compose build --no-cache app`（有swap后编译不会OOM）
- [ ] `docker-compose up -d`
- [ ] 验证 GD：`docker exec eve-esi-app php -r "var_dump(gd_info());"`（需 FreeType Support: true）
- [ ] `git pull origin main`（拉取 `4bc5e05` KM图片功能）
- [ ] 测试 KM 图片端点：`GET /api/killmails/kill/22460248/image`

---

## ✅ KM 图片生成功能（代码完成）

**状态**: ✅ 代码完成，待服务器恢复后验证
**优先级**: 高
**负责**: [Claude Code]
**commit**: `4bc5e05`

### 完成内容
- [x] `KillmailImageService.php`（843行）：服务端 PHP GD 图片生成
- [x] 布局：标题栏 → 头部（头像+舰船渲染+信息）→ 统计栏 → [左:攻击者列表 | 右:装备槽位]→ 页脚
- [x] EVE in-game KM UI 风格（深蓝黑主题，黄色标题）
- [x] `KillmailController::killImage()` 路由：`GET /api/killmails/kill/{killId}/image`
- [x] `/killmails` 页面添加"生成 KM 图片"面板（紫色主题，输入KM ID，预览+下载）
- [x] NotoSansSC 字体上传至服务器 `/opt/eve-esi/storage/fonts/`
- [x] Dockerfile 已修改支持 GD FreeType/JPEG/WebP（需 rebuild 才能生效）
- [x] 修复 PHP 命名空间 GD 函数调用问题（`use function imagettftext;` 等）

### 待验证（服务器恢复后）
- [ ] 测试中文字符渲染
- [ ] 测试图片缓存机制
- [ ] 测试大量攻击者时的布局

---

## ✅ P0代码优化（已完成）

**状态**: ✅ 已完成
**优先级**: 最高
**负责**: [Qoder]
**commit**: `2c6f586`

### P0-1: Token重复查询问题
- [x] 创建 TokenService 统一Token获取服务
- [x] 同一请求周期内只查询数据库一次
- [x] 修改10个控制器使用TokenService替代重复查询
- [x] 预计减少约80%的Token数据库查询

### P0-2: 空间站名称代码重复问题
- [x] 增强 StationNameService 为统一服务
- [x] 本地数据优先 + ESI兜底 + 私有建筑处理
- [x] 重构 AssetDataService、MarketService、CharacterLocationController
- [x] 消除约200行重复代码

---

## 🔴 代码审查问题修复

**状态**: 🔄 进行中（部分完成）
**优先级**: 高
**负责**: [Claude Code]

### 背景
2026-04-01 [Claude Code] 对全项目进行了安全/性能/代码质量审查，发现 6 个严重、8 个高危、10+ 个中等问题。详见 `knowledge/code-review.md`。

### 严重问题修复清单
- [x] 添加 CSRF meta 标签到布局文件（`3e6a96a`）
- [ ] OAuth 添加 state 参数验证
- [ ] 替换 Tailwind CDN 为编译版本
- [ ] 压缩首页背景视频 (20MB → 2-3MB)
- [ ] Token 数据库存储加密
- [ ] 管理员验证改用角色 ID

### 高危问题修复清单
- [ ] 提取 getNames() 到共享服务
- [x] 修复缓存闭包中的 token 过期问题（`3e6a96a`）
- [ ] 舰队功能添加权限检查
- [x] 合同页面恢复正常缓存（`3e6a96a`）
- [x] 技能页面改为移除自动刷新（`3e6a96a`）
- [ ] 添加手机端响应式布局
- [ ] 错误信息脱敏

### 中等问题已修复
- [x] 补全 ESI datasource=serenity（`3e6a96a`）
- [x] TokenRefreshService 添加 Redis 锁防并发（`3e6a96a`）
- [x] 删除 CharacterController::refresh() 空方法（`3e6a96a`）
- [x] 新增通知 summary 轻量端点（`3e6a96a`）
- [x] 首页视频加载前星空动画（`3e6a96a`）

---

## 🔴 通知接口 ESI 错误

**状态**: 🔄 待修复
**优先级**: 最高
**负责**: [待分配]

### 问题
服务器日志报错：`ESI request failed for universe/names`
- 文件：`NotificationDataController.php` 第 422 行
- 在 `resolveNames()` 方法中调用 ESI `universe/names` 接口失败
- 错误被 `Cache::remember` 缓存了（1小时 TTL），会持续影响

### 日志原文
```
[2026-04-02 19:56:37] production.WARNING: ESI names 解析失败: ESI request failed for universe/names
[2026-04-02 19:56:37] production.ERROR: 🔔 [Notifications] 获取提醒数据失败
```

---

## 待办队列

### 高优先级
- [ ] **服务器 OOM 恢复** + rebuild Docker + 验证 KM 图片功能 `[Qoder]`
- [ ] 修复通知接口 `ESI request failed for universe/names` 错误 `[待分配]`
- [ ] 修复代码审查剩余严重问题（见上方清单）`[Claude Code]`
- [ ] 验证市场搜索功能 `[Qoder]`

### 中优先级
- [ ] KM 图片布局细化（攻击者分组、装备图标对齐）`[Claude Code]`
- [ ] 修复代码审查中等问题 `[Claude Code]`
- [ ] 测试后台管理缓存功能 `[待分配]`
- [ ] Redis 缓存预热优化 `[待分配]`

### 低优先级
- [ ] CacheKeyService 统一使用
- [ ] 提取公共 JS/CSS 到共享文件

---

## 历史任务归档

### 2026-04-04 完成
- [x] **KM图片生成功能**: KillmailImageService (843行) + 前端面板 + 字体上传 `[Claude Code]`
- [x] GitHub 推送 `4bc5e05` `[Claude Code]`

### 2026-04-02 完成
- [x] **P0代码优化**: TokenService统一服务 + StationNameService重构（+1118/-382行）`[Qoder]`
- [x] 部署 `2c6f586` 到服务器（15个文件）`[Qoder]`
- [x] Claude Code 全面 Review + Bug 修复（20个文件，+622/-182）`[Claude Code]`
- [x] 部署 `3e6a96a` 到服务器 `[Qoder]`

### 2026-04-01 完成
- [x] 全面代码审查 `[Claude Code]`
- [x] GitHub 同步（4个虫洞优化文件）`[Qoder]`

### 2026-03-25 ~ 3-28 完成
- [x] 虫洞查询功能完整开发 `[Qoder]`
- [x] 虫洞页面多轮优化 `[Qoder]`

### 2026-03-24 完成
- [x] 修复 commit 59f0549 误删的数据文件 `[Qoder]`
- [x] 修复市场搜索静态文件加载 `[Qoder]`

### 2026-03-23 完成
- [x] 斥候工具（Scout）开发 `[Qoder]`

### 2026-03-22 完成
- [x] LP Store 价格计算模式 `[Qoder]`
- [x] Guide 使用指南页面 `[Qoder]`
