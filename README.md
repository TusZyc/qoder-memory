# ⚠️ AI 开发协作记忆系统 — 必读

> **你正在阅读一个由多个 AI 工具协作开发的项目的共享记忆库。**
> 无论你是 Qoder、OpenClaw、Claude Code、Cursor 还是其他任何 AI 工具，
> 在开始任何开发工作之前，你**必须**完整阅读本文件并遵守以下规范。
> 这是项目负责人 Tus 设定的强制要求。

**项目负责人**: Tus（GitHub: TusZyc）
**项目**: EVE ESI Tools — EVE Online（国服）数据工具平台
**项目仓库**: https://github.com/TusZyc/eve-esi-qoder
**线上地址**: https://51-eve.online

---

## 一、阅读顺序（每次接手开发必须执行）

按以下顺序阅读文件，恢复开发上下文：

| 步骤 | 文件 | 目的 | 是否必读 |
|------|------|------|----------|
| 1 | **本文件 README.md** | 了解协作规范和规则 | ✅ 必读 |
| 2 | **HANDOFF.md** | 了解当前项目状态、最近进展、待办事项 | ✅ 必读 |
| 3 | **knowledge/dev-rules.md** | 开发规范（代码组织、ESI调用、部署验证） | ✅ 必读 |
| 4 | **tasks/active.md** | 了解进行中的任务详情 | ✅ 必读 |
| 5 | knowledge/code-review.md | 代码审查报告（如果要修 bug） | 按需 |
| 6 | knowledge/pitfalls.md | 踩坑记录（如果遇到技术问题） | 按需 |
| 7 | knowledge/project-overview.md | 项目功能全貌（如果不了解项目） | 按需 |
| 8 | knowledge/deployment.md | 部署和运维（如果要操作服务器） | 按需 |
| 9 | tasks/backlog.md | 待办队列（如果要找新任务） | 按需 |

---

## 二、参与开发的 AI 工具

本项目由多个 AI 工具交替开发，每个工具负责不同阶段的工作。

| 工具 | 标识 | 参与时段 | 说明 |
|------|------|----------|------|
| OpenClaw | `[OpenClaw]` | 2026-03 早期 | 项目初始开发，已停用 |
| Qoder | `[Qoder]` | 2026-03-12 ~ 03-24 | 主力开发阶段，已停用 |
| Claude Code | `[Claude Code]` | 2026-04-01 ~ | 代码审查与修复阶段 |
| Codex | `[Codex]` | 2026-04-09 ~ | 装配模拟器继续开发与上下文接手 |

> 如果你是一个新的 AI 工具，请在上表中添加自己的信息。

---

## 三、记忆更新规范（强制）

### 3.1 标注规则
所有对本仓库文件的修改，**必须**遵守以下标注规则：

- 每条新增内容必须标注 `[工具名]` + 日期
- 示例：`- [x] 修复登录bug [Claude Code] 2026-04-01`
- 更新 HANDOFF.md 时，修改顶部的 `更新者` 和 `最后更新` 字段

### 3.2 不覆盖原则
- **禁止**删除或修改其他工具写的记录
- **只追加**新内容
- 如果发现其他工具的记录有误，用追加的方式标注更正，例如：
  `> [Claude Code] 更正：上述缓存TTL实际为300秒而非600秒`

### 3.3 结束工作时必须更新的文件

| 文件 | 更新什么 |
|------|---------|
| **HANDOFF.md** | 更新「当前任务」状态、「最近完成」表格、「下一步计划」 |
| **tasks/active.md** | 更新任务进度，勾选已完成项，添加新任务 |
| **knowledge/pitfalls.md** | 如果遇到并解决了新问题，追加记录 |

### 3.4 开发日志归档
每次开发结束后，在 `archive/logs/` 下创建日志文件：
- **命名格式**: `YYYY-MM-DD-工具名.md`（例如 `2026-04-01-claude-code.md`）
- **内容**: 本次开发做了什么、遇到了什么问题、怎么解决的

> 注：历史日志（2026-03-24 之前）由 Qoder 创建，当时尚未建立工具名标注规范，
> 因此文件名中不含工具名，但内容均为 Qoder 的开发记录。

---

## 四、目录结构

```
qoder-memory/
├── README.md                  # 【入口】协作规范（本文件）
├── HANDOFF.md                 # 【状态】当前项目快照
│
├── knowledge/                 # 【知识库】技术文档（按需查阅）
│   ├── dev-rules.md           #   ⭐ 开发规范（必读）
│   ├── code-review.md         #   代码审查报告 [Claude Code]
│   ├── esi-endpoints.md       #   ESI API 完整端点参考（公开/认证分类）
│   ├── pitfalls.md            #   踩坑记录与调试经验
│   ├── project-overview.md    #   项目功能概览
│   └── deployment.md          #   部署与运维手册
│
├── tasks/                     # 【任务】进度管理
│   ├── active.md              #   进行中的任务
│   └── backlog.md             #   待办队列
│
├── ideas/                     # 【设计】方案文档
│   └── admin-panel-design.md  #   后台管理设计方案
│
└── archive/                   # 【归档】历史记录
    ├── logs/                  #   开发日志（按日期+工具名）
    └── projects/              #   项目存档
```

---

## 五、项目技术速查

| 项 | 值 |
|----|-----|
| 框架 | Laravel 10 / PHP 8.2 |
| 前端 | Blade + Tailwind CSS |
| 数据库 | SQLite |
| 缓存 | Redis |
| 部署 | Docker Compose (Nginx + PHP-FPM + Redis) |
| 服务器 | 阿里云 ECS 47.116.125.182（中国大陆） |
| EVE API | ESI Serenity (ali-esi.evepc.163.com) |
| 项目目录(服务器) | /opt/eve-esi |

### 部署命令速查
```bash
# SSH 连接（密钥路径因设备而异）
ssh -i "<密钥路径>" root@47.116.125.182

cd /opt/eve-esi
git pull origin main
docker compose exec -T app php artisan cache:clear
docker compose exec -T app php artisan config:clear
docker compose exec -T app php artisan view:clear
docker compose exec -T app php artisan route:clear
docker restart eve-esi-app  # 仅在修改 PHP 静态变量时需要
```

---

## 六、给 Tus 的使用提示

每次切换到新的 AI 工具时，第一句话可以说：

> "请阅读 qoder-memory 仓库的 README.md，按照规范恢复上下文并开始工作。"

这一句话就够了，AI 会按照本文件的阅读顺序自动完成上下文恢复。
