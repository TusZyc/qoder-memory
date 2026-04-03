# ESI API 端点完整参考

> 数据来源：https://ali-esi.evepc.163.com/latest/swagger.json (v1.19)
> Base URL: `https://ali-esi.evepc.163.com/latest`
> 国服必须带参数: `datasource=serenity`（曙光服用 `infinity`）
> 语言参数: `language=zh`（部分端点支持中文）

---

## 公开端点（无需 Token）

### Alliance（联盟）
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/alliances/` | 所有联盟ID | 3600s |
| GET | `/alliances/{id}/` | 联盟信息 | 3600s |
| GET | `/alliances/{id}/corporations/` | 联盟成员军团 | 3600s |
| GET | `/alliances/{id}/icons/` | 联盟图标 | 3600s |

### Character（角色公开信息）
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/characters/{id}/` | 角色公开信息 | 86400s |
| POST | `/characters/affiliation/` | 批量查角色所属 | 3600s |

### Corporation（军团公开信息）
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/corporations/{id}/` | 军团公开信息 | 3600s |
| GET | `/corporations/{id}/alliancehistory/` | 联盟历史 | 3600s |
| GET | `/corporations/{id}/icons/` | 军团图标 | 3600s |

### Universe（宇宙数据）
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| POST | `/universe/names/` | 批量ID→名称（最多1000个） | - |
| POST | `/universe/ids/` | 批量名称→ID | - |
| GET | `/universe/types/` | 所有物品类型ID | 3600s |
| GET | `/universe/types/{id}/` | 物品详情（支持language=zh） | 3600s |
| GET | `/universe/categories/` | 所有物品分类 | 3600s |
| GET | `/universe/categories/{id}/` | 分类详情 | 3600s |
| GET | `/universe/groups/` | 所有物品组 | 3600s |
| GET | `/universe/groups/{id}/` | 组详情（支持language=zh） | 3600s |
| GET | `/universe/systems/` | 所有星系ID | 3600s |
| GET | `/universe/systems/{id}/` | 星系详情（支持language=zh） | 3600s |
| GET | `/universe/constellations/` | 所有星座 | 3600s |
| GET | `/universe/constellations/{id}/` | 星座详情 | 3600s |
| GET | `/universe/regions/` | 所有星域 | 3600s |
| GET | `/universe/regions/{id}/` | 星域详情 | 3600s |
| GET | `/universe/stations/{id}/` | NPC空间站详情 | 3600s |
| GET | `/universe/stargates/{id}/` | 星门详情 | 3600s |
| GET | `/universe/stars/{id}/` | 恒星详情 | 3600s |
| GET | `/universe/planets/{id}/` | 行星详情 | 3600s |
| GET | `/universe/moons/{id}/` | 卫星详情 | 3600s |
| GET | `/universe/factions/` | 所有势力 | 3600s |
| GET | `/universe/graphics/` | 图形资源 | 3600s |
| GET | `/universe/graphics/{id}/` | 图形详情 | 3600s |
| GET | `/universe/structures/{id}/` | 建筑信息（公开建筑无需认证，玩家建筑需要） | 3600s |

### Market（市场）
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/markets/prices/` | 全服均价 | 300s |
| GET | `/markets/{region_id}/orders/` | 区域订单 | 300s |
| GET | `/markets/{region_id}/history/` | 历史成交 | 86400s |
| GET | `/markets/groups/` | 市场分组 | 3600s |
| GET | `/markets/groups/{id}/` | 分组详情 | 3600s |
| GET | `/markets/{region_id}/types/` | 区域活跃物品 | 600s |

### 其他公开端点
| 方法 | 路径 | 说明 | 缓存 |
|------|------|------|------|
| GET | `/status/` | 服务器状态 | 无 |
| GET | `/route/{origin}/{destination}/` | 路径规划 | 86400s |
| GET | `/search/` | 公开搜索 | 3600s |
| GET | `/incursions/` | 入侵活动 | 300s |
| GET | `/industry/facilities/` | 工业设施 | 3600s |
| GET | `/industry/systems/` | 系统工业指数 | 3600s |
| GET | `/insurance/prices/` | 保险价格 | 3600s |
| GET | `/killmails/{id}/{hash}/` | 击毁详情 | 86400s |
| GET | `/loyalty/stores/{corp_id}/offers/` | LP商店报价 | 3600s |
| GET | `/sovereignty/map/` | 主权地图 | 3600s |
| GET | `/sovereignty/campaigns/` | 主权战役 | 300s |
| GET | `/sovereignty/structures/` | 主权建筑 | 3600s |
| GET | `/wars/` | 战争列表 | 3600s |
| GET | `/wars/{id}/` | 战争详情 | 3600s |
| GET | `/wars/{id}/killmails/` | 战争KM | 3600s |
| GET | `/dogma/attributes/` | 属性列表 | 3600s |
| GET | `/dogma/effects/` | 效果列表 | 3600s |

---

## 需要认证的端点（需要 Token + Scope）

