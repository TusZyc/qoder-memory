# KM 图片生成器

**状态**: 技术验证完成，待开发
**讨论日期**: 2026-04-03
**讨论者**: Tus + Claude Code

---

## 功能目标

定时轮询特定军团/联盟的 KM，有新 KM 时生成一张模仿游戏内样式的图片，后期通过 WS 推送到微信/QQ 等 IM 平台。

---

## 技术方案（已验证可行）

### 渲染工具
- **PHP GD + TTF 字体**，无需 Browsershot/Chrome
- 服务器需安装字体包：`apt-get install fonts-dejavu`
- Docker 容器内已验证可生成 PNG

### 图片资源来源
EVE 官方图片服务器，TypeID 通用（国服同 TQ）：
```
舰船渲染图: https://images.evetech.net/types/{type_id}/render?size=256
物品图标:   https://images.evetech.net/types/{type_id}/icon?size=64
角色头像:   https://images.evetech.net/characters/{character_id}/portrait?size=64
军团 logo:  https://images.evetech.net/corporations/{corp_id}/logo?size=64
联盟 logo:  https://images.evetech.net/alliances/{alliance_id}/logo?size=64
```

### 数据来源
- zKillboard API 或国服 KB 接口轮询新 KM
- ESI `/killmails/{id}/{hash}/` 获取完整装配和参战信息

---

## 布局结构（三区）

```
┌─────────────────────────────────────────── 780px ──┐
│ [舰船渲染图 160x160]  舰船名（黄色）                  │  顶部区
│                       Victim / Corp / Alliance      │
│                       Time / System / ISK Lost      │
├─────────────────────────────────────────────────── │
│ HIGH  [icon][icon][icon][空][空][空]                 │  装配区
│ MED   [icon][icon][空][空]                          │
│ LOW   [icon][icon][icon][空][空]                    │
│ RIG   [icon][icon][icon]                            │
├─────────────────────────────────────────────────── │
│ [头像] 角色名  Corp名  舰船名  伤害  FINAL BLOW      │  参战区
│ [头像] 角色名  Corp名  舰船名  伤害                  │
└─────────────────────────────────────────────────── │
```

---

## 关键技术点

### slot flag → 槽位映射表
```
11-18  = 高槽 1-8
19-26  = 中槽 1-8
27-34  = 低槽 1-8
92-99  = 改装槽 1-8
125-132 = 子系统槽
87     = 无人机仓
133-142 = 舰载机挂架
5      = 货舱
```

### 空槽补全
ESI 只返回有装备的槽，空槽数量需查 dogma 属性：
- attributeID 14 = 高槽数
- attributeID 13 = 中槽数
- attributeID 12 = 低槽数
- attributeID 1137 = 改装槽数

简化方案：每类槽固定显示最大格数（高8/中5/低5/改3），空槽用 ✕ 框占位，无需额外 API 调用。

### 图片缓存
EVE 图片服务器图标应本地缓存（storage/km-icons/），避免每次生成都请求网络。

---

## 待开发事项

- [ ] 实现图片本地缓存层
- [ ] 修复舰船渲染图拉取（当前偶发失败）
- [ ] 加入角色头像
- [ ] 完善布局细节（字体大小、间距）
- [ ] 封装成 Laravel Service/Command
- [ ] 接入真实 KB 接口轮询逻辑
- [ ] WS 推送到 IM 平台
