# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-10
**更新者**: [Claude Code]
**当前设备**: Aliyun ECS + 本地开发

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
| **装配模拟器** | 🔄 已进入阶段1重写：新版基础页开发中 | 最高 | [Codex] |
| **SDE-First 静态数据架构** | 📋 方案设计完成，待 Tus 拍板 D1-D8 决策项后启动 Phase 0 | 高（长期） | [待分配] |
| KM 图片生成器 | ✅ 功能完整，代码重构为 KillmailImageRenderer | 高 | [Qoder] |
| 战场报告功能 | ✅ 框架完成（阵营预览、ISK 千位分隔、KM 弹窗） | 高 | [Qoder] |
| P0代码优化 | ✅ 已完成（TokenService + StationNameService） | 最高 | [Qoder] |
| 修复通知接口 ESI universe/names 错误 | 🔄 待修复 | 高 | [待分配] |
| /notifications 目录权限错误 | ✅ 已修复（chmod 755） | 高 | [Claude Code] |

详细任务清单见 `tasks/active.md`

---

## 最近进展

| 日期 | 内容 | commit | 操作者 |
|------|------|--------|--------|
| 4-10 | 装配模拟器阶段1重写启动：替换旧版超大前端页，改为更简单的基础可用版骨架（选船/选槽/搜索装备/装卸/资源检查） | — | [Codex] |
| 4-10 | 服务器与 GitHub 同步：将服务器工作区 6 个未提交文件合入 main，备份 Qoder 11 个 commit 到 backup 分支 | `daa3c2d` | [Claude Code] |
| 4-10 | SDE-First 架构方案文档：扩展 Qoder 的 sde-data-architecture.md 为完整迁移方案 `knowledge/sde-first-architecture.md` | — | [Claude Code] |
| 4-09 | 装配模拟器：修复舰船势力分类展开慢/空白，修复装备安装误报“该槽位已满” | — | [Codex] |
| 4-09 | 装配模拟器：确认 `/fitting-simulator` 已部署 MVP，前端开始补基础属性实时联动 | — | [Codex] |
| 4-08 | /notifications 目录权限修复（drwx---rwx → 755） | — | [Claude Code] |
| 4-06 | KM 图片生成器大重构：拆分为 ImageRenderer/EnrichService/FilterService | `32f4cc3` | [Qoder] |
| 4-06 | 战场报告功能完成：BattleReportController/Service/View(871行) | `4034842` | [Qoder] |
| 4-06 | 字体升级：WQY ZenHei → Noto Sans SC | `32f4cc3` | [Qoder] |
| 4-05 | 总价值加入船体价格，多轮 KM 图片优化完成 | `e3aec36` | [Claude Code] |
| 4-02 | P0优化完成：TokenService 统一服务 + StationNameService 重构 | `2c6f586` | [Qoder] |
| 4-01 | 全面代码审查（6严重/8高危/10中等） | — | [Claude Code] |

---

## 服务器状态

| 项目 | 值 |
|------|-----|
| 最后成功部署 | 2026-04-05 [Claude Code via SSH] |
| 服务器当前 commit | `daa3c2d` (2026-04-10 [Claude Code] 同步) |
| Docker 容器 | ✅ 正常运行（含 GD FreeType 支持、Noto Sans SC 字体） |
| Swap | 2GB（已持久化） |
| HTTPS 证书 | ZeroSSL，90天，每日 14:01 自动续期 |
| /notifications 目录权限 | ✅ 已修复为 755 |

---

## KM 图片生成器技术要点

**文件**:
- `app/Services/Killmail/KillmailImageRenderer.php` — GD 图片绘制
- `app/Services/Killmail/KillmailEnrichService.php` — 数据丰富化
- `app/Services/Killmail/KillmailFilterService.php` — 数据筛选

**路由**: `GET /api/killmails/kill/{killId}/image?force=1`

**布局尺寸**:
- 画布 900px，头部 192px，统计栏 68px，底部 70px
- 参与者栏 360px，头像/舰船渲染 各 160px，攻击者行高 62px

**图片接口**: 国服 `https://image.evepc.163.com/`
- 头像: `/Character/{id}_{size}.jpg`
- 舰船渲染: `/Render/{id}_{size}.png`
- 物品/军团/联盟: `/Type|Corporation|Alliance/{id}_{size}.png`

**市场价格**: 国服 ali-esi 吉他区域（10000002）最低卖单，缓存 24h

**数据注意**:
- `supporters` 独立于 `attackers`，`damage_done=0` 的攻击者不是支援者
- `constellation_name` 和 `victim.position` 已加入 KillmailService 返回数据
- 物品总价值 = 装备价值 + 船体最低卖单

