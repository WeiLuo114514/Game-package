---
name: tech-patterns-effects-triggers
description: 运算牌效果和王牌效果的技术规范 — apply vs trigger vs signal 三套模式、AnimationRequest、飘字、UI刷新、音效系统、bonus_requeue插队、UI音效架构、效果范围预览系统、商店特价区、倍增魔法、利息上限+魔法容量动态变量、结局面板全局统计、SettlementSequencer超时修复、教程AnimationPlayer集成、绿色封锁多入口模式、效果削弱范式（2026-07-19 更新）
metadata: 
  node_type: memory
  type: reference
  originSessionId: b794dd90-ecd7-4b3a-8970-3d8b78086b7a
---

# 效果系统技术规范

> 最后更新：2026-07-14

## CardData 等级系统（2026-07-05 更新）

**重要**：`level` 字段保持简单 `@export var level := 1`，不要用 setter/getter 覆写（会导致 Godot 资源初始化字段偏移）。

超体标签通过 `get_effective_level()` 方法实现等级覆写：
```gdscript
func get_effective_level() -> int:
    if has_tag(TagType.SUPERBODY):
        return 3
    return level
```
所有需要等级判定的代码（显示、等级闸门、效果参数索引）统一用 `get_effective_level()` 而非 `card_data.level`。

## 标签系统（2026-07-05 更新）

| 标签 | ID | 颜色 | 效果 |
|------|-----|------|------|
| CHAMELEON | 64 | #00ff88 | `has_color()` 对任意颜色返回 true |
| SUPERBODY | 128 | #00e5ff | `get_effective_level()` 返回 3 |
| ETERNAL | 32 | #ff4081 | 删牌魔法(20001)不可选中 |

### has_color() 统一入口

```gdscript
func has_color(target: String) -> bool:
    if has_tag(TagType.CHAMELEON):
        return true
    return color == target or extra_colors.has(target)
```
全项目所有颜色判定走此函数，变色龙一处修改全局生效。`color` 字段使用单字母（"R"/"Y"/"B"/"G"/"W"），事件改色时也必须用单字母。

### ghost_magic_ids

清除魔法印记时，被清除的魔法 ID 移入 `ghost_magic_ids` 数组——释放了 `applied_magic_ids` 的槽位（`can_accept_magic()` 只检查 `applied_magic_ids.size()`），但效果保留（需要效果执行时也遍历 `ghost_magic_ids`）。

## 底数计算

```gdscript
func get_display_base_number() -> int:
    return max(0, floori((base_number + turn_base_delta) * turn_base_mult))
```
先加后乘，最终结果钳在 0 以上。详见 [[base-number-formula-20260704]]。

## 事件系统（2026-07-05 大幅扩展）

### 调度表
每单元（3关）结束触发：Lv1/3/6/9/12/15/18。PlayField 中 `pending_event_id` 必须在 `reset_for_new_game()` **之后**设置。

### 选项结构
支持 `"effect"` (单效果) 和 `"effects"` (多效果数组) 两种格式。多效果顺序执行，`PAY_MONEY` 返回 false 时阻断链条。

### EventEffect 全表（17 个）

| 效果 | 参数 | 说明 |
|------|------|------|
| GAIN_MONEY | {amount} | 给钱 |
| PICK_S_CARD | {count} | 三选一开包 |
| REMOVE_CARDS | {count} | 选牌删除 |
| APPLY_VARIANT | {count} | 随机异彩 |
| GAIN_MAGIC | {pool} | 随机魔法 |
| SET_TAG | {tag, count} | 随机打标签 |
| SELECT_TAG | {tag, count} | 选牌打标签 |
| AUTO_TAG_BY_KEYWORD | {tag, keyword} | 按脚本名自动打标签 |
| BOOST_INIT_BASE | {multiplier} | 全局初始底数×N |
| CHANGE_RANDOM_COLOR | {count} | 随机换色（R/Y/B/G） |
| GAIN_SPECIFIC_WILD | {wild_id} | 获得指定王牌 |
| BUY_EXTRA_SLOT | {cost} | 买槽位 |
| BUY_EXTRA_ROUND | {cost} | 买回合 |
| GAIN_SPECIFIC_MAGIC | {magic_id, cost} | 买魔法 |
| INCREASE_SLOT/HAND/MAX_COST/MAX_ROUND | {amount} | 免费升级 |
| PAY_MONEY | {amount} | 扣钱（失败阻断链） |
| ENTER_ENDLESS | {} | 无尽模式 |
| TRIGGER_ENDING | {} | 结局 |
| CLEAR_MAGIC_SLOTS | {count} | 清印记（保留效果） |

