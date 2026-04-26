# 2026-04-26 Codex - KB 改动兼容与 KM 详情恢复总结

## 一句话结论

KB 这次更新后，问题不在搜索，而在“完整详情依赖的 hash 获取链路”被新版 WASM PoW 门禁拦住了。现在项目已经完成服务端兼容，`/api/killmails/kill/{id}` 能重新拿到 hash，并通过官方 ESI 返回完整 KM。

## KB 这次改了什么

### 1. 搜索接口还在，但协议细节变了
- KB 搜索/列表接口仍然可用，但已稳定依赖新版 protobuf headers 和 cookies。
- 关键请求特征：
  - `Content-Type: application/alicegearaegis`
  - `Accept: application/alicegearaegis`
  - `FinalFantasy-XIV: {ReDive cookie}`
  - cookies: `GranblueFantasy`, `ReDive`
- 搜索参数新增/确认了 `groups[]`、`regions[]`、`next` 等字段。

### 2. KM 详情接口前面加了 WASM 门禁
- 受影响接口：
  - `/app/kill/{id}/info`
  - `/app/kill/{id}/item`
  - `/app/kill/{id}/fitting`
  - `/app/kill/{id}/atk`
  - `/app/kill/{id}/support`
  - `/app/kill/{id}/img`
- 直接 HTTP 请求这些接口，即使带 cookie 和 header，也会被 403 拦下。

### 3. `api认证` 链接不是静态页面自带的
- 页面上的 `api认证` 最终指向官方 ESI：
  - `https://ali-esi.evepc.163.com/latest/killmails/{KillID}/{Hash}/?datasource=serenity`
- 但这个链接是 KB 前端运行后，用 `km.Hash` 动态拼出来的。
- 静态 HTML 本身不包含 hash，所以不能靠抓页面源码恢复完整详情。

## 这次改动带来的直接影响

### 对现有项目的影响
- KM 搜索列表基本不受影响。
- 点击查看 KM 详情时，后端拿不到 hash，就无法再走官方 ESI `/killmails/{id}/{hash}/`。
- 结果是详情接口会退回旧 KB fallback，而新版/新 KM 常常拿不到，最终前端看到“KM 不存在或页面暂时不可用”。

### 真正卡住的关键点
- 不是“详情接口坏了”，而是“拿不到 hash”。
- 没有 hash，就拿不到完整攻击者、装备、掉落、坐标等权威详情。

## 已确认的 KB 门禁流程

服务端最终复现的是这条链：

1. 加载 `/assets/KBPoWR_bg.wasm?v2`
2. 用 WASM `encrypt(...)` 生成指纹 payload
3. `POST /app/el_psy_congroo`
4. 拿到 `question + difficulty`
5. 用 WASM `find_gate_async(...)` 计算 answer
6. `POST /app/Steins;Gate`
7. 再请求 `/app/kill/{id}/info`
8. 从 protobuf 里取出 `km.Hash`
9. 调官方 ESI `/latest/killmails/{id}/{hash}/?datasource=serenity`

额外确认：
- `Steins;Gate` 阶段必须沿用挑战接口返回后更新过的 cookies，尤其是 `ReDive` 和 `Princess.Connect`。

## 项目里做了哪些升级

### 搜索与列表兼容
- 更新了 beta KB protobuf 请求头、cookie 和参数编码。
- autocomplete 改为优先使用新版 beta API。
- ship group 搜索改为原生 `groups[]`，不再本地展开大量 type id。
- 搜索摘要继续缓存到 `kb:kill_summary:{killId}`，供必要时回退使用。

### 详情链路恢复
- 新增 `scripts/kb_gate_hash.mjs`
  - 在 Node 中复用 KB 官方风格 wasm-bindgen glue
  - 自动跑完整个 KB 门禁流程并输出 hash JSON
- 新增 `app/Services/Killmail/KbGateHashResolver.php`
  - 从 Laravel 调 Node 脚本获取 hash
  - 成功后缓存到 `kb:esi_hash:{killId}`
- `KillmailService` 改为：
  - 先尝试已有 hash
  - 拿不到时走 KB gate resolver
  - 成功后直接走官方 ESI 获取完整 KM

### 最后的稳定性修复
- 在 PHP-FPM / Docker 环境下，Node 脚本虽然成功输出了 JSON hash，但 `proc_close()` 可能返回 `-1`。
- 如果还按退出码硬判失败，就会出现“脚本明明成功，接口仍报错”的假失败。
- 已修成：
  - 只要 stdout 里已经拿到合法 40 位 hash，就直接视为成功并缓存。
  - 退出码仅作为辅助诊断信息。

## 当前效果

### 线上结果
- 搜索功能维持正常。
- KM 详情接口已恢复完整数据，不再只剩摘要兜底。
- 已在服务器验证：
  - `/api/killmails/kill/22490593` 返回 `success: true`
  - `esi_verified: true`
  - `attackers`、`items`、受害者装备和掉落均已完整返回

### 代表性提交
- 主项目：
  - `4de8213` `fix: adapt killmail kb api changes`
  - `9e380b3` `feat: resolve killmail hash through kb gate`
  - `261f012` `fix: accept kb hash resolver output before proc_close`
- 记忆仓库：
  - `1ca7a1f` `docs: record kb killmail api update`

## 后续建议

### 1. 继续观察 KB 是否再次调整 WASM 或 cookie 流程
- 这次方案已经跑通，但本质上仍然依赖 KB 当前前端门禁实现。
- 如果 KB 再改 challenge 字段、protobuf 格式或 cookie 语义，resolver 需要跟进。

### 2. 给 hash resolver 增加更明确的监控日志
- 重点记录：
  - `el_psy_congroo` 是否成功
  - `Steins;Gate` 是否返回 202
  - `/app/kill/{id}/info` 是否 200
  - 是否命中 `kb:esi_hash:{killId}` 缓存

### 3. 尽量把 KB gate 相关逻辑维持在单独脚本/服务里
- 这样以后 KB 再改，只需要集中修改 resolver，不容易把整个 `KillmailService` 再搅乱。
