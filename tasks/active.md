# 进行中的任务

**最后更新**: 2026-04-05
**更新者**: [Claude Code]

---

## ✅ KM 图片生成器（功能完整，持续优化）

**状态**: ✅ 功能完整，已经过多轮布局优化
**最新 commit**: `e3aec36`

### 已完成
- [x] `KillmailImageService.php`：服务端 PHP GD 图片生成
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

### 待优化（用户反馈驱动）
- [ ] 布局细节继续调整

---

## ✅ 服务器 OOM 恢复（已完成）

**状态**: ✅ 已完成
**日期**: 2026-04-05

- [x] Swap 扩容至 2GB（持久化）
- [x] Docker 镜像重建（GD + FreeType + JPEG + WebP）
- [x] 字体文件修复（WQY ZenHei 17MB 真字体替换假 HTML 文件）
- [x] 验证 GD FreeType: `FreeType Support: 1`

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
- [ ] 手机端响应式布局 `[Claude Code]`

### 低优先级
- [ ] CacheKeyService 统一使用
- [ ] 提取公共 JS/CSS 到共享文件

---

## 历史任务归档

### 2026-04-05 完成
- [x] **KM图片生成器多轮优化**: 11项布局大改 + 视觉细节 + 数据修复 + 船体价格 `[Claude Code]` commits: `c3adbbc` `bc72a51` `4693124` `26bb20b` `e3aec36`
- [x] **服务器OOM恢复**: Swap 2GB + Docker 重建 + 字体修复 `[Claude Code]`
- [x] **KM图片布局初版优化**: 国服接口 + 军团联盟上下排 + 物品名白色 `[Claude Code]` commit: `8b81605`

### 2026-04-04 完成
- [x] **KM图片生成器**: 完整实现 `[Claude Code]` commit: `4bc5e05`

### 2026-04-02~03 完成
- [x] P0代码优化 `[Qoder]`
- [x] Claude Code 全面 Review + Bug 修复 `[Claude Code]`
- [x] 部署 `cde8d06` 到服务器 `[Claude Code]`
