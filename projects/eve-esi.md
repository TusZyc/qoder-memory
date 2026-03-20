# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-20**

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
- **异步加载架构**（2026-03-13 重构）：SkillController 仅返回视图壳，数据由 SkillDataController API 异步加载
- **API 端点**：
  - `GET /api/dashboard/skills/overview` — 总SP、未分配SP、训练剩余时间
  - `GET /api/dashboard/skills/queue` — 技能队列（带名称）
  - `GET /api/dashboard/skills/groups` — 所有技能按分组（含未学习）
- **全量技能显示**（2026-03-13）：
  - 从 ESI /universe/categories/16/ 获取全部 25 个技能分组
  - 每个分组从 /universe/groups/{id}/ 获取所有技能 type_ids（共 673 个）
  - 已学技能显示等级和蓝色进度条，未学技能灰色半透明 + "未学习" 标签
  - 分组标签显示 "已学/总数"（如 43/123），全学满显绿色
  - 缓存：eve_skill_category_groups (24h), eve_skillgroup_full_{id} (24h), skills_{char_id} (5min)
- 训练中/等待中/已完成三种状态 + 进度条
- 技能队列默认显示前 5 个，可展开/折叠
- 60 秒自动刷新

### 4. 资产页面（100%）
- **两步加载架构**：先加载位置列表（快速），再按需加载每个位置的物品详情
- **API 端点**：
  - `GET /api/dashboard/assets/locations` — 返回位置列表+物品数量
  - `GET /api/dashboard/assets/location/{locationId}` — 返回某位置的物品树
  - `GET /api/dashboard/assets/search?q=关键词` — 搜索物品
- **搜索功能**（2026-03-12）：
  - 结果按位置分组为可折叠卡片，与正常模式样式一致
  - 展开位置卡片可查看完整树形物品（调用 loadLocationItems）
  - 位置名称显示中文（优先读缓存，不再显示为 ID）
- **舰船/物品机库分类**（2026-03-12）：
  - 空间站/建筑内物品按 category_id 分为「舰船机库」和「物品机库」两个分组
  - getGroupNames() 返回 category_id，buildNode() 带入前端
  - 前端 renderHangarSections() 按 category_id === 6 分组渲染
- **物品计数修正**（2026-03-12）：
  - 舰船/集装箱内物品不再重复计算，以容器本身为 1 个单位
- **三层缓存**：
  - 原始资产数据 15 分钟（assets_raw_{characterId}）
  - 每个位置物品树 15 分钟（assets_loc_{characterId}_{locationId}）
  - 类型详情/分组名称/位置名称各 24 小时
- **并发 HTTP 请求**：Http::pool() 每批并发查询类型详情和分组名称
- **中文分组名称**：ESI API 调用添加 language=zh 和 datasource=serenity
- **前端懒加载**：位置默认折叠，点击展开时加载物品
- **自动预加载**：单星系用户自动展开星系和最大位置
- **树形展示**：通过 item_id/location_id 父子关系构建，支持展开/折叠
- **位置标志中文映射**：机库/货柜仓/无人机仓/高槽/中槽/低槽/改装件等

### 5. 本地数据服务（100% - 2026-03-13 重构）
- **数据来源**：ceve-market.org 的 evedata.xlsx（每日更新）
- **Python 脚本 update_evedata.py**：下载 xlsx，解析 6 张表，生成 JSON 数据文件
- **数据文件**：
  - `data/eve_names.json`：43,305 条 ID→中文名称映射（物品28K + 星域109 + 星座1.1K + 星系8.2K + NPC站5.2K + 玩家建筑247）
  - `data/eve_station_systems.json`：5,464 条站点→星系ID映射
  - `data/evedata_meta.json`：元数据（更新时间、各类计数）
- **EveDataService**：
  - getItemDatabase()：读取 eve_names.json，缓存 2 小时
  - getStationSystemMap()：读取 eve_station_systems.json，缓存 2 小时
  - getNameById() / getNamesByIds()：本地优先，ESI API 兜底
  - updateData()：调用 Python 脚本更新数据
- **AssetDataController 集成**：
  - getLocationInfo()：NPC空间站/玩家建筑优先从本地数据查找名称和所属星系
  - getSolarSystemNames()：星系名称优先从本地数据查找
  - 仅对本地找不到的实体才回退到 ESI API
