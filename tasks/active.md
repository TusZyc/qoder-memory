# 进行中的任务

**最后更新**: 2026-03-24 23:30

---

## 记忆系统重构

**状态**: ✅ 完成中  
**优先级**: 高

### 目标
建立结构化的记忆系统，支持多设备无缝切换开发

### 新目录结构
```
qoder-memory/
├── HANDOFF.md                 # 每次切换必看
├── README.md                  # 使用指南
├── knowledge/
│   ├── pitfalls.md            # 踩坑记录
│   ├── project-overview.md    # 项目概览
│   └── deployment.md          # 部署手册
├── tasks/
│   ├── active.md              # 当前任务（本文件）
│   └── backlog.md             # 待办队列
├── ideas/
│   └── admin-panel-design.md  # 设计文档
└── archive/
    └── logs/                  # 历史日志
```

### 完成情况
- [x] 创建目录结构
- [x] 创建 HANDOFF.md
- [x] 创建 knowledge/pitfalls.md
- [x] 创建 knowledge/project-overview.md
- [x] 创建 knowledge/deployment.md
- [x] 创建 tasks/active.md
- [ ] 推送到 GitHub
- [ ] 验证完整性

---

## 市场搜索优化验证

**状态**: 🔄 待验证  
**优先级**: 中

### 背景
2026-03-24 修复了市场搜索在 Redis 缓存清空后超时的问题

### 修改内容
- 文件：`app/Services/MarketService.php`
- 方法：`getMarketGroupsTree()`
- 改动：优先从静态文件 `public/data/market_groups.json` 加载

### 验证步骤
1. 访问 https://51-eve.online/market
2. 在搜索框输入物品名称
3. 确认能正常显示搜索结果
4. 选择分类，确认能加载物品列表

### 相关 Commit
- `31e9fab` - fix: getMarketGroupsTree优先从静态文件加载避免ESI超时

---

## 待办队列

### 高优先级
- [ ] 验证市场搜索功能

### 中优先级
- [ ] 测试后台管理缓存功能
- [ ] Redis 缓存预热优化

### 低优先级
- [ ] CacheKeyService 采用率提升（当前 53%）
- [ ] 缓存键命名风格统一

---

## 历史任务归档

### 2026-03-24 完成
- [x] 修复 commit 59f0549 误删的数据文件
- [x] 修复市场搜索静态文件加载

### 2026-03-23 完成
- [x] 斥候工具（Scout）开发

### 2026-03-22 完成
- [x] LP Store 价格计算模式
- [x] Guide 使用指南页面
