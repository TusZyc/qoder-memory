# 进行中的任务

**最后更新**: 2026-04-08
**更新者**: [Claude Code - 合并冲突]

---

## ✅ KM 图片生成器（功能完整，代码重构完成）

**状态**: ✅ 功能完整，代码架构优化完成
**最新 commit**: `32f4cc3` (战场报告功能优化修复)

### 已完成
- [x] `KillmailImageRenderer.php`：GD 图片绘制服务
- [x] `KillmailEnrichService.php`：数据丰富化服务
- [x] `KillmailFilterService.php`：数据筛选服务
- [x] 国服图片接口 `https://image.evepc.163.com/`
- [x] 头像与舰船渲染同尺寸（160px）
- [x] 角色名白色，军团/联盟图标并排（40px），名称在图标右侧
- [x] 舰船名后显示分类和船体价格（国服 ali-esi 吉他卖单最低价）
- [x] 位置格式：`星系（安等）< 星座 < 星域`，安等着色，合并字符串测宽避免重叠
- [x] 时间显示在舰船名与位置之间
- [x] 标题栏右上角：`由 Tus Esi System生成 Kill ID：xxx`
- [x] 统计栏：黑色背景，无多余边框和竖线
- [x] 支援信息从 ESI `supporters` 字段读取（非 damage_done=0 攻击者），格式 `N（总量）`
- [x] 参与者栏宽 360px，行高 62px，各行间距 6px，全白文字
- [x] 装备明细：无左侧颜色条，掉落物品深绿背景 `[5,55,18]`
- [x] 物品价格（xxx ISK）显示在数量左边，右对齐
- [x] 总价值 = 装备价值 + 船体最低卖单，右对齐绿色
- [x] 总掉落（仅掉落物品）右对齐淡蓝色
- [x] `KillmailService` 新增 `constellation_name`/`supporter_count`/`victim.position`
- [x] `/killmails` 页面 KM 图片弹窗（appended to body，z-index=9999）

### 待优化
- [ ] 部署到服务器（需同步 Docker 配置、新字体、新服务）

---

## ✅ 战场报告功能（框架完成）

**状态**: ✅ 基础功能完成，待服务器验证
**优先级**: 高
**负责**: [Qoder]
**commit**: `4034842` & `32f4cc3`

### 已完成
- [x] `BattleReportController.php` — 页面和 API 控制器
- [x] `BattleReportService.php` (515 行) — 战场数据聚合
- [x] `resources/views/battlereport/index.blade.php` (871 行) — 前端展示
- [x] 搜索星系 KM 历史（指定时间范围）
- [x] 阵营预览（蓝队 vs 红队对称布局）
- [x] ISK 千位分隔符格式化
- [x] KM 详情弹窗（从战报中查看单个 KM）
- [x] 击杀数统计
- [x] 物品获取统计

### 待办
- [ ] 服务器部署验证
- [ ] 性能测试（大时间跨度数据量）

---

## ✅ 服务器 OOM 恢复（已完成）

**状态**: ✅ 已完成
**日期**: 2026-04-05

- [x] Swap 扩容至 2GB（持久化）
- [x] Docker 镜像重建（GD + FreeType + JPEG + WebP）
- [x] 字体文件修复（WQY ZenHei → Noto Sans SC）
- [x] 验证 GD FreeType: `FreeType Support: 1`

---

## ✅ P0代码优化（已完成）

**状态**: ✅ 已完成
**优先级**: 最高
**负责**: [Qoder]
**commit**: `2c6f586`

### P0-1: Token重复查询问题
- [x] 创建 TokenService 统一Token获取服务
- [x] 同一请求周期内只查询数据库一次
- [x] 修改10个控制器使用TokenService替代重复查询
- [x] 预计减少约80%的Token数据库查询

### P0-2: 空间站名称代码重复问题
- [x] 增强 StationNameService 为统一服务
- [x] 本地数据优先 + ESI兜底 + 私有建筑处理
- [x] 重构 AssetDataService、MarketService、CharacterLocationController
- [x] 消除约200行重复代码