### SelectCard 事件模式
`open_event_select_tag(count, tag_type)` — 多段选牌打标签
`open_event_clear_magic(count)` — 选牌清印记

## 王牌配置（2026-07-05 更新）

### 字段结构
TEXTURE 字段已从 WildConfig/LevelWildConfig 删除。现为 6 字段：
```
[NAME, DESC, QUALITY, SCRIPT, TRIGGERS, COLOR]
```
Idx 枚举：`{ NAME=0, DESC=1, QUALITY=2, SCRIPT=3, TRIGGERS=4, COLOR=5 }`

卡面纹理完全依赖分层 PNG：`{id}B.png` / `{id}T.png`（`%04d` 零补齐格式）。

### 管道架构（WildEffect extends Node2D）
```
try_execute → _can_trigger → _execute → _on_executed(signal)
```
挂载方式：WildCardUI 场景 EffectNode 子节点，`_try_load_effect()` 中 set_script + 注入 wild_data/trigger_list。全局缓存已废弃。

## 系统参数（2026-07-05 更新）

| 参数 | 默认值 | 变量 | 可扩展 |
|------|--------|------|:--:|
| 生效槽位 | 4 | `active_zone_capacity` | 事件 INCREASE_SLOT |
| 手牌上限 | 6 | `stat.hand.val` | 事件 INCREASE_HAND |
| 最大费用 | 8 | `stat.cost.base_max` | 事件 INCREASE_MAX_COST |
| 回合上限 | 3~4 | `max_round` | 事件 INCREASE_MAX_ROUND |
| 王牌上限 | 5 | `MAX_OWNED_WILDS` | 固定 |
| 魔法上限 | 2 | `MAX_OWNED_MAGICS` | 固定 |

## 预览系统

`get_preview_score()` 遍历 `active_zone` 动态数组，自适应槽位容量（4/5/6 皆可）。详见 [[dev-log-20260705]] 预览系统章节。

## 品质系统

| 品质 | 数值 | 印记槽 | 字母 | 王牌价格 | 回收价 |
|------|:--:|:-----:|:--:|:------:|:-----:|
| C | 0 | 1 | C | 4$ | 2$ |
| B | 1 | 2 | B | 6$ | 3$ |
| A | 2 | 3 | A | 12$ | 6$ |
| S | 3 | 4 | S | 20$ | 10$ |
| X | - | 4 | X | - | - |

价格定义：`FileManager.QUALITY_PRICE`（购买）/ `WildCardUI.RECYCLE_PRICES`（回收，严格半价）

## 商店品质动态权重（2026-07-06 新增）

王牌刷新权重 = 颜色权重 × 品质权重

品质权重由两个因素叠加：
- `QUALITY_BASE_WEIGHTS[unit]`：按单元1-6递进（前期B高S低→后期S高B低）
- `QUALITY_REFRESH_BONUS[quality] × _wild_refresh_count`：付费刷新修正，单元切换归零

详见 `shop_page.gd` 中的 `_get_wild_quality_weight()`。

## BaseCardData 数据层（2026-07-06 新增）

`stat.base` 字典已重构为 `BaseCardData` Resource：

```gdscript
class_name BaseCardData
extends Resource
@export var base_card_id: int = 0
@export var init: float = 30.0
var bonus: float = 0.0
var next: float = 0.0
var val: float = 30.0
```

- `GameData.base_card` 为唯一权威数据源
- `BaseCardConfig` 静态配置表（当前仅默认 id=0）
- 后续：DeckConfig 接入 `base_card_id` + InfoBoard 换 BaseCardUI
- 全局 `stat.base.xxx` → `base_card.xxx`（0残留）

## 商品双击购买（2026-07-06 新增）

三个 goods 脚本统一模式：
- `_on_gui_input` 检测左键，0.2s 窗口内双击→购买，超时→飘字「双击购买」
- `_process` 手动计时（非 Godot 内置双击，可控窗口）
- 购买前检查：余额不足→「货币不足」，已拥有→「已拥有」，成功→「购买成功！」

## 事件飘字反馈（2026-07-06 新增）

23 个 EventEffect 全部在 `event_page.gd` 的 `_exec_*` 函数末尾调用 `GameData.show_tip()`。
颜色语义：gold=收益 / tomato=代价 / #88ff88=系统升级 / #88aaff=魔法 / #ff88ff=异彩/印记 / #ffaa00=特殊

## 事件牌系统（2026-07-07 新增）

### 架构
- 强制事件（ID 1-7）：关卡里程碑触发，记入 `opened_event_ids` 不可复触
- 事件牌事件（ID 110+）：商店事件牌商品触发，独立池 `event_card_pool`，不记入 `opened_event_ids` 可复抽

