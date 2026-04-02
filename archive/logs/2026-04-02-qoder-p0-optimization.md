# P0代码优化 - TokenService与StationNameService重构

**日期**: 2026-04-02
**工具**: [Qoder]
**commit**: `2c6f586`

---

## 背景

基于代码审查发现的两个P0级别问题：
1. Token重复查询问题 - 同一请求周期内多次查询数据库获取相同Token
2. 空间站名称代码重复问题 - 多个服务中存在大量重复的空间站名称翻译代码

---

## P0-1: Token重复查询问题

### 问题分析
- WalletDataController 中有10处 `User::where('eve_character_id', $characterId)->value('access_token')`
- 钱包页面加载时会触发7个API，每个都独立查询Token
- 同一个用户的Token被查询了多次

### 解决方案
创建 `TokenService` 统一服务：
- 使用静态变量 `$tokenCache` 存储请求级别的Token缓存
- 同一请求周期内只查询数据库一次
- 提供 `getToken()` 和 `getTokens()` 方法

### 修改的控制器
1. WalletDataController (10处)
2. MailDataController (6处)
3. SkillDataController (3处)
4. NotificationDataController (2处)
5. ContactDataController (1处)
6. StandingDataController (1处)
7. ContractDataController (1处)
8. FittingDataController (1处)
9. BookmarkDataController (3处)
10. CharacterKillmailDataController (1处)

### 优化效果
| 页面 | 优化前 | 优化后 |
|-----|-------|-------|
| 钱包页面 | Token查询7次 | Token查询1次 |
| 邮件页面 | Token查询5次 | Token查询1次 |
| 技能页面 | Token查询3次 | Token查询1次 |

---

## P0-2: 空间站名称代码重复问题

### 问题分析
- AssetDataService、MarketService、CharacterLocationController 中各有类似的空间站名称翻译逻辑
- 每个地方都实现了：本地数据查找 → ESI调用 → 星系名翻译 → 军团名翻译
- StationNameService.php 存在但从未被调用（孤儿服务）

### 解决方案
重构 `StationNameService` 为统一服务：
- 本地数据优先（eve_stations.json, eve_structures.json）
- ESI API 兜底
- 私有建筑返回 "未开放的玩家建筑"
- 提供统一的 `getName()` 和 `getNames()` 接口

### 修改的文件
1. StationNameService.php - 增强为完整服务
2. AssetDataService.php - 委托给StationNameService
3. MarketService.php - 委托给StationNameService
4. CharacterLocationController.php - 使用StationNameService

### 代码变化
- 消除约200行重复代码
- 新增TokenService约135行
- 增强StationNameService约100行

---

## 部署

### 部署文件
```
app/Services/TokenService.php (新增)
app/Services/StationNameService.php (修改)
app/Services/AssetDataService.php (修改)
app/Services/MarketService.php (修改)
app/Http/Controllers/Api/CharacterLocationController.php (修改)
app/Http/Controllers/Api/WalletDataController.php (修改)
app/Http/Controllers/Api/MailDataController.php (修改)
app/Http/Controllers/Api/SkillDataController.php (修改)
app/Http/Controllers/Api/NotificationDataController.php (修改)
app/Http/Controllers/Api/ContactDataController.php (修改)
app/Http/Controllers/Api/StandingDataController.php (修改)
app/Http/Controllers/Api/ContractDataController.php (修改)
app/Http/Controllers/Api/FittingDataController.php (修改)
app/Http/Controllers/Api/BookmarkDataController.php (修改)
app/Http/Controllers/Api/CharacterKillmailDataController.php (修改)
```

### 部署步骤
1. SCP 上传所有文件到服务器 `/opt/eve-esi/`
2. 清除Laravel缓存：`php artisan cache:clear && config:clear && view:clear`
3. 验证PHP语法：`php -l` 检查通过

---

## 后续建议

1. **性能监控**: 观察钱包页面加载速度是否有明显改善
2. **扩展TokenService**: 可考虑添加Token过期检测和自动刷新
3. **统一更多服务**: EveHelper 中的 `getNameById` 可考虑统一到 TokenService 模式