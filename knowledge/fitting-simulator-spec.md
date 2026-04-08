# 装配模拟器技术规格文档

> **状态**: 技术调研完成，待开发
> **最后更新**: 2026-04-08
> **负责 AI**: [Claude Code] 技术调研 + 架构设计

---

## 一、功能目标

模拟游戏内装配页面，允许用户在网页上：
1. 选择舰船
2. 将装备拖入槽位（高/中/低/改装件）
3. 实时查看装备后各项属性（护盾HP/抗性/速度/电容/DPS等）
4. 查看装配总价格估算
5. 保存/分享装配方案

---

## 二、核心概念

### 2.1 Dogma 系统（属性计算引擎）

EVE 游戏内所有属性计算的核心规则。每个物品（舰船/装备/弹药/技能/植入体）有：
- **属性（Attributes）**：数值属性，如 `maxVelocity=365`、`cpuOutput=130`
- **效果（Effects）**：包含一组修正器（Modifiers），描述装备装上后如何修改目标属性

**修正器的 5 个组成部分**：
```
1. 源属性 (sourceAttributeID)     — 从装备上读取的值
2. 目标位置 (domain)              — CurrentShip / CurrentTarget / Gang
3. 目标属性 (modifiedAttributeID) — 要修改哪个属性
4. 操作类型 (operator)            — 如何计算（见下表）
5. 过滤器 (func + filter)         — 可选，按组/技能限制作用范围
```

**9 步计算顺序（严格按序执行）**：

| 顺序 | 操作 | 计算方式 | 受堆叠惩罚 |
|------|------|---------|-----------|
| 1 | PreAssign | `val = src`（直接覆盖基础值） | 否 |
| 2 | PreMul | `val *= src` | 是 |
| 3 | PreDiv | `val /= src` | 是 |
| 4 | ModAdd | `val += src`（平面加值） | 否 |
| 5 | ModSub | `val -= src` | 否 |
| 6 | PostMul | `val *= src` | 是 |
| 7 | PostDiv | `val /= src` | 是 |
| 8 | PostPercent | `val *= (1 + src/100)` | 是 |
| 9 | PostAssign | `val = src`（最终覆盖） | 否 |

**堆叠惩罚公式**：`S(n) = e^(-(n/2.67)²)`，n 为从 0 开始的位置索引

| 第几个模块 | 有效率 |
|-----------|--------|
| 第 1 个 | 100.0% |
| 第 2 个 | 86.9% |
| 第 3 个 | 57.1% |
| 第 4 个 | 28.3% |
| 第 5 个 | 10.6% |
| 第 6 个 | 3.0% |

**堆叠惩罚规则**：
- 正向和负向修正**分开**计算各自的惩罚序列
- 按绝对值从大到小排序（最强的排第一，惩罚最轻）
- **免堆叠惩罚**：技能加成、舰船船体加成、植入体/硬接线、ModAdd/ModSub（平面加值）
- **永远不受惩罚**：Damage Control 模块

**属性完整计算流程（4 步 Pass）**：
```
Pass 1: 收集舰船 + 所有已装装备的属性基础值
Pass 2: 收集所有生效的效果（按模块状态：Offline/Online/Active/Overloaded）
Pass 3: 应用所有修正器，按 9 步顺序 + 堆叠惩罚计算最终属性值（支持递归：属性可以依赖另一个属性的计算结果）
Pass 4: 计算衍生属性（DPS / EHP / 电容稳定性等，这些不在 Dogma 系统内，需要额外公式）
```

**模块状态**：
- `Offline`：已装配但离线，不消耗资源，无任何效果
- `Online`：在线，消耗 CPU/电网，被动效果生效
- `Active`：激活状态，主动效果生效（如护盾加硬/装甲修复器）
- `Overloaded`：过载，额外效果生效但会损伤模块（超频）

### 2.2 抗性值注意

SDE 中抗性存储为**共振值（resonance）**，不是百分比：
```
共振值 0.4 = 抗性 60%
转换公式：抗性% = (1 - 共振值) × 100
```

---

## 三、数据来源