### 事件牌池结构
```gdscript
static var event_card_pool := {
    1: [110, 111, 112],  # 常见事件（3$）
    2: [],                # 精良事件（待填充）
    3: [],                # 稀有事件（待填充）
}
```

### 售价 & 品质分布
3$: T1=60%, T2=30%, T3=10%
5$: T1=10%, T2=40%, T3=50%

### 通用概率门
`event_page._execute_effect` 入口处检测 `params.chance`。任意效果加 `"chance": 0.5` 即可零成本获得概率判定。未命中返回 true（不是错误，只是无事发生）。

### GAIN_MONEY 随机范围
params 使用 `"min": 2, "max": 6` 替代 `"amount": 5` 即可在范围内随机。

### APPLY_VARIANT 指定类型
params 加 `"variant": 5` 可指定折损异彩（CardData.VariantType.DAMAGED=5）。

### OPEN_RANDOM_BASE_PACK
新 EventEffect，随机开 R/Y/B/G 基础包。通过 `GameData.pending_event_pack_type` 传递包类型。

### 事件牌退场链路
- `GameData.pending_event_is_card` flag 区分事件来源
- `RoundMenu._event_back()` 统一路由：事件牌→回商店，强制事件→InfoPage
- 延迟效果（删牌/打标签/清印记/开包）完成后也走 `_event_back()`
- `_on_draw_card_page_pack_opening_done` 检查 `pending_event_is_card` 决定去向

### goods_event.gd API
与其他 goods 脚本一致：`purchased(tier, price)` 信号 / `init_data(tier, price)` / `restore_display()` / `fade_out()` / `refresh_state()` / 双击购买

## 效果飘字系统（2026-07-08 新增）

### BaseEffect.show_effect_tip(target_card, value_string)
- 在目标卡牌的 `effect_node.global_position` 处弹 TipsLabel
- 卡牌名自动从 `FileManager.card_id_mapping[Card_data.card_id][0]` 获取
- 颜色使用源卡牌的 `Card_data.color`
- value 为 String，调用方自行拼接：`"+3"` / `"×2"` / `"[color=red]符号变为×"` 等
- 备选：TipsLabel 的 BBCode 颜色需要全称（red/yellow/dodgerblue/limegreen/white），非单字母

### 调用位置
- 直接写在 `apply()` 中数据修改之后，紧接 `add_effect()` 调用
- 不要放在 `on_effect_triggered()`：apply 只在结算时调用一次，不是预览时
- 受影响卡牌多张时逐张调用

### 撞击卡牌名飘字
- `SettlementAnimator._on_label_hit`：FlyingLabel 撞击时用 `settlement_queue[current_card_index]` 取当前卡牌名和颜色，在撞击点弹飘字
- 颜色映射 `{"R":"red","Y":"yellow","B":"dodgerblue","G":"limegreen","W":"white"}` 必须转换

### 已完成的卡牌
- 红卡 10001-10018：全部
- 黄卡 20001-20017：全部
- 蓝卡 30001-30015：全部（不含 30010/30016/30017）

## 王牌飘字系统（2026-07-09 新增）

### BaseWildEffect 辅助方法
```gdscript
func get_wild_name() -> String
func show_self_tip(text: String, color_override := "") -> void
static func show_card_effect_tip(card_node: CanvasItem, cd: CardData, value: String) -> void
```

### 飘字规则
| 生效对象 | 王牌自己飘什么 | 目标牌飘什么 |
|---------|:----------:|:--------:|
| 运算牌 | 王牌名字 | 效果值（+6, ×2, 底数翻倍...） |
| 战场信息 | 效果描述 | — |

- 王牌名通过 `show_self_tip(get_wild_name())`，效果通过 `show_card_effect_tip(card_node, cd, "×2")`
- ON_SETTLE_STEP 的 context 提供 `active_zone_cards` 数组用于定位目标 Card 节点
- 手牌目标的特效（4008/4009）无 Card 节点，退化为 `show_self_tip` 显示效果描述
- `show_card_effect_tip` 参数类型为 `CanvasItem`（非 `Node2D`），因为 Card 是 Control 类型

### 非结算阶段 activate 动画
- `GameData.trigger_wild_effects()` 执行成功后自动播放 `WildCardUI.ani.play("activate")`
- 跳过 ON_SETTLE_STEP（该触发器由 PlayField._execute_wild_card_phase 通过 "play" 动画链路处理）
- ON_WILD_ACTIVATE 由 PlayField._trigger_active_wild 独立处理（已有 activate 动画）
- 信号驱动效果（1002/2008）需手动调用 `GameData.play_wild_activate_anim(id)`

