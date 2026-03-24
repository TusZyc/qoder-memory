# Qoder 记忆系统

**项目负责人**: 图斯（Tus）  
**GitHub**: TusZyc

---

## 目录结构

```
qoder-memory/
├── HANDOFF.md                 # 【每次切换设备必看】当前状态
├── README.md                  # 本文件 - 使用指南
│
├── knowledge/                 # 【知识库】
│   ├── pitfalls.md            # 踩坑记录与调试经验
│   ├── project-overview.md    # 项目功能概览
│   └── deployment.md          # 部署与运维手册
│
├── tasks/                     # 【任务管理】
│   ├── active.md              # 进行中的任务详情
│   └── backlog.md             # 待办队列
│
├── ideas/                     # 【设计文档】
│   └── admin-panel-design.md  # 后台管理设计方案
│
└── archive/                   # 【归档】
    └── logs/                  # 历史日志
```

---

## 使用流程

### 切换设备开始开发

1. **拉取最新记忆**
   ```bash
   cd d:\Qoder-work\qoder-memory
   git pull
   ```

2. **让 Qoder 读取交接文件**
   > "请阅读 HANDOFF.md 和 tasks/active.md，恢复开发上下文"

3. **如果遇到技术问题**
   > "请查阅 knowledge/pitfalls.md 中的相关经验"

### 结束开发准备切换

1. **让 Qoder 更新记忆文件**
   > "请更新 HANDOFF.md 和 tasks/active.md，总结今天的开发内容"

2. **如果遇到新问题并解决**
   > "请将这个问题和解决方案记录到 knowledge/pitfalls.md"

3. **提交并推送**
   ```bash
   cd d:\Qoder-work\qoder-memory
   git add -A
   git commit -m "sync: [设备] [日期]"
   git push
   ```

---

## 快速参考

| 需求 | 查阅文件 |
|------|---------|
| 当前在做什么 | `HANDOFF.md` |
| 任务详情 | `tasks/active.md` |
| 遇到技术问题 | `knowledge/pitfalls.md` |
| 了解项目功能 | `knowledge/project-overview.md` |
| 部署和运维 | `knowledge/deployment.md` |
| 设计思路 | `ideas/` 目录 |

---

## 活跃项目

### EVE ESI Tools (eve-esi-qoder)
- **仓库**: https://github.com/TusZyc/eve-esi-qoder
- **域名**: https://51-eve.online
- **技术栈**: Laravel 10 + PHP 8.2 + Blade + Tailwind CSS + Alpine.js + Redis
- **部署**: Docker Compose → 阿里云 ECS (47.116.125.182)

---

## 服务器速查

```bash
# SSH 连接
ssh -i "d:\Qoder-work\.qoder\openclaw.pem" root@47.116.125.182

# 项目目录
cd /opt/eve-esi

# 清理缓存
docker compose exec -T app php artisan cache:clear

# 重启应用
docker restart eve-esi-app
```