### 3.1 SDE（Static Data Export）

CCP 官方发布的完整游戏静态数据导出，包含所有物品、属性、效果定义。

**下载地址**：
- 官方（YAML/JSONL）：`https://developers.eveonline.com/static-data/`
- Fuzzwork 转换版（SQLite/MySQL/PostgreSQL）：`https://www.fuzzwork.co.uk/dump/`
  - `sqlite-latest.sqlite.bz2` —— ~129MB 压缩，~350MB 解压
  - `mysql-latest.tar.bz2` —— ~71MB 压缩

**为什么必须用 SDE，ESI 不够**：
- ESI 的 `/dogma/effects/{id}` 不返回修正器（modifiers）表达式树
- 效果的 modifierInfo 字段只存在于 SDE 的 `dgmEffects` 表
- 单独用 ESI 需要对每个 type_id 单独请求，速度极慢，无批量接口

### 3.2 关键数据库表

| 表名 | 用途 | 估计行数 |
|------|------|---------|
| `invTypes` | 所有物品（舰船/装备/弹药/技能）基础信息 | ~45,000 |
| `invGroups` | 物品组（如"强袭型护卫舰"） | ~1,000 |
| `invCategories` | 物品分类（舰船=6/装备=7/弹药=8/技能=16） | ~50 |
| `invMetaTypes` | 科技等级（T1/T2/海盗/派系等） | ~15,000 |
| `dgmAttributeTypes` | 属性定义（名称/单位/默认值/是否受堆叠） | ~2,500 |
| `dgmTypeAttributes` | 每个物品的属性值 | ~500,000 |
| `dgmEffects` | 效果定义（含 modifierInfo YAML） | ~5,000-7,000 |
| `dgmTypeEffects` | 物品-效果映射 | ~50,000 |
| `dgmExpressions` | 旧式效果表达式树（部分效果仍用） | ~30,000 |
| `eveUnits` | 属性单位定义 | ~200 |

**装配功能所需子集体积估算**：
- 涉及物品类型：舰船(~400) + 装备(~8,000) + 弹药(~2,000) + 技能(~500) + 植入体(~200) ≈ 11,000 种物品
- 对应的 dgmTypeAttributes：~200,000-300,000 行
- 相关 dgmEffects：~5,000 条
- **压缩后 JSON 估计：~5-15MB**

### 3.3 效果修正器格式

**现代格式（modifierInfo，YAML，大部分效果使用）**：
```yaml
modifierInfo:
  - domain: shipID
    func: ItemModifier
    modifiedAttributeID: 76     # 目标属性（maxTargetRange）
    modifyingAttributeID: 1991  # 源属性
    operator: 5                 # PostMul
```

`func` 类型说明：
| func | 说明 |
|------|------|
| `ItemModifier` | 修改装备所在物品（通常是舰船本身） |
| `LocationModifier` | 修改指定位置的所有物品 |
| `LocationGroupModifier` | 修改指定位置指定 groupID 的物品 |
| `LocationRequiredSkillModifier` | 修改需要特定技能的物品 |
| `OwnerRequiredSkillModifier` | 修改角色持有的技能 |

**旧式格式（dgmExpressions 表达式树）**：
部分效果仍使用递归表达式树，需要遍历 `dgmExpressions` 表构建 modifier。建议参考 Pyfa 的 `eos/effectHandlerOperands.py` 进行解析。

### 3.4 国服数据校正方案

**问题**：CCP 只发布欧服（Tranquility）SDE，国服（Serenity）版本通常较旧，数值可能有差异。

**解决方案**：以欧服 SDE 为基础，用国服 ESI 覆盖差异属性值。

**校正流程**：
```
1. 导入 Fuzzwork SQLite → 本地数据库（一次性操作）
2. 运行校正脚本：
   - 遍历所有装配相关 type_id
   - 调用 ali-esi.evepc.163.com/latest/universe/types/{id}/?datasource=serenity
   - 取返回的 dogma_attributes 数组
   - 用国服值覆盖本地数据库中对应的属性值
3. 保存为校正后的装配数据库
4. 游戏更新后重新运行步骤 2
```