- **自动更新**：每周一凌晨 2:00 cron 执行 eve:update-data（host 级别 crontab）
- EveHelper：静态门面

### 6. 安全改进（2026-03-12）
- TokenRefreshService 统一 Token 刷新（消除三处重复）
- 缓存 key 添加 user_id 前缀（防止碰撞）
- 前端用 textContent/createElement 替代 innerHTML（防 XSS）
- Controller 添加角色 ID 归属验证

### 7. 市场功能（100% - 2026-03-16 大重构）
- **MarketController**：公开访问，无需登录，支持游客/已登录两种模式
- **MarketDataController API 端点**：
  - `GET /api/public/market/groups` — 市场分组树
  - `GET /api/public/market/search` — 物品模糊搜索（本地 items.json）
  - `GET /api/public/market/regions` — 星域列表
  - `GET /api/public/market/active-types` — 区域内有订单的物品 ID 列表
  - `GET /api/public/market/orders?region_id=X&type_id=Y` — 市场订单
  - `GET /api/public/market/history?region_id=X&type_id=Y` — 价格历史
  - `GET /api/public/market/types/{id}` — 物品详情
  - `GET /api/market/character-orders`（认证）— 角色订单
  - `GET /api/market/my-order-ids`（认证）— 角色订单 ID 列表
- **MarketService**：
  - ESI API 封装，含分组树构建、订单/历史/物品查询、角色订单
  - `enrichOrdersWithLocation()`：中文站名翻译（星系名、卫星、军团名、27种设施类型映射）
  - `enrichOrdersWithExpires()`：计算订单到期时间
  - `getActiveTypeIds()`：区域内有订单的物品类型
  - `buildTypeCategoryMap()`：搜索结果分类路径构建
  - 站名缓存 key：`eve_locinfo_{id}`（与资产页共享）
- **CacheMarketGroups artisan 命令**：预缓存 2141 个市场分组 + 115 个星域
- **前端三栏布局**：
  - 左栏：区域列表（伏尔戈（吉他）默认 + 全部 + 115 星域，中文排前）
  - 中栏：市场分组树 / 搜索结果（300ms 防抖搜索 + 分类路径显示）
  - 右栏：物品信息 + 订单表（价格/数量/中文位置/到期时间） + 价格历史图表
  - 无订单物品灰色半透明 + "仅有订单"过滤复选框
  - 订单分页（10条/页 + 加载更多/折叠按钮）
  - "我的订单"标签仅登录用户可见
- **中文站名翻译**：参考 AssetDataController 模式，6 步翻译流程
  1. ESI 获取站点详情（名称、star_system_id、owner）
  2. Http::pool() 并行获取中英文星系名
  3. /universe/names 获取中英文军团名
  4. 27 种设施类型英中映射（Assembly Plant → 组装工厂 等）
  5. 替换：星系名→中文、Moon→卫星、军团名→中文、设施类型→中文
  6. 缓存到 `eve_locinfo_{id}`（24h TTL，与资产页共享）

### 8. 游客仪表盘（100% - 2026-03-14）
- **GuestDashboardController**：无需授权，显示服务器状态 + 功能预览
- 路由：`GET /guest`
- 三服务器状态卡片 + 授权提示 + 功能预览（角色/技能/资产锁定状态）

### 9. KM 查询功能（100% - 2026-03-14，家里电脑开发）
- **KillmailController**（Api 命名空间）：页面渲染 + 搜索/列表/详情 API
- **KillmailService**：KB API 集成 + Beta KB protobuf 解析 + ESI killmail 详情
- **API 端点**：
  - `GET /killmails` — KM 查询页面
  - `GET /api/killmails/search?q=角色名` — 搜索角色
  - `GET /api/killmails/pilot/{pilotId}/kills` — 角色 KM 列表
  - `GET /api/killmails/kill/{killId}?hash=xxx` — KM 详情
- **前端**：角色搜索 + KM 列表 + 模态框详情，支持 KM ID / KB 链接 / ESI 链接直接查询
- **ESI hash 获取策略**：前端优先 Beta KB API 提取 hex hash，降级旧 KB HTML，再降级后端

