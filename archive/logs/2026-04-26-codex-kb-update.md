# 2026-04-26 Codex - KB 新版接口与 KM 详情门禁

## 背景

Tus 反馈 KM 页面搜索可正常获取列表，但点击查看 KM 详情报“KM 不存在或页面暂时不可用”。经纠正，搜索功能之前也可用，真正问题一直是详情页无法获取完整 KM。

## KB 新版接口变化

- `beta.ceve-market.org` 已更新为 Angular SPA，静态页面本身不包含 KM hash。
- 搜索/列表接口仍可通过 protobuf 调用：
  - `/app/search/search`
  - `/app/list/pilot|corp|alli/{id}/kill|loss`
  - `/app/search/search_char`
  - `/app/search/search_item`
  - `/app/search/search_sys`
- protobuf 请求需要：
  - `Content-Type: application/alicegearaegis`
  - `Accept: application/alicegearaegis`
  - `FinalFantasy-XIV: {ReDive cookie}`
  - cookies: `GranblueFantasy`, `ReDive`
- 搜索参数新增/确认：
  - `types[]` 字段 5
  - `groups[]` 字段 6
  - `systems[]` 字段 7
  - `regions[]` 字段 8
  - `start_date` 字段 9
  - `end_date` 字段 10
  - `next` 字段 11

## 详情门禁

- `/app/kill/{id}/info`、`/item`、`/fitting`、`/atk`、`/support`、`/img` 均需要 KB 前端完成 WASM PoW 门禁。
- 直接 curl 即使带 cookies 和 `FinalFantasy-XIV` header，也返回 403。
- KB 页面上的“api认证”链接确实包含官方 ESI hash，但它是前端拿到 `km.Hash` 后拼出来的：
  - `https://ali-esi.evepc.163.com/latest/killmails/{KillID}/{Hash}/?datasource={server}`
- `km.Hash` 来自受保护的 `/app/kill/{id}/info`，所以不能直接从静态 HTML 抓。

## 已做代码变更

- `BetaKbApiClient`
  - 增加新版 beta protobuf headers/cookie 统一封装。
  - 增加 `postBetaProto` / `getBetaProto`。
  - 增加 beta autocomplete 解析。
  - 搜索缓存命中时也会按 `kill_id` 写入摘要缓存。
  - `parseBetaKillListProtobuf` 会缓存 `kb:kill_summary:{killId}`。
- `ProtobufCodec`
  - 增加 `encodePackedInt32`。
- `KillmailSearchService`
  - autocomplete 优先尝试 beta API，失败再 fallback。
- `KillmailService`
  - 新版 ship group 直接走 `groups[]`，不再本地展开成大量 type id。
  - 详情无 hash 时先旧 KB fallback；若旧 KB 也失败，则使用搜索摘要构造 `detail_unavailable=true` 的降级详情。
- 前端视图
  - 移除浏览器直接请求 `beta.ceve-market.org/app/kill/{id}/info` 的旧逻辑。
  - 统一走后端 `/api/killmails/kill/{id}`。
  - 如果返回 `detail_unavailable_reason`，详情弹窗显示提示。

## 部署与验证

- 已通过 SCP 覆盖服务器 `/opt/eve-esi` 对应文件。
- 已在容器内执行：
  - `php -l app/Services/Killmail/BetaKbApiClient.php`
  - `php -l app/Services/KillmailService.php`
  - `php artisan cache:clear`
  - `php artisan view:clear`
  - `php artisan config:clear`
  - `php artisan route:clear`
  - `docker restart eve-esi-app`
- 烟测：
  - `/killmails` 返回 200。
  - `/api/killmails/advanced-search?...` 返回 50 条。
  - 先搜索再请求 `/api/killmails/kill/22490593` 返回 `success: true`，不再 `fetch_failed`。

## hash 获取方案实验

目标仍然是完整 KM 详情，摘要兜底不是最终方案。

优先方案 2：服务端复刻 KB 前端门禁流程。

已验证：
- 可 `POST /app/el_psy_congroo` 获取 `question + difficulty`。
- 本地 Node 可加载 `KBPoWR_bg.wasm`。
- WASM `encrypt` 可生成 fp bytes。
- WASM `find_gate_async` 可算出 gate answer。

待继续：
- 补齐或复用 wasm-bindgen glue 的 `externref` 行为。
- 完成 `POST /app/Steins;Gate`。
- 请求 `/app/kill/{id}/info` 并解析 `km.Hash`。
- 拿到 hash 后走官方 ESI `/latest/killmails/{id}/{hash}/?datasource=serenity`。
