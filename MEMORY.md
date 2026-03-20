# Qoder 长期记忆

**最后更新：2026-03-20**

## 项目负责人
- 名称：图斯（Tus）
- GitHub：TusZyc

## 活跃项目

### EVE ESI Tools (eve-esi-qoder)
- **仓库**: https://github.com/TusZyc/eve-esi-qoder
- **原始仓库**（OpenClaw 开发）：git@github.com:TusZyc/eve-esi.git
- **域名**: https://51-eve.online（2026-03-18 配置 HTTPS）
- **技术栈**: Laravel 10 + PHP 8.2 + Blade + Tailwind CSS + Alpine.js + Redis
- **部署**: Docker Compose → 阿里云 ECS (47.116.125.182)
- **状态**: 持续开发中，由 Qoder 接替 OpenClaw 继续开发
- **功能**: 21 个主要功能（SSO登录、仪表盘、技能、资产、钱包、书签、联系人、合同、装配、击毁报告、通知、声望、KM查询、市场、角色信息、旗舰导航等）
- **详细记忆**: 见 `projects/eve-esi.md`

## 服务器信息
- 阿里云 ECS：47.116.125.182
- 操作系统：Ubuntu
- SSH 用户：root
- SSH 密钥：d:\Qoder-work\.qoder\openclaw.pem
- 项目目录：/opt/eve-esi
- Docker 容器：eve-esi-app (php-fpm), eve-esi-nginx, eve-esi-redis
- SSH 命令：ssh -i "d:\Qoder-work\.qoder\openclaw.pem" root@47.116.125.182

## 开发环境
- 公司电脑：Windows + WSL，SSH 密钥 qoder_server.pem / ed25519
- 家里电脑：Windows + WSL，SSH 密钥 openclaw.pem / ed25519，2026-03-12 晚配置完成
- 本地工作目录：d:\Qoder-work\（仅部分文件，完整项目在服务器 /opt/eve-esi）
- 完整项目在服务器 /opt/eve-esi，本地通过 SCP 上传修改的文件

## 部署注意事项
- PHP 文件通过 SCP 上传到服务器 /tmp，再 docker cp 到容器内
- Docker 服务名是 `app`（不是 php），artisan 命令：docker compose exec -T app php artisan ...
- 部署后必须清理缓存：php artisan cache:clear / view:clear
- 历史问题：tarball 方式部署曾导致 PHP $ 变量符号丢失，已改用 SCP 直传
- git push 后服务器不会自动部署，需手动 SCP + docker cp

## EVE ESI API 经验

### 中文翻译陷阱
- `universe/stations/{id}/?datasource=serenity` 直接返回中文站名（无需 language 参数）
- `universe/stations/{id}/` 国服 Serenity 数据源直接返回完整中文，不需要逐词翻译
- `universe/names/` 批量端点不支持 language 参数
- `universe/systems/{id}/?language=zh` 支持中文（可获取中文星系名）
- 逐词 str_replace 翻译会导致空格问题，应直接调用 ESI 接口获取原生中文名

