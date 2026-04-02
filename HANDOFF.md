# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-02 20:30
**更新者**: [Qoder]（基于 [Claude Code] 提交部署）
**当前设备**: 家里电脑

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
| 修复通知接口 ESI universe/names 错误 | 🔄 待修复 | 最高 | [待分配] |
| 修复代码审查剩余问题 | 🔄 进行中 | 高 | [Claude Code] |
| 验证市场搜索功能 | ⏳ 待验证 | 中 | [待分配] |

详细任务清单见 `tasks/active.md`

---

## 最近进展

| 日期 | 内容 | 操作者 |
|------|------|--------|
| 4-02 | 部署 `3e6a96a` 到服务器（git pull + 缓存清理） | [Qoder] |
| 4-02 | 全面修复：Token缓存闭包/通知summary端点/星空动画/CSRF/datasource/Redis锁 | [Claude Code] |
| 4-01 | 全面代码审查：发现6个严重、8个高危、10个中等问题 | [Claude Code] |
| 4-01 | 建立多AI协作记忆规范 | [Claude Code] |
| 4-01 | GitHub同步：提交4个虫洞优化文件 | [Qoder] |
| 3-28 | 虫洞功能优化（行星类型解析/搜索错误处理/KM限5条） | [Qoder] |
| 3-26 | 虫洞页面优化（侧边栏/样式/效果中文化/天体信息） | [Qoder] |
| 3-25 | 虫洞查询功能上线（2604星系/90类型/6效果） | [Qoder] |

---

## 服务器状态

| 项目 | 值 |
|------|-----|
| 最后部署 | 2026-04-02 [Qoder] |
| 最后commit | `3e6a96a` (fix: 全面修复Token缓存/通知性能/星空效果/安全优化) |
| 待部署变更 | 无 |
| Docker容器 | eve-esi-app / eve-esi-nginx / eve-esi-redis ✅ 运行中 |
| HTTPS证书 | ZeroSSL，90天，每日14:01自动续期 |
| GitHub同步 | ✅ 已同步至 `3e6a96a` |

---

## ⚠️ 重要注意事项

1. **PHP 静态变量缓存**：修改 PHP 代码后需 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 不能删除，提交前用 git status 检查
4. **虫洞行星类型**：使用 `/universe/types/{type_id}/?language=zh` 获取中文名，已缓存30天
5. **Token缓存闭包**：`Cache::remember` 闭包必须在内部获取最新Token，不可在外部获取后传入
6. **ESI datasource**：国服ESI所有调用必须带 `datasource=serenity`
7. **海外SSH受限**：从海外网络连不上服务器（阿里云安全组限制），需从中国大陆网络操作
8. **多AI协作**：[Claude Code] 负责代码审查与修复（走GitHub），[Qoder] 负责部署与服务器操作

---

## 下一步计划

1. 修复通知接口 `ESI request failed for universe/names` 错误（日志 ERROR）
2. 修复代码审查剩余问题（详见 `knowledge/code-review.md`）
3. 验证市场搜索功能
4. 测试后台管理缓存功能
