# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-15（第七次会话 - 家里电脑）**

## 已完成功能

### 1. OAuth2 认证系统（100%）
- 3 步授权流程（清除缓存 → 授权 → 填写授权码）
- EVE 国服 OAuth2，Client ID: bc90aa496a404724a93f41b4f4e97761
- 3V 完整 73 个权限（含 esi-clones.read_clones.v1 / read_implants.v1）
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
- 全量技能显示：25 个分组 673 个技能，已学/未学区分（已过滤虚构技能分类）
- 训练中/等待中/已完成三种状态 + 进度条
- 60 秒自动刷新

### 4. 资产页面（100%）
- 两步加载架构：先位置列表，再按需加载物品
- API：/api/dashboard/assets/locations, location/{id}, search
- 搜索、舰船/物品机库分类、树形展示、位置标志中文映射
- 并发 HTTP 请求 + 三层缓存

### 5. 角色信息页面（100% - 2026-03-15）
- 四标签切换界面：角色描述 / 属性&植入体 / 克隆体 / 雇佣历史
- **角色描述**：完整 EVE 富文本解码（Python unicode 转义、HTML 实体、\xNN hex 序列→Unicode、ARGB 颜色→CSS RGB）
- **属性&植入体**：五大属性值 + 当前植入体列表，异步加载
- **克隆体**：跳跃克隆体位置与植入体，NPC 空间站逐段中文翻译（星系名 + Moon→卫星 + 军团名 + 设施类型映射）
- **雇佣历史**：军团历史含起止时间与天数统计（格式：start_date ~ end_date (N天)）
- API：/api/dashboard/character/attributes, implants, clones, corphistory
- CharacterController 含 resolveLocationName() + translateStationName() 逐段翻译

### 6. 本地数据服务（100% - 2026-03-15 更新）
- ceve-market.org evedata.xlsx → Python 脚本 → JSON
- items.json (~36,800条，含 8,534 个星系中文名从 ESI 批量导入)
- eve_station_systems.json (5,464条)
- EveDataService：本地优先，ESI API 兜底
- 星系中文名支持位置搜索模糊匹配

### 7. 安全改进（2026-03-12）
- TokenRefreshService 统一刷新、缓存 key 前缀、XSS 防护、角色 ID 验证

### 8. 市场功能（100% - 2026-03-14）
- **MarketController**：公开访问，支持游客/已登录两种模式
- **MarketDataController API**：市场分组树、分组详情、市场订单、价格历史、物品详情
- **MarketService**：ESI API 封装，分组树构建、订单/历史/物品查询
- **CacheMarketGroups artisan 命令**：预缓存 2141 个市场分组
- **前端**：分组树浏览 + 订单表 + 价格历史图表

### 9. 游客仪表盘（100% - 2026-03-14）
- **GuestDashboardController**：无需授权，GET /guest
- 三服务器状态卡片 + 授权提示 + 功能预览

### 10. 首页 "Tus Esi System (Beta)"（100% - 2026-03-13）
- 视频背景 eve-esi-bg.webm + 三服务器状态
- 授权使用/无授权使用入口

### 11. KM 查询页面（100% - 2026-03-15 更新）
- **数据源**：beta.ceve-market.org REST API（protobuf 响应）
- **自定义 Protobuf 解码器**：pbDecodeVarint / pbParseMessage 等
- **高级搜索**：角色/军团/联盟/舰船/星系多维度搜索
- **位置搜索**：中文星系名模糊匹配自动补全（本地 items.json + ESI universe/ids 兜底）
- **舰船搜索**：支持具体舰船和舰船类别自动补全（已去除多余"(类别)"后缀）
- **时间搜索**：精确到秒（datetime-local step=1），支持仅时间条件搜索
- **搜索条件**：允许仅位置、仅时间、或任意组合作为搜索条件
- **KM 详情弹窗**：参与者列表、损失装配、伤害统计（max-w-5xl 居中弹窗）

### 12. 统一导航栏（2026-03-14）
- 认证页面统一 5 图标 + 登出
- 当前页面 bg-white/10 高亮
- 市场页面保持独立导航

### 13. API 端点
- **公开**：/api/public/server-status, /api/public/market/*
- **认证仪表盘**：/api/dashboard/server-status, character-info, character-location, character-online, skills, skill-queue
- **认证技能**：/api/dashboard/skills/overview, queue, groups
- **认证资产**：/api/dashboard/assets/locations, location/{id}, search
- **认证角色**：/api/dashboard/character/attributes, implants, clones, corphistory
- **认证市场**：/api/market/character-orders, my-order-ids
- **KM 查询**：/api/killmails/autocomplete, advanced-search

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
- 角色信息：app/Http/Controllers/CharacterController.php + resources/views/characters/index.blade.php
- 市场：app/Http/Controllers/MarketController.php + Api/MarketDataController.php + MarketService.php
- KM 查询：app/Http/Controllers/Api/KillmailController.php + KillmailService.php + resources/views/killmails/index.blade.php
- 游客仪表盘：app/Http/Controllers/GuestDashboardController.php
- 市场缓存命令：app/Console/Commands/CacheMarketGroups.php
- 数据服务：app/Services/EveDataService.php
- 数据更新：scripts/update_evedata.py + app/Console/Commands/UpdateEveData.php
- Token 刷新：app/Services/TokenRefreshService.php
- ESI 配置：config/esi.php（含克隆体/植入体权限）
- 缓存配置：config/cache.php（文件驱动）