### 10. 共享布局系统（2026-03-16）
- **layouts/app.blade.php**：认证页面主布局，含 Tailwind CDN、共享样式
  - `@stack('head-scripts')` / `@stack('styles')` / `@yield('content')` / `@stack('scripts')`
  - 内容无容器包装，各页面自行添加 `<div class="container">`
- **layouts/guest.blade.php**：游客页面布局，结构同 app.blade.php
- **layouts/partials/navbar.blade.php**：共享导航栏
  - 所有用户：🏠 仪表盘 | 📊 市场 | ⚔️ KM
  - 仅认证用户：📚 技能 | 📦 资产 | 👥 角色
  - `$activePage` 高亮当前页，`$isLoggedIn` 控制显示
- 所有页面已迁移（dashboard、skills、assets、characters、market、killmails、guest-dashboard）

### 11. 统一导航栏（2026-03-16 升级）
- 通过 layouts/partials/navbar.blade.php 共享，不再各页面独立维护
- 当前页面 `bg-white/10` 高亮

### 13. 钱包功能（100% - 2026-03-19/20）
- **WalletController**：钱包页面控制器
- **WalletDataController API 端点**：
  - `GET /api/dashboard/wallet/balance` — 角色钱包余额
  - `GET /api/dashboard/wallet/journal` — 角色钱包流水
  - `GET /api/dashboard/wallet/corp-balance` — 军团钱包余额（Division 1-7）
  - `GET /api/dashboard/wallet/corp-journal` — 军团钱包流水（Division 1-7）
- **前端**：角色钱包/军团钱包双标签页，Division 选择器，流水分页

### 14. 书签/保存的地点（100% - 2026-03-19）
- **BookmarkController**：页面控制器
- **BookmarkDataController API 端点**：
  - `GET /api/dashboard/bookmarks` — 角色书签列表
  - `GET /api/dashboard/bookmarks/folders` — 书签文件夹
- **前端**：书签列表 + 文件夹分组

### 15. 联系人（100% - 2026-03-19）
- **ContactController**：页面控制器
- **ContactDataController API 端点**：
  - `GET /api/dashboard/contacts` — 角色联系人列表
- **前端**：联系人列表 + 好感度显示

### 16. 合同（100% - 2026-03-19）
- **ContractController**：页面控制器
- **ContractDataController API 端点**：
  - `GET /api/dashboard/contracts` — 角色合同列表
  - `GET /api/dashboard/contracts/{id}/items` — 合同物品详情
- **前端**：合同列表 + 物品详情模态框

### 17. 装配（100% - 2026-03-19）
- **FittingController**：页面控制器
- **FittingDataController API 端点**：
  - `GET /api/dashboard/fittings` — 角色装配列表
- **前端**：装配列表 + 详情展开

### 18. 击毁报告（100% - 2026-03-19）
- **CharacterKillmailController**：页面控制器
- **CharacterKillmailDataController API 端点**：
  - `GET /api/dashboard/killmails` — 角色 KM 列表
- **前端**：KM 列表 + 详情跳转

### 19. 通知/提醒（100% - 2026-03-19）
- **NotificationController**：页面控制器
- **NotificationDataController API 端点**：
  - `GET /api/dashboard/notifications` — 角色通知列表
- **前端**：通知列表 + 分类筛选

### 20. 声望（100% - 2026-03-19）
- **StandingController**：页面控制器
- **StandingDataController API 端点**：
  - `GET /api/dashboard/standings` — 角色声望列表
- **前端**：声望列表 + 势力/军团/个人分类

### 21. 旗舰导航功能（2026-03-17）
- **CapitalNavController**：公开访问页面控制器
- **CapitalNavApiController API 端点**：
  - `GET /api/capital-nav/autocomplete?q=关键词` — 星系中文名模糊搜索
  - `GET /api/capital-nav/distance?from=ID&to=ID` — 计算两星系距离（光年）
  - `GET /api/capital-nav/reachable?system_id=ID&ship_type=TYPE&jdc=0-5&fuel_eff=0-5&jf=0-5` — 一跳可达星系
  - `GET /api/capital-nav/route?from=ID&to=ID&ship_type=TYPE&jdc=0-5&fuel_eff=0-5&jf=0-5&algorithm=bfs|dijkstra` — 路线规划
