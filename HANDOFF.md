# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-05
**更新者**: [Claude Code]
**当前设备**: 用户本地

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
| KM 图片生成器开发 | ✅ 功能完整，布局多轮优化完成 | 高 | [Claude Code] |
| 服务器 Docker 重建（OOM后恢复） | ✅ 已完成 | 最高 | [Qoder/Claude Code] |
| 验证市场搜索功能 | ⏳ 待验证 | 中 | [待分配] |

详细任务清单见 `tasks/active.md`

---

## 最近进展

| 日期 | 内容 | commit | 操作者 |
|------|------|--------|--------|
| 4-05 | 总价值加入船体价格 | `e3aec36` | [Claude Code] |
| 4-05 | 船体价格修复为国服 ESI 端点（ali-esi.evepc.163.com） | `26bb20b` | [Claude Code] |
| 4-05 | 8项视觉优化：统计栏黑背景/去左侧色条/行距加大/颜色调整 | `4693124` | [Claude Code] |
| 4-05 | supporters/position字段修复：正确解析官方ESI数据 | `bc72a51` | [Claude Code] |
| 4-05 | KM图片11项布局大改（头像160/船体价格/位置格式/宽参与者栏等） | `c3adbbc` | [Claude Code] |
| 4-05 | KM图片布局优化：国服接口/军团联盟上下排/物品名白色 | `8b81605` | [Claude Code] |
| 4-05 | 服务器 OOM 恢复：Swap 2GB + Docker 重建 + 字体修复 | — | [Claude Code] |
| 4-03 | UI修复：页面标题/合同NaN/服务器维护检测/装配搜索 | `cde8d06` | [Claude Code] |

---

## 服务器状态

| 项目 | 值 |
|------|-----|
| 最后成功部署 | 2026-04-05 [Claude Code via SSH] |
| 服务器当前 commit | `e3aec36` |
| Docker容器 | ✅ 正常运行（含 GD FreeType 支持） |
| Swap | 2GB（已持久化） |
| HTTPS证书 | ZeroSSL，90天，每日14:01自动续期 |

---

## KM 图片生成器技术要点

**文件**: `app/Services/Killmail/KillmailImageService.php`

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

**字体**: WQY ZenHei（`storage/fonts/NotoSansSC-Regular/Bold.ttf`，实为 .ttc）

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
9. **PHP GD 命名空间**：GD函数在命名空间类中必须用 `use function imagettftext;` 等显式导入
10. **括号宽度测量**：用合并字符串测宽（`$this->tw($str . '（')`），不要逐字符累加，否则会重叠

---

## 下一步计划

1. 继续优化 KM 图片布局细节（用户反馈驱动）
2. 验证市场搜索功能
3. 继续修复代码审查剩余问题（详见 `knowledge/code-review.md`）
