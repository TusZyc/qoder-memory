# 开发规范（所有 AI 工具必须遵守）

> 本项目采用"补丁式开发"——功能按需添加，bug 即时修复。
> 以下规范用于减少这种开发模式的常见问题：代码重复、改A坏B、结构混乱。

---

## 一、改代码前：先查再写

**问题**：同一段逻辑在多个文件中重复实现（如 `getNames()` 曾在 5 个控制器中各写一份）

**规则**：
- 写新功能前，先搜索项目中是否已有类似实现
- 如果找到了，优先复用或提取为共享 Service
- 如果没找到，写在 `app/Services/` 而非控制器里，方便下一个人复用

**检查方法**：
```bash
grep -r "你要写的关键函数名或逻辑" app/
```

---

## 二、改代码后：检查影响范围

**问题**：修改公共方法的参数或返回格式，导致其他调用方报错（如修 Token 缓存导致通知接口异常）

**规则**：
- 修改 Service / Helper 方法时，先搜索所有调用方
- 修改返回格式时，确认所有前端页面兼容
- 改完后检查服务器日志，确认无新 ERROR

**检查方法**：
```bash
# 搜索谁调用了这个方法
grep -r "方法名" app/ resources/
# 部署后检查日志
docker compose exec app tail -50 storage/logs/laravel.log
```

---

## 三、代码组织规则

| 类型 | 放在哪里 | 说明 |
|------|---------|------|
| API 端点 | `app/Http/Controllers/Api/` | 保持精简，逻辑放 Service |
| 业务逻辑 | `app/Services/` | 可复用的核心逻辑 |
| Token 获取 | `TokenService::getToken($characterId)` | **统一入口**，禁止直接查 User 表 |
| 本地数据读取 | `EveDataService` | 优先本地 JSON，ESI 兜底 |
| 缓存键 | 通过 `CacheKeyService` 管理 | 避免键名冲突 |

---

## 四、ESI API 调用规范

```php
// ✅ 正确
$response = Http::withToken(TokenService::getToken($characterId))
    ->timeout(15)
    ->get("{$baseUrl}characters/{$characterId}/skills/", [
        'datasource' => 'serenity'  // 国服必须带
    ]);

// ❌ 错误：不带 datasource、用旧 token、没设超时
$response = Http::withToken($user->access_token)
    ->get("{$baseUrl}characters/{$characterId}/skills/");
```

**关键点**：
- 所有调用必须带 `'datasource' => 'serenity'`
- Token 通过 `TokenService::getToken()` 获取，不要用 `$user->access_token`
- 在 `Cache::remember` 闭包内获取 Token，不要在外部捕获
- ESI 失败时：核心数据抛异常（不缓存失败），辅助数据静默降级
- **区分公开/私有端点**（见下表），公开端点**不要**传 Token

### ESI 端点认证分类

| 类型 | 端点前缀/示例 | 是否需要 Token |
|------|-------------|---------------|
| 世界数据 | `universe/names`、`universe/types/{id}`、`universe/systems/{id}`、`universe/stations/{id}`、`universe/regions/`、`universe/groups/{id}`、`universe/categories/{id}` | ❌ 不需要 |
| 市场数据 | `markets/{region}/orders/`、`markets/{region}/history/`、`markets/groups/` | ❌ 不需要 |
| 服务器状态 | `status/` | ❌ 不需要 |
| 角色数据 | `characters/{id}/skills/`、`characters/{id}/wallet/`、`characters/{id}/notifications/`、`characters/{id}/assets/`、`characters/{id}/mail/` 等 | ✅ 需要 |
| 军团数据 | `corporations/{id}/wallets/`、`corporations/{id}/members/` 等 | ✅ 需要 |
| 建筑数据 | `universe/structures/{id}` | ✅ 需要（玩家建筑） |

**判断标准**：查世界/公开数据不需要 Token，查角色/军团私有数据需要 Token。`universe/structures/{id}` 是例外——玩家建筑需要认证。

---

## 五、前端规范

- 所有用户界面文本使用中文
- 使用 Tailwind CSS（CDN 方式）
- 错误提示使用 toast 而非 `alert()`
- 动态链接加 `rel="noopener noreferrer"`

---

## 六、部署后验证

每次部署后必须执行：

```bash
cd /opt/eve-esi
git pull origin main
docker compose exec -T app php artisan cache:clear
docker compose exec -T app php artisan config:clear
docker compose exec -T app php artisan view:clear
docker compose exec -T app php artisan route:clear
docker restart eve-esi-app  # 修改 PHP 代码时需要

# 验证无报错
docker compose exec app tail -30 storage/logs/laravel.log
```

---

## 七、错误返回格式

API 统一返回格式：
```json
{"success": false, "error": "error_code", "message": "用户可读的中文描述"}
```

---

## 八、禁止事项

- ❌ 直接 `User::where()->value('access_token')` — 用 `TokenService`
- ❌ `Cache::remember` 闭包中用外部捕获的 `$token`
- ❌ ESI 调用不带 `datasource=serenity`
- ❌ 删除 `data/*.json` 数据文件
- ❌ 删除或覆盖其他 AI 工具的代码而不检查影响
- ❌ 在项目仓库中放与代码无关的文件（规范、日志等放记忆仓库）
