# EVE ESI Tools - 项目记忆

**最后更新：2026-03-17（代码优化重构后）**

## 项目概述

- **名称**: EVE ESI Tools (eve-esi-qoder)
- **仓库**: https://github.com/TusZyc/eve-esi-qoder
- **服务器**: 阿里云 ECS 47.116.125.182, /opt/eve-esi
- **技术栈**: Laravel 10 + PHP 8.2 + Blade + Tailwind CSS + Alpine.js + Redis
- **部署**: Docker Compose (eve-esi-app, eve-esi-nginx, eve-esi-redis)

## 目录结构

```
eve-esi-qoder/
├── app/
│   ├── Console/Commands/           # Artisan 命令
│   │   ├── CacheMarketGroups.php   # 市场分组缓存
│   │   ├── UpdateEveData.php       # EVE 数据更新
│   │   ├── SyncUniverseSystems.php # 星系数据同步
│   │   └── TestEsiConnection.php
│   ├── Http/Controllers/
│   │   ├── Api/                    # 异步数据 API
│   │   │   ├── AssetDataController.php    # 资产数据 (353行，瘦身自955行)
│   │   │   ├── DashboardDataController.php
│   │   │   ├── KillmailController.php     # KM API
│   │   │   ├── MarketDataController.php   # 市场 API
│   │   │   ├── CapitalNavApiController.php # 旗舰导航 API
│   │   │   └── SkillDataController.php
│   │   ├── AuthController.php      # OAuth2 认证
│   │   ├── DashboardController.php
│   │   ├── SkillController.php
│   │   ├── AssetController.php
│   │   ├── MarketController.php
│   │   ├── CharacterController.php
│   │   ├── CapitalNavController.php
│   │   └── GuestDashboardController.php
│   ├── Services/
│   │   ├── EveEsiService.php       # ESI API 基础
│   │   ├── EveDataService.php      # 本地数据服务
│   │   ├── KillmailService.php     # KM 门面 (1066行，拆分自2600行)
│   │   ├── Killmail/               # KM 子服务
│   │   │   ├── ProtobufCodec.php   # Protobuf 编解码
│   │   │   ├── BetaKbApiClient.php # Beta KB API 客户端
│   │   │   └── KillmailSearchService.php # 搜索聚合
│   │   ├── MarketService.php
│   │   ├── SkillService.php
│   │   ├── AssetService.php
│   │   ├── AssetDataService.php    # 资产数据业务逻辑
│   │   ├── CapitalNavigationService.php
│   │   ├── CharacterDataService.php # 角色数据并行获取
│   │   ├── CacheKeyService.php     # 统一缓存键管理
│   │   ├── ApiErrorHandler.php     # 统一错误处理
│   │   ├── TokenRefreshService.php # Token 刷新统一服务
│   │   └── SystemDistanceService.php
│   ├── Exceptions/
│   │   └── EveApiException.php     # EVE API 统一异常
│   ├── Middleware/
│   │   └── AutoRefreshEveToken.php # Token 自动刷新
│   └── Helpers/EveHelper.php
├── config/
│   ├── esi.php                     # ESI 配置 + OAuth 权限
│   └── market.php
├── data/
│   ├── items.json                  # 物品名称
│   ├── eve_names.json              # ID→名称映射
│   ├── eve_station_systems.json    # 站点→星系
│   ├── eve_systems_full.json       # 星系坐标 (1.26MB)
│   └── solar_system_jumps.json
├── resources/views/
│   ├── layouts/
│   │   ├── app.blade.php           # 认证用户布局
│   │   ├── guest.blade.php         # 游客布局
│   │   └── partials/navbar.blade.php
│   ├── dashboard.blade.php
│   ├── guest-dashboard.blade.php
│   ├── skills/index.blade.php
│   ├── assets/index.blade.php
│   ├── characters/index.blade.php
│   ├── killmails/index.blade.php
│   ├── market/index.blade.php
│   └── capital-nav/index.blade.php
├── routes/
│   ├── web.php
│   └── api.php
├── scripts/update_evedata.py       # 数据更新脚本
└── docker-compose.yml
```

## 功能与关键文件映射

| 功能 | Controller | Service | View |
|------|------------|---------|------|
| OAuth2 登录 | AuthController | - | auth/guide.blade.php |
| 仪表盘 | DashboardController | - | dashboard.blade.php |
| 技能队列 | SkillController, Api/SkillDataController | SkillService | skills/index.blade.php |
| 资产管理 | AssetController, Api/AssetDataController | AssetService | assets/index.blade.php |
| 击杀报告 | Api/KillmailController | KillmailService | killmails/index.blade.php |
| 市场工具 | MarketController, Api/MarketDataController | MarketService | market/index.blade.php |
| 角色信息 | CharacterController | - | characters/index.blade.php |
| 旗舰导航 | CapitalNavController, Api/CapitalNavApiController | CapitalNavigationService | capital-nav/index.blade.php |

## 常用命令

