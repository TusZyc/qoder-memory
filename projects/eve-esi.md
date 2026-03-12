# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-13**

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
- 服务端渲染，技能中文名（本地数据 + API 兜底）
- 训练中/等待中/已完成三种状态 + 进度条
- 已学技能列表（按分组展示）
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

### 7. API 端点
- GET /api/dashboard/server-status
- GET /api/dashboard/skills
- GET /api/dashboard/skill-queue
- GET /api/dashboard/assets/locations
- GET /api/dashboard/assets/location/{locationId}
- GET /api/dashboard/assets/search?q=关键词
- GET /api/dashboard/character-info
- GET /api/dashboard/character-location
- GET /api/dashboard/character-online

## 待开发功能

| 优先级 | 功能 | 状态 |
|--------|------|------|
| 高 | 钱包查询页面 | 路由和 Controller 方法存在，缺视图 |
| 高 | 资产估值（价格数据） | API 框架完成，缺价格数据 |
| 中 | 市场订单页面 | 无代码 |
| 中 | 军团管理页面 | 无代码 |
| 低 | 技能按游戏内分类展示 | 骨架存在，分组逻辑未实现 |
| 低 | 击杀记录 | 无代码 |
| 低 | 合同查询 | 无代码 |
| 低 | 数据可视化 | 无代码 |

## 已知问题
- 前端全部内联在 Blade 模板中，无组件化
- 技能进度条的 SP 阈值是固定值，不精确（不同技能倍率不同）
- characters/index.blade.php 视图代码缺失
- evedata.xlsx 中少量物品翻译可能不准确（如护盾增强器显示舰船属性）

## 关键文件
- 路由：routes/web.php
- 认证：app/Http/Controllers/AuthController.php
- 仪表盘：app/Http/Controllers/DashboardController.php + resources/views/dashboard.blade.php
- API 数据：app/Http/Controllers/Api/DashboardDataController.php
- 技能：app/Http/Controllers/SkillController.php + resources/views/skills/index.blade.php
- 资产：app/Http/Controllers/Api/AssetDataController.php + resources/views/assets/index.blade.php
- 数据服务：app/Services/EveDataService.php + app/Helpers/EveHelper.php
- 数据更新脚本：scripts/update_evedata.py
- 数据更新命令：app/Console/Commands/UpdateEveData.php
- Token 刷新：app/Services/TokenRefreshService.php
- 用户模型：app/Models/User.php
- Token 中间件：app/Http/Middleware/AutoRefreshEveToken.php
- ESI 配置：config/esi.php
