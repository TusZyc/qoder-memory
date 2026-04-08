# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-08
**更新者**: [Claude Code - 合并冲突]
**当前设备**: Aliyun ECS + 本地开发

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
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
| 服务器当前 commit | `e3aec36`（待部署 Qoder 的 `32f4cc3`） |
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

1. 服务器部署 Qoder 的最新代码（`32f4cc3` — 战场报告 + KM 重构）
2. 验证战场报告功能和 KM 图片生成
3. 修复通知接口 `ESI request failed for universe/names` 错误（日志 ERROR）
4. 继续修复代码审查剩余问题（详见 `knowledge/code-review.md`）