- **CapitalNavigationService**：
  - 7 种舰船类型常量（战略货舰/长须鲸级/黑隐特勤舰/航母/无畏/超航/泰坦）
  - 基础跳跃距离配置（5-7.5 光年）
  - 技能修正计算（JDC +20%/级，燃料效率 -10%/级，JF -10%/级）
  - BFS 算法：纯跳跃最少跳数
  - Dijkstra 算法：星门+跳跃最少燃料（星门边代价 0，跳跃边代价为燃料）
- **SyncUniverseSystems artisan 命令**：从 ESI 同步 5432 个 K-space 星系到 eve_systems_full.json
- **前端三标签页面**：
  - 星系距离：选择起点终点，显示距离和坐标差
  - 一跳可达：选择舰船类型和技能等级，显示可达星系列表（支持前端过滤）
  - 路线规划：选择起点终点、舰船、技能、算法，显示路线表格（含跳跃/星门标识）
- **限制**：旗舰不能进入高安（>=0.5）和 Pochven（27 个星系硬编码）
- **导航栏**：📍 图标，所有用户可见

### 11. 首页 "Tus Esi System (Beta)"（100% - 2026-03-13）
- **视频背景**（第四次会话）：eve-esi-bg.webm (19MB) 全屏视频背景 + bg-black/55 遮罩层确保文字可读
- **可读性优化**（第四次会话）：卡片改用 bg-black/40 backdrop-blur-lg，标签文字透明度全面提升
- **备案信息**（2026-03-18）：页脚添加苏ICP备17024259号-9，链接到工信部网站
- 实时服务器状态：晨曦(Serenity)/曙光(Infinity)/欧服(Tranquility) 三卡片
  - 显示：在线状态灯、在线人数、启动时间、版本号
  - ServerStatusController 公开 API 代理三个 ESI 端点，5 分钟缓存
- 两个入口按钮：
  - "无授权使用" → 弹出"功能开发中"模态框
  - "授权使用" → 跳转 /auth/guide
- **页面跳转逻辑**（第四次会话）：
  - 首页不再自动跳转，已登录用户也能正常访问首页
  - AuthController::guide 检测已授权用户直接跳转 /dashboard

### 12. API 端点
- GET /api/public/server-status（公开，无需认证）
- GET /api/public/market/groups, search, regions, active-types, orders, history, types/{id}（公开市场 API）
- GET /api/dashboard/server-status
- GET /api/dashboard/skills, skill-queue
- GET /api/dashboard/skills/overview, queue, groups
- GET /api/dashboard/assets/locations, location/{id}, search
- GET /api/dashboard/character-info, character-location, character-online
- GET /api/market/character-orders, my-order-ids（认证市场 API）
- GET /api/killmails/autocomplete, advanced-search, search, pilot/{id}/kills, kill/{id}（KM API）

## 待开发功能

| 优先级 | 功能 | 状态 |
|--------|------|------|
| 高 | 资产估值（价格数据） | API 框架完成，缺价格数据 |
| 中 | 军团管理页面 | 无代码 |
| 低 | 数据可视化 | 无代码 |

## 代码优化

### 2026-03-20 重构

- **BasePageController 基础控制器**：新增抽象基类
  - 提供 `renderPage()` 方法，自动注入 `$user` 和 `$isLoggedIn`
  - 11 个简单页面控制器重构继承此基类
- **User 模型**：新增 `hasEveCharacter()` 辅助方法
- **Token 刷新中间件修复**：
  - 修复 `AutoRefreshEveToken` 静默失败 bug
  - 刷新失败时正确处理：过期则 401/重定向，未过期则警告日志
- **项目清理**：
  - 删除所有垃圾文件（.md 文档、.sh 脚本、测试文件）
  - 删除测试视图（test.blade.php、system-distance-test.blade.php）
  - 更新 .gitignore 规则
  - 删除导航栏"保存的地点"按钮

### 2026-03-17 重构

