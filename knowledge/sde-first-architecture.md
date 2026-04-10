# SDE-First 静态数据架构改造方案

> **状态**: 设计完成，待分阶段实施
> **创建日期**: 2026-04-10
> **作者**: [Claude Code]
> **范围**: 整站静态数据从 ESI 调用反向迁移到本地 SDE 子集
> **预计周期**: 长期，分 4 个 Phase 推进，慢工出细活

---

## 0. 文档定位

本文档是 `sde-data-architecture.md`（[Qoder] 2026-04-09）的**扩展和升级版**：

| Qoder 原版 | 本版本 |
|---|---|
| 聚焦"为新功能（装配/工业）建子集" | 聚焦"把已有 26 个 ESI 静态数据调用迁回本地" |
| 多个独立子集（fitting/industry/...） | 推荐**单一 `static.sqlite`** + 保留 fitting.sqlite 独立 |
| 命令行驱动 | **管理后台驱动** + 命令行兜底 |
| 无 ESI 兜底机制 | **未命中自动落 missing_log + 引导补数据** |
| 无国服差异处理 | **`sde_overrides` 表 + admin 录入界面** |
| 单次设计 | **Phase 0~4 灰度迁移**，每步可回滚 |

> Qoder 原版的子集表设计仍然有效，本文档**复用**它的命令名、表结构、属性 ID 清单。

---

## 1. 背景与问题

### 1.1 现状盘点（量化）

截至 2026-04-10，对 `D:\Qoder-work\eve-esi-qoder\app\` 全量 grep `universe/(types|groups|categories|systems|stations|regions|constellations|races|factions)`：

| 项目 | 数量 |
|---|---|
| 调用 universe/* 端点的源文件 | **26 个** |
| 总调用次数 | **61 处** |
| 已有本地汉化文件 (`data/eve_items.json`) | **20.7 万条** typeID 中文名 + 中文分类 |
| 已有本地静态文件 (`data/eve_*.json`) | 6 个（systems / stations / regions / constellations / structures / items） |
| 已有 SDE 子集先例 (`fitting.sqlite`) | 15 MB / 7174 类型 / 159k 属性 |
| 已有静态优化先例 (`market_groups.json`) | 由 commit `31e9fab` 引入，避免清缓存后市场搜索 500 |

**结论**：项目早已沿"静态化"方向走，但**碎片化** —— 6 个 JSON + 1 个 fitting.sqlite + 各种内联缓存，没有统一架构。

### 1.2 当前 ESI 静态数据调用的痛点

| 痛点 | 来源 |
|---|---|
| **响应慢** | KM 详情解析时几十个 typeID 走 N+1 ESI，累计 3-10 秒 |
| **国服 ESI 不稳** | `pitfalls.md` 已记录多次连接异常、5xx |
| **token 闭包陷阱** | 用户路径上调 ESI 触发 `Cache::remember` 闭包陷阱（pitfalls.md 4-02） |
| **并发刷新陷阱** | 多请求同时刷新 token（pitfalls.md 4-02） |
| **datasource 漏参** | `datasource=serenity` 漏参得欧服数据（pitfalls.md 4-02） |
| **stations 中文 6 步法** | 国服 ESI `/universe/stations` 不支持 `language=zh`，需要 6 步翻译（pitfalls.md） |
| **缓存清空连锁反应** | redis 清空后市场搜索 30s 超时 → 已临时用静态文件兜底 |
| **error rate limit 风险** | ESI 错误率达阈值会被限流，影响所有功能 |

> **绝大部分痛点在 SDE-First 改造后自然消失**，因为根本不调 ESI。

### 1.3 为什么现在做

1. 已有 fitting.sqlite 跑通了 SDE 提取链路，**工具链就位**
2. 已有 eve_items.json 提供高质量汉化覆盖，**翻译数据就位**
3. 装配模拟器要继续推进，**没有 SDE 寸步难行**
4. 用户体验问题（KM 慢、市场搜索断）**已经在生产环境暴露**

---

## 2. 目标与非目标

### 2.1 目标
- **G1**：把 26 个文件中**绝大多数静态数据 ESI 调用**改为查本地 `static.sqlite`
- **G2**：建立**单一入口** `StaticDataService`，所有静态数据查询走它
- **G3**：建立**管理后台** `/admin/sde`，管理员可一键更新 SDE / 校正国服差异 / 查看 ESI 兜底命中
- **G4**：保留 **ESI 兜底路径**，新物品/缺失数据不会返回 unknown
- **G5**：彻底解决 token 闭包陷阱、并发陷阱、datasource 漏参等 ESI 调用根源问题
- **G6**：为代理人查询、行星开发、装配模拟器进阶等**新功能预留数据基础**

### 2.2 非目标
- **不替换动态数据 ESI 调用**：角色信息、市场价格、邮件、合同、钱包、立场、击杀邮件本身、主权占领等
- **不重写 fitting.sqlite**：装配模拟器子集已存在且独立，保持现状
- **不一次性迁移所有 26 个文件**：分 Phase 灰度，每个 Phase 都可独立回滚
- **不做欧服数据校验**：本项目只服务国服，欧服数据仅作为初始来源

---

## 3. 数据层架构

### 3.1 文件布局

```
storage/sde/
├── sqlite-latest.sqlite             # 全量原始 SDE (~350 MB, .gitignore)
├── sqlite-latest.sqlite.bz2         # 下载缓存 (.gitignore)
└── last_update.txt                  # 最近一次成功提取的 SDE 版本号

