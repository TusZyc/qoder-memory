# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-05
**更新者**: [Claude Code]
**当前设备**: 用户本地

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
| KM 图片生成器开发 | ✅ 功能完成，布局持续优化中 | 高 | [Claude Code] |
| 服务器 Docker 重建（OOM后恢复） | ✅ 已完成 | 最高 | [Qoder/Claude Code] |
| 修复通知接口 ESI universe/names 错误 | ✅ 已完成 | 高 | [Claude Code] |
| 验证市场搜索功能 | ⏳ 待验证 | 中 | [待分配] |

详细任务清单见 `tasks/active.md`

---

## 最近进展

| 日期 | 内容 | 操作者 |
|------|------|--------|
| 4-05 | KM图片布局优化：国服图片接口/军团联盟上下排列/物品名白色/去掉红背景，推送 `8b81605` | [Claude Code] |
| 4-05 | 服务器 OOM 恢复：Swap 扩至 2GB，Docker 镜像重建成功（GD FreeType 支持） | [Claude Code] |
| 4-05 | 字体文件修复：原文件是 HTML 假文件，替换为 WQY ZenHei（17MB 真字体） | [Claude Code] |
| 4-04 | KM图片生成器代码完成，推送 `4bc5e05`；服务器部署受阻（Docker build OOM） | [Claude Code] |
| 4-03 | UI修复：页面标题/合同NaN/服务器维护检测/装配搜索，部署 `cde8d06` | [Claude Code] |
| 4-02 | **P0优化完成**: TokenService统一服务 + StationNameService重构，减少80%重复查询 | [Qoder] |

---

## 服务器状态

| 项目 | 值 |
|------|-----|
| 最后成功部署 | 2026-04-05 [Claude Code via SSH] |
| 项目最新 commit | `8b81605` (feat: 优化 KM 图片布局) |
| 服务器当前 commit | `cde8d06`（代码文件单独上传，未 git pull） |
| Docker容器 | ✅ 正常运行（新镜像，含 GD FreeType 支持） |
| Swap | 2GB（已持久化） |
| HTTPS证书 | ZeroSSL，90天，每日14:01自动续期 |
| GitHub同步 | ✅ 已同步至 `8b81605` |

---

## KM 图片生成器技术要点

**文件**: `app/Services/Killmail/KillmailImageService.php`

**图片接口**: 国服 `https://image.evepc.163.com/`
- 头像: `/Character/{id}_{size}.jpg`
- 舰船渲染: `/Render/{id}_{size}.png`
- 物品图标: `/Type/{id}_{size}.png`
- 军团图标: `/Corporation/{id}_{size}.png`
- 联盟图标: `/Alliance/{id}_{size}.png`

**字体**: WQY ZenHei（`/opt/eve-esi/storage/fonts/NotoSansSC-Regular/Bold.ttf`，实际是 wqy-zenhei.ttc）

**路由**: `GET /api/killmails/kill/{killId}/image?force=1`（`force=1` 强制重新生成）

**本地物品名**: 优先从 `data/eve_items.json` 查询中文名，API 数据兜底

**布局**: 头像(128) | 舰船渲染(160) | 文字信息 → 统计栏 → [参与者(310px) | 装备(590px)] → 总价值

---

## ⚠️ 重要注意事项

1. **PHP 静态变量缓存**：修改 PHP 代码后需 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 不能删除，提交前用 git status 检查
4. **Token缓存闭包**：`Cache::remember` 闭包必须在内部获取最新Token，不可在外部获取后传入
5. **ESI datasource**：国服ESI所有调用必须带 `datasource=serenity`
6. **海外SSH受限**：从海外网络连不上服务器（阿里云安全组限制），需从中国大陆网络操作
7. **多AI协作**：[Claude Code] 负责代码审查与修复（走GitHub），[Qoder] 负责部署与服务器操作
8. **Docker build 内存**：服务器已加 2GB swap，编译 PHP 扩展不再 OOM
9. **PHP GD 命名空间**：GD函数在命名空间类中必须用 `use function imagettftext;` 等显式导入
10. **KM图片缓存清除**：换接口/改代码后需 `rm -f storage/app/km-images/img_*.png` 清图片缓存

---

## 下一步计划

1. 继续优化 KM 图片布局细节（用户反馈驱动）
2. 验证市场搜索功能
3. 继续修复代码审查剩余问题（详见 `knowledge/code-review.md`）
