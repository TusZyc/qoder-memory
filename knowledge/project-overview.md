# EVE ESI 项目概览

**最后更新**: 2026-03-24

---

## 项目基本信息

| 项目 | 信息 |
|------|------|
| **名称** | EVE ESI Tools (eve-esi-qoder) |
| **域名** | https://51-eve.online |
| **仓库** | https://github.com/TusZyc/eve-esi-qoder |
| **技术栈** | Laravel 10 + PHP 8.2 + Blade + Tailwind CSS + Alpine.js + Redis |
| **部署** | Docker Compose → 阿里云 ECS (47.116.125.182) |
| **状态** | 持续开发中 |

---

## 功能清单

### 公开功能（无需登录）
| 功能 | 路由 | 说明 |
|------|------|------|
| 首页 | `/` | 视频背景 + 三服务器状态 |
| 游客仪表盘 | `/guest` | 功能预览 |
| 市场查询 | `/market` | 订单查询、价格历史 |
| KM 查询 | `/killmails` | 击杀报告搜索 |
| 旗舰导航 | `/capital-nav` | 星系距离、跳跃路线 |
| LP Store | `/lp-store` | 忠诚点利润计算 |
| 斥候工具 | `/scout` | 扫描数据解析+分享 |
| 使用指南 | `/guide` | 功能介绍 |

### 认证功能（需要 EVE SSO 登录）
| 功能 | 路由 | 说明 |
|------|------|------|
| 仪表盘 | `/dashboard` | 角色信息、技能概览 |
| 技能队列 | `/skills` | 全部 673 技能、25 分组 |
| 资产管理 | `/assets` | 树形展示、搜索、分类 |
| 钱包 | `/wallet` | 余额、流水、军团钱包 |
| 邮件 | `/mail` | 邮件读取、标签筛选 |
| 书签 | `/bookmarks` | 书签列表、文件夹 |
| 联系人 | `/contacts` | 联系人、好感度 |
| 合同 | `/contracts` | 合同列表、物品详情 |
| 装配 | `/fittings` | 装配方案列表 |
| 击毁报告 | `/character-killmails` | 个人 KM |
| 通知 | `/notifications` | 游戏通知 |
| 声望 | `/standings` | 势力/军团/个人声望 |
| 角色信息 | `/characters` | 属性、植入体、克隆体 |

### 管理功能
| 功能 | 路由 | 说明 |
|------|------|------|
| 后台管理 | `/admin` | 用户管理、日志、缓存、数据 |
| 舰队管理 | `/fleet` | 开发中 |

---

## 缓存体系

| 分类 | 前缀 | TTL |
|------|------|-----|
| 角色数据 | `char:*` | 5 分钟 |
| 资产数据 | `assets_raw_*` | 15 分钟 - 24 小时 |
| KM 数据 | `kb:*` | 5 分钟 - 24 小时 |
| 市场数据 | `market_*` | 5 分钟 - 7 天 |
| 导航数据 | `esi_*` | 24 小时 - 7 天 |
| 位置名称 | `eve_locinfo_*` | 24 小时 |

---

## 关键文件路径

### 路由与控制器
```
routes/web.php                              # Web 路由
routes/api.php                              # API 路由
app/Http/Controllers/AuthController.php     # 认证控制器
app/Http/Controllers/DashboardController.php
app/Http/Controllers/MarketController.php
app/Http/Controllers/Api/                   # API 控制器目录
```

### 服务层
```
app/Services/MarketService.php              # 市场服务
app/Services/KillmailService.php            # KM 服务（门面）
app/Services/Killmail/                      # KM 子服务
  ├── ProtobufCodec.php                     # Protobuf 编解码
  ├── BetaKbApiClient.php                   # Beta KB API 客户端
  └── KillmailSearchService.php             # 搜索聚合
app/Services/TokenRefreshService.php        # Token 刷新
app/Services/EveDataService.php             # 静态数据服务
app/Services/CacheKeyService.php            # 缓存键管理
app/Services/CapitalNavigationService.php   # 旗舰导航
app/Services/LpStoreService.php             # LP Store
app/Services/ScoutService.php               # 斥候工具
```

### 视图
```
resources/views/layouts/
  ├── app.blade.php                         # 认证页面主布局
  ├── guest.blade.php                       # 游客页面布局
  └── partials/navbar.blade.php             # 共享导航栏
resources/views/dashboard.blade.php
resources/views/market/index.blade.php
resources/views/skills/index.blade.php
resources/views/assets/index.blade.php
```

### 数据文件
```
data/eve_items.json                         # 物品 ID→中文名（206987条）
data/eve_systems.json                       # 星系中文名
data/eve_stations.json                      # NPC 空间站
data/eve_regions.json                       # 星域
data/eve_constellations.json                # 星座
data/eve_structures.json                    # 玩家建筑
data/eve_systems_full.json                  # 完整星系数据（含坐标）
public/data/market_groups.json              # 市场分组（静态）
```

### 配置
```
config/esi.php                              # ESI API 配置
config/admin.php                            # 管理员配置
config/cache.php                            # 缓存配置
```

---

## API 端点

### 公开 API
```
GET /api/public/server-status               # 服务器状态
GET /api/public/market/groups               # 市场分组
GET /api/public/market/search               # 物品搜索
GET /api/public/market/orders               # 市场订单
GET /api/public/market/history              # 价格历史
GET /api/capital-nav/autocomplete           # 星系搜索
GET /api/capital-nav/distance               # 距离计算
GET /api/capital-nav/reachable              # 一跳可达
GET /api/capital-nav/route                  # 路线规划
GET /api/public/lp-store/factions           # LP 势力
GET /api/public/lp-store/offers             # LP 报价
```

### 认证 API
```
GET /api/dashboard/skills/overview          # 技能概览
GET /api/dashboard/skills/queue             # 技能队列
GET /api/dashboard/skills/groups            # 技能分组
GET /api/dashboard/assets/locations         # 资产位置
GET /api/dashboard/assets/location/{id}     # 位置物品
GET /api/dashboard/wallet/balance           # 钱包余额
GET /api/dashboard/wallet/journal           # 钱包流水
GET /api/mail                               # 邮件列表
GET /api/mail/{id}                          # 邮件详情
GET /api/killmails/search                   # KM 搜索
GET /api/killmails/pilot/{id}/kills         # 角色 KM
```

---

## 代码优化记录

### 2026-03-17 重构
- **KillmailService**：2600行 → 1066行门面 + 3个子服务
- **AssetDataController**：955行 → 353行 + AssetDataService
- **统一错误处理**：EveApiException + ApiErrorHandler
- **统一缓存键管理**：CacheKeyService
- **角色数据并行加载**：CharacterDataService（Http::pool 10-20s→3-5s）

### 2026-03-20 重构
- **BasePageController**：11 个简单页面控制器继承基类
- **Token 刷新中间件修复**：修复 AutoRefreshEveToken 静默失败

---

## HTTPS 配置

| 项目 | 值 |
|------|-----|
| 域名 | 51-eve.online |
| 证书类型 | 通配符证书（*.51-eve.online） |
| 颁发机构 | ZeroSSL（90天有效期） |
| 验证方式 | DNS-01（阿里云 DNS API） |
| 自动续期 | 每天 14:01 cron 检查 |
| 证书目录 | /etc/nginx/ssl/ |

---

## 待开发功能

- [ ] 军团管理后台
- [ ] 资产估值系统
- [ ] 数据可视化图表
- [ ] 多军团支持
