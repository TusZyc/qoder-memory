# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-14（第五次会话 - 公司电脑）**

## 已完成功能

### 1. OAuth2 认证系统（100%）
- 3 步授权流程（清除缓存 → 授权 → 填写授权码）
- EVE 国服 OAuth2，Client ID: bc90aa496a404724a93f41b4f4e97761
- 3V 完整 73 个权限
- Refresh Token 自动刷新（提前 5 分钟，通过中间件 AutoRefreshEveToken）
- TokenRefreshService 统一刷新逻辑（解决三处重复问题）

### 2. Dashboard 仪表盘（100%）
- 异步数据加载 + 骨架屏
- 服务器状态卡片（状态指示灯 + 在线玩家 + 版本 + 启动时间 + VIP 模式）
- 角色信息卡片（角色名 + 军团 + 联盟 + 位置 + 在线状态）
- 技能信息卡片（总 SP + 未分配 SP + 已学技能数）
- 导航栏紧凑布局（图标导航）

### 3. 技能队列页面（100%）
- 异步加载架构：SkillController 返回视图壳，SkillDataController API 异步加载
- API：/api/dashboard/skills/overview, queue, groups
- 全量技能显示：25 个分组 673 个技能，已学/未学区分
- 训练中/等待中/已完成三种状态 + 进度条
- 60 秒自动刷新

### 4. 资产页面（100%）
- 两步加载架构：先位置列表，再按需加载物品
- API：/api/dashboard/assets/locations, location/{id}, search
- 搜索、舰船/物品机库分类、树形展示、位置标志中文映射
- 并发 HTTP 请求 + 三层缓存

### 5. 本地数据服务（100% - 2026-03-13）
- ceve-market.org evedata.xlsx → Python 脚本 → JSON
- eve_names.json (43,305条)、eve_station_systems.json (5,464条)
- EveDataService：本地优先，ESI API 兜底

### 6. 安全改进（2026-03-12）
- TokenRefreshService 统一刷新、缓存 key 前缀、XSS 防护、角色 ID 验证

### 7. 市场功能（100% - 2026-03-14，第五次会话添加）
- **MarketController**：公开访问，支持游客/已登录两种模式
- **MarketDataController API**：
  -  — 市场分组树
  -  — 分组详情
  -  — 市场订单
  -  — 价格历史
  -  — 物品详情
  - （认证）— 角色订单
  - （认证）— 角色订单 ID 列表
- **MarketService**：ESI API 封装，分组树构建、订单/历史/物品查询
- **CacheMarketGroups artisan 命令**：预缓存 2141 个市场分组（避免 Web 超时）
- **前端**：分组树浏览 + 订单表 + 价格历史图表

### 8. 游客仪表盘（100% - 2026-03-14，第五次会话添加）
- **GuestDashboardController**：无需授权，GET /guest
- 三服务器状态卡片 + 授权提示 + 功能预览（角色/技能/资产锁定）

### 9. 首页 "Tus Esi System (Beta)"（100% - 2026-03-13）
- 视频背景 eve-esi-bg.webm + 三服务器状态
- 授权使用/无授权使用入口

### 10. KM 查询页面（100% - 2026-03-14，家里电脑开发）
- **数据源**：beta.ceve-market.org REST API（protobuf 响应）
- **自定义 Protobuf 解码器**：pbDecodeVarint / pbParseMessage 等
- **击杀列表**：Beta KB /app/list/pilot/{id}/kill，50 条/页
- **ESI Hash 提取**：Beta KB /app/kill/{id}/info → 40 字符 hex hash
- **KM 详情**：ESI /killmails/{id}/{hash}/ 完整击杀数据
- **前端**：角色搜索 + KM 列表 + 模态框详情，支持 KM ID/KB 链接/ESI 链接

### 11. 统一导航栏（2026-03-14）
- 认证页面统一 5 图标：🏠📚📦👥⚔️ + 登出
- 当前页面 bg-white/10 高亮
- 市场页面保持独立导航（游客/已登录两种模式）

### 12. API 端点
- GET /api/public/server-status（公开）
- GET /api/public/market/groups, groups/{id}, orders, history, types/{id}（公开市场）
- GET /api/dashboard/server-status, skills, skill-queue
- GET /api/dashboard/skills/overview, queue, groups
- GET /api/dashboard/assets/locations, location/{id}, search
- GET /api/dashboard/character-info, character-location, character-online
- GET /api/market/character-orders, my-order-ids（认证市场）
- GET /api/killmails/search, pilot/{id}/kills, kill/{id}（KM）

## 待开发功能

| 优先级 | 功能 | 状态 |
|--------|------|------|
| 高 | 钱包查询页面 | 路由和 Controller 方法存在，缺视图 |
| 高 | 资产估值（价格数据） | API 框架完成，缺价格数据 |
| 中 | 军团管理页面 | 无代码 |
| 低 | 合同查询 | 无代码 |
| 低 | 数据可视化 | 无代码 |

## 已知问题
- 前端全部内联在 Blade 模板中，无组件化（导航栏各页面独立维护）
- 技能进度条 SP 阈值固定，不精确
- evedata.xlsx 少量物品翻译可能不准确

## 关键文件
- 路由：routes/web.php
- 认证：app/Http/Controllers/AuthController.php
- 首页：resources/views/welcome.blade.php
- 仪表盘：app/Http/Controllers/DashboardController.php + resources/views/dashboard.blade.php
- API 数据：app/Http/Controllers/Api/DashboardDataController.php
- 服务器状态 API：app/Http/Controllers/Api/ServerStatusController.php
- 技能 API：app/Http/Controllers/Api/SkillDataController.php
- 技能页面：app/Http/Controllers/SkillController.php + resources/views/skills/index.blade.php
- 资产：app/Http/Controllers/Api/AssetDataController.php + resources/views/assets/index.blade.php
- 市场：app/Http/Controllers/MarketController.php + Api/MarketDataController.php + MarketService.php
- KM 查询：app/Http/Controllers/Api/KillmailController.php + KillmailService.php
- 游客仪表盘：app/Http/Controllers/GuestDashboardController.php
- 市场缓存命令：app/Console/Commands/CacheMarketGroups.php
- 数据服务：app/Services/EveDataService.php
- 数据更新：scripts/update_evedata.py + app/Console/Commands/UpdateEveData.php
- Token 刷新：app/Services/TokenRefreshService.php
- ESI 配置：config/esi.php
- 缓存配置：config/cache.php（文件驱动）
