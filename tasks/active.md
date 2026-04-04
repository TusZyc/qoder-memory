# 进行中的任务

**最后更新**: 2026-04-04
**更新者**: [Claude Code]

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

## ✅ 通知接口 ESI 错误（已修复）

**状态**: ✅ 已完成
**负责**: [Claude Code]
**commit**: `cde8d06`

- [x] 修复 `NotificationDataController.php` 第 422 行 ESI universe/names 调用错误
- [x] 公共端点移除不必要 token 认证
- [x] ESI names 批次 404 自动拆分重试

---

## 🚧 KM 图片生成器（进行中，待服务器恢复）

**状态**: 🚧 代码已完成，服务器部署受阻（OOM）
**优先级**: 中
**负责**: [Claude Code]
**commit**: `4bc5e05`

### 已完成
- [x] 布局设计：参考游戏内击毁报告 UI（900px 宽，仿 EVE 深色主题）
- [x] `KillmailImageService.php`（843行）：PHP GD 图片生成服务
  - 标题栏 + 角色头像/舰船渲染 + 参与者列表 + 装备明细（按槽位分组）
  - 从 EVE 图片服务器获取头像/图标/渲染图（带本地缓存）
  - NotoSansSC TTF 字体，字体文件已上传到服务器 `storage/fonts/`
  - "最后一击"/"造成伤害最多" 参与者分组
- [x] `KillmailController` 新增 `killImage()` 接口
- [x] 路由：`GET /api/killmails/kill/{id}/image`
- [x] `/killmails` 页面右栏新增"生成 KM 图片" UI（输入框+预览+下载）
- [x] 修复 PHP 命名空间下 GD 函数调用问题（添加 `use function` 声明）
- [x] 修改 `Dockerfile`：加入 FreeType/JPEG/WebP 依赖 + 阿里云镜像源

### 待完成（需服务器恢复）
- [ ] 服务器恢复 + 加 Swap（Tus 操作）
- [ ] 重建 Docker 镜像（已换阿里云源，约3-5分钟）
- [ ] 验证 GD FreeType 支持：`php -r "print_r(gd_info());"`
- [ ] 测试接口 `GET /api/killmails/kill/22460248/image`

### 待优化（功能验证后）
- [ ] 研究如何更接近游戏内截图风格（间距、字体大小等微调）
- [ ] 图片生成加水印/生成者信息
- [ ] 考虑接入 WebSocket 实时通知 KM

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

## 待办队列

### 高优先级
- [ ] **恢复服务器**（Tus 操作，见 HANDOFF.md 恢复步骤）
- [ ] 验证 KM 图片生成接口 `[Claude Code]`
- [ ] 修复代码审查剩余严重问题 `[Claude Code]`

### 中优先级
- [ ] 验证市场搜索功能 `[Qoder]`
- [ ] 修复代码审查中等问题 `[Claude Code]`
- [ ] 测试后台管理缓存功能 `[待分配]`
- [ ] Redis 缓存预热优化 `[待分配]`

### 低优先级
- [ ] CacheKeyService 统一使用
- [ ] 提取公共 JS/CSS 到共享文件
- [ ] KM 图片生成器后续优化（水印、WebSocket 通知）

---

## 历史任务归档

### 2026-04-04 完成
- [x] KM 图片生成器代码实现（`4bc5e05`）`[Claude Code]`
- [x] 上传字体文件 NotoSansSC 到服务器 `[Claude Code]`
- [x] Dockerfile 换阿里云镜像源 + FreeType 支持 `[Claude Code]`

### 2026-04-03 完成
- [x] 多项 UI 修复（页面标题/合同NaN/服务器维护/装配搜索）`[Claude Code]`
- [x] 部署 `cde8d06` `[Claude Code]`

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