```bash
# 部署相关
docker compose up -d
docker compose exec app php artisan cache:clear
docker compose exec app php artisan view:clear
docker compose exec app php artisan migrate

# 数据更新
docker compose exec app php artisan eve:update-data      # 更新 EVE 数据
docker compose exec app php artisan eve:sync-universe    # 同步星系坐标
docker compose exec app php artisan cache:market-groups  # 缓存市场分组

# SCP 部署单文件
scp -i ~/.ssh/openclaw.pem file.php root@47.116.125.182:/tmp/
ssh -i ~/.ssh/openclaw.pem root@47.116.125.182 "docker cp /tmp/file.php eve-esi-app:/var/www/html/path/"
```

## API 端点

### 公开 API（无需认证）

- `GET /api/public/server-status` - 三服务器状态
- `GET /api/public/market/*` - 市场数据 (groups, search, regions, orders, history)
- `GET /api/capital-nav/*` - 旗舰导航 (autocomplete, distance, reachable, route)
- `GET /api/killmails/*` - KM 查询 (autocomplete, advanced-search, search)

### 认证 API（/api/dashboard/）

- `/server-status` - 服务器状态
- `/character-info`, `/character-location`, `/character-online` - 角色信息
- `/skills/overview`, `/skills/queue`, `/skills/groups` - 技能
- `/assets/locations`, `/assets/location/{id}`, `/assets/search` - 资产

## 代码优化记录（2026-03-17）

### 已完成优化

- [x] **KillmailService 重构**: 2600行 → 门面模式 + 3子服务
  - `KillmailService.php` - 门面（1066行，保留原有接口）
  - `Killmail/ProtobufCodec.php` - Protobuf 编解码
  - `Killmail/BetaKbApiClient.php` - Beta KB API 客户端
  - `Killmail/KillmailSearchService.php` - 搜索聚合服务
- [x] **AssetDataController 瘦身**: 955行 → 353行，业务逻辑提取到 `AssetDataService`
- [x] **统一错误处理**: `EveApiException` + `ApiErrorHandler`
- [x] **统一缓存键管理**: `CacheKeyService`（TTL常量 + 缓存键方法）
- [x] **角色数据并行**: `CharacterDataService`（Http::pool 并行获取4个API，10-20s→3-5s）
- [x] **Token 刷新统一**: 所有服务使用 `TokenRefreshService`
- [x] **数据库索引**: corporation_id, alliance_id, token_expires_at
- [x] **Dashboard 缓存**: 服务器状态 5 分钟 Redis 缓存
- [x] **角色属性/植入体缓存**: 5 分钟缓存

### 待完成优化

- [ ] **安全加固**（待正式上线）: HTTPS、Redis密码、Token加密、OAuth state验证、Session加密
- [ ] **CacheKeyService 采用率**: 目前 53%，部分服务仍硬编码缓存键
- [ ] **缓存键命名风格统一**: 新 `char:` 风格 vs 旧 `assets_raw_` 风格混用
- [ ] 缺少数据转换层 (DTO)

### 待开发功能

- [ ] 钱包查询页面
- [ ] 资产估值功能
- [ ] 军团管理页面

## 缓存体系概览（80+ 缓存键）

| 分类 | 前缀/模式 | TTL |
|------|-----------|-----|
| 角色数据 | `char:*` | 5 分钟 |
| 资产数据 | `assets_raw_*`, `eve_locinfo_*`, `eve_sysname_*` | 15分钟/24小时 |
| KM 数据 | `kb:*` | 5分钟-24小时 |
| 市场数据 | `market_*` | 5分钟-7天 |
| 导航数据 | `esi_*` | 24小时-7天 |
| 基础数据 | `eve_names_database`, `eve_station_system_map` | 2小时 |

## EVE ESI API 注意事项

### 中文翻译陷阱

- `universe/stations/{id}/` **不支持** `language=zh`，永远返回英文
- `universe/names/` 批量端点也不支持 language 参数
- `universe/systems/{id}/?language=zh` 支持中文
- NPC 空间站中文名需逐段翻译：星系名 + "Moon"→"卫星" + 军团名 + 设施类型映射

### 数据格式

- 角色描述 `\xNN`：Unicode 码点 U+00NN，需特殊解码
- EVE 颜色格式：ARGB (#AARRGGBB)，CSS 需去掉 alpha 变为 RGB

## 旗舰导航功能细节

- 7 种舰船类型：战略货舰/长须鲸/黑隐特勤舰/航母/无畏/超航/泰坦
- 技能修正：JDC +20%/级距离，燃料效率 -10%/级
- 限制：不能进入高安 (>=0.5) 和 Pochven (27 个星系)
- 数据源：eve_systems_full.json (5432 个 K-space 星系)

## KM 搜索技术细节

### Beta KB API

- 地址：`https://beta.ceve-market.org/app/search/search` (POST)
- 协议：gRPC-web 风格 protobuf
- XSRF：需要 ReDive + GranblueFantasy cookies + FinalFantasy-XIV header
- 重要限制：entity(角色/军团/联盟) 不能与 types/systems 组合使用

### 建筑处理

- ESI category 65 = Structures
- 常见建筑：Astrahus=35833, Fortizar=35834, Keepstar=35835
