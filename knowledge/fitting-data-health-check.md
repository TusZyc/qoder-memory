# 装配模拟器数据健康检查

## 核心原则
- 装配规则以官方 SDE 和自建装配 SDE 为主要依据。
- 第三方整理数据可以参考，但不作为最终规则来源。
- 不只凭页面表现猜规则，优先回到 SDE 里查字段和效果。

## 当前数据来源
- 本地官方 SDE 压缩包：
  - `D:\codex-work\official-sde\eve-online-static-data-latest-jsonl.zip`
- 服务器自建装配子集：
  - `/opt/eve-esi/database/fitting_official.sqlite`
- 装配模拟器当前的装备浏览、搜索、舰船详情，已经优先读取自建官方 SDE 子集。

## 常用检查脚本
- 主健康检查：
  - `scripts/fitting_data_health_check.php`
- 重点探查：
  - `scripts/fitting_data_probe.php`
- 服务器示例：
  - `docker exec eve-esi-app php scripts/fitting_data_probe.php database/fitting_official.sqlite`

## 已确认的舰船字段
- `12 / lowSlots`：低槽数量。
- `13 / medSlots`：中槽数量。
- `14 / hiSlots`：高槽数量。
- `48 / cpuOutput`：舰船 CPU。
- `11 / powerOutput`：舰船能量栅格。
- `101 / launcherSlotsLeft`：发射器硬点。
- `102 / turretSlotsLeft`：炮台硬点。
- `1132 / upgradeCapacity`：校准值上限。
- `1137` 和 `1154`：改装槽数量。
- `1547 / rigSize`：改装件尺寸。
- `283 / droneCapacity`：无人机舱容量。
- `1271 / droneBandwidth`：无人机带宽。
- `1785 / isCapitalSize`：旗舰体量标记。

## 已确认的装备字段
- 槽位效果：
  - `loPower`：低槽。
  - `medPower`：中槽。
  - `hiPower`：高槽。
  - `rigSlot`：改装件。
  - `subSystem`：子系统。
- 硬点效果：
  - `launcherFitted`：占用发射器硬点。
  - `turretFitted`：占用炮台硬点。
- 资源消耗：
  - `50 / cpu`：CPU 占用。
  - `30 / power`：能量栅格占用。
  - `1153 / upgradeCost`：校准值占用。
- 尺寸相关：
  - `1547 / rigSize`：改装件硬性尺寸规则。
  - `128 / chargeSize`：弹药或武器尺寸分类，不能单独当作通用装配合法性规则。
- 无人机：
  - 物品 `volume`：无人机舱占用体积。
  - `1272 / droneBandwidthUsed`：无人机带宽占用。
- 技能：
  - `182`、`183`、`184`：所需技能 ID。
  - `277`、`278`、`279`：所需技能等级。
- 舰船限制：
  - `canFitShipGroup*`：只能安装到指定舰船分组。
  - `canFitShipType*`：只能安装到指定具体舰船。
- 安装数量限制：
  - `1544 / maxGroupFitted`：同组最多安装数量。
  - `2431 / maxTypeFitted`：同一具体装备最多安装数量。
  - `763 / maxGroupActive`：同组最多激活数量。
  - `978 / maxGroupOnline`：同组最多在线数量。

## 重要规则结论
- 改装件尺寸是硬规则：舰船改装件尺寸必须和改装件自身尺寸一致。
- 武器的 `chargeSize` 不能单独决定能不能装，它更适合用于弹药匹配、武器分组、后续 DPS 计算。
- 硬点不参与“适合当前舰船”和“资源足够”两个筛选。
- 硬点只在用户真正安装炮台或发射器时提示并阻止。
- 未勾选“资源足够”时，允许 CPU、能量栅格、校准值等超上限，右侧资源面板显示“超上限”提醒。
- 页面用词统一为游戏内说法：“能量栅格”，不再使用“电网”。

## 已验证样例
- 裂谷级 / `587`：
  - 高槽 `3`，中槽 `3`，低槽 `4`，改装槽 `3`。
  - 炮台硬点 `3`，发射器硬点 `2`。
  - 改装件尺寸 `1`，非旗舰。
- 长须鲸级 / `28352`：
  - 旗舰舰船。
  - 改装件尺寸 `4`。
  - 无人机舱 `8800`。
- 泽尼塔 / `52907`：
  - 旗舰改装件尺寸 `4`。
  - 对应超大型先驱者武器 `52915`。
  - `Ultratidal Entropic Disintegrator I` 具有 `chargeSize = 4` 和 `canFitShipType = 52907`。
- 压缩装备：
  - 海豚级可见 2 个中型压缩装备。
  - 逆戟鲸级可见 5 个大型压缩装备。
  - 长须鲸级可见 5 个旗舰压缩装备。
  - 裂谷级在“适合当前舰船”筛选下看不到压缩装备。
- 指挥脉冲：
  - 裂谷级看不到。
  - 海豚级、逆戟鲸级、长须鲸级都能看到对应指挥脉冲。
- 损伤控制 I：
  - `group_id = 60`
  - `max_group_fitted = 1`
- 脉冲激活式关联全能核心：
  - `max_type_fitted = 1`
- 地精灵无人机：
  - 体积 `5`
  - 带宽 `5`

## 当前风险记录
- 一些旧物品、NPC 物品、废案物品已经被排除在普通装配列表外。
- 一些旗舰级或超大型装备没有明确的 `canFitShipGroup*` 或 `canFitShipType*`，所以当前额外使用“旗舰体量”规则兜底。
- `maxGroupOnline` 和 `maxGroupActive` 暂未实现，等后续有装备在线/激活状态后再做。

## 后续开发优先级
- 优先处理明确的舰船类型/分组限制。
- 再处理槽位、硬点、资源、改装件尺寸、无人机舱和带宽。
- 技能筛选需要角色技能数据，放到后续阶段。
- DPS、抗性等计算可以继续推进，但需要明确当前先使用“无技能基础值”。