### BaseCardData 迁移注意事项
- `GameData.stat` 是 Dictionary，不含 `base` 键
- 底数操作统一用 `GameData.base_card.val` / `GameData.base_card.init`
- 07-06 重构后部分王牌脚本仍有 `stat.base` 残留，需检查

## 音效系统（2026-07-09 新增）

### AudioConfig
- `sfx_mapping` — 单文件 key → AudioStream，`play_sfx_key(key)` 播放
- `sfx_sets` — 音频集 key → Array[AudioStream]，`play_sfx_set(key)` 随机选一/`play_sfx_set_multi(key, count)` 随机选 N

### 金币音效架构
- 写在 `InfoBoard.update_info()` 中，检测 `delta = target - _displayed_money`
- delta ≥ 15 → `play_sfx_set("coin_large")`，否则 `play_sfx_set_multi("coin", randi_range(1, 3))`
- 全项目 18 处 `player_money +=` 无侵入覆盖

### 卡牌音效接入点
- 拿牌：`op_card._state_drag_enter()` → `card_pickup`
- 放牌：`op_card._state_slot_enter()` → `card_place`
- 抽牌：`GameData.draw_cards()` → `card_draw`（仅实际抽到牌时播放）

## UI 音效系统（2026-07-11 新增）

### 架构
- **AudioManager（Autoload）**：`play_ui_click(key_override)` — 默认播 `ui_click` 音效集；`play_ui_hover(key_override)` — 默认播 `ui_hover` 单音效。key_override 可指定特殊音效。
- **场景根节点路由**：StartMenu / RoundMenu / PlayField 各有 `ui_sfx_click(key)` 和 `ui_sfx_hover(key)` 方法，内部调用 AudioManager。
- **编辑器连线**：UI 按钮的 `pressed` / `mouse_entered` 信号直接在编辑器连接到根节点的这两个方法，零代码接入。

### 设计决策
- 不在每个 UI 节点挂音效子节点（会增加场景复杂度）
- 不要求 UI 按钮挂脚本（很多纯场景节点无脚本）
- 根节点做路由、AudioManager 做播放，分离关注点

## 结算插队系统：bonus_requeue（2026-07-11 新增）

### 用途
效果牌触发「再次触发左侧牌的效果」时（如燃尽），需要让左侧牌**完整重播效果动画 + 重新 apply()**，而不是静默调用 apply()。

### 与 settlement_requeue 的区别
| 队列 | skip_effect | apply() | 用途 |
|------|:--:|:--:|------|
| `settlement_requeue` | true | 跳过 | 再放送：只重新参与数学计算 |
| `bonus_requeue` | 无（is_bonus=true） | 重新调用 | 燃尽：完整重播效果动画 |

### 机制
1. 效果在 `apply()` 中 `GameData.bonus_requeue.append(target_card)`
2. `_handle_post_effect_variants()` 检测 `bonus_requeue` 非空，将目标卡以 `is_bonus: true` 插入 `settlement_queue` 当前处理位置之后
3. 下一轮 while 循环处理该卡：`_animate_single_card_effect()` → `start_math_ani()` → `execute_slot_effect()` → `do_effect()` → `apply()` 重新执行
4. `is_bonus: true` 防止变种检查套娃

### 相关文件
- `GameData.gd`：`bonus_requeue: Array`（与 settlement_requeue 并列）
- `PlayField.gd`：`_build_settlement_context()` 清空、`_handle_post_effect_variants()` 处理

## 效果范围预览系统（2026-07-12 新增）

### BaseEffect.get_affected_indices()
在 `BaseEffect.gd` 中新增虚方法，所有效果脚本按需覆写：

```gdscript
func get_affected_indices(_slot_cards: Array, _self_index: int) -> Dictionary:
    return {}  # 默认无高亮（弃牌/手牌触发等非槽位目标效果）
```

### 返回格式
`{ slot_index: {"color": "R", "rng": bool} }`
- `color`: 单字母 "R"/"Y"/"B"/"G"/"W"
- `rng`: true 时显示较浅色（alpha=0.7），false/省略时显示实色（alpha=1.0）
- 空字典 = 无需高亮