**注意**：Dogma 计算规则（操作类型/堆叠惩罚公式）两服相同，只有**数值**可能不同，因此引擎代码无需区分服务器。

---

## 四、关键属性 ID 速查表

### 舰船资源输出

| 属性名 | ID | 说明 |
|--------|-----|------|
| `cpuOutput` | 48 | 舰船总 CPU（tf） |
| `powerOutput` | 11 | 舰船总电网（MW） |
| `capacitorCapacity` | 482 | 电容总量（GJ） |
| `rechargeRate` | 55 | 电容充能时间（ms） |
| `upgradeCapacity` | 1132 | 改装件校准值上限 |

### 装备资源消耗

| 属性名 | ID | 说明 |
|--------|-----|------|
| `cpu` | 50 | 装备 CPU 消耗 |
| `power` | 30 | 装备电网消耗 |
| `upgradeCost` | 1153 | 改装件校准消耗 |

### 槽位数量

| 属性名 | ID | 说明 |
|--------|-----|------|
| `hiSlots` | 14 | 高槽数 |
| `medSlots` | 13 | 中槽数 |
| `lowSlots` | 12 | 低槽数 |
| `rigSlots` | 1137 | 改装件槽数 |

### 硬点

| 属性名 | ID | 说明 |
|--------|-----|------|
| `turretSlotsLeft` | 102 | 炮台硬点 |
| `launcherSlotsLeft` | 101 | 发射器硬点 |

### HP 与抗性（共振值）

| 属性名 | ID | 说明 |
|--------|-----|------|
| `shieldCapacity` | 263 | 护盾 HP |
| `armorHP` | 265 | 装甲 HP |
| `hp` | 9 | 结构 HP |
| `shieldEmDamageResonance` | 271 | 护盾 EM 共振 |
| `shieldExplosiveDamageResonance` | 272 | 护盾 爆炸 共振 |
| `shieldKineticDamageResonance` | 273 | 护盾 动能 共振 |
| `shieldThermalDamageResonance` | 274 | 护盾 热能 共振 |
| `armorEmDamageResonance` | 267 | 装甲 EM 共振 |
| `armorExplosiveDamageResonance` | 268 | 装甲 爆炸 共振 |
| `armorKineticDamageResonance` | 269 | 装甲 动能 共振 |
| `armorThermalDamageResonance` | 270 | 装甲 热能 共振 |

### 机动性

| 属性名 | ID | 说明 |
|--------|-----|------|
| `maxVelocity` | 37 | 最大速度（m/s） |
| `agility` | 70 | 惯性系数（越低越灵活） |
| `signatureRadius` | 552 | 信号半径（m） |
| `warpSpeedMultiplier` | 600 | 曲速速度倍率 |

### 锁定

| 属性名 | ID | 说明 |
|--------|-----|------|
| `maxTargetRange` | 76 | 最大锁定距离 |
| `scanResolution` | 564 | 扫描分辨率 |
| `maxLockedTargets` | 192 | 最大锁定目标数 |

### 无人机

| 属性名 | ID | 说明 |
|--------|-----|------|
| `droneBandwidth` | 1271 | 无人机带宽（Mbit/s） |
| `droneCapacity` | 283 | 无人机舱容量（m³） |

---

## 五、装备槽位映射（flag 值）

ESI 装配接口 `/characters/{id}/fittings/` 中用 `flag` 标识槽位位置：

| flag 值 | 槽位 |
|---------|------|
| 11-18 | 低槽 1-8 |
| 19-26 | 中槽 1-8 |
| 27-34 | 高槽 1-8 |
| 92-99 | 改装件槽 1-8 |
| 125-132 | 子系统槽 1-8（战略型巡洋舰） |
| 87 | 无人机舱 |
| 5 | 货舱 |

---

## 六、开源参考实现

