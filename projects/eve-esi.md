# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-12**

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
- 服务端渲染，技能中文名（本地 items.json + API 兜底）
- 训练中/等待中/已完成三种状态 + 进度条
- 已学技能列表（按分组展示）
- 60 秒自动刷新

### 4. 资产页面（100% - 2026-03-12 完全重写）
- **两步加载架构**：先加载位置列表（快速），再按需加载每个位置的物品详情
- **API 端点**：
  - `GET /api/dashboard/assets/locations` — 返回位置列表+物品数量
  - `GET /api/dashboard/assets/location/{locationId}` — 返回某位置的物品树
- **三层缓存**：
  - 原始资产数据 15 分钟（assets_raw_{characterId}）
  - 每个位置物品树 15 分钟（assets_loc_{characterId}_{locationId}）
  - 类型详情/分组名称/位置名称各 24 小时
- **并发 HTTP 请求**：Http::pool() 每批 50 个并发查询类型详情和分组名称
- **中文分组名称**：ESI API 调用添加 language=zh 和 datasource=serenity
- **前端懒加载**：位置默认折叠，点击展开时加载物品，后台自动预加载其他位置
- **树形展示**：通过 item_id/location_id 父子关系构建，支持展开/折叠
- **搜索**：支持按物品名、分组名搜索，自动展开匹配路径
- **位置标志中文映射**：机库/货柜仓/无人机仓/高槽/中槽/低槽/改装件等

### 5. 数据服务（100%）
- EveDataService：本地 items.json（28,294 条） + ESI API 兜底
- EveHelper：静态门面
- 缓存 24 小时

### 6. 安全改进（2026-03-12）
- TokenRefreshService 统一 Token 刷新（消除三处重复）
- 缓存 key 添加 user_id 前缀（防止碰撞）
- 前端用 textContent/createElement 替代 innerHTML（防 XSS）
- Controller 添加角色 ID 归属验证

### 7. API 端点
- GET /api/dashboard/server-status
- GET /api/dashboard/skills
- GET /api/dashboard/skill-queue
- GET /api/dashboard/assets/locations（新）
- GET /api/dashboard/assets/location/{locationId}（新）
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
- 物品名称依赖本地 items.json + ESI universe/names/，部分新物品可能缺失

## 关键文件
- 路由：routes/web.php
- 认证：app/Http/Controllers/AuthController.php
- 仪表盘：app/Http/Controllers/DashboardController.php + resources/views/dashboard.blade.php
- API 数据：app/Http/Controllers/Api/DashboardDataController.php
- 技能：app/Http/Controllers/SkillController.php + resources/views/skills/index.blade.php
- 资产：app/Http/Controllers/Api/AssetDataController.php + resources/views/assets/index.blade.php
- 数据服务：app/Services/EveDataService.php + app/Helpers/EveHelper.php
- Token 刷新：app/Services/TokenRefreshService.php
- 用户模型：app/Models/User.php
- Token 中间件：app/Http/Middleware/AutoRefreshEveToken.php
- ESI 配置：config/esi.php