**字体**: Noto Sans SC (`storage/fonts/NotoSansSC-Regular.ttf` / `NotoSansSC-Bold.ttf`)

---

## 战场报告功能

**文件**:
- `app/Http/Controllers/Api/BattleReportController.php`
- `app/Services/BattleReport/BattleReportService.php` (515 行)
- `resources/views/battlereport/index.blade.php` (871 行)

**功能**:
- 搜索星系 KM 历史（指定时间范围）
- 阵营预览（蓝队 vs 红队）
- ISK 千位分隔符格式化
- KM 详情弹窗
- 击杀数统计
- 物品获取统计

**路由**:
```
GET  /battlereport
POST /api/battlereport/search (需验证参数：system_id, start_time, end_time, include_nearby)
```

---

## 缓存清理命令

```bash
# 清除生成的 KM 图片（改代码后必须执行）
rm -f storage/app/km-images/km_*.png

# 清除军团图标缓存（图标显示错误时）
rm -f storage/app/km-images/img_corporations_*.png

# 清除船体价格缓存
rm -f storage/app/km-images/km_hull_price_*.json
```

---

## ⚠️ 重要注意事项

1. **PHP 静态变量缓存**：修改 PHP 代码后需 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 不能删除，提交前用 git status 检查
4. **Token缓存闭包**：`Cache::remember` 闭包必须在内部获取最新Token，不可在外部获取后传入
5. **ESI datasource**：国服ESI所有调用必须带 `datasource=serenity`，且用 `ali-esi.evepc.163.com`
6. **海外SSH受限**：从海外网络连不上服务器（阿里云安全组限制），需从中国大陆网络操作
7. **多AI协作**：[Claude Code] 负责代码审查与修复（走GitHub），[Qoder] 负责部署与服务器操作
8. **Docker build 内存**：服务器已加 2GB swap，编译 PHP 扩展不再 OOM
9. **PHP GD 命名空间**：GD 函数在命名空间类中必须用 `use function imagettftext;` 等显式导入
10. **括号宽度测量**：用合并字符串测宽（`$this->tw($str . '（')`），不要逐字符累加，否则会重叠
11. **字体文件位置**：Noto Sans SC 位置为 `storage/fonts/NotoSansSC-Regular.ttf`（非 WQY ZenHei）

---

## 下一步计划

1. **继续装配模拟器开发**：验证前端基础属性联动，补更多常见模块规则 `[Codex] 2026-04-09`
   > [Tus] 2026-04-10 通过 [Claude Code] 反馈：装配模拟器一稿质量较差，可能整体重做，详见 ideas 或后续讨论
2. 排查 `fitting.sqlite` 中大量 `effects.modifiers` 为空的问题，决定是重导入还是增强解析 `[Codex] 2026-04-09`
3. 服务器部署 Qoder 的最新代码（`32f4cc3` — 战场报告 + KM 重构）
   > [Claude Code] 2026-04-10 更正：服务器已同步至 `daa3c2d`（包含 32f4cc3 + 后续 Qoder 工作 + 服务器未提交改动），此项已完成
4. 修复通知接口 `ESI request failed for universe/names` 错误（日志 ERROR）
   > [Claude Code] 2026-04-10：此问题在 SDE-First 改造的 Phase 2 后会自然消失（universe/names 改走 static.sqlite）
5. **SDE-First 静态数据架构改造**（长期，分 4 Phase） `[Claude Code] 2026-04-10`
   - 方案文档：`knowledge/sde-first-architecture.md`
   - 待 Tus 拍板 §9 决策清单（D1-D8）
   - Phase 0 预计 1-2 天，整体周期长，慢工出细活

---

## 装配模拟器快速参考

**技术文档**：`knowledge/fitting-simulator-spec.md`（完整规格，包含架构/数据库/API设计/代码示例）

**核心要点**：
- 数据来源：欧服 SDE（Fuzzwork MySQL dump）+ 国服 ESI 属性值校正
- Dogma 引擎：前端 JS 实现（可升级为 WASM），9 步操作链 + 堆叠惩罚
- 开源参考：EVEShipFit/dogma-engine（Rust/WASM），Pyfa/eos（Python）
- 抗性注意：SDE 存储共振值，转换：`抗性% = (1 - 共振值) × 100`
- 三阶段开发：①基础装配界面 → ②Dogma引擎 → ③完整功能

> [Codex] 2026-04-09：已确认服务器 `/fitting-simulator` 不是空白页，而是“公开页面 + SDE 查询 API + 基础资源检查”的 MVP。
> [Codex] 2026-04-09：当前主要缺口不是 UI，而是 `fitting.sqlite` 中大量模块 `effects.modifiers` 为空，暂时无法直接实现完整 Dogma。