database/
├── database.sqlite                  # 用户数据（订单、缓存、设置等）
├── fitting.sqlite                   # 装配模拟器子集（保留独立，~15 MB）
└── static.sqlite                    # ★ 本次新增：应用静态数据子集 (~30 MB)

data/
├── eve_items.json                   # 中文翻译来源（保留，作为 import 源之一）
├── eve_systems.json                 # 同上
├── eve_stations.json                # 同上
└── ... (其他保留作为兜底)
```

**关键决策**：
- **single static.sqlite** 而非多子集 —— 简化数据库连接配置，避免跨库 join，admin 一处管理
- **fitting.sqlite 保留独立** —— 表结构特殊（dogma 属性多），且已存在
- **data/*.json 保留不删** —— 在 Phase 4 之前作为 fallback；Phase 4 改为从 static.sqlite 反向导出（保持外部依赖兼容）

### 3.2 `static.sqlite` 表结构

```sql
-- ============================
-- 元信息
-- ============================
CREATE TABLE sde_meta (
    key TEXT PRIMARY KEY,
    value TEXT,
    updated_at INTEGER
);
-- 例: ('sde_version', '2026-04-05'), ('extracted_at', '1712304000'),
--    ('cn_sync_at', '1712390400')

-- ============================
-- 物品/分组/分类
-- ============================
CREATE TABLE sde_types (
    type_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,                     -- 来自 eve_items.json，可为空
    group_id INTEGER,
    category_id INTEGER,
    market_group_id INTEGER,
    mass REAL,
    volume REAL,
    capacity REAL,
    portion_size INTEGER,
    radius REAL,
    published INTEGER,                -- 0/1
    meta_level INTEGER,
    icon_id INTEGER
);
CREATE INDEX idx_types_group ON sde_types(group_id);
CREATE INDEX idx_types_category ON sde_types(category_id);
CREATE INDEX idx_types_market_group ON sde_types(market_group_id);
CREATE INDEX idx_types_name_cn ON sde_types(name_cn);

CREATE TABLE sde_groups (
    group_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    category_id INTEGER,
    published INTEGER,
    icon_id INTEGER
);
CREATE INDEX idx_groups_category ON sde_groups(category_id);

CREATE TABLE sde_categories (
    category_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    published INTEGER
);

CREATE TABLE sde_market_groups (
    market_group_id INTEGER PRIMARY KEY,
    parent_id INTEGER,
    name_en TEXT,
    name_cn TEXT,
    description TEXT,
    has_types INTEGER
);
CREATE INDEX idx_mgroups_parent ON sde_market_groups(parent_id);

CREATE TABLE sde_meta_types (
    type_id INTEGER PRIMARY KEY,
    parent_type_id INTEGER,
    meta_group_id INTEGER             -- T1=1, T2=2, faction=4, T3=14 等
);

-- ============================
-- 宇宙
-- ============================
CREATE TABLE sde_regions (
    region_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    faction_id INTEGER
);

CREATE TABLE sde_constellations (
    constellation_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    region_id INTEGER,
    faction_id INTEGER
);
CREATE INDEX idx_const_region ON sde_constellations(region_id);

CREATE TABLE sde_systems (
    system_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    constellation_id INTEGER,
    region_id INTEGER,
    security REAL,
    security_class TEXT,              -- 'A' / 'B' / 'C' / 'C1'..'C6' (虫洞)
    faction_id INTEGER
);
CREATE INDEX idx_systems_const ON sde_systems(constellation_id);
CREATE INDEX idx_systems_region ON sde_systems(region_id);
CREATE INDEX idx_systems_name_cn ON sde_systems(name_cn);

