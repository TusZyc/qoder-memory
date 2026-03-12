# EVE ESI 管理网站 - 项目状态

**最后更新：2026-03-12**

## 已完成功能

### 1. OAuth2 认证系统（100%）
- 3 步授权流程（清除缓存 → 授权 → 填写授权码）
- EVE 国服 OAuth2，Client ID: bc90aa496a404724a93f41b4f4e97761
- 3V 完整 73 个权限
- Refresh Token 自动刷新（提前 5 分钟，通过中间件 AutoRefreshEveToken）

### 2. Dashboard 仪表盘（100%）
- 异步数据加载 + 骨架屏
- 服务器状态卡片（状态指示灯 + 在线玩家 + 版本 + 启动时间 + VIP 模式）
- 角色信息卡片（角色名 + 军团 + 联盟 + 位置 + 在线状态）
- 技能信息卡片（总 SP + 未分配 SP + 已学技能数）
- 导航栏紧凑布局（图标导航）

### 3. 技能队列页面（100%）
- 服务端渲染，技能中文名（本地 items.json + API 兜底）
- 训练中/等待中/已完成三种状态 + 进度条
- 已学技能列表（分组展示骨架存在，分组逻辑暂未实现）
- 60 秒自动刷新

### 4. 资产页面（阶段 1 完成）
- 异步加载资产列表
- 物品名称/位置名称查询
- 搜索过滤
- 缺失：总价值计算、分页

### 5. 数据服务（100%）
- EveDataService：本地 items.json（28,294 条）+ ESI API 兜底
- EveHelper：静态门面
- 缓存 24 小时

### 6. API 端点（100%）
- GET /api/dashboard/server-status
- GET /api/dashboard/skills
- GET /api/dashboard/skill-queue
- GET /api/dashboard/assets
- GET /api/dashboard/character-info
- GET /api/dashboard/character-location
- GET /api/dashboard/character-online

## 待开发功能

| 优先级 | 功能 | 状态 |
|--------|------|------|
| 高 | 钱包查询页面 | 路由和 Controller 方法存在，缺视图 |
| 高 | 资产估值 | API 框架完成，缺价格数据 |
| 中 | 市场订单页面 | 无代码 |
| 中 | 军团管理页面 | 无代码 |
| 低 | 技能按游戏内分类展示 | 骨架存在，分组逻辑未实现 |
| 低 | 击杀记录 | 无代码 |
| 低 | 合同查询 | 无代码 |

## 已知问题
- Token 刷新逻辑在中间件、DashboardController、SkillController 三处重复
- characters/index.blade.php 视图代码缺失
- 前端全部内联在 Blade 模板中，无组件化
- 技能进度条 SP 阈值固定，不精确（不同技能倍率不同）

## 关键文件
- 路由：routes/web.php
- 认证：app/Http/Controllers/AuthController.php
- 仪表盘：app/Http/Controllers/DashboardController.php
- 仪表盘视图：resources/views/dashboard.blade.php
- API 数据：app/Http/Controllers/Api/DashboardDataController.php
- 技能：app/Http/Controllers/SkillController.php
- 技能视图：resources/views/skills/index.blade.php
- 资产 API：app/Http/Controllers/Api/AssetDataController.php
- 资产视图：resources/views/assets/index.blade.php
- 数据服务：app/Services/EveDataService.php
- 辅助类：app/Helpers/EveHelper.php
- 用户模型：app/Models/User.php
- Token 中间件：app/Http/Middleware/AutoRefreshEveToken.php
- ESI 配置：config/esi.php