### Skills（技能）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/skills/` | `esi-skills.read_skills.v1` |
| GET | `/characters/{id}/skillqueue/` | `esi-skills.read_skillqueue.v1` |
| GET | `/characters/{id}/attributes/` | `esi-skills.read_skills.v1` |

### Assets（资产）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/assets/` | `esi-assets.read_assets.v1` |
| POST | `/characters/{id}/assets/locations/` | `esi-assets.read_assets.v1` |
| POST | `/characters/{id}/assets/names/` | `esi-assets.read_assets.v1` |

### Wallet（钱包）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/wallet/` | `esi-wallet.read_character_wallet.v1` |
| GET | `/characters/{id}/wallet/journal/` | `esi-wallet.read_character_wallet.v1` |
| GET | `/characters/{id}/wallet/transactions/` | `esi-wallet.read_character_wallet.v1` |
| GET | `/corporations/{id}/wallets/` | `esi-wallet.read_corporation_wallets.v1` |
| GET | `/corporations/{id}/wallets/{div}/journal/` | `esi-wallet.read_corporation_wallets.v1` |
| GET | `/corporations/{id}/wallets/{div}/transactions/` | `esi-wallet.read_corporation_wallets.v1` |

### Location（位置）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/location/` | `esi-location.read_location.v1` |
| GET | `/characters/{id}/online/` | `esi-location.read_online.v1` |
| GET | `/characters/{id}/ship/` | `esi-location.read_ship_type.v1` |

### Mail（邮件）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/mail/` | `esi-mail.read_mail.v1` |
| POST | `/characters/{id}/mail/` | `esi-mail.send_mail.v1` |
| GET | `/characters/{id}/mail/{mail_id}/` | `esi-mail.read_mail.v1` |
| PUT | `/characters/{id}/mail/{mail_id}/` | `esi-mail.organize_mail.v1` |
| DELETE | `/characters/{id}/mail/{mail_id}/` | `esi-mail.organize_mail.v1` |
| GET | `/characters/{id}/mail/labels/` | `esi-mail.read_mail.v1` |
| GET | `/characters/{id}/mail/lists/` | `esi-mail.read_mail.v1` |

### Notifications（通知）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/notifications/` | `esi-characters.read_notifications.v1` |
| GET | `/characters/{id}/notifications/contacts/` | `esi-characters.read_notifications.v1` |

### Contacts（联系人）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/contacts/` | `esi-characters.read_contacts.v1` |
| POST | `/characters/{id}/contacts/` | `esi-characters.write_contacts.v1` |
| PUT | `/characters/{id}/contacts/` | `esi-characters.write_contacts.v1` |
| DELETE | `/characters/{id}/contacts/` | `esi-characters.write_contacts.v1` |
| GET | `/characters/{id}/contacts/labels/` | `esi-characters.read_contacts.v1` |

### Contracts（合同）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/contracts/` | `esi-contracts.read_character_contracts.v1` |
| GET | `/characters/{id}/contracts/{cid}/items/` | `esi-contracts.read_character_contracts.v1` |
| GET | `/characters/{id}/contracts/{cid}/bids/` | `esi-contracts.read_character_contracts.v1` |

### Bookmarks（书签）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/bookmarks/` | `esi-bookmarks.read_character_bookmarks.v1` |
| GET | `/characters/{id}/bookmarks/folders/` | `esi-bookmarks.read_character_bookmarks.v1` |

### Fittings（装配）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/fittings/` | `esi-fittings.read_fittings.v1` |
| POST | `/characters/{id}/fittings/` | `esi-fittings.write_fittings.v1` |
| DELETE | `/characters/{id}/fittings/{fid}/` | `esi-fittings.write_fittings.v1` |

### Killmails（击毁报告）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/killmails/recent/` | `esi-killmails.read_killmails.v1` |

### Clones（克隆体）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/clones/` | `esi-clones.read_clones.v1` |
| GET | `/characters/{id}/implants/` | `esi-clones.read_implants.v1` |

### Standings（声望）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/standings/` | `esi-characters.read_standings.v1` |

### Roles（角色权限）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/roles/` | `esi-characters.read_corporation_roles.v1` |

### Fleets（舰队）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/characters/{id}/fleet/` | `esi-fleets.read_fleet.v1` |
| GET | `/fleets/{id}/members/` | `esi-fleets.read_fleet.v1` |

### Market（建筑市场 - 需认证）
| 方法 | 路径 | Scope |
|------|------|-------|
| GET | `/markets/structures/{structure_id}/` | `esi-markets.structure_markets.v1` |

---

## 注意事项

1. **公开端点不要传 Token**：会导致 Token 过期时请求失败，而本来不需要认证
2. **universe/names 的 ID 限制**：int32 范围（最大 2147483647），建筑 ID 超出此范围会 404
3. **universe/names 批量限制**：单次最多 1000 个 ID，超过需分批
4. **分页**：很多端点支持 `page` 参数，响应头 `X-Pages` 表示总页数
5. **language=zh**：仅部分 universe 端点支持，其他端点忽略此参数
6. **缓存时间**：表中的缓存时间是 ESI 服务器侧的建议缓存时间
