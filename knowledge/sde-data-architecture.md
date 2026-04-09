# SDE 统一数据架构设计方案

> **状态**: 设计完成，待实施
> **创建日期**: 2026-04-09
> **涉及功能**: 装配模拟器、工业制造、以及其他依赖静态数据的功能模块

---

## 一、背景与问题

### 1.1 当前数据架构问题

| 问题 | 影响 |
|------|------|
| 数据来源分散 | 物品数据、星系数据、虫洞数据等分布在不同JSON文件和ESI接口 |
| 维护成本高 | 游戏更新后需要分别更新多个数据源，容易遗漏 |
| 扩展困难 | 新增功能时需要寻找新数据源，无法复用现有数据 |
| 版本不一致 | 各数据源版本可能不同步，导致数据不一致 |

### 1.2 解决思路

建立以 **SDE（Static Data Export）为统一数据源** 的架构，为不同功能生成轻量子集，ESI 作为国服校正和实时数据补充。

---

## 二、核心架构设计

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    SDE 统一数据架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  第一层：完整 SDE 数据源                                      │
│  ┌─────────────────────────────────────┐                   │
│  │   sqlite-latest.sqlite (350MB)      │                   │
│  │   来源: Fuzzwork                     │                   │
│  │   更新: 游戏版本更新后重新下载         │                   │
│  └─────────────────────────────────────┘                   │
│                       ↓ 提取子集                            │
│                                                             │
│  第二层：功能子集数据库                                       │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐              │
│  │fitting.    │ │industry.   │ │其他功能    │              │
│  │sqlite      │ │sqlite      │ │子集...     │              │
│  │~10MB       │ │~15MB       │ │            │              │
│  └────────────┘ └────────────┘ └────────────┘              │
│                       ↓ 国服校正                            │
│                                                             │
│  第三层：ESI 数据补充                                         │
│  ┌─────────────────────────────────────┐                   │
│  │   ali-esi.evepc.163.com             │                   │
│  │   ├── 属性值校正（国服差异）          │                   │
│  │   ├── 中文名称补充                   │                   │
│  │   └── 实时数据（价格、市场等）        │                   │
│  └─────────────────────────────────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 数据流向

```
游戏发布新版本
      ↓
下载最新 SDE (sqlite-latest.sqlite)
      ↓
运行提取命令，生成各功能子集
      ↓
运行校正命令，用国服 ESI 覆盖差异
      ↓
各功能模块使用更新后的数据
```

---

## 三、SDE 数据全览

### 3.1 物品相关表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `invTypes` | ~45,000 | typeID, typeName, groupID, published | 所有物品基础信息 |
| `invGroups` | ~1,000 | groupID, categoryName, categoryID | 物品分组 |
| `invCategories` | ~50 | categoryID, categoryName | 物品分类 |
| `invMetaTypes` | ~15,000 | typeID, metaGroupID, parentTypeID | 科技等级关系 |
| `invMarketGroups` | ~1,500 | marketGroupID, parentGroupID | 市场分组树 |
| `invTraits` | ~16,000 | typeID, skillID, bonus | 舰船加成描述 |

### 3.2 工业相关表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `industryBlueprints` | ~3,000 | blueprintTypeID, productTypeID, maxProductionLimit | 蓝图基础信息 |
| `industryActivityMaterials` | ~50,000 | typeID, materialTypeID, quantity, activityID | 制造材料清单 |
| `industryActivityProducts` | ~3,000 | typeID, productTypeID, quantity | 制造产出 |
| `industryActivitySkills` | ~8,000 | typeID, skillID, level, activityID | 制造所需技能 |
| `industryActivityProbabilities` | ~1,000 | typeID, probability | 发明成功率 |

### 3.3 属性系统表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `dgmAttributeTypes` | ~2,500 | attributeID, attributeName, unitID | 属性定义 |
| `dgmTypeAttributes` | ~500,000 | typeID, attributeID, valueFloat | 物品属性值 |
| `dgmEffects` | ~5,000 | effectID, effectName, modifierInfo | 效果定义 |
| `dgmTypeEffects` | ~50,000 | typeID, effectID | 物品-效果映射 |
| `dgmExpressions` | ~30,000 | expressionID, operandID | 效果表达式树 |

