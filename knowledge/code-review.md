# 代码审查报告

**审查日期**: 2026-04-01
**审查者**: [Claude Code]
**审查范围**: 全项目（后端 + 前端 + 安全 + 性能）

---

## 🔴 严重问题（必须修复）

### 1. OAuth 登录缺少 state 参数验证
- **文件**: `app/Http/Controllers/AuthController.php`
- **问题**: 登录回调没有验证 state 参数，存在 CSRF 攻击和授权码注入风险
- **修复**: 登录重定向时生成随机 state 存入 session，回调时验证匹配
- **状态**: ⬜ 未修复

### 2. Token 明文存储在数据库
- **文件**: `database/migrations/2026_03_09_000001_create_users_table.php`
- **问题**: access_token 和 refresh_token 以明文存储，数据库泄露则所有用户令牌暴露
- **修复**: 使用 Laravel 的 `encrypted` cast 或 `Crypt` 门面加密
- **状态**: ⬜ 未修复

### 3. CSRF 保护被不当关闭
- **文件**: `app/Http/Middleware/VerifyCsrfToken.php`
- **问题**: `auth/callback` 和 `api/*` 被排除在 CSRF 保护外
- **修复**: 配合 OAuth state 参数解决 callback 的问题；API 路由应迁移到 api.php
- **状态**: ⬜ 未修复

### 4. 前端使用 Tailwind CDN（严重性能问题）
- **文件**: 所有布局文件的 `<head>` (`layouts/app.blade.php`, `layouts/guest.blade.php`, `layouts/admin.blade.php`, `welcome.blade.php`, `guide.blade.php`)
- **问题**: 加载 ~300KB 的 JS 实时生成 CSS，Tailwind 官方明确标注"仅用于开发"
- **修复**: 使用 Vite 或 Laravel Mix 构建编译版 Tailwind CSS
- **状态**: ⬜ 未修复

### 5. 首页 20MB 背景视频
- **文件**: `public/eve-esi-bg.webm` (19,977,900 bytes)
- **问题**: 用户首次访问需下载 20MB，在国内网络环境下体验极差
- **修复**: 压缩到 2-3MB，提供 poster 图片，考虑仅桌面端加载
- **状态**: ⬜ 未修复

### 6. 布局文件缺少 CSRF meta 标签
- **文件**: `resources/views/layouts/app.blade.php`
- **问题**: 邮件和舰队功能中 `document.querySelector('meta[name="csrf-token"]')` 返回 null，导致功能报错
- **修复**: 在 `<head>` 中添加 `<meta name="csrf-token" content="{{ csrf_token() }}">`
- **状态**: ⬜ 未修复

---

## 🟠 高危问题（尽快修复）

### 7. getNames() 方法重复 6 次
- **文件**: ContactDataController, ContractDataController, StandingDataController, CharacterKillmailDataController, WalletDataController, CharacterController
- **问题**: 同一段 ESI 名称查询 + 缓存逻辑复制粘贴了 6 次（~300 行重复代码）
- **修复**: 提取到 EveDataService 的共享方法
- **状态**: ⬜ 未修复

### 8. 缓存闭包中的 Token 过期问题
- **文件**: WalletDataController, SkillDataController, MailDataController 等
- **问题**: `Cache::remember()` 闭包捕获了当时的 token，token 刷新后缓存中仍是旧 token；ESI 调用失败的空结果也会被缓存
- **修复**: 在闭包内获取最新 token；ESI 失败时不缓存结果
- **状态**: ⬜ 未修复

### 9. 舰队功能缺少权限检查
- **文件**: `app/Http/Controllers/FleetController.php`
- **问题**: show/members/report/export 方法任何登录用户都可访问任意舰队
- **修复**: 添加所有者或参与者权限验证
- **状态**: ⬜ 未修复

### 10. 合同页面每次绕过缓存
- **文件**: `app/Http/Controllers/Api/ContractDataController.php` line 53
- **问题**: `Cache::forget()` 在每次请求时调用，缓存形同虚设
- **修复**: 移除 Cache::forget，依赖正常的 TTL 过期机制
- **状态**: ⬜ 未修复