### EVEShipFit/dogma-engine（最推荐）
- 语言：Rust → 编译为 WebAssembly
- npm 包：`@EVEShipFit/dogma-engine`
- GitHub：`https://github.com/EVEShipFit/dogma-engine`
- 特点：
  - 完整实现 4 步 Pass 计算流程
  - 通过 JavaScript 回调获取数据（不直接加载数据库到 WASM 内存）
  - 现代 web 方案的标杆
- 所需 JS 回调接口（WASM 调用这些函数获取数据）：
  ```javascript
  get_dogma_attributes(type_id)   // 返回该物品所有属性
  get_dogma_attribute(attr_id)    // 返回单个属性定义
  get_dogma_effects(type_id)      // 返回该物品所有效果
  get_dogma_effect(effect_id)     // 返回单个效果定义（含 modifiers）
  get_type(type_id)               // 返回物品基础信息
  ```

### Pyfa/eos（最成熟，Python）
- GitHub：`https://github.com/pyfa-org/Pyfa`
- 特点：
  - 桌面端金标准，社区验证多年
  - `eos` 引擎可独立使用
  - `eos/gamedata.py`：数据模型定义
  - `eos/fit.py`：`calculateModifiedAttributes()` 完整实现
- 用途：理解计算流程、验证结果

### EVE-Fitting.js（唯一纯 JS 实现）
- GitHub：`https://github.com/tiktrimo/EVE-Fitting.js`
- 语言：JavaScript + React
- 用途：参考 JS 端实现思路

---

## 七、推荐技术架构

### 技术栈
- 后端：Laravel 10（现有）
- 前端：Blade + Alpine.js + Tailwind CSS（现有）
- 数据库：在现有 SQLite 旁新建一个 `fitting.sqlite`，存储 SDE 数据
- Dogma 引擎：前端 JavaScript（初期可简化，二期引入 WASM）

### 整体架构图

```
浏览器端
┌──────────────────────────────────────────────┐
│  装配 UI（Blade + Alpine.js）                 │
│  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│  │ 舰船选择  │  │ 槽位面板  │  │ 属性面板   │  │
│  │ 搜索过滤  │  │ 拖拽装备  │  │ 实时数据   │  │
│  └──────────┘  └──────────┘  └────────────┘  │
│                     ↕                         │
│         fitting.js（Dogma 计算引擎）            │
│         ↑ 从页面缓存中读取物品数据               │
└──────────────────────────────────────────────┘
              ↕ API 请求
服务端（Laravel）
┌──────────────────────────────────────────────┐
│  /fitting                → 装配页面           │
│  /api/fitting/ships      → 舰船列表           │
│  /api/fitting/types/{id} → 物品属性+效果      │
│  /api/fitting/search     → 装备搜索           │
│  /api/fitting/save       → 保存装配           │
│  /api/fitting/price      → 价格估算           │
├──────────────────────────────────────────────┤
│  FittingDataService      ← SDE 数据查询服务   │
│  FittingPriceService     ← 复用现有市场接口    │
│  FittingRepository       ← 用户装配保存/读取   │
├──────────────────────────────────────────────┤
│  fitting.sqlite（SDE 数据，只读）              │
│  app.sqlite（用户数据，读写）                  │
└──────────────────────────────────────────────┘
```

### 数据流

```
首次进入页面：
  → 请求 /api/fitting/ships → 返回舰船列表（含分类/图片）→ 渲染选择器

选择舰船：
  → 请求 /api/fitting/types/{shipId} → 返回槽位数/硬点/基础属性 → 渲染槽位

搜索装备：
  → 请求 /api/fitting/search?q=xxx&slot=mid → 返回匹配装备列表

拖入装备（核心流程）：
  → 前端 fitting.js 接收新装备
  → 从本地缓存查 type 属性
  → 若缓存无：请求 /api/fitting/types/{moduleId} → 缓存结果
  → 运行 Dogma 计算引擎
  → 更新属性面板 UI

保存装配：
  → POST /api/fitting/save { ship_id, slots: [{type_id, flag}], name }
  → 存入 SQLite user_fittings 表
```

### SDE 数据导入流程