### 优化效果
| 页面 | 优化前 | 优化后 |
|-----|-------|-------|
| 钱包页面 `/wallet` | Token查询7次 | Token查询1次 |
| 邮件页面 `/mail` | Token查询5次 | Token查询1次 |
| 技能页面 `/skills` | Token查询3次 + 逐个查询 | Token查询1次 + 批量查询 |

---

## ✅ /notifications 目录权限修复（已完成）

**状态**: ✅ 已完成
**优先级**: 高
**负责**: [Claude Code]

### 问题
- 500 错误：View [notifications.index] not found
- 根本原因：`drwx---rwx` 目录权限，group 无读取权限
- PHP-FPM 容器内无法访问

### 解决
- [x] 执行 `chmod 755 /opt/eve-esi/resources/views/notifications/`
- [x] 执行 `docker exec eve-esi-app php artisan view:clear`
- [x] 验证通过：`docker exec eve-esi-app ls /var/www/html/resources/views/notifications/`

---

## 🔴 代码审查问题修复

**状态**: 🔄 进行中
**优先级**: 高
**负责**: [Claude Code]

### 背景
2026-04-01 [Claude Code] 对全项目进行了安全/性能/代码质量审查，发现 6 个严重、8 个高危、10+ 个中等问题。详见 `knowledge/code-review.md`。

### 严重问题修复清单
- [x] 添加 CSRF meta 标签到布局文件（`3e6a96a`）
- [ ] OAuth 添加 state 参数验证
- [ ] 替换 Tailwind CDN 为编译版本
- [ ] 压缩首页背景视频 (20MB → 2-3MB)
- [ ] Token 数据库存储加密
- [ ] 管理员验证改用角色 ID

### 高危问题修复清单
- [ ] 提取 getNames() 到共享服务
- [x] 修复缓存闭包中的 token 过期问题（`3e6a96a`）
- [ ] 舰队功能添加权限检查
- [x] 合同页面恢复正常缓存（`3e6a96a`）
- [x] 技能页面改为移除自动刷新（`3e6a96a`）
- [ ] 添加手机端响应式布局
- [ ] 错误信息脱敏

### 中等问题已修复
- [x] 补全 ESI datasource=serenity（`3e6a96a`）
- [x] TokenRefreshService 添加 Redis 锁防并发（`3e6a96a`）
- [x] 删除 CharacterController::refresh() 空方法（`3e6a96a`）
- [x] 新增通知 summary 轻量端点（`3e6a96a`）
- [x] 首页视频加载前星空动画（`3e6a96a`）

---

## 🔴 通知接口 ESI 错误

**状态**: 🔄 待修复
**优先级**: 最高
**负责**: [待分配]

### 问题
服务器日志报错：`ESI request failed for universe/names`
- 文件：`NotificationDataController.php` 第 422 行
- 在 `resolveNames()` 方法中调用 ESI `universe/names` 接口失败
- 错误被 `Cache::remember` 缓存了（1小时 TTL），会持续影响

### 日志原文
```
[2026-04-02 19:56:37] production.WARNING: ESI names 解析失败: ESI request failed for universe/names
[2026-04-02 19:56:37] production.ERROR: 🔔 [Notifications] 获取提醒数据失败
```

---

## 记忆系统重构

**状态**: ✅ 已完成
**优先级**: 高