### 3.4 宇宙相关表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `mapRegions` | ~100 | regionID, regionName | 星域信息 |
| `mapConstellations` | ~1,000 | constellationID, regionID | 星座信息 |
| `mapSolarSystems` | ~8,000 | solarSystemID, constellationID | 星系信息 |
| `mapSolarSystemJumps` | ~14,000 | fromSolarSystemID, toSolarSystemID | 星系跳跃连接 |
| `mapRegionJumps` | ~300 | fromRegionID, toRegionID | 星域跳跃连接 |
| `staStations` | ~5,000 | stationID, solarSystemID | NPC空间站 |

### 3.5 势力与NPC相关表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `chrFactions` | ~20 | factionID, factionName | 势力信息 |
| `chrRaces` | ~4 | raceID, raceName | 种族信息 |
| `crpNPCCorporations` | ~180 | corporationID, factionID | NPC公司 |
| `crpNPCDivisions` | ~30 | divisionID, divisionName | NPC部门 |

### 3.6 行星开发表

| 表名 | 数据量 | 主要字段 | 用途 |
|------|--------|---------|------|
| `planetSchematics` | ~100 | schematicID, cycleTime | 行星开发配方 |
| `planetSchematicsTypeMap` | ~500 | schematicID, typeID, isInput | 配方材料映射 |

---

## 四、功能子集设计

### 4.1 装配模拟器子集 (fitting.sqlite)

**目标大小**: ~10MB

**提取范围**:
- Category ID: 6(舰船), 7(装备), 8(弹药), 16(技能), 18(无人机), 24(植入体), 32(子系统)
- 物品数量: ~11,000 种
- 属性过滤: 保留核心计算所需属性（约80-100个属性ID）

**表结构**:
```sql
-- 物品基础信息
fitting_types (type_id, group_id, category_id, name_en, name_cn, ...)

-- 物品属性值
fitting_attributes (type_id, attribute_id, value_int, value_float)

-- 效果定义（含修正器）
fitting_effects (effect_id, name, effect_category, modifiers_json)

-- 物品-效果映射
fitting_type_effects (type_id, effect_id, is_default)

-- 属性定义
fitting_attr_types (attribute_id, name, unit_id, stackable, high_is_good)

-- 辅助表
fitting_groups, fitting_categories, fitting_meta_types
```

**关键属性ID清单**:
```
资源类: 48(CPU输出), 11(电网输出), 482(电容), 55(充能), 1132(校准)
消耗类: 50(CPU消耗), 30(电网消耗), 1153(校准消耗)
槽位: 12(低槽), 13(中槽), 14(高槽), 1137(改装), 102(炮台), 101(发射器)
HP: 263(护盾), 265(装甲), 9(结构)
抗性: 271-274(护盾四抗), 267-270(装甲四抗)
机动: 37(速度), 70(灵活性), 552(信号), 600(曲速)
锁定: 76(距离), 564(分辨率), 192(目标数)
无人机: 1271(带宽), 283(容量)
武器: DPS计算相关属性
电容: 消耗相关属性
```

### 4.2 工业制造子集 (industry.sqlite)

**目标大小**: ~15MB

**提取范围**:
- 所有蓝图（industryBlueprints 全量）
- 材料/技能需求（按蓝图ID关联）
- 制造产出信息

**表结构**:
```sql
-- 蓝图基础信息
industry_blueprints (blueprint_type_id, product_type_id, production_time, 
                     waste_factor, max_production_limit, tech_level)

-- 材料需求
industry_requirements (blueprint_type_id, required_type_id, quantity, 
                       activity_id, damage_per_job, is_skill)

-- 发明概率
industry_invention_chance (product_type_id, base_chance)

-- 产品信息
industry_products (type_id, name_en, name_cn, group_id, meta_level)
```

**Activity ID 对照**:
```
1  = 制造
3  = 时间效率研究
4  = 材料效率研究
5  = 复制
8  = 发明
11 = 反应
```

### 4.3 其他潜在子集