```bash
# 一次性操作：
# 1. 下载 Fuzzwork SQLite
wget https://www.fuzzwork.co.uk/dump/sqlite-latest.sqlite.bz2
bunzip2 sqlite-latest.sqlite.bz2

# 2. 运行 Laravel 导入命令（需开发）
php artisan fitting:import-sde sqlite-latest.sqlite

# 3. 运行国服校正
php artisan fitting:sync-serenity
# 遍历所有 type_id，用 ali-esi 覆盖属性值

# 4. 后续更新（每次游戏补丁后）
php artisan fitting:sync-serenity  # 只需重新校正，不需要重新导入
```

---

## 八、前端 Dogma 计算引擎设计（JS 简化版）

适用于初期 MVP，覆盖 80% 场景：

```javascript
class DogmaEngine {
  constructor(typeData) {
    // typeData: { [typeId]: { attributes: {attrId: value}, effects: [...] } }
    this.data = typeData;
  }

  calculateFit(shipId, fittedModules) {
    // fittedModules: [ {typeId, state: 'online'|'active'|'offline'} ]

    const attrs = {};  // 最终属性值

    // Step 1: 加载舰船基础属性
    const shipAttrs = this.data[shipId].attributes;
    Object.assign(attrs, shipAttrs);

    // Step 2: 收集所有生效的 modifiers
    const modifiers = this.collectModifiers(shipId, fittedModules);

    // Step 3: 应用 9 步计算 + 堆叠惩罚
    return this.applyModifiers(attrs, modifiers);
  }

  collectModifiers(shipId, modules) {
    const modifiers = [];
    for (const mod of modules) {
      if (mod.state === 'offline') continue;
      const effects = this.data[mod.typeId].effects;
      for (const effect of effects) {
        if (!this.effectActiveInState(effect, mod.state)) continue;
        for (const modifier of effect.modifiers) {
          if (modifier.domain === 'shipID') {
            modifiers.push({
              targetAttrId: modifier.modifiedAttributeID,
              operator: modifier.operator,
              value: this.data[mod.typeId].attributes[modifier.modifyingAttributeID],
              sourceTypeId: mod.typeId,
            });
          }
        }
      }
    }
    return modifiers;
  }

  applyModifiers(baseAttrs, modifiers) {
    const result = { ...baseAttrs };
    const ORDER = ['PreAssign','PreMul','PreDiv','ModAdd','ModSub',
                   'PostMul','PostDiv','PostPercent','PostAssign'];

    for (const op of ORDER) {
      const group = modifiers.filter(m => m.operator === op);
      for (const [attrId, value] of Object.entries(result)) {
        const mods = group.filter(m => m.targetAttrId == attrId);
        if (!mods.length) continue;
        result[attrId] = this.applyGroup(value, mods, op);
      }
    }
    return result;
  }

  applyGroup(baseVal, mods, op) {
    const STACKABLE_OPS = ['PreMul','PreDiv','PostMul','PostDiv','PostPercent'];
    if (!STACKABLE_OPS.includes(op)) {
      // 不受堆叠惩罚，直接全部应用
      return mods.reduce((v, m) => this.applyOp(v, m.value, op), baseVal);
    }
    // 堆叠惩罚：正/负分组，按强度排序
    const pos = mods.filter(m => m.value >= 0).sort((a,b) => b.value - a.value);
    const neg = mods.filter(m => m.value < 0).sort((a,b) => a.value - b.value);
    let val = baseVal;
    for (let i = 0; i < pos.length; i++) {
      const penalized = pos[i].value * this.stackPenalty(i);
      val = this.applyOp(val, penalized, op);
    }
    for (let i = 0; i < neg.length; i++) {
      const penalized = neg[i].value * this.stackPenalty(i);
      val = this.applyOp(val, penalized, op);
    }
    return val;
  }

  stackPenalty(n) {
    return Math.exp(-Math.pow(n / 2.67, 2));
  }

  applyOp(val, src, op) {
    switch(op) {
      case 'PreAssign':
      case 'PostAssign':  return src;
      case 'PreMul':
      case 'PostMul':     return val * src;
      case 'PreDiv':
      case 'PostDiv':     return val / src;
      case 'ModAdd':      return val + src;
      case 'ModSub':      return val - src;
      case 'PostPercent': return val * (1 + src / 100);
    }
  }

  effectActiveInState(effect, state) {
    // effectCategory: 0=passive, 1=online, 2=active, 4=overload
    if (state === 'online')     return [0, 1].includes(effect.effectCategory);
    if (state === 'active')     return [0, 1, 2].includes(effect.effectCategory);
    if (state === 'overloaded') return [0, 1, 2, 4].includes(effect.effectCategory);
    return false;
  }
}
```