CREATE TABLE sde_system_jumps (
    from_system_id INTEGER,
    to_system_id INTEGER,
    PRIMARY KEY (from_system_id, to_system_id)
);
CREATE INDEX idx_jumps_from ON sde_system_jumps(from_system_id);

CREATE TABLE sde_stations (
    station_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,                     -- 已组装好的中文名（"系统名 X - 公司名 设施类型"）
    system_id INTEGER,
    type_id INTEGER,
    owner_corp_id INTEGER,
    operation_id INTEGER
);
CREATE INDEX idx_stations_system ON sde_stations(system_id);

-- ============================
-- 势力 / 种族
-- ============================
CREATE TABLE sde_factions (
    faction_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    description TEXT,
    militia_corp_id INTEGER,
    icon_id INTEGER
);

CREATE TABLE sde_races (
    race_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    icon_id INTEGER
);

CREATE TABLE sde_npc_corps (
    corp_id INTEGER PRIMARY KEY,
    name_en TEXT,
    name_cn TEXT,
    faction_id INTEGER,
    icon_id INTEGER
);

-- ============================
-- 国服差异覆盖（admin 可写）
-- ============================
CREATE TABLE sde_overrides (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_type TEXT,                 -- 'type' / 'group' / 'system' / 'station' / 'faction' / ...
    entity_id INTEGER,
    field TEXT,                       -- 'name_cn' / 'mass' / 'volume' / ...
    value TEXT,
    note TEXT,                        -- 录入原因
    created_by TEXT,                  -- admin user
    created_at INTEGER,
    updated_at INTEGER
);
CREATE INDEX idx_overrides_entity ON sde_overrides(entity_type, entity_id);

-- ============================
-- ESI 兜底命中日志（StaticDataService 自动写入）
-- ============================
CREATE TABLE sde_missing_log (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    entity_type TEXT,                 -- 'type' / 'system' / ...
    entity_id INTEGER,
    hit_count INTEGER DEFAULT 1,
    first_seen_at INTEGER,
    last_seen_at INTEGER,
    resolved INTEGER DEFAULT 0,       -- 1 = 已纳入 SDE 子集
    UNIQUE(entity_type, entity_id)
);
CREATE INDEX idx_missing_unresolved ON sde_missing_log(resolved, last_seen_at);
```

### 3.3 数据来源映射

| 表 | SDE 源表 | 中文名来源 | 备注 |
|---|---|---|---|
| `sde_types` | `invTypes` | `data/eve_items.json` | 优先 `eve_items.json`，缺失走 ESI `language=zh` 一次性同步 |
| `sde_groups` | `invGroups` | ESI `/universe/groups/{id}?language=zh` | 一次性同步 |
| `sde_categories` | `invCategories` | ESI `/universe/categories/{id}?language=zh` | 一次性同步 |
| `sde_market_groups` | `invMarketGroups` | `data/market_groups.json`（已有） | 已经汉化 |
| `sde_meta_types` | `invMetaTypes` | — | 无中文需求 |
| `sde_regions` | `mapRegions` | `data/eve_regions.json` | 已有 |
| `sde_constellations` | `mapConstellations` | `data/eve_constellations.json` | 已有 |
| `sde_systems` | `mapSolarSystems` | `data/eve_systems.json` | 已有 |
| `sde_system_jumps` | `mapSolarSystemJumps` | — | 无中文需求 |
| `sde_stations` | `staStations` | `data/eve_stations.json` | 已有，包含 6 步法翻译结果 |
| `sde_factions` | `chrFactions` | ESI `language=zh` 一次性同步 | |
| `sde_races` | `chrRaces` | ESI `language=zh` 一次性同步 | |
| `sde_npc_corps` | `crpNPCCorporations` | ESI `language=zh` 一次性同步 | |

> **关键洞察**：本地 `data/*.json` 已经覆盖了大部分宇宙数据的汉化，新方案是**把它们整合进 SQL** 而不是从零做汉化。

---

## 4. 服务层架构

### 4.1 `StaticDataService` 门面

新文件：`app/Services/StaticData/StaticDataService.php`

```php
<?php

namespace App\Services\StaticData;

class StaticDataService
{
    /**
     * 查物品（含国服汉化、覆盖、兜底）
     * @return array|null  null 表示连兜底都失败
     */
    public function type(int $typeId): ?array;

    /**
     * 批量查询，效率优化（单次 IN 查询 vs N 次单查）
     * @param int[] $typeIds
     * @return array<int, array>  type_id => data
     */
    public function types(array $typeIds): array;

    public function group(int $groupId): ?array;
    public function groups(array $groupIds): array;

    public function category(int $categoryId): ?array;

    public function marketGroup(int $marketGroupId): ?array;
    public function marketGroupTree(): array;       // 替代 MarketService::getMarketGroupsTree

    public function system(int $systemId): ?array;
    public function systems(array $systemIds): array;

    public function station(int $stationId): ?array;
    public function stations(array $stationIds): array;

    public function region(int $regionId): ?array;
    public function constellation(int $constellationId): ?array;

    public function faction(int $factionId): ?array;
    public function race(int $raceId): ?array;
    public function npcCorp(int $corpId): ?array;

    /**
     * 自动识别 location_id 类型（system/station/structure）
     * structure 走 ESI（玩家建筑 SDE 没有）
     */
    public function resolveLocation(int $locationId, ?string $token = null): ?array;

    /**
     * 系统跳跃距离（用 sde_system_jumps BFS）
     */
    public function jumpDistance(int $fromSystemId, int $toSystemId): ?int;

    /**
     * 路径计算
     */
    public function jumpRoute(int $fromSystemId, int $toSystemId, string $preference = 'shortest'): array;
}
```

### 4.2 内部实现核心规则

```php
public function type(int $typeId): ?array
{
    // 1. 应用层缓存（Laravel cache，避免高频重复 SQL）
    return Cache::remember("static.type.{$typeId}", 3600, function () use ($typeId) {
        // 2. 主查询
        $row = DB::connection('static')
            ->table('sde_types')
            ->where('type_id', $typeId)
            ->first();

        if ($row) {
            $data = (array) $row;
            // 3. 应用国服差异覆盖
            $this->applyOverrides('type', $typeId, $data);
            return $data;
        }

        // 4. 未命中：写 missing_log + 走 ESI 兜底
        $this->logMissing('type', $typeId);
        return $this->fallbackToEsi('type', $typeId);
    });
}

