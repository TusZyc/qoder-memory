# 踩坑记录与调试经验

> 每次遇到问题解决后，在这里记录，避免重复踩坑

---

## 目录
- [ESI API 陷阱](#esi-api-陷阱)
- [部署与缓存](#部署与缓存)
- [数据文件](#数据文件)
- [Git 操作](#git-操作)
- [前端问题](#前端问题)
- [OAuth 认证](#oauth-认证)
- [工具使用](#工具使用)

---

## ESI API 陷阱

### KB 新版 KM 详情 hash 获取（2026-04-26 Codex 发现）
**问题**：KB 更新后，搜索/列表接口仍可用，但点击 KM 详情时无法再通过普通 HTTP 获取 ESI hash，导致 `/api/killmails/kill/{id}` 只能走旧 KB HTML fallback，而新 KM 旧页经常返回“世界线已发生混乱”。

**确认结果**：
- `https://beta.ceve-market.org/kill/{id}` 静态 HTML 不包含 `api认证`、`ali-esi` 或 `killmails/{id}/{hash}`。
- KB 前端会在运行后将 `km.KillID` 和 `km.Hash` 拼成官方 ESI 链接：`https://ali-esi.evepc.163.com/latest/killmails/{KillID}/{Hash}/?datasource={server}`。
- `km.Hash` 来自 `/app/kill/{id}/info`，该接口以及 `/item`、`/fitting`、`/atk`、`/support`、`/img` 都需要先完成 KB 的 WASM PoW 门禁。
- 仅获取 `GranblueFantasy` / `ReDive` cookies，并带 `FinalFantasy-XIV: {ReDive}` header 不够；直接请求详情接口会 403。

**KB 前端门禁流程**：
1. 加载 `/assets/KBPoWR_bg.wasm?v2`。
2. 用 WASM `encrypt(fingerprint_json_bytes)` 生成 fp payload。
3. `POST /app/el_psy_congroo`，body 是 protobuf `{ payload: fp }`。
4. 返回 `question` 和 `difficulty`。
5. 用 WASM `find_gate_async(question, difficulty, progressCb)` 算 answer。
6. `POST /app/Steins;Gate`，body 是 protobuf `{ question: "", answer }`。
7. 再请求 `/app/kill/{id}/info`，解 protobuf 后才能拿到 `km.Hash`。

**实验状态**：
- 已验证 `el_psy_congroo` 可返回 `question + difficulty`。
- 已验证本地 Node 可以加载 KB WASM，并算出 gate answer。
- 临时手写 Node wasm-bindgen glue 卡在 `externref` 兼容，需继续补齐或复用官方 glue；这不是 hash 获取原理障碍。

**当前线上兜底**：
- 搜索结果会按 `kill_id` 缓存摘要。
- 详情无 hash 且旧 KB fallback 失败时，返回 `detail_unavailable=true` 的摘要详情，避免直接报“KM 不存在或页面暂时不可用”。
- 这只是临时兜底，不满足“完整 KM 详情”目标；完整修复仍要拿到 hash 后走官方 ESI。

### 中文翻译参数支持表
| 端点 | 支持 language=zh | 备注 |
|------|-----------------|------|
| `universe/systems/{id}` | ✅ 支持 | 直接返回中文星系名 |
| `universe/stations/{id}` | ❌ 不支持 | 需要逐段翻译（见下方方案） |
| `universe/names` | ❌ 不支持 | 返回服务器默认语言 |
| `universe/groups/{id}` | ✅ 支持 | **必须显式指定**，否则返回英文 |

### NPC 空间站中文名翻译方案
ESI `universe/stations/{id}` 不支持 `language=zh`，需要 6 步翻译：

```php
// 1. 获取站点详情（名称、system_id、owner）
// 2. Http::pool() 并行获取中英文星系名
// 3. /universe/names 获取中英文军团名
// 4. 27 种设施类型映射表（Assembly Plant → 组装工厂 等）
// 5. 替换：星系名→中文、Moon→卫星、军团名→中文、设施类型→中文
// 6. 缓存到 eve_locinfo_{id}（24h TTL）
```

### ESI 邮件 API 限制
- **不支持按群组 ID 筛选**：ESI `/characters/{id}/mail/` 不支持 `mailing_list_id` 参数
- **解决方案**：获取全部邮件，前端实现群组过滤

### ESI datasource 参数缺失（2026-04-02 发现）
**问题**：国服 ESI 调用未带 `datasource=serenity` 参数，导致返回全球服数据  
**解决**：所有 ESI API 调用必须显式添加 `datasource=serenity`，包括 `universe/names` 等端点

### ESI 数据格式
| 问题 | 解决方案 |
|------|---------|
| 角色描述 `\xNN` | Unicode 码点 U+00NN，需 `pack('H*','00'.$m[1])` → UCS-2BE → UTF-8 |
| EVE 颜色格式 ARGB | CSS 需 RGB，用 `substr($color, 2)` 去掉 alpha |
| 合同 API | 仅返回 30 天内或未完成的合同 |

### 装配模拟器 modifiers 为空（2026-04-09 Codex 发现）
**问题**：`/fitting-simulator` 页面已接入 `fitting.sqlite`，但很多模块的 `effects.modifiers` 为空数组，导致前端无法直接做完整 Dogma 计算。  
**现象**：
- `Damage Control II` 的 `damageControl` effect 存在，但 `modifiers: []`
- `Large Shield Extender II`、`1MN Afterburner II`、`1600mm Steel Plates II` 等也是 effect 名称存在、modifier 为空
**影响**：当前可做基础资源检查和部分启发式属性联动，但不能依赖 effect tree 实现完整 9 步计算链。  
**后续方向**：
- 先检查 `ImportFittingSde` 的 `modifierInfo -> modifiers_json` 解析链是否漏掉了复杂 YAML
- 若导入链没问题，再检查源 SDE 字段格式是否与当前简单 parser 不兼容

---

## 部署与缓存

### PHP 静态变量缓存
**问题**：修改 PHP 代码后，静态变量仍然保留旧值  
**原因**：PHP-FPM 进程不会自动重载静态变量  
**解决**：
```bash
docker restart eve-esi-app
```

### Redis 缓存清空后市场搜索超时
**问题**：清空 Redis 后，市场搜索 500 错误  
**原因**：`getMarketGroupsTree()` 依赖 ESI API 重建，耗时超过 30 秒  
**解决**：优先从静态文件加载（commit `31e9fab`）

```php
public function getMarketGroupsTree(): array
{
    return Cache::remember('market_groups_tree', 86400 * 7, function () {
        // 优先从静态文件加载
        $staticFile = public_path('data/market_groups.json');
        if (file_exists($staticFile)) {
            $data = json_decode(file_get_contents($staticFile), true);
            if (is_array($data) && !empty($data)) {
                return $data;
            }
        }
        // 静态文件不存在，从 ESI 获取
        return $this->buildMarketGroupsTreeFromApi();
    });
}
```

### 部署后必须清理缓存
```bash
docker compose exec -T app php artisan cache:clear
docker compose exec -T app php artisan config:clear
docker compose exec -T app php artisan view:clear
docker compose exec -T app php artisan route:clear
```

### tarball 部署问题
**问题**：tarball 方式部署导致 PHP `$` 变量符号丢失  
**解决**：改用 SCP 直传文件

### Token 缓存闭包陷阱（2026-04-02 Claude Code 发现）
**问题**：API 控制器在 `Cache::remember` 闭包外部获取 Token，闭包执行时 Token 可能已过期  
**原因**：缓存未命中时闭包延迟执行，闭包捕获的是外部旧 Token  
**解决**：闭包内部重新获取最新 Token，获取失败时不缓存结果
```php
// ✗ 错误写法
$token = $user->eve_access_token;
return Cache::remember($key, 300, function () use ($token) {
    // token 可能已过期
    return $this->callEsiApi($token);
});

// ✓ 正确写法
return Cache::remember($key, 300, function () use ($user) {
    $token = $user->fresh()->eve_access_token; // 闭包内获取最新
    return $this->callEsiApi($token);
});
```

### TokenRefreshService 并发刷新问题（2026-04-02）
**问题**：多个请求同时触发 Token 刷新，导致重复调用 OAuth  
**解决**：添加 Redis 分布式锁 `Cache::lock("token_refresh:{$userId}", 10)`

---

## 数据文件

### 必需的数据文件清单
| 文件 | 用途 | 缺失影响 |
|------|------|---------|
| `data/eve_items.json` | 物品中文名（206987条） | 物品显示英文 |
| `data/eve_systems.json` | 星系中文名 | 位置显示英文 |
| `data/eve_stations.json` | NPC空间站数据 | 克隆体位置显示英文 |
| `data/eve_regions.json` | 星域数据 | 星域选择异常 |
| `data/eve_constellations.json` | 星座数据 | 星座显示异常 |
| `data/eve_structures.json` | 玩家建筑数据 | 建筑位置显示异常 |

### 误删数据文件恢复（2026-03-24 案例）
**问题**：commit `59f0549` 误删了 6 个数据文件  
**恢复**：
```bash
git checkout 9ad687a -- data/eve_items.json data/eve_systems.json data/eve_stations.json data/eve_constellations.json data/eve_regions.json data/eve_structures.json
```

---

## Git 操作

### 代码同步时检查删除
**教训**：提交前必须检查 `git status`，特别注意大文件是否被意外删除

### git status 误报同步
**问题**：`git status` 显示 up to date 但实际不同步  
**验证方法**：
```bash
git fetch origin
git diff HEAD origin/main --stat
```

### 推送策略
**原则**：所有代码以服务器为准，推送从服务器执行
```bash
# 在服务器上执行
cd /opt/eve-esi
git add -A
git commit -m "feat: xxx"
git push origin main
```

### CSRF 保护（2026-04-02 补全）
**问题**：布局文件缺少 CSRF meta 标签，导致 AJAX 请求可能被拒绝  
**解决**：在 `app.blade.php`、`guest.blade.php`、`admin.blade.php` 的 `<head>` 中添加：
```html
<meta name="csrf-token" content="{{ csrf_token() }}">
```

---

## OAuth 认证

### Refresh Token 过期识别
- **错误信息**：`{"error":"invalid_grant","error_description":"Invalid refresh token..."}`
- **HTTP 状态码**：400 或 401
- **处理**：强制登出用户，引导重新授权

### EVE 国服 OAuth 特性
| 特性 | 说明 |
|------|------|
| refresh_token 长度 | 24 字符（正常，非截断） |
| Token Rotation | 不使用，refresh_token 保持不变 |
| access_token 有效期 | 约 20 分钟 |
| refresh_token 有效期 | 约 7-30 天 |

### Session 与 Token 状态解耦
**问题**：Session 有效但 OAuth Token 过期，用户被困在死循环  
**原因**：AuthController 检查已登录就跳转，但 Token 已失效  
**解决**：Token 刷新失败时强制登出用户

---

## 前端问题

### 复制按钮在 Promise 回调中 event 对象不可用
**问题**：异步回调中 `event.target` 为 undefined  
**解决**：在异步操作前保存按钮引用

---

## 工具使用

### Docker 容器中 tree 命令不可用
**替代方案**：
```bash
find . -type f -name "*.php" | head -20
```

### PowerShell 中替代 head 命令
```powershell
Get-Content file.txt | Select-Object -First 10
```

### Nginx 配置变量在 SSH 重定向中被展开
**问题**：SSH 传输 Nginx 配置时 `$uri` 等变量被本地 shell 展开  
**解决**：使用 base64 编码传输

---

## KM 搜索技术细节

### Beta KB Search API
- **地址**：`POST https://beta.ceve-market.org/app/search/search`
- **协议**：gRPC-web 风格 protobuf 编码
- **XSRF 保护**：需要 ReDive + GranblueFantasy cookies + FinalFantasy-XIV header
- **XSRF cookies 获取**：从根路径 `/` 获取（比 `/search` 快 10 倍）

### Protobuf 字段映射
| 字段 | 含义 |
|------|------|
| F1 | Alliances |
| F2 | Corporations |
| F3 | Characters |
| F4 | chartype (victim→lost, finalblow→win, attacker→atk) |
| F5 | types |
| F6 | groups |
| F7 | systems |
| F8 | regions |
| F9 | startdate |
| F10 | enddate |

### 重要限制
- entity(F1-F3) **不能**与 types(F5)/systems(F7) 组合使用，会返回空
- entity + chartype + daterange 可以组合

### 建筑搜索
- ESI category 65 = Structures
- 常见建筑：Astrahus=35833, Fortizar=35834, Keepstar=35835
- 建筑搜索跳过 chartype 过滤

### 连接稳定性
- beta.ceve-market.org 从阿里云 ECS 访问不稳定
- XSRF cookies 缓存 30 分钟，负缓存 60 秒
- 重试逻辑：2 次尝试，500ms 间隔
