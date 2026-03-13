# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-14**

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

### 7. 首页 "Tus Esi System (Beta)"（100% - 2026-03-13）
- **视频背景**（第四次会话）：eve-esi-bg.webm (19MB) 全屏视频背景 + bg-black/55 遮罩层确保文字可读
- **可读性优化**（第四次会话）：卡片改用 bg-black/40 backdrop-blur-lg，标签文字透明度全面提升
- 实时服务器状态：晨曦(Serenity)/曙光(Infinity)/欧服(Tranquility) 三卡片
  - 显示：在线状态灯、在线人数、启动时间、版本号
  - ServerStatusController 公开 API 代理三个 ESI 端点，5 分钟缓存
- 两个入口按钮：
  - "无授权使用" → 弹出"功能开发中"模态框
  - "授权使用" → 跳转 /auth/guide
- **页面跳转逻辑**（第四次会话）：
  - 首页不再自动跳转，已登录用户也能正常访问首页
  - AuthController::guide 检测已授权用户直接跳转 /dashboard

### 8. KM 查询页面（100% - 2026-03-14）
- **数据源**：beta.ceve-market.org REST API（protobuf 响应）
- **自定义 Protobuf 解码器**：
  - pbDecodeVarint / pbParseMessage / pbGetVarint / pbGetString / pbGetDouble
  - 解析 wire type 0 (varint), 1 (double), 2 (length-delimited), 5 (32-bit)
- **击杀列表**：
  - Beta KB `/app/list/pilot/{id}/kill` → 50 条记录/页
  - 字段映射：F1=kill_id, F2=victim, F8=ship, F9=time, F10=system, F13=ISK, F16=hash
  - 旧 KB HTML 解析作为备用
- **ESI Hash 提取**：
  - Beta KB `/app/kill/{id}/info` → 正则提取 40 字符十六进制 hash
  - Hash 缓存 86400s TTL
- **KM 详情**：ESI `/killmails/{id}/{hash}/` 获取完整击杀数据
- **前端**：
  - 击杀列表从后端 API 获取，显示 ISK 值（黄色）
  - ESI hash 预加载到 data-hash 属性，点击即时加载详情
  - 支持按角色名搜索（POST /universe/ids/ 获取 character_id）
  - 支持直接输入 Kill ID 查询

### 9. 导航统一（2026-03-14）
- 全部 6 个页面右上角统一 5 图标导航栏：🏠📚📦👥⚔️
- 当前页面图标 bg-white/10 高亮
- 涉及页面：dashboard, skills, assets, characters/index, characters/show, killmails

### 10. API 端点
- GET /api/public/server-status（公开，无需认证）
- GET /api/dashboard/server-status
- GET /api/dashboard/skills
- GET /api/dashboard/skill-queue
- GET /api/dashboard/skills/overview
- GET /api/dashboard/skills/queue
- GET /api/dashboard/skills/groups
- GET /api/dashboard/assets/locations
- GET /api/dashboard/assets/location/{locationId}
- GET /api/dashboard/assets/search?q=关键词
- GET /api/dashboard/character-info
- GET /api/dashboard/character-location
- GET /api/dashboard/character-online
- GET /api/killmails/search?character_name=xxx
- GET /api/killmails/{killId}/detail
- GET /api/killmails/{killId}/hash
- GET /api/killmails/pilot/{pilotId}/kills

## 待开发功能

| 优先级 | 功能 | 状态 |
|--------|------|------|
| 高 | 钱包查询页面 | 路由和 Controller 方法存在，缺视图 |
| 高 | 资产估值（价格数据） | API 框架完成，缺价格数据 |
| 中 | 无授权使用功能页面 | 首页入口已有，页面未开发 |
| 中 | 市场订单页面 | 无代码 |
| 中 | 军团管理页面 | 无代码 |
| 低 | 合同查询 | 无代码 |
| 低 | 数据可视化 | 无代码 |

## 已知问题
- 前端全部内联在 Blade 模板中，无组件化
- 技能进度条的 SP 阈值是固定值，不精确（不同技能倍率不同）
- evedata.xlsx 中少量物品翻译可能不准确（如护盾增强器显示舰船属性）

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
- KM 查询：app/Http/Controllers/Api/KillmailController.php + app/Services/KillmailService.php + resources/views/killmails/index.blade.php
- 角色：app/Http/Controllers/CharacterController.php + resources/views/characters/index.blade.php + show.blade.php
- 数据服务：app/Services/EveDataService.php + app/Helpers/EveHelper.php
- 数据更新脚本：scripts/update_evedata.py
- 数据更新命令：app/Console/Commands/UpdateEveData.php
- Token 刷新：app/Services/TokenRefreshService.php
- 用户模型：app/Models/User.php
- Token 中间件：app/Http/Middleware/AutoRefreshEveToken.php
- ESI 配置：config/esi.php