private function applyOverrides(string $type, int $id, array &$data): void
{
    $overrides = DB::connection('static')
        ->table('sde_overrides')
        ->where('entity_type', $type)
        ->where('entity_id', $id)
        ->get();

    foreach ($overrides as $o) {
        // value 是字符串，按字段类型转
        $data[$o->field] = $this->castValue($o->field, $o->value);
    }
}

private function logMissing(string $type, int $id): void
{
    DB::connection('static')->statement('
        INSERT INTO sde_missing_log (entity_type, entity_id, hit_count, first_seen_at, last_seen_at)
        VALUES (?, ?, 1, ?, ?)
        ON CONFLICT(entity_type, entity_id) DO UPDATE SET
            hit_count = hit_count + 1,
            last_seen_at = excluded.last_seen_at
    ', [$type, $id, time(), time()]);
}

private function fallbackToEsi(string $type, int $id): ?array
{
    // 调用 ESI，结果短期缓存 24h，避免重复打
    return Cache::remember("static.esi_fallback.{$type}.{$id}", 86400, function () use ($type, $id) {
        // ... 调用对应 ESI 端点 ...
    });
}
```

### 4.3 数据库连接配置

```php
// config/database.php 新增
'connections' => [
    // ... 现有 ...
    'static' => [
        'driver' => 'sqlite',
        'database' => database_path('static.sqlite'),
        'prefix' => '',
        'foreign_key_constraints' => false,
    ],
    // fitting 已存在，保留
    'fitting' => [...],
],
```

### 4.4 调用方式（业务代码改造示例）

**改造前** (`KillmailEnrichService.php`):
```php
foreach ($typeIds as $tid) {
    $info = Http::get("ali-esi.evepc.163.com/latest/universe/types/{$tid}/?datasource=serenity")->json();
    $names[$tid] = $info['name'] ?? 'Unknown';
}
```

**改造后**:
```php
$types = $this->staticData->types($typeIds);   // 单次查询
foreach ($typeIds as $tid) {
    $names[$tid] = $types[$tid]['name_cn'] ?? $types[$tid]['name_en'] ?? 'Unknown';
}
```

延迟从 ~2-5 秒（N 次 ESI）降到 ~5-20 ms（单次 SQLite IN 查询）。

---

## 5. 管理后台 `/admin/sde`

### 5.1 页面结构

```
/admin/sde
│
├── 主面板
│   ├── 当前 SDE 版本：2026.04.05
│   ├── 子集大小：28.4 MB
│   ├── 上次提取：2026-04-05 13:22:15
│   ├── 上次国服汉化同步：2026-04-05 14:01:32
│   └── 数据完整性校验：✅ types: 45123 | groups: 1042 | systems: 8421
│
├── 一键操作
│   ├── [⬇ 下载最新 SDE] (后台异步任务)
│   ├── [🔧 重新提取 static.sqlite]
│   ├── [🌐 同步国服中文名]
│   ├── [♻ 清除应用缓存] (执行 cache:clear + 清 StaticDataService 内部缓存)
│   └── [📊 校验数据完整性]
│
├── 国服差异覆盖管理
│   ├── 表格列出所有 sde_overrides 记录
│   ├── [+ 新增覆盖项] (entity_type / entity_id / field / value / note)
│   ├── [✏ 编辑] / [🗑 删除]
│   └── 支持 CSV 导入导出
│
├── ESI 兜底命中记录
│   ├── 列出 sde_missing_log 表，按 hit_count 倒序
│   ├── 显示：entity_type / entity_id / hit_count / first_seen / last_seen
│   ├── [🔧 一键纳入] 单条
│   ├── [🔧 一键纳入全部未解决] (全量)
│   └── [✅ 标记为已解决]
│
└── 提取/同步任务历史
    └── 显示最近 30 次任务的执行日志
```

### 5.2 路由与控制器

```php
// routes/web.php
Route::middleware(['auth', 'admin'])->prefix('admin/sde')->group(function () {
    Route::get('/', [AdminSdeController::class, 'index'])->name('admin.sde.index');
    Route::post('/download', [AdminSdeController::class, 'download']);
    Route::post('/extract', [AdminSdeController::class, 'extract']);
    Route::post('/sync-cn', [AdminSdeController::class, 'syncCn']);
    Route::post('/clear-cache', [AdminSdeController::class, 'clearCache']);
    Route::post('/validate', [AdminSdeController::class, 'validate']);

    // overrides CRUD
    Route::resource('overrides', AdminSdeOverrideController::class)->except(['show']);

    // missing log
    Route::post('/missing/{id}/resolve', [AdminSdeMissingController::class, 'resolve']);
    Route::post('/missing/import', [AdminSdeMissingController::class, 'import']);
});
```

### 5.3 后台任务（避免阻塞页面）

下载 + 提取 + 汉化同步可能各需 1-5 分钟，必须**异步**：

```php
// app/Jobs/SdeDownloadJob.php
// app/Jobs/SdeExtractJob.php
// app/Jobs/SdeSyncCnJob.php

// 用 Laravel queue (database driver 即可，本项目无 redis queue)
// 状态写到 sde_meta 表的 'task_status' 键
// 前端轮询 GET /admin/sde/task-status 显示进度
```

---

## 6. 国服汉化与差异处理

### 6.1 中文名同步策略

**初始化**（一次性）：
```bash
php artisan sde:sync-cn-from-esi
```
- 遍历 `sde_types` 中所有 `name_cn IS NULL` 的记录
- 调用国服 ESI `/universe/types/{id}?datasource=serenity&language=zh`
- 写回 `name_cn` 字段
- 同时记录到 `data/eve_items.json` 用于 git 持久化

**日常维护**（游戏更新后）：
- admin 后台 [🌐 同步国服中文名] 按钮
- 增量：只同步 `last_sync_at` 之后新增的 typeID

**优先级链**：
1. `sde_overrides` 表的覆盖值（最高优先级）
2. `sde_types.name_cn`
3. `sde_types.name_en`
4. `'Unknown'` 兜底

### 6.2 国服差异校正

**已知国服与欧服差异类型**：
- 物品文案（极少，主要是历史性翻译差异）
- 个别物品属性（罕见，但确有发生）
- 新版本物品的发布时间差（国服通常滞后 1-2 周）

**校正方式**：

#### 方式 A：手工录入（推荐用于个别已知差异）
admin 后台手工添加 `sde_overrides` 记录。

#### 方式 B：脚本批量校正（适用于游戏大版本更新后）
```bash
php artisan sde:diff-against-esi --types=12345,12346,12347
```
- 拉取国服 ESI 数据
- 与 `sde_types` 对比
- 差异写入 `sde_overrides`
- 记录到 admin 可见的 diff 报告

#### 方式 C：被动发现
通过 `sde_missing_log` 自然发现新物品。

---

## 7. 迁移路径

### 7.1 总览

```
Phase 0  搭骨架        (1-2 天)   ← 基础设施，无业务影响
Phase 1  数据导入       (1 天)    ← 填充 static.sqlite，业务尚未切换
Phase 2  P0 改造        (3-4 天)  ← KM/Asset/Station，用户可感知改善
Phase 3  P1-P2 改造     (3-5 天)  ← Market/Distance/装配/LP/...
Phase 4  清理收尾       (1-2 天)  ← 移除老 ESI 路径与 data/*.json
```

### 7.2 Phase 0：搭骨架（基础设施）

**输出**：可工作的 `static.sqlite` schema + `StaticDataService` 门面 + admin 页面骨架，**没有业务代码切换**

**任务清单**：
- [ ] 创建 `database/migrations/` 中关于 `static.sqlite` 的迁移文件（或直接 SQL schema 文件，因为是 SDE 库不需要 migration 历史）
- [ ] 实现 `StaticDataService` 类（先返回空，所有方法可调用）
- [ ] 在 `config/database.php` 注册 `static` 连接
- [ ] 创建 `app/Console/Commands/SdeExtract.php`（基于 `ImportFittingSde.php` 模式）
- [ ] 创建 `app/Console/Commands/SdeSyncCn.php`
- [ ] 创建 `app/Console/Commands/SdeDownloadFromFuzzwork.php`
- [ ] 创建 admin 路由与 `AdminSdeController` 骨架
- [ ] 创建 `resources/views/admin/sde/index.blade.php` 骨架
- [ ] 在 admin 侧边栏添加菜单项

**验收**：访问 `/admin/sde` 能看到面板（数据为空），命令行 `php artisan sde:extract` 不报错（即使没数据）

### 7.3 Phase 1：数据导入

**输出**：填充完整的 `static.sqlite`，所有表都有数据

**任务清单**：
- [ ] 实现 `SdeDownloadFromFuzzwork`：从 `https://www.fuzzwork.co.uk/dump/latest/sqlite-latest.sqlite.bz2` 下载并解压到 `storage/sde/`
- [ ] 实现 `SdeExtract`：从 `sqlite-latest.sqlite` 提取所需子集到 `database/static.sqlite`
  - 13 个表的提取 SQL
  - 字段映射（SDE camelCase → 项目 snake_case）
  - 进度日志
- [ ] 实现 `SdeSyncCnFromLocal`：从 `data/eve_items.json` / `eve_systems.json` 等 JSON 写入 `name_cn` 字段
- [ ] 在 admin 后台触发完整流程，验证生成的 `static.sqlite` 数据完整性
- [ ] 单元测试：每个表的数据条数 vs SDE 期望条数

**验收**：
- `static.sqlite` ~25-35 MB
- `sde_types` 至少 45000 条
- `sde_systems` 8000+ 条
- 90%+ 的 `name_cn` 字段非空（来自 eve_items.json + data/*.json）

### 7.4 Phase 2：P0 改造（用户可感知）

**目标文件**（按改造收益排序）：

#### 2.1 KillmailEnrichService.php
- ESI 调用：`universe/types`、`universe/groups`、`universe/systems`、`universe/stations`
- **改造步骤**：
  1. 注入 `StaticDataService`
  2. 把所有 `Http::get(...universe/types...)` 替换为 `$this->staticData->types($ids)`
  3. 系统/空间站/分组同理
  4. 保留 character/corp/alliance 的 ESI 调用（动态数据）
- **回归测试**：
  - 前后对比 KM 22395435 的图片输出（像素级或字段级）
  - 计时：首次请求和缓存命中分别测
- **预期收益**：KM 详情页加载从 3-10s → < 500ms

#### 2.2 StationNameService.php
- 当前：6 步法翻译（pitfalls.md 记录），每个站调 ESI 多次
- **改造步骤**：
  1. 注入 `StaticDataService`
  2. NPC 站直接 `$this->staticData->station($id)`，命中即返回
  3. 玩家建筑（Citadel）保留 6 步法 ESI 路径
  4. 自动识别：station_id < 100000000 视为 NPC 站，>= 视为玩家建筑
- **回归测试**：所有依赖 `StationNameService` 的地方（资产、克隆体、订单等）

#### 2.3 AssetDataService.php
- 6 处 ESI 调用，主要是物品名 + 位置名
- **改造步骤**：
  1. 物品 typeID → `StaticDataService::types`
  2. 位置 location_id → `StaticDataService::resolveLocation`
- **回归测试**：资产页面、克隆体页面

**验收**：
- KM 详情页平均加载 < 500ms（之前 3-10s）
- 资产页面平均加载 < 1s（之前 5-15s）
- `pitfalls.md` 中 token 闭包陷阱在这 3 个文件不再可触发

### 7.5 Phase 3：P1-P2 改造

| 文件 | 优先级 | 备注 |
|---|---|---|
| `MarketService.php` | P1 | 静态查询走 SDE，价格走 ESI |
| `EsiSystemDistanceService.php` / `SystemDistanceService.php` | P1 | 跳跃距离走 `sde_system_jumps` |
| `EveDataService.php` | P1 | 综合数据服务 |
| `KillmailSearchService.php` | P2 | 搜索时的物品/系统过滤 |
| `LpStoreService.php` | P2 | LP 商店物品基础信息 |
| `ScoutService.php` | P2 | 侦察相关的星系/物品 |
| `WormholeService.php` | P3 | 虫洞数据，调用很少 |
| 其他 13 个文件 | 视情况 | 大多是 Controller 直接调 ESI 的小段，逐个清理 |

### 7.6 Phase 4：清理收尾

- [ ] `data/*.json` 改为从 `static.sqlite` **导出生成**（保持外部依赖兼容）
- [ ] 删除业务代码中的所有"先查 SDE，失败查 JSON"的双路径
- [ ] 删除老的 `EveDataService` 中已被 `StaticDataService` 取代的方法
- [ ] 更新 README / dev-rules.md，要求新功能必须走 `StaticDataService`
- [ ] `pitfalls.md` 标记已失效的 ESI 陷阱

---

## 8. 风险与对策

| 风险 | 概率 | 影响 | 对策 |
|---|---|---|---|
| 游戏更新后 SDE 未发布期间新物品 unknown | 高 | 个别 type 显示英文 | ESI 兜底 + missing_log + 提示管理员 |
| 国服特有属性差异 | 中 | 个别属性错误 | `sde_overrides` 表 + diff 校正脚本 |
| `static.sqlite` 入 git 后膨胀仓库 | 中 | clone 慢 | 选项：(a) 入 git，30 MB 可接受 (b) git-lfs (c) release attachment + 部署时下载 |
| 双数据源切换期不一致 | 中 | KM/资产显示不同名 | 严格走 `StaticDataService` 单一出口；不允许业务代码直接读 `data/*.json` |
| SDE 表结构变更 | 低 | 提取脚本崩 | 提取脚本写防御性代码 + 字段存在性检查 + Phase 1 测试 |
| Fuzzwork 服务下线 | 极低 | 无法下载 SDE | 备份方案：从游戏客户端导出（CCP 官方工具） |
| 改造引入回归 bug | 中 | 用户感知功能异常 | 每个 Phase 独立提交、灰度发布、保留 ESI fallback |
| admin 误操作覆盖 | 低 | `sde_overrides` 数据混乱 | 所有覆盖操作记录到 audit log + 软删除 |

---

## 9. 决策清单（实施前需要 Tus 拍板）

| 决策项 | 选项 | 推荐 |
|---|---|---|
| **D1**: `static.sqlite` 是否入 git | (a) 入 git (b) git-lfs (c) 不入 git，部署时下载 | (a)，30 MB 可接受，简化部署 |
| **D2**: SDE 全量 `sqlite-latest.sqlite` 是否入 git | 350 MB，**强烈不建议入 git** | 不入 git，gitignore，部署/admin 按需下载 |
| **D3**: queue driver | (a) database (b) redis | (a)，无需新增依赖 |
| **D4**: admin 权限 | (a) 复用现有 `auth + admin` middleware (b) 新建权限组 | (a) |
| **D5**: 是否合并 `fitting.sqlite` 到 `static.sqlite` | 合并 / 保持独立 | 保持独立，理由见 §3.1 |
| **D6**: 新物品 ESI 兜底缓存 TTL | 1h / 24h / 7d | 24h，平衡及时性和 ESI 负载 |
| **D7**: 中文名优先级 | sde_overrides > eve_items.json > ESI 当前值 / 其他 | 见 §6.1 |
| **D8**: Phase 2 第一个改造选哪个 | KM / Station / Asset | KM（最痛、最有感、改造范围相对清晰） |

---

## 10. 实施 checklist（顺序执行）

### Phase 0
- [ ] 决定 D1-D8
- [ ] 创建 `app/Services/StaticData/StaticDataService.php` 骨架
- [ ] 创建 `database/static.sqlite.schema.sql`
- [ ] 创建 `app/Console/Commands/SdeExtract.php` 骨架
- [ ] 创建 `app/Console/Commands/SdeSyncCn.php` 骨架
- [ ] 创建 `app/Console/Commands/SdeDownloadFromFuzzwork.php`
- [ ] 创建 `app/Http/Controllers/Admin/AdminSdeController.php`
- [ ] 创建 `resources/views/admin/sde/index.blade.php`
- [ ] 添加路由 `routes/web.php`
- [ ] 添加 admin 侧边栏链接
- [ ] 配置 `config/database.php` 的 `static` 连接

### Phase 1
- [ ] 实现 SDE 下载命令并验证
- [ ] 实现 13 个表的提取 SQL
- [ ] 实现汉化数据导入
- [ ] 验证 `static.sqlite` 数据完整性
- [ ] 写单元测试

### Phase 2.1 (KM)
- [ ] 备份当前 `KillmailEnrichService.php`
- [ ] 注入 `StaticDataService`
- [ ] 改造 type/group/system/station 调用
- [ ] 单测覆盖
- [ ] 部署到测试环境
- [ ] 抽样 10 个 KM 做前后对比
- [ ] 部署到生产并观察 1-2 天

### Phase 2.2 (Station)
- [ ] 同 Phase 2.1 模式

### Phase 2.3 (Asset)
- [ ] 同 Phase 2.1 模式

### Phase 3
- [ ] 按文件优先级表逐个改造

### Phase 4
- [ ] 清理 data/*.json 引用
- [ ] 更新文档
- [ ] 移除 ESI fallback 代码（仅保留必要的）

---

## 11. FAQ

### Q1: 为什么不用 ORM model 而是直接 DB facade？
A: `static.sqlite` 是只读数据（除 overrides 和 missing_log），用 model 反而引入额外开销。`StaticDataService` 内部直接 `DB::connection('static')` 更简单。

### Q2: 为什么不用单文件 JSON 而要建 SQLite？
A: JSON 全量加载 → 内存占用大、查找 O(n)；SQLite 索引查找 O(log n)、内存可控、支持复杂 join（如 type → group → category）。`market_groups.json` 那种纯前端用的小数据可以保留 JSON，但服务端查询必须走 SQL。

### Q3: 已有的 `data/eve_items.json` 还要保留吗？
A: **保留**。理由：
1. Phase 4 之前，作为 `static.sqlite` 损坏时的兜底
2. 是中文名的权威来源（你已确认质量比 ESI `language=zh` 更好）
3. 可以直接被前端 JS 加载（避免每个查询走 API）
4. Phase 4 后改为从 `static.sqlite` 导出生成

### Q4: 这跟 `fitting.sqlite` 有冲突吗？
A: **没有**。`fitting.sqlite` 是装配模拟器专用的高度优化子集（含完整 dogma 属性 + effects + modifiers），用于客户端 Dogma 计算。`static.sqlite` 是面向通用查询（item/system/station 名称解析等）。两者职责不同，保持独立。装配模拟器不依赖 `static.sqlite`。

### Q5: 改造期间会破坏现有功能吗？
A: 不会。每个 Phase 都保留 ESI fallback，灰度切换。具体策略：
- Phase 2-3 改造时，先添加 SDE 路径，**保留** ESI 路径作为兜底
- 部署后观察日志，确认 SDE 路径覆盖率 > 95%
- Phase 4 才删除 ESI 路径

### Q6: SDE 多久更新一次？
A: Fuzzwork 每周更新一次。游戏大版本更新后 1-3 天内会有对应版本。日常开发只需要每月或每个游戏版本更新一次即可。

### Q7: 如果服务器连不上 Fuzzwork 怎么办？
A: 可以从本地 Windows 下载后 scp 上传到服务器 `storage/sde/`。admin 后台需要支持"上传 SDE 文件"功能作为兜底。

---

## 12. 参考资料

- Qoder 原版子集设计：`knowledge/sde-data-architecture.md`
- 装配模拟器规格：`knowledge/fitting-simulator-spec.md`
- 已踩过的 ESI 坑：`knowledge/pitfalls.md` (ESI API 陷阱章节)
- 已有的 SDE 提取实现：`app/Console/Commands/ImportFittingSde.php`
- Fuzzwork SDE 下载：https://www.fuzzwork.co.uk/dump/
- Fuzzwork SDE 文档：https://www.fuzzwork.co.uk/2021/07/17/understanding-the-eve-online-sde-1/
- 国服 ESI 文档：https://ali-esi.evepc.163.com/ui/

---

## 变更记录

| 日期 | 工具 | 变更 |
|---|---|---|
| 2026-04-10 | [Claude Code] | 创建文档（在 Qoder `sde-data-architecture.md` 基础上扩展为完整迁移方案） |