### 实现规则（统一所有卡牌的预览行为）
| 效果类型 | 预览策略 | 示例 |
|----------|----------|------|
| 确定方向目标 | 实色高亮目标槽位 | 加法在前→右侧×牌，星火→右侧牌 |
| 确定自身目标 | 实色高亮自己 | 纯数值牌、二次利用、群众路线 |
| 条件触发（条件可提前判断） | 满足→实色，不满足→浅色 | 燃尽(effect_count)、燃点(红牌数)、乌合之众(弃牌数) |
| 条件触发（条件不可提前判断） | 始终浅色 | 爆燃（依赖结算时effect_count） |
| RNG 概率效果 | 始终浅色 | 押注、千术、老虎机、更高更强 |
| 半确定（部分RNG） | 确定部分实色，RNG部分浅色 | 播种（左侧实色，右侧浅色） |
| 全局状态/手牌/弃牌触发 | 空字典（不显示） | 蒸腾作用、春风、三十三小队、投资 |
| 无效果 | 空字典 | 还魂（apply返回false） |

### UI 架构
```
Card._on_mouse_entered (active_zone_index >= 0)
    → 0.2s SceneTreeTimer
    → _on_hover_preview_ready()
    → card_container.show_effect_preview(self)
        → effect_node.get_affected_indices(active_zone_cards, active_zone_index)
        → _effect_highlight_map = result
        → _update_slot_guide_highlights()

Card._on_mouse_exited / Card._state_drag_enter
    → _cancel_hover_preview()
    → card_container.hide_effect_preview()
        → _effect_highlight_map.clear()
        → _update_slot_guide_highlights()
```

### 相关文件
- `BaseEffect.gd` — `get_affected_indices()` 虚方法
- `ActiveCardContainer.gd` — `_effect_highlight_map`, `show_effect_preview()`, `hide_effect_preview()`, `EFFECT_COLORS`, `EFFECT_COLORS_RNG`
- `op_card.gd` — `_hover_preview_timer`, `_preview_wanted`, `_on_hover_preview_ready()`, `_cancel_hover_preview()`

### 覆盖统计（2026-07-13）
- 红牌 17 张 + 黄牌 20 张 + 蓝牌 6 张 + pure_numeric + BaseEffect = 45 个 `get_affected_indices` 实现

## 商店特价区（2026-07-12 重构）

### 变更
- 删除 GoodsEvent（事件商品）和 CardImprove（升级卡牌按钮）
- Panel1 → 特价区（"特价区"标签 + 1 个 GoodsPack 槽位），固定 8 折
- Panel2 → 3 个普通卡包槽位（不再有随机折扣）
- Panel3 → 王牌（3 个槽位，不变）
- Panel4 → 魔法（2 个槽位，不变）
- 颜色偏好按钮移至 ShopBackground 直接子节点

### 关键常量
- `ON_SALE_DISCOUNT = 0.8` — 特价区固定折扣
- `_purchased_on_sale: bool` — 独立于普通卡包的购买追踪
- 特价区不跟颜色偏好联动（纯随机，增强惊喜感）
- 刷新时特价区独立重新随机

### 升级卡牌
- 删除了商店升级按钮（`_on_card_improve_pressed` 及所有相关回调）
- 升级现在只能通过升级魔法（40001, 4$）实现
- 这提升了升级魔法的稀缺性和商店价值

## 倍增魔法 90001（2026-07-12 新增）

- **效果脚本**: `magicScript/effects/magic_double_score.gd`
- **类型**: 非目标魔法（NEEDS_TARGET=false），一次性消耗（consumes_slot() → false）
- **效果**: `player_money *= 2`，分数为 0 时 `show_tip("当前没有分数可以翻倍！")` 返回 false 不消耗
- **价格**: 18$（MagicConfig.gd 90001），产出：商店刷新 + Boss盲盒 HIGH档位
- **激活**: MagicSlot._on_magic_activated → effect_node.activate(null) → 成功后 _consume_magic() 移除

## InfoPage Lv1 商店按钮隐藏（2026-07-14 新增）

- **位置**: `info_page.gd:_setup_panels()`，所有 InfoPage 刷新路径的汇聚点
- **逻辑**: `shop_button.visible = GameData.now_game_level != 1`
- **为什么放 _setup_panels**: 覆盖三条刷新路径（首次揭幕 `_play_new_unit`、同单元返回 `_check_unit_state`、跨单元过渡 `_play_quite_then_new`），一处修改全路径覆盖
- **visible vs mouse_filter**: `_set_nav_buttons_enabled()` 仍控制动画期间的交互屏蔽，visible 控制 Level 1 期间的可见性，两者互补不冲突

## 系统参数扩展：利息上限 + 魔法容量（2026-07-14 新增）

### 利息上限 `interest_cap`

- **变量**: `GameData.interest_cap`，默认 5
- **利息公式**: `mini(floori(player_money / 5.0), GameData.interest_cap)`（`game_settlement_info.gd:60`）
- **效果**: 古战场遗迹「华贵的羽袍」→ `interest_cap += 5` → 变为 10$
- **EventEffect**: `INCREASE_INTEREST_CAP` #{amount: int} → `_exec_increase_interest_cap()`