- **KillmailService 门面模式重构**：2600行 → 1066行门面 + 3个子服务
  - `Killmail/ProtobufCodec.php` - Protobuf 编解码逻辑
  - `Killmail/BetaKbApiClient.php` - Beta KB API 客户端（含 XSRF 处理）
  - `Killmail/KillmailSearchService.php` - 搜索聚合服务
- **AssetDataController 瘦身**：955行 → 353行 + `AssetDataService`
- **统一错误处理**：`EveApiException` + `ApiErrorHandler`
- **统一缓存键管理**：`CacheKeyService`（TTL 常量 + 缓存键方法）
- **角色数据并行加载**：`CharacterDataService`（Http::pool 10-20s→3-5s）
- **Token 刷新统一**：所有服务使用 `TokenRefreshService`
- **数据库索引优化**：corporation_id, alliance_id, token_expires_at
- **Dashboard/角色属性缓存**：5分钟 Redis 缓存

### 新增文件

- `app/Services/CacheKeyService.php`
- `app/Services/ApiErrorHandler.php`
- `app/Services/CharacterDataService.php`
- `app/Services/AssetDataService.php`
- `app/Services/Killmail/ProtobufCodec.php`
- `app/Services/Killmail/BetaKbApiClient.php`
- `app/Services/Killmail/KillmailSearchService.php`
- `app/Exceptions/EveApiException.php`
- `database/migrations/2026_03_17_000001_add_indexes_to_users_table.php`

## 已知问题
- 技能进度条的 SP 阈值是固定值，不精确（不同技能倍率不同）
- evedata.xlsx 中少量物品翻译可能不准确（如护盾增强器显示舰船属性）

## 关键文件
- 路由：routes/web.php
- 认证：app/Http/Controllers/AuthController.php
- 首页：resources/views/welcome.blade.php
- 布局系统：resources/views/layouts/app.blade.php, guest.blade.php, partials/navbar.blade.php
- 仪表盘：app/Http/Controllers/DashboardController.php + resources/views/dashboard.blade.php
- API 数据：app/Http/Controllers/Api/DashboardDataController.php
- 服务器状态 API：app/Http/Controllers/Api/ServerStatusController.php
- 技能 API：app/Http/Controllers/Api/SkillDataController.php
- 技能页面：app/Http/Controllers/SkillController.php + resources/views/skills/index.blade.php
- 资产：app/Http/Controllers/Api/AssetDataController.php + resources/views/assets/index.blade.php
- 市场：app/Http/Controllers/MarketController.php + app/Http/Controllers/Api/MarketDataController.php + app/Services/MarketService.php
- KM 查询：app/Http/Controllers/Api/KillmailController.php + app/Services/KillmailService.php
- 游客仪表盘：app/Http/Controllers/GuestDashboardController.php
- 数据服务：app/Services/EveDataService.php + app/Helpers/EveHelper.php
- 数据更新脚本：scripts/update_evedata.py
- 数据更新命令：app/Console/Commands/UpdateEveData.php
- 市场缓存命令：app/Console/Commands/CacheMarketGroups.php
- Token 刷新：app/Services/TokenRefreshService.php
- 用户模型：app/Models/User.php
- Token 中间件：app/Http/Middleware/AutoRefreshEveToken.php
- ESI 配置：config/esi.php
- 缓存配置：config/cache.php（文件驱动）

## HTTPS 配置（2026-03-18）

### 域名与证书
- **域名**: 51-eve.online
- **证书类型**: 通配符证书（*.51-eve.online）
- **颁发机构**: ZeroSSL（90天有效期）
- **验证方式**: DNS-01（阿里云 DNS API）

### 证书管理
- **工具**: acme.sh v3.1.3
- **证书目录**: `/etc/nginx/ssl/`
- **自动续期**: 每天 14:01 cron 检查
- **续期后命令**: `docker restart eve-esi-nginx`

### Nginx 配置更新
- **docker-compose.yml**: 添加 443 端口映射，挂载 `/etc/nginx/ssl`
- **default.conf**: 
  - HTTP 80 端口 301 重定向到 HTTPS
  - HTTPS 443 端口 TLSv1.2/1.3
  - ECDHE 加密套件
  - 安全头: X-Frame-Options, X-Content-Type-Options

### 访问地址
- **HTTPS**: https://51-eve.online
- **HTTP**: 自动跳转 HTTPS