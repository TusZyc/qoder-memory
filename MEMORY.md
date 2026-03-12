# Qoder 长期记忆

## 项目负责人
- 名称：图斯（Tus）
- GitHub：TusZyc

## 活跃项目

### EVE ESI 管理网站
- 仓库：git@github.com:TusZyc/eve-esi-qoder.git
- 原始仓库（OpenClaw 开发）：git@github.com:TusZyc/eve-esi.git
- 技术栈：Laravel 10 + PHP 8.2 + Blade + Tailwind CSS + Alpine.js + SQLite + Redis
- 部署：Docker Compose → 阿里云 ECS (47.116.125.182)
- 状态：约 75% 完成，由 Qoder 接替 OpenClaw 继续开发

## 服务器信息
- 阿里云 ECS：47.116.125.182（测试服务器）
- 操作系统：Ubuntu
- SSH 用户：root
- SSH 密钥：~/.ssh/qoder_server.pem
- 项目目录：/opt/eve-esi
- Docker 容器：eve-esi-app (php-fpm), eve-esi-nginx, eve-esi-redis

## 开发环境
- 公司电脑（当前）：Windows + WSL，SSH 密钥已配置（qoder_server.pem, ed25519）
- 家里电脑：待配置（需要生成 SSH 密钥并添加到 GitHub 和服务器）
- 本地工作目录：d:\Qoder-work\eve-esi（仅部分文件，完整项目在服务器 /opt/eve-esi）

## 部署注意事项
- PHP 文件通过 SCP 上传（/mnt/d/Qoder-work/... → 服务器路径）
- 部署后必须清理缓存：php artisan view:clear / cache:clear / route:clear
- 部署后重启容器：docker compose restart app
- 历史问题：tarball 方式部署曾导致 PHP $ 变量符号丢失，已改用 SCP 直传

## 偏好
- 使用中文交流
- 项目之前由 OpenClaw（AI 助手名"小图"）开发，现由 Qoder 接替
- 代码推送到 eve-esi-qoder 仓库，不动原始 eve-esi 仓库
