# 进行中的任务

**最后更新**: 2026-04-05
**更新者**: [Claude Code]

---

## ✅ KM 图片生成器（功能完成，持续优化）

**状态**: ✅ 功能完整，布局已经过两轮优化
**commit**: `8b81605`

### 已完成
- [x] `KillmailImageService.php`：服务端 PHP GD 图片生成
- [x] 国服图片接口 `https://image.evepc.163.com/`
- [x] 布局：头像→舰船渲染→文字信息（军团/联盟上下排列）
- [x] 参与者列表：最后一击/最高伤害置顶，[头像][舰船+武器图标][文字]
- [x] 装备明细：掉落物品绿色背景，销毁物品无特殊背景，名称统一白色
- [x] 本地物品名：优先 `data/eve_items.json`，API 兜底
- [x] 舰船名格式：`舰船名（舰船类型）`
- [x] 路由支持 `?force=1` 强制重新生成
- [x] `/killmails` 页面添加 KM 图片生成面板

### 待优化（根据用户反馈）
- [ ] 布局细节继续调整

---

## ✅ 服务器 OOM 恢复（已完成）

**状态**: ✅ 已完成
**日期**: 2026-04-05

- [x] Swap 扩容至 2GB（持久化）
- [x] Docker 镜像重建（GD + FreeType + JPEG + WebP）
- [x] 字体文件修复（WQY ZenHei 17MB 真字体替换假 HTML 文件）
- [x] 验证 GD FreeType: `FreeType Support: 1`
- [x] KM 图片生成接口测试通过

---

## ✅ P0代码优化（已完成）

**状态**: ✅ 已完成
**commit**: `2c6f586`

- [x] TokenService 统一服务，减少约80%Token数据库查询
- [x] StationNameService 重构，消除约200行重复代码

---

## 🔄 代码审查问题修复（进行中）

**状态**: 🔄 部分完成
**负责**: [Claude Code]

### 严重问题
- [x] 添加 CSRF meta 标签
- [ ] OAuth 添加 state 参数验证
- [ ] 替换 Tailwind CDN 为编译版本
- [ ] 压缩首页背景视频 (20MB → 2-3MB)
- [ ] Token 数据库存储加密
- [ ] 管理员验证改用角色 ID

### 高危问题
- [x] 修复缓存闭包中的 token 过期问题
- [x] 合同页面恢复正常缓存
- [x] 技能页面移除自动刷新
- [ ] 舰队功能添加权限检查
- [ ] 添加手机端响应式布局

---

## 待办队列

### 高优先级
- [ ] 继续优化 KM 图片布局（用户反馈驱动）`[Claude Code]`
- [ ] 验证市场搜索功能 `[Qoder]`
- [ ] 修复代码审查剩余严重问题 `[Claude Code]`

### 中优先级
- [ ] KM 图片：添加图片压缩优化（当前 ~600KB）`[Claude Code]`
- [ ] 手机端响应式布局 `[Claude Code]`

### 低优先级
- [ ] CacheKeyService 统一使用
- [ ] 提取公共 JS/CSS 到共享文件

---

## 历史任务归档

### 2026-04-05 完成
- [x] **服务器OOM恢复**: Swap 2GB + Docker 重建 + 字体修复 `[Claude Code]`
- [x] **KM图片布局优化**: 国服接口 + 军团联盟上下排 + 物品名白色 `[Claude Code]`

### 2026-04-04 完成
- [x] **KM图片生成器**: 完整实现 `[Claude Code]`
- [x] GitHub 推送 `4bc5e05` `[Claude Code]`

### 2026-04-02~03 完成
- [x] P0代码优化 `[Qoder]`
- [x] Claude Code 全面 Review + Bug 修复 `[Claude Code]`
- [x] 部署 `cde8d06` 到服务器 `[Claude Code]`