### 11. 技能页面每 60 秒强制刷新整页
- **文件**: `resources/views/skills/index.blade.php` line 250
- **问题**: `setTimeout(function() { location.reload(); }, 60000)` 导致用户失去滚动位置和展开状态
- **修复**: 改为 AJAX 局部刷新数据
- **状态**: ⬜ 未修复

### 12. 无手机端响应式布局
- **文件**: 所有布局文件
- **问题**: 侧边栏固定 256px，无 hamburger 菜单，小屏幕基本不可用
- **修复**: 添加移动端响应式设计（折叠侧边栏 + hamburger 按钮）
- **状态**: ⬜ 未修复

### 13. 错误信息泄露内部细节
- **文件**: WalletDataController, ContractDataController, MailDataController, AuthController
- **问题**: `$e->getMessage()` 直接返回给前端，可能暴露内部路径、数据库结构等
- **修复**: 生产环境返回通用错误信息，详细信息只写入日志
- **状态**: ⬜ 未修复

### 14. 管理员验证用角色名而非角色 ID
- **文件**: `app/Http/Middleware/EnsureSiteAdmin.php`, `config/admin.php`
- **问题**: EVE 角色名可以修改，用名字判断管理员不安全
- **修复**: 改用 `eve_character_id`
- **状态**: ⬜ 未修复

---

## 🟡 中等问题（计划修复）

### 15. CacheKeyService 定义了但大部分代码未使用
- **问题**: 大部分控制器自行硬编码缓存键名，与 CacheKeyService 不一致
- **状态**: ⬜ 未修复

### 16. 大 JSON 文件每次请求重新加载到内存
- **文件**: NotificationDataController 的 getSystemsData() / getItemsData()
- **问题**: file_get_contents() 每次请求都读磁盘，应用 Laravel 缓存
- **状态**: ⬜ 未修复

### 17. set_time_limit(120) 用于 Web 请求
- **文件**: AssetDataController, MarketDataController
- **问题**: 应改为队列异步处理
- **状态**: ⬜ 未修复

### 18. API 路由写在 web.php
- **问题**: `/api/*` 路由在 web.php 中，走了不必要的 session 中间件
- **状态**: ⬜ 未修复

### 19. escapeHtml() 等函数在 18 个文件中重复定义
- **修复**: 提取到 public/js/utils.js 共享文件
- **状态**: ⬜ 未修复

### 20. CSS 在三个布局文件中重复 200+ 行
- **修复**: 提取到 public/css/app.css 共享文件
- **状态**: ⬜ 未修复

### 21. 令牌刷新竞争条件
- **文件**: AutoRefreshEveToken 中间件 + TokenRefreshService
- **问题**: 并发请求可能同时刷新 token，导致其中一个失效
- **修复**: 使用 Laravel 的 Cache::lock() 实现分布式锁
- **状态**: ⬜ 未修复

### 22. 通知 item 查找 O(n*m) 复杂度
- **文件**: NotificationDataController lines 403-414
- **问题**: 嵌套循环查找 item 名称，应改为 hashmap 索引
- **状态**: ⬜ 未修复

### 23. Chart.js 始终加载（即使用户未使用图表）
- **文件**: market/index.blade.php
- **修复**: 改为懒加载
- **状态**: ⬜ 未修复

### 24. alert() 用于错误提示
- **文件**: killmails/index.blade.php, mail/index.blade.php
- **修复**: 改为 toast 通知
- **状态**: ⬜ 未修复

---

## 🟢 低优先级

- 日志打印过多（每次成功请求都 Log::info）
- 错误返回格式不统一
- CharacterController::refresh() 方法无实际逻辑
- 部分 ESI 调用缺少 datasource=serenity
- 缺少 `<noscript>` 标签
- 动态外部链接缺少 rel="noopener noreferrer"
- Token 片段被写入日志（即使截断了仍有风险）

---

## 修复进度跟踪

| 类别 | 总数 | 已修复 | 剩余 |
|------|------|--------|------|
| 🔴 严重 | 6 | 0 | 6 |
| 🟠 高危 | 8 | 0 | 8 |
| 🟡 中等 | 10 | 0 | 10 |
| 🟢 低优先级 | 7 | 0 | 7 |