### 完成情况
- [x] 创建目录结构
- [x] 创建 HANDOFF.md、knowledge/*.md、tasks/*.md
- [x] 推送到 GitHub
- [x] Claude Code 重构为多AI协作规范

---

## 🔵 装配模拟器（新功能，待开发）

**状态**: 🔵 技术调研完成，待开始开发
**优先级**: 最高
**技术文档**: `knowledge/fitting-simulator-spec.md`

### 阶段一（MVP）- 待开始
- [ ] SDE 数据导入命令 (`php artisan fitting:import-sde`)
- [ ] 国服 ESI 校正命令 (`php artisan fitting:sync-serenity`)
- [ ] `FittingDataService`：舰船/装备数据查询服务
- [ ] `/api/fitting/ships`：舰船列表 API
- [ ] `/api/fitting/types/{id}`：物品属性+效果 API
- [ ] `/api/fitting/search`：装备搜索 API
- [ ] 装配页面 Blade 模板（舰船选择器 + 槽位面板 + 属性面板）
- [ ] Alpine.js 装配交互（拖拽/点击装备入槽）
- [ ] 基础资源检查（CPU/电网/槽位/校准值）

### 阶段二 - 待阶段一完成
- [ ] 前端 Dogma 计算引擎（fitting.js）
- [ ] 9步操作链 + 堆叠惩罚实现
- [ ] 修正后属性实时显示
- [ ] EHP 计算

### 阶段三 - 待阶段二完成
- [ ] 技能加成、弹药/脚本、DPS计算、电容稳定性
- [ ] 装配保存/分享
- [ ] EFT 格式导入导出
- [ ] 价格估算

---

## 待办队列

### 最高优先级
- [ ] **开始装配模拟器阶段一开发** `[待分配]`
- [ ] 部署 Qoder 的最新代码（`32f4cc3`）到服务器 `[待操作]`
- [ ] 修复通知接口 `ESI request failed for universe/names` 错误 `[待分配]`

### 高优先级
- [ ] 继续修复代码审查严重问题（OAuth/CDN/视频压缩/Token加密） `[Claude Code]`
- [ ] 验证战场报告功能 `[待分配]`

### 中优先级
- [ ] 手机端响应式布局 `[Claude Code]`
- [ ] 修复代码审查高危问题（权限检查/脱敏等） `[Claude Code]`

### 低优先级
- [ ] CacheKeyService 统一使用
- [ ] 提取公共 JS/CSS 到共享文件

---

## 历史任务归档

### 2026-04-08 完成
- [x] /notifications 目录权限修复 (chmod 755) `[Claude Code]`
- [x] 合并两个 AI 的记忆冲突（HANDOFF.md/tasks/active.md） `[Claude Code]`

### 2026-04-06 完成
- [x] **KM 图片生成器架构重构**: 拆分为 ImageRenderer/EnrichService/FilterService `[Qoder]`
- [x] **战场报告功能完成**: BattleReportController/Service/View(871行) `[Qoder]`
- [x] **字体升级**: WQY ZenHei → Noto Sans SC `[Qoder]`

### 2026-04-05 完成
- [x] **KM图片生成器多轮优化**: 11项布局大改 + 视觉细节 + 数据修复 + 船体价格 `[Claude Code]` commits: `c3adbbc` `bc72a51` `4693124` `26bb20b` `e3aec36`
- [x] **服务器OOM恢复**: Swap 2GB + Docker 重建 + 字体修复 `[Claude Code]`

### 2026-04-04 完成
- [x] **KM图片生成器**: 完整实现 `[Claude Code]` commit: `4bc5e05`

### 2026-04-02 完成
- [x] **P0代码优化**: TokenService统一服务 + StationNameService重构（+1118/-382行）`[Qoder]`
- [x] 部署 `2c6f586` 到服务器（15个文件）`[Qoder]`
- [x] Claude Code 全面 Review + Bug 修复（20个文件，+622/-182）`[Claude Code]`
- [x] 部署 `3e6a96a` 到服务器 `[Qoder]`

### 2026-04-01 完成
- [x] 全面代码审查 `[Claude Code]`
- [x] GitHub 同步（4个虫洞优化文件）`[Qoder]`

### 2026-03-25 ~ 3-28 完成
- [x] 虫洞查询功能完整开发 `[Qoder]`
- [x] 虫洞页面多轮优化 `[Qoder]`

### 2026-03-24 完成
- [x] 修复 commit 59f0549 误删的数据文件 `[Qoder]`
- [x] 修复市场搜索静态文件加载 `[Qoder]`

### 2026-03-23 完成
- [x] 斥候工具（Scout）开发 `[Qoder]`

### 2026-03-22 完成
- [x] LP Store 价格计算模式 `[Qoder]`
- [x] Guide 使用指南页面 `[Qoder]`
