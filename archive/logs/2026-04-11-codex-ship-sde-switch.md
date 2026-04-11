# 2026-04-11 Codex 舰船数据切换到自建 SDE 记录

## 本次目标
- 减少混用数据源带来的问题。
- 把舰船浏览和舰船详情也迁移到自建官方 SDE 子集。

## 已完成修改
- 在 `App\Services\FittingOfficialDataService` 中新增舰船方法：
  - `getShipCategoryTree`
  - `getShipsByCategoryPath`
  - `getShipDetails`
- 修改 `FittingSimulatorDataController`：
  - 舰船分类树优先读取 `fitting_official.sqlite`。
  - 分类下舰船列表优先读取 `fitting_official.sqlite`。
  - 舰船详情优先读取 `fitting_official.sqlite`。
- 如果自建 SDE 不可用，仍然回退到旧装配数据库。

## 为什么要做
- 装备数据已经切到自建 SDE。
- 舰船数据如果还主要来自旧库，就容易出现数据不一致。
- 可能影响的地方包括：
  - 槽位数量。
  - 硬点数量。
  - 改装件尺寸。
  - 无人机舱。
  - 旗舰体量标记。

## 服务器验证
- 裂谷级 / `587`：
  - 高槽 `3`
  - 中槽 `3`
  - 低槽 `4`
  - 改装槽 `3`
  - 炮台硬点 `3`
  - 发射器硬点 `2`
  - CPU `130`
  - 能量栅格 `41`
  - 改装件尺寸 `1`
  - 非旗舰舰船
- 长须鲸级 / `28352`：
  - 分组 `883`
  - 旗舰舰船
  - 改装件尺寸 `4`
  - 无人机舱 `8800`
- 标准米玛塔尔护卫舰路径返回 6 艘舰船，其中包含裂谷级。
- “裂谷级不能看到超大型 / 旗舰装备”的检查仍然正常。