### 衍生属性计算（Pass 4，不在 Dogma 内）

```javascript
// EHP（有效 HP）计算
function calcEHP(attrs) {
  const shieldEHP = attrs[263] / Math.min(attrs[271], attrs[272], attrs[273], attrs[274]);
  const armorEHP  = attrs[265] / Math.min(attrs[267], attrs[268], attrs[269], attrs[270]);
  const hullEHP   = attrs[9];  // 结构无抗性（通常）
  return { shieldEHP, armorEHP, hullEHP, total: shieldEHP + armorEHP + hullEHP };
}

// 电容稳定性判断
function calcCapStability(attrs) {
  const capacity    = attrs[482];   // 电容总量（GJ）
  const rechargeMs  = attrs[55];    // 充能时间（ms）
  // 充能量 = capacity * 0.2 / (sqrt(capacitorCharge/capacity) * rechargeMs/5000)
  // 复杂，需要迭代计算，参考 Pyfa eos/capSim.py
}

// DPS 计算（需要武器类型特殊处理，此处省略）
```

---

## 九、Laravel 后端接口设计

### 路由

```php
// routes/web.php
Route::get('/fitting', [FittingController::class, 'index'])->name('fitting');

// routes/api.php
Route::prefix('fitting')->group(function () {
    Route::get('/ships',          [FittingController::class, 'ships']);
    Route::get('/types/{typeId}', [FittingController::class, 'typeData']);
    Route::get('/search',         [FittingController::class, 'search']);
    Route::post('/price',         [FittingController::class, 'price']);
    Route::middleware('auth')->group(function () {
        Route::get('/saved',           [FittingController::class, 'savedList']);
        Route::post('/save',           [FittingController::class, 'save']);
        Route::delete('/saved/{id}',   [FittingController::class, 'delete']);
    });
});
```

### API 响应格式

**GET /api/fitting/ships**
```json
{
  "groups": [
    {
      "category": "护卫舰",
      "types": [
        { "type_id": 587, "name": "Rifter", "name_cn": "裂谷号",
          "group_id": 25, "group_name": "突击型护卫舰",
          "icon_url": "https://image.evepc.163.com/Render/587_128.png" }
      ]
    }
  ]
}
```

**GET /api/fitting/types/{typeId}**
```json
{
  "type_id": 587,
  "name": "Rifter",
  "attributes": {
    "48": 130,
    "11": 41,
    "14": 3,
    "13": 3,
    "12": 4
  },
  "effects": [
    {
      "effect_id": 11,
      "effect_category": 0,
      "modifiers": [
        {
          "domain": "shipID",
          "func": "ItemModifier",
          "modifiedAttributeID": 76,
          "modifyingAttributeID": 1991,
          "operator": 5
        }
      ]
    }
  ],
  "slot_info": {
    "hi": 3, "med": 3, "low": 4, "rig": 3,
    "turret": 3, "launcher": 2
  }
}
```

**GET /api/fitting/search?q=装甲&slot=low&category=装备**
```json
{
  "results": [
    {
      "type_id": 11269,
      "name": "Damage Control II",
      "name_cn": "伤害控制 II",
      "slot": "low",
      "cpu": 0,
      "power": 0,
      "meta_level": 5
    }
  ]
}
```

### 数据库表（新增）