| 子集名称 | 数据来源 | 目标大小 | 用途 |
|----------|---------|---------|------|
| 宇宙导航 | mapSolarSystems, mapJumps | ~2MB | 星系导航、旗舰跳跃 |
| 虫洞数据 | mapWormholeClasses + 外部数据 | ~1MB | 虫洞查询 |
| 代理人 | agtAgents, crpNPCDivisions | ~1MB | 代理人搜索 |
| 行星开发 | planetSchematics | ~0.5MB | 行星开发计算 |

---

## 五、实施计划

### 5.1 命令设计

```bash
# 主命令：导入完整SDE
php artisan sde:import 
    {--force : 强制重新下载}
    {--file= : 使用本地文件}

# 提取子集
php artisan sde:extract-fitting
php artisan sde:extract-industry
php artisan sde:extract-universe

# 国服校正
php artisan sde:sync-serenity
    {--types= : 指定校正的物品类型}
    {--force : 强制全部重新校正}

# 一键更新
php artisan sde:update-all
    # 等价于: import + extract + sync
```

### 5.2 目录结构

```
storage/app/sde/
├── sqlite-latest.sqlite      # 原始SDE（不提交git）
└── last_update.txt           # 更新记录

database/
├── database.sqlite           # 用户数据
├── fitting.sqlite            # 装配子集
└── industry.sqlite           # 工业子集
```

### 5.3 配置更新

```php
// config/database.php 新增连接
'fitting' => [
    'driver' => 'sqlite',
    'database' => database_path('fitting.sqlite'),
],

'industry' => [
    'driver' => 'sqlite',
    'database' => database_path('industry.sqlite'),
],
```

---

## 六、ESI 校正策略

### 6.1 需要校正的数据

| 数据类型 | 校正原因 | ESI 接口 |
|----------|---------|---------|
| 物品属性值 | 国服可能有版本差异 | `/universe/types/{id}/` |
| 中文名称 | SDE 只有英文名 | `/universe/types/{id}/` |
| 市场价格 | 实时数据 | `/markets/prices/` |

### 6.2 校正流程

```
1. 遍历子集中的所有 type_id
2. 调用国服 ESI 获取最新数据
3. 对比并更新差异字段
4. 记录校正日志
```

### 6.3 校正频率

| 场景 | 频率 |
|------|------|
| 游戏版本更新后 | 必须 |
| 新增物品类型 | 必须 |
| 日常维护 | 可选（差异通常很小） |

---

## 七、与现有数据的关系

### 7.1 数据迁移对照

| 现有数据文件 | 迁移到 | 说明 |
|-------------|--------|------|
| `data/eve_items.json` | fitting.sqlite | 装配相关物品 |
| `data/eve_systems.json` | 保留或合并 | 星系数据（项目已有完善方案） |
| `data/eve_stations.json` | 保留或合并 | 空间站数据 |
| `data/eve_regions.json` | 保留或合并 | 星域数据 |

### 7.2 兼容性考虑

- 现有 Service 类（如 EveDataService）需要更新数据源
- API 接口保持不变，只更换底层数据源
- 前端 JSON 文件（如 market_groups.json）保持原有静态文件方案

---

## 八、风险与对策

| 风险 | 对策 |
|------|------|
| SDE 文件过大（350MB） | 不提交到 git，部署时按需下载 |
| 提取脚本复杂 | 参考现有 ImportFittingSde.php，逐步完善 |
| 国服版本落后 | 通过 ESI 校正覆盖差异 |
| SDE 表结构变化 | 查询前用 `.schema` 验证字段名 |

---

## 九、预期收益

| 收益 | 说明 |
|------|------|
| **维护效率提升 80%** | 一键更新替代分散更新 |
| **扩展成本降低** | 新功能直接从 SDE 提取数据 |
| **数据一致性保证** | 统一版本，避免不一致问题 |
| **未来功能支持** | 为代理人搜索、行星开发等功能预留数据基础 |

---

## 十、参考资料

- Fuzzwork SDE 下载: https://www.fuzzwork.co.uk/dump/
- Fuzzwork SDE 说明: https://www.fuzzwork.co.uk/2021/07/17/understanding-the-eve-online-sde-1/
- 国服 ESI 文档: https://ali-esi.evepc.163.com/ui/
- 现有导入命令: `app/Console/Commands/ImportFittingSde.php`
- 装配模拟器规格: `qoder-memory/knowledge/fitting-simulator-spec.md`