### 数据格式
- EVE 角色描述 `\xNN`：是 Unicode 码点 U+00NN，需 pack('H*','00'.$m[1]) → UCS-2BE → UTF-8
- EVE 颜色格式：ARGB 8位 (#AARRGGBB)，CSS 需 RGB 6位，需 substr($color,2) 去掉 alpha
- items.json 原本不含星系数据，已从 ESI 批量导入 8534 个星系中文名

## 偏好
- 使用中文交流
- 项目之前由 OpenClaw（AI 助手名"小图"）开发，2026-03-12 由 Qoder 接替继续开发
- 代码推送到 eve-esi-qoder 仓库，不动原始 eve-esi 仓库

## 代码优化重构（2026-03-17）

### 已完成优化

- **KillmailService 重构**: 2600行 → 门面模式(1066行) + 3子服务
  - `Killmail/ProtobufCodec.php` - Protobuf 编解码
  - `Killmail/BetaKbApiClient.php` - Beta KB API 客户端
  - `Killmail/KillmailSearchService.php` - 搜索聚合
- **AssetDataController 瘦身**: 955行 → 353行 + `AssetDataService`
- **统一错误处理**: `EveApiException` + `ApiErrorHandler`
- **统一缓存键管理**: `CacheKeyService` (TTL常量 + 缓存键方法)
- **角色数据并行**: `CharacterDataService` (Http::pool 10-20s→3-5s)
- **Token 刷新统一**: 所有服务使用 `TokenRefreshService`
- **数据库索引**: corporation_id, alliance_id, token_expires_at
- **Dashboard/角色属性缓存**: 5分钟 Redis缓存

### 缓存体系 (80+ 缓存键)

| 分类 | 前缀 | TTL |
|------|------|-----|
| 角色数据 | `char:*` | 5分钟 |
| 资产数据 | `assets_raw_*` | 15分钟-24小时 |
| KM数据 | `kb:*` | 5分钟-24小时 |
| 市场数据 | `market_*` | 5分钟-7天 |
| 导航数据 | `esi_*` | 24小时-7天 |

### 待完成

- ~~安全加固（HTTPS、Redis密码、Token加密等）~~ ✅ HTTPS 已完成（2026-03-18）
- CacheKeyService 采用率提升（53%）
- 缓存键命名风格统一

## HTTPS 配置（2026-03-18）

### 证书配置
- **域名**: 51-eve.online（通配符证书 *.51-eve.online）
- **证书颁发机构**: ZeroSSL
- **有效期**: 90 天（到期前自动续期）
- **验证方式**: DNS-01（阿里云 DNS API）
- **工具**: acme.sh v3.1.3

### 证书文件
- 证书目录: `/etc/nginx/ssl/`
- 证书文件: `51-eve.online.crt`（fullchain）
- 私钥文件: `51-eve.online.key`

### 自动续期
- Cron: `1 14 * * *` 每天 14:01 检查续期
- 续期后自动执行: `docker restart eve-esi-nginx`

### Nginx 配置
- HTTP (80): 301 重定向到 HTTPS
- HTTPS (443): TLSv1.2/1.3，ECDHE 加密套件
- 证书安装命令已配置 reloadcmd

### 阿里云 AccessKey（DNS 验证）
- 已配置在 acme.sh 配置文件中
- 位置: `/root/.acme.sh/account.conf`

### 注意事项
- Nginx 1.29.6 不支持 `immutable` 参数，已移除
- SSH 传输 Nginx 配置时变量会被本地 shell 展开，需用 base64 编码

## 旗舰导航功能（2026-03-17）
- 三标签页面：星系距离 / 一跳可达 / 路线规划
- 公开访问（无需认证），导航栏 📍 图标
- 舰船类型 7 种：战略货舰/长须鲸级/黑隐特勤舰/航母/无畏/超航/泰坦
- 技能修正：JDC +20%/级跳跃距离，燃料效率 -10%/级，JF -10%/级（仅战略货舰）
- 距离公式：sqrt((x2-x1)^2+(y2-y1)^2+(z2-z1)^2) / 9.461e15 光年
- 路线算法：BFS（纯跳跃最少跳数）/ Dijkstra（星门+跳跃最少燃料）
- 数据源：eve:sync-universe artisan 命令从 ESI 同步 5432 个 K-space 星系到 eve_systems_full.json (1.26MB)
- 限制：旗舰不能进入高安（>=0.5）和 Pochven（27 个星系硬编码）
- API 端点：/api/capital-nav/autocomplete, distance, reachable, route（均 throttle:30,1）

## EVE ESI 后台管理规划（2026-03-18）

### 功能规划
- **军团管理后台**：军团概况、成员管理、权限分级、资产统计、战报统计、导航配置
- **站点管理后台**：用户角色管理、SSO 授权监控、日志监控、配置管理

### 鉴权方案
- 不新建账号系统，在现有 EVE SSO + Cookie 登录基础上叠加权限层
- 极简版：config/admin.php 写死管理员角色ID + 中间件判断
- 完整版：users 表加 role 字段（user/corp_admin/site_admin）

### 详细设计
见 `ideas/admin-panel-design.md`

---

## KM 搜索技术细节

### Beta KB Search API
- 地址：https://beta.ceve-market.org/app/search/search (POST)
- 协议：gRPC-web 风格的 protobuf 编码
- XSRF 保护：需要 ReDive + GranblueFantasy cookies + FinalFantasy-XIV header
- XSRF cookies 从 beta.ceve-market.org 根路径 `/` 获取（比 `/search` 快 10 倍）
- Protobuf 字段：F1=Allis, F2=Corps, F3=Chars, F4=chartype, F5=types, F6=groups, F7=systems, F8=regions, F9=startdate, F10=enddate
- 重要限制：entity(F1-F3) 不能与 types(F5)/systems(F7) 组合使用，会返回空
- entity + chartype + daterange 可以组合
- chartype 映射：victim→lost, finalblow→win, attacker→atk

### Beta KB List API
- 地址：https://beta.ceve-market.org/app/list/{type}/{id}/{kill|loss}
- URL 路径映射：pilot→pilot, corporation→corp, alliance→alli
- 不需要 XSRF cookies，直接 GET
- 返回 protobuf 格式，需检测 HTML 响应 (ord($body[0]) === 0x3C)

### 建筑/Structure 处理
- ESI category 65 = Structures
- 常见建筑：Astrahus=35833, Fortizar=35834, Keepstar=35835
- 建筑搜索跳过 chartype 过滤（建筑 KM 一般都是被摧毁）
- isStructureTypeId() 基于 ESI category 65 的 group type_ids 判断

### beta.ceve-market.org 连接注意事项
- 该站点从阿里云 ECS Docker 容器访问不稳定，经常超时
- XSRF cookies 缓存 30 分钟，负缓存 60 秒避免阻塞
- 重试逻辑：2 次尝试，500ms 间隔