### 魔法数量上限 `max_owned_magics`

- **变量**: `GameData.max_owned_magics`，默认 3（原为 `const MAX_OWNED_MAGICS`）
- **改 const 为 var**: 所有魔法容量检查（`can_hold_magic()`/`buy_magic()`/`acquire_magic()`）内部引用 `max_owned_magics`，自动适配动态值
- **外部引用同步更新**: `blind_box.gd`、`debug.gd` 中 `GameData.MAX_OWNED_MAGICS` → `GameData.max_owned_magics`
- **效果**: 古战场遗迹「磨损的腰包」→ `max_owned_magics += 1` → 变为 4
- **EventEffect**: `INCREASE_MAGIC_CAP` #{amount: int} → `_exec_increase_magic_cap()`

### 新增 EventEffect 汇总

| EventEffect | 参数 | 处理函数 | 飘字 |
|-------------|------|----------|------|
| INCREASE_INTEREST_CAP (27) | {amount: int} | `_exec_increase_interest_cap` | `利息上限+N (N$)` |
| INCREASE_MAGIC_CAP (28) | {amount: int} | `_exec_increase_magic_cap` | `魔法数量上限+N (N个)` |

均属于「始终成功」类型（`return true`），排布在 `PAY_MONEY` 之前，与已有 INCREASE_* 系列语义一致。

### 古战场遗迹重设计（EventConfig id=6, Lv9）

| 选项 | 效果 | EventEffect(s) |
|------|------|---------------|
| 华贵的羽袍 | 利息上限 5→10$ | INCREASE_INTEREST_CAP {amount: 5} |
| 磨损的腰包 | 魔法容量 3→4 | INCREASE_MAGIC_CAP {amount: 1} |
| 折断的魔杖 | 手牌上限 6→7 | INCREASE_HAND {amount: 1} |

旧版三个选项（先锋战旗/军备仓库/不朽意志）已废弃，新选项每个只有单一效果，且手牌+1是复用已有 INCREASE_HAND 零新增成本。

## 结局面板架构（2026-07-14 规划中）

### 现状
- `GameData.trigger_game_ending`（回归门下）和 `GameData.endless_mode`（迈向无尽）已被 Lv18 事件设置但**从未被任何代码检测**
- 无 EndingPage 场景/脚本
- Lv18 事件完成后直接进商店→InfoPage→玩家点击「下一把」进不存在的 Level 19

### 设计决策
- **EndingPage 放 CanvasLayer**：与 SelectCard 同级全屏覆盖层，不受 RoundMenu 页面切换影响，语义明确（终点而非中转站）
- **分支结局**: 归途（RETURN_TO_MASTER）和征程再续（ENDLESS_JOURNEY）均终止游戏→返回 StartMenu，差异仅为叙事文案
- **only Boss 关回合用尽即死**: 经济落后=慢性惩罚，非 Boss 关失败只是跳奖励而非结束游戏，与 Balatro 式经济循环一致

## 结局面板全局统计（2026-07-15 实现）

### 全局统计扩展

`global_stats` 字典新增两个字段：`total_score`（累计总得分）、`total_discard`（累计弃牌总数）。`commit_stats_to_global()` 在每局结束时累加当前局的 `current_score` 和 `total_discard_count`。

### capture_ending_snapshot 重设计

调用顺序调整为 **commit_stats_to_global → capture_ending_snapshot**（先累加全局再拍快照），确保快照拿到的是包含本局贡献的最新全局数据。

快照字段：`final_score`（本局）、`completed_level`、`is_endless`、`seed`、`total_score`（全局）、`total_cards_used`（全局）、`total_discard`（全局）、`mvp_card_id`（全局）。

MVP 卡牌从 `global_stats.card_usage` 中取累计使用次数最高的卡牌 ID，而非 `stats.card_usage`（仅本局）。

## SettlementSequencer 超时假阳性（2026-07-15 修复）

### 根因

`play_phase_requests()` 无条件创建 5 秒 `SceneTreeTimer`，空阶段（无请求）函数瞬间返回但定时器继续倒计时，5 秒后触发假阳性 warning。

### 修复（两层防御）

1. **空数组提前 return**：`_active.is_empty()` 时直接返回，不创建定时器
2. **`_phase_completed` 标志位**：循环结束后设 `_phase_completed = true`，定时器回调中检查该标志，已完成则静默 return

### 关键代码模式

```gdscript
var _phase_completed := false
var timer := get_tree().create_timer(PHASE_TIMEOUT)
timer.timeout.connect(func():
    if _phase_completed: return  # 已完成，静默
    timed_out = true
    push_warning(...)
)
# ... 处理请求 ...
_phase_completed = true
```