```sql
-- 用户保存的装配方案
CREATE TABLE user_fittings (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id     INTEGER NOT NULL,
    name        VARCHAR(100) NOT NULL,
    ship_type_id INTEGER NOT NULL,
    description TEXT,
    slots_json  TEXT NOT NULL,  -- JSON: [{type_id, flag, state}]
    created_at  DATETIME,
    updated_at  DATETIME,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- SDE 数据表（从 Fuzzwork 导入，只读）
-- fitting_types   ← 从 invTypes 提取子集
-- fitting_attrs   ← 从 dgmTypeAttributes 提取子集
-- fitting_effects ← 从 dgmEffects 提取（含解析后的 modifiers JSON）
-- fitting_type_effects ← 从 dgmTypeEffects 提取子集
```

---

## 十、开发阶段规划

### 阶段一：MVP（基础装配界面）
**目标**：可以装配，检查资源限制，显示基础属性（不含效果修正）

- [ ] SDE 导入命令 (`php artisan fitting:import-sde`)
- [ ] 国服 ESI 校正命令 (`php artisan fitting:sync-serenity`)
- [ ] `FittingDataService`：查询舰船/装备数据
- [ ] `/api/fitting/ships`：舰船列表 API
- [ ] `/api/fitting/types/{id}`：物品数据 API（属性+效果）
- [ ] `/api/fitting/search`：装备搜索 API
- [ ] 装配页面 Blade 模板（舰船选择器 + 槽位面板 + 属性面板）
- [ ] Alpine.js 装配交互逻辑（拖拽/点击装备到槽位）
- [ ] 基础资源检查（CPU/电网/槽位类型/校准值）

### 阶段二：Dogma 引擎
**目标**：装备效果实时计算，显示修正后的真实属性

- [ ] 前端 `fitting.js` Dogma 计算引擎（参考上方 JS 伪代码）
- [ ] 实现 9 步操作链 + 堆叠惩罚
- [ ] 支持 modifierInfo YAML 格式解析（服务端预处理存为 JSON）
- [ ] 属性面板更新：HP/抗性/速度/电容/锁定等
- [ ] EHP 计算（Pass 4 衍生属性）

### 阶段三：完整功能
**目标**：接近游戏内装配功能

- [ ] 技能等级设置 + 技能加成计算
- [ ] 弹药装载（影响武器 DPS/射程）
- [ ] 脚本（Script）装载（影响装备参数）
- [ ] 无人机舱管理
- [ ] DPS 计算（需要武器种类特殊处理）
- [ ] 电容稳定性计算（迭代算法）
- [ ] 装配保存/分享（URL 分享装配 code）
- [ ] 价格估算（复用现有 `getHullLowestSellPrice` + 装备市场查询）
- [ ] EFT 格式装配导入/导出（兼容 Pyfa 格式）

---

## 十一、已知难点与风险

| 难点 | 风险等级 | 说明 |
|------|---------|------|
| 旧式 expression tree 解析 | 高 | 部分效果仍用 dgmExpressions，递归解析复杂 |
| 电容稳定性计算 | 中 | 需要迭代模拟，参考 Pyfa capSim.py |
| DPS 计算 | 中 | 不同武器类型（炮台/导弹/无人机）公式不同 |
| 国服版本差异 | 中 | 关键舰船/装备属性值与欧服不同，需校正 |
| SDE 数据量 | 低 | 子集处理后可控，~5-15MB JSON |
| 技能加成 | 中 | 需要技能等级输入，完整技能树实现复杂 |

---

## 十二、参考资料

- EVEShipFit Dogma Engine（Rust/WASM）：`https://github.com/EVEShipFit/dogma-engine`
- EVEShipFit 数据转换器：`https://github.com/EVEShipFit/data`
- EVE-Fitting.js（纯 JS 实现）：`https://github.com/tiktrimo/EVE-Fitting.js`
- Pyfa 装配引擎（Python）：`https://github.com/pyfa-org/Pyfa`
- Fuzzwork SDE 下载：`https://www.fuzzwork.co.uk/dump/`
- EVE Ref 数据集（SDE+ESI 合并）：`https://ref-data.everef.net/`
- EVE 大学 堆叠惩罚：`https://wiki.eveuniversity.org/Stacking_penalties`
- 国服 ESI：`https://ali-esi.evepc.163.com/latest/`
