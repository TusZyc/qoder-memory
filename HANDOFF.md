# 项目状态快照

> 本文件是项目的实时状态。每次接手开发时，先读 README.md 了解规范，再读本文件了解现状。

**最后更新**: 2026-04-04
**更新者**: [Claude Code]
**当前设备**: 用户本地

---

## 当前任务

| 任务 | 状态 | 优先级 | 负责 |
|------|------|--------|------|
| P0代码优化 | ✅ 已完成 | 最高 | [Qoder] |
| 修复通知接口 ESI universe/names 错误 | ✅ 已完成 | 高 | [Claude Code] |
| KM 图片生成器开发 | 🚧 代码完成，服务器待恢复 | 中 | [Claude Code] |
| 验证市场搜索功能 | ⏳ 待验证 | 中 | [待分配] |

详细任务清单见 `tasks/active.md`

---

## 最近进展

| 日期 | 内容 | 操作者 |
|------|------|--------|
| 4-04 | KM图片生成器代码完成，推送 `4bc5e05`；服务器部署受阻（Docker build OOM，需重建镜像） | [Claude Code] |
| 4-04 | 修改 Dockerfile 换用阿里云镜像源 + 添加 FreeType/JPEG/WebP GD 支持 | [Claude Code] |
| 4-04 | 服务器上传字体文件 NotoSansSC-Regular/Bold.ttf → storage/fonts/ | [Claude Code] |
| 4-03 | KM图片生成器技术验证：PHP GD + EVE图片服务器方案可行，见 ideas/km-image-generator.md | [Claude Code] |
| 4-03 | UI修复：页面标题/合同NaN/服务器维护检测/装配搜索，部署 `cde8d06` | [Claude Code] |
| 4-02 | **P0优化完成**: TokenService统一服务 + StationNameService重构，减少80%重复查询 | [Qoder] |
| 4-02 | 部署 `2c6f586` 到服务器（15个文件） | [Qoder] |
| 4-02 | 全面修复：Token缓存闭包/通知summary端点/星空动画/CSRF/datasource/Redis锁 | [Claude Code] |
| 4-01 | 全面代码审查：发现6个严重、8个高危、10个中等问题 | [Claude Code] |
| 4-01 | 建立多AI协作记忆规范 | [Claude Code] |

---

## 服务器状态

| 项目 | 值 |
|------|-----|
| 最后成功部署 | 2026-04-03 [Claude Code via SSH]，commit `cde8d06` |
| 最新commit | `4bc5e05` (feat: KM 图片生成器) |
| 待部署变更 | `4bc5e05` KM图片生成器（已上传文件到服务器，待Docker重建） |
| Docker容器 | ⚠️ 需要恢复！OOM 杀掉了 php-fpm 和 systemd |
| HTTPS证书 | ZeroSSL，90天，每日14:01自动续期 |
| GitHub同步 | ✅ 已同步至 `4bc5e05` |

---

## ⚠️ 服务器紧急状态（2026-04-04）

Docker build 时编译 PHP GD 扩展导致服务器 **OOM（内存不足）**，Linux OOM Killer 杀掉了多个进程：
- `php-fpm`（进程 686607、686608）已被杀
- `AliYunDunMonito`（进程 1748）已被杀
- `systemd`（进程 586727）已被杀

**恢复步骤**（需要 Tus 操作）：

```bash
# 1. 重启 Docker 并恢复容器
systemctl restart docker
cd /opt/eve-esi && docker compose up -d

# 2. 添加 2GB Swap（防止下次 build 再次 OOM）
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile && swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab

# 3. 重建 Docker 镜像（已换阿里云源，速度快）
cd /opt/eve-esi
docker build -t eve-esi-app docker/app/
docker compose up -d --force-recreate app

# 4. 验证 GD FreeType 支持
docker exec eve-esi-app php -r "print_r(gd_info());" | grep FreeType

# 5. 清除缓存
docker exec eve-esi-app php artisan config:clear
docker exec eve-esi-app php artisan route:clear
```

---

## ⚠️ 重要注意事项

1. **PHP 静态变量缓存**：修改 PHP 代码后需 `docker restart eve-esi-app` 才能生效
2. **Redis 缓存清空后**：市场搜索可能超时，已改为优先从静态文件加载
3. **数据文件保护**：`data/*.json` 不能删除，提交前用 git status 检查
4. **Token缓存闭包**：`Cache::remember` 闭包必须在内部获取最新Token，不可在外部获取后传入
5. **ESI datasource**：国服ESI所有调用必须带 `datasource=serenity`
6. **海外SSH受限**：从海外网络连不上服务器（阿里云安全组限制），需从中国大陆网络操作
7. **多AI协作**：[Claude Code] 负责代码审查与修复（走GitHub），[Qoder] 负责部署与服务器操作
8. **Docker build 内存**：服务器内存较小，编译 PHP 扩展容易 OOM，建议先加 2GB swap 再 build
9. **Debian 镜像源**：Dockerfile 已配置阿里云镜像 `mirrors.aliyun.com`，国内 build 快很多
10. **PHP GD 命名空间**：GD函数在命名空间类中必须用 `use function imagettftext;` 等显式导入

---

## 下一步计划

1. **恢复服务器**（Tus 操作）：重启 docker、加 swap、重建 Docker 镜像
2. **验证 KM 图片生成**：`GET /api/killmails/kill/{id}/image` 接口
3. 验证市场搜索功能
4. 继续修复代码审查剩余问题（详见 `knowledge/code-review.md`）