## 事件 PAY_MONEY 门控模式（2026-07-15）

### 用法

在事件的 `effects` 数组首位放置 `PAY_MONEY`，`event_page._on_button_pressed` 逐条执行效果，`PAY_MONEY` 返回 false（余额不足）时 `break` 中断链条，按钮恢复可用并弹出「金额不足！」提示。

```json
"effects": [
    {"effect": EventEffect.PAY_MONEY, "params": {"amount": 15}},
    {"effect": EventEffect.INCREASE_SLOT, "params": {"amount": 1}},
]
```

`PAY_MONEY` 放在首位是必要条件——它必须是 first gate，后续效果才是奖励。

## Resource duplicate 后 typed array 赋值（2026-07-15）

### 问题

Godot 4.7 对 `Resource.duplicate(true)` 后的副本执行 `typed_array_prop = []`（untyped Array 赋值给 `Array[int]`）会触发属性赋值校验失败。

### 解决

使用 `.clear()` 原地清空数组，不经过 setter：

```gdscript
# 错误
copy.applied_magic_ids = []
# 正确
copy.applied_magic_ids.clear()
```

同样适用于 `tags` 等其他 typed array 属性。

## 新手教程系统（2026-07-15 新增）

### 架构

`TutorialOverlay`（Control，挂在 GameWorld/CanvasLayer）负责步骤状态机 + 输入权限管理，不处理遮罩/高亮（由用户在场景中自行布置视觉引导）。

### 输入门控模式

不在 overlay 层吞鼠标事件，而是在目标脚本入口处通过 GameData flag 做门控：

```gdscript
# GameData.gd
var is_tutorial := false
var tutorial_allows_drag := false
var tutorial_allows_settle := false

# op_card.gd _gui_input 入口
if GameData.is_tutorial and not GameData.tutorial_allows_drag: return

# PlayField.gd _on_settlement_button_clicked 入口
if GameData.is_tutorial and not GameData.tutorial_allows_settle: return
```

### 步骤推进

通过监听已有信号自动推进，不轮询：
- `card_container.slot_changed` → 检测槽位被填充
- `SettlementButton.pressed` → 额外连接（与 PlayField 原有连接并存）
- `_SettlementAnimator.all_done` → 检测结算动画完成

### mouse_filter 策略

WELCOME 步骤设为 `MOUSE_FILTER_STOP`（捕获点击推进），其余步骤设为 `MOUSE_FILTER_IGNORE`（事件穿透到游戏元素）。

### 入口与退路

- **入口**：`StartMenu._on_tutorial_pressed()` → `TutorialOverlay.start_tutorial()` → `is_tutorial=true` → `init_game_from_deck()` → `change_screen("PlayField")` → 等待 `load_finish` → 开始引导
- **跳过**：`_on_skip_pressed()` → 清 flag → `reset_for_new_game()` → 回 StartMenu
- **完成**：`_finish_tutorial()` → 同上
- **兜底**：`_exit_tree()` 确保 flag 被清

### 注意事项

- `tutorial_allows_*` flag 不在 `_reset_state()` 中重置（教程模式标志不属于局内状态）
- 教程结束必须显式调用 `_cleanup_tutorial()` 清除所有 flag
- 详见 [[tutorial-system-20260715]]

## 教程 AnimationPlayer 集成（2026-07-19 新增）

### 架构

TutorialOverlay 场景中挂载 `$AnimationPlayer`，编辑器中创建 5 个动画：
- `step1`～`step4`：各步骤的循环演示动画（箭头指引、高亮等）
- `RESET`：停止循环动画 + 重置视觉状态（不循环）

### _play_anim 调用链

```
_enter_step(PLACE_FIRST) → _play_anim("step1") → RESET → step1(loop)
_enter_step(PLACE_SECOND) → _play_anim("step2") → RESET → step2(loop)
_on_settle_pressed()     → _play_anim("step3") → RESET → step3
_enter_step(DISCARD)     → _play_anim("step4") → RESET → step4(loop)
_enter_step(COMPLETE)    → _play_anim("")      → RESET only
```

- step3 特殊：在 `_on_settle_pressed` 中播放（非 `_enter_step`），因为需要在玩家点击结算时触发，与游戏结算动画同步
- step1/2/4 在 `_enter_step` 中播放，均为循环动画

### RESET 竞态防护

```gdscript
var _anim_chain_id: int = 0

func _play_anim(name: String) -> void:
    _anim_chain_id += 1
    var my_id := _anim_chain_id
    animation_player.play("RESET")
    animation_player.animation_finished.connect(func():
        if _anim_chain_id == my_id:  # 只有最新链路的回调才生效
            animation_player.play(name)
    , CONNECT_ONE_SHOT)
```

每次调用递增 ID，RESET 完成时检查 ID 是否匹配。若玩家快速操作导致两次 `_play_anim` 触发（如前一次 RESET 未播完就进入下一步），旧链路的回调因 ID 不匹配而跳过。

### 教程 DISCARD 步骤

- DISCARD 步骤进入时需强制显示弃牌区：`play_field.card_container._show_fold_zone()`
- 权限同拖拽步骤：`tutorial_allows_drag = true`, `tutorial_allows_settle = false`
- 弃牌检测：`_on_slot_changed` 中 `_current_step == Step.DISCARD` 时推进（`_discard_to_fold` 末尾会 emit `slot_changed`）
- FoldZone 平时由 `try_apply_focus`/`release_card_focus` 管理，教程中需绕过此机制手动展示

### SETTLE 步骤推进变更

旧版 `_on_settle_pressed` 直接 `_advance_step()`（SETTLE→WATCH），新版改为等待结算完成：

```
玩家点击结算 → _on_settle_pressed 播放 step3（不推进）
→ 游戏结算动画执行
→ _SettlementAnimator.all_done 信号
→ _on_settlement_done → _advance_step（SETTLE→DISCARD）
```

`_on_settle_pressed` 中检查 `play_field.seltting` 防重入，避免连点导致 step3 反复重播。

## 绿色内容封锁模式（2026-07-19 新增）

### 设计意图

绿色套牌已实现但搭配效果有待验证，测试版发布前全部封锁。所有代码和数据完整保留，恢复只需删除对应过滤行。

### 多入口封锁清单

封锁内容涉及多个独立系统入口，每个入口的过滤方式不同：

| 入口 | 过滤方式 | 关键代码 |
|------|----------|----------|
| 商店卡包 | 移出枚举列表 | `_all_pack_types()` 不含 GREEN_BASE/ADVANCED |
| 跨色卡包 | black_list 字段 | `GachaConfig.GREEN_BLACKLIST` + `open_gacha_pack()` 中过滤 |
| 商店王牌 | 按颜色字段过滤 | `_refresh_wild_cards` 跳过 `config[COLOR].color == "G"` |
| 商店魔法 | 黑名单数组 | `SHOP_MAGIC_BLACKLIST` +60004 |
| 盲盒魔法 | 硬编码跳过 | `_rnd_magic_from_pool` 跳过 60004 |
| 颜色偏好按钮 | 循环跳过 | `_on_color_button_pressed` B→"" 跳过 G |
| 初始套牌 | 可用列表移除 | `available_decks.erase("green_starter")` |
| 随机基础包事件 | 移出候选池 | 删除 GREEN_BASE |

### GachaConfig black_list 接线

`black_list` 字段在 GachaConfig 中一直存在但 `open_gacha_pack()` 从未消费（死字段）。新增过滤逻辑：

```gdscript
var black_list: Array = config.get("black_list", [])
if not black_list.is_empty():
    var filtered: Array[int] = []
    for cid in eligible_ids:
        if not black_list.has(cid):
            filtered.append(cid)
    eligible_ids = filtered
```

放在 `filter_tags` 筛选之后、品质分组之前。使用 card_id（int）匹配。

## 效果削弱模式：自举切断 + 方差控制（2026-07-19 新增）

### 案例：20005 更高更强

**原始问题**：效果引用「场上底数最大的牌」且包含自身，炫彩提升本牌底数后形成正反馈环（self-bootstrapping），与淬炼 combo 第 8 关打出 100 万+。

**修改方案（双管齐下）**：

1. **自举切断**：max 搜索排除自身（`c != Card_node`），最小改动切断正反馈
2. **方差控制**：RNG 从「必乘 × 概率再乘（×²）」改为「加法兜底 + 概率乘法」
   - 失败：`base + max_other × 0.5`（加法，黄色流派本家特色）
   - 成功：`base × max_other × 0.5`（乘法，保留爆发可能）

### 通用范式

当一个效果同时存在自举循环和方差过大的问题时：
1. 先切断自举（排除自身引用），这是 bug 修复
2. 再把方差从指数级拉到线性级（×² → + vs ×），这是强度调整

### preview_contribution 期望值公式

```gdscript
var prob: float = Effect_params.get("double_chance", 60) / 100.0
var expected := prob * (base * factor) + (1.0 - prob) * (base + factor)
sim_bases[index] = expected
```

用概率加权让预览分数反映长期平均，避免欺骗性高亮。
