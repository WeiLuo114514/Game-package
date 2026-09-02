---
name: tech-patterns-effects-triggers
description: 运算牌效果和王牌效果的技术规范 — 王牌修改规范(判定放_can_trigger失败静默/bonus_requeue重触发/ON_CARD_RESOLVED的is_bonus一牌一次判定/ON_MATH_BEGIN触发器/状态惰性初始化/结算期移除卡同步清队列/pending_blind_pack_type跨reset存活)、apply vs trigger vs signal 三套模式、ON_CARD_RESOLVED触发器、王牌触发器统一派发、种子系统+RNG四流架构、AnimationRequest、批量王牌效果串行同步(apply_card_effects_serial)、王牌结算重复飘名抑制、飘字、UI刷新、音效系统、bonus_requeue插队、效果范围预览系统、绿色封锁多入口模式、效果削弱范式（2026-09-01 更新）
metadata: 
  node_type: memory
  type: reference
  originSessionId: b794dd90-ecd7-4b3a-8970-3d8b78086b7a
---

# 效果系统技术规范

> 最后更新：2026-09-01

## 王牌修改规范（2026-09-01 从实测提炼，改王牌优先查这里）

### 1. 概率判定放 `_can_trigger`（失败静默）
- 概率类王牌把 RNG 判定写在 `_can_trigger`（返回 bool）。失败时 `try_execute` 返回 false → 框架 `_signal_wild_success(false)` 不播音效/动画。
- 判定放 `_execute` 里则失败也返回 true → 误播反馈。成功时框架自动播 activate 动画+声效，**不要手动再调 `play_wild_activate_anim`**（会重复播/打断）。
- 判定所需的 context（如 `card_node`）在 `_can_trigger` 里也能读。

### 2. 重触发用 `bonus_requeue`（禁止 `do_effect`）
- 「再结算一次」类效果（子母弹/暴怒/东山再起）：`GameData.bonus_requeue.append(card_node)`。框架 `_handle_post_effect_variants` 以 `is_bonus` 插回结算队列 → 该牌完整重播 math 动画+音效，播完才进下一张。
- 直接 `card_node.do_effect()` 绕过队列 → 动画并发乱序（bug 类）。

### 3. ON_CARD_RESOLVED 的 `is_bonus`：一牌一次判定
- `PlayField._execute_card_effect_phase` 派发 ON_CARD_RESOLVED 时 context 已带 `is_bonus`（取 entry）。重触发类王牌必须在 `_can_trigger`/`_execute` 检查 `context.get("is_bonus")` 并跳过 → 防无限插队。TagSystem.dispatch_one 同 context。

### 4. `ON_MATH_BEGIN` 触发器（stage_mods 类数学效果）
- `TriggerDefines.gd` 已新增 `ON_MATH_BEGIN`，在 `PlayField._animate_math_begin`（MATH_BEGIN 阶段，运算结算过渡）派发。
- 数学类 stage_mods（反方向的钟 `reverse_math` / 斗转星移 `swap_slot_1_and_n`）用此触发器 → 激活动画与效果生效同步。**不要挂 ON_PRE_SETTLE**（动画早播、效果脱节）。

### 5. ON_GAME_START = 通用开局触发器（2026-09-02 定稿）+ 状态惰性初始化
- **写法**：开局类效果（回合-1 / 目标分提升 / 状态初始化等）普通王牌与试炼牌**统一挂 `ON_GAME_START`**。派发点只有一处：`GameData.trigger_wild_game_start()` 在**每关（盲注）战斗开始 = PlayField load_finish（owned 装配后、round 1 前）**对 `owned_wild_cards` 全体派发（temp 实例，不依赖 UI 节点）；试炼牌也在 owned 中故天然覆盖。
- **触发语义**：每关触发一次。普通王牌每关都触发；要「整局一次」的效果需自身 once-guard。试炼牌只在它激活的那关触发（激活关次才有它的 owned/level 成员）。
- **试炼牌应用 vs 派发解耦**：`apply_boss_trial`/`force_boss_trial` 只负责「把试炼设为激活」（加入 level_wild_cards + owned），**不再自行派发**。debug `givewild` 对 9xxx 路由到 `force_boss_trial`（使 9xxx 成为激活试炼而非普通拥有），<9000 仍为普通拥有添加。
- 需「首次访问时确定并持久化」的状态（如 9002 的 N）仍用惰性辅助函数：ON_GAME_START / `get_current_description` / 首次结算任一先到即设，`set_state` flag 保证每 Boss 只执行一次（`wild_data.data` 与节点共享，WildData 每 Boss 重建自动清零）。
- 一次性开局惩罚/加成建议套「once-guard」：`if get_state(key,false): return; set_state(key,true)`。

### 6. 结算期移除槽位牌：同步清 settlement_queue
- WILD_CARD 阶段移除槽位牌时四件事：`discard_pile.append` + `on_card_to_discard` + **从 `settlement_queue` 删对应 entry**（防 `_prepare_math_queue` 访问已释放节点崩溃）+ `active_zone_cards[i]`/`active_zone[i]` 置 null + `queue_free()`（或右侧飞出动画）。

### 7. `pending_blind_pack_type`/`pending_ultimate_pack` 跨 reset 存活
- 盲盒在 `reset_for_new_game()` **之前**设值、RoundMenu **之后**消费 → `_reset_state()` 不清理这两个 flag，仅 `init_game_from_deck()`（新局）显式清除。

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


## ON_CARD_RESOLVED 触发器（2026-08-07 新增）

### 定位
填补王牌触发器体系的空白——在每张运算牌效果结算完毕后派发，context 含 `card_data`（CardData）和 `card_node`（Card 节点）。

### 定义
`TriggerDefines.gd`: `const ON_CARD_RESOLVED := "card_resolved"`

### 派发点
`PlayField._execute_card_effect_phase()`: 每张牌 `await _animate_single_card_effect()` 后调用：
```gdscript
GameData.trigger_wild_effects(Trigger.ON_CARD_RESOLVED, {"card_data": cd, "card_node": card})
```

### 用途
- 试炼王牌（9005/9006）：react to specific color card resolution
- 普通王牌（1002/1004/1006/2005/2008/2009/3008）：累计计数/概率再触发

### 与 card_effect_triggered 信号的区别
| | ON_CARD_RESOLVED 触发器 | card_effect_triggered 信号 |
|--|----------------------|--------------------------|
| 派发方式 | `trigger_wild_effects` 统一遍历 | 手动 connect/disconnect |
| 适用对象 | slot wild + level wild（统一） | 仅 slot wild（需持久节点） |
| 配置方式 | config trigger_list 声明式 | `_init` 中手动连接 |
| 架构定位 | 主要路径（2026-08-07 后） | 遗留（仅 audio/background 系统使用） |

### 迁移规范
所有「运算牌效果生效时」逻辑应走 ON_CARD_RESOLVED，不再使用 card_effect_triggered 信号：
```gdscript
func _execute(trigger: String, _context: Dictionary):
    if trigger == Trigger.ON_CARD_RESOLVED:
        var cd: CardData = _context.get("card_data")
        # 逐卡逻辑
```

### 已迁移的普通王牌
1002 子母弹、1004 永恒火焰、1006 普罗米修斯、2005 利息、2008 金黄扑满、2009 奇货可居、3008 死战不退


## 王牌触发器统一派发（2026-08-07 重构）

### trigger_wild_effects 覆盖率
`GameData.trigger_wild_effects(trigger, context)` 现在统一遍历两个来源：
```gdscript
func trigger_wild_effects(trigger: String, context: Dictionary = {}) -> void:
    # 槽位王牌
    for node in wild_slot.wild_card_nodes: ...
    # 试炼王牌（level_wild_cards）
    for wd in level_wild_cards:
        var effect := _make_temp_effect_instance(wd)
        effect.try_execute(trigger, context)
```

### 影响
- 试炼王牌现在参与所有触发器（ON_PRE_SETTLE/ON_CARD_RESOLVED/ON_DRAW_CARD/ON_DISCARD_CARD/ON_AFTER_SETTLE/ON_SETTLE_STEP），和槽位王牌完全一致
- `apply_level_wild_pre_settle` 仅保留 `stage_mods.clear()`，不再独立遍历试炼王牌
- `trigger_all_wild_effects` 已删除（冗余）


## 种子系统 & RNG 架构（2026-08-31 更新为 4 流拆分）

### 播种
`GameData.init_game_from_deck()` 开头：
```gdscript
if _rng_seed == 0:
    set_seed(hash(int(Time.get_unix_time_from_system())) & 0x7FFFFFFF)
else:
    set_seed(_rng_seed)  # 用户指定种子，每局重新应用
```

### 4 条独立随机流（都从 `_rng_seed` 确定性派生，互不消费）
| 流 | 状态变量 | 用途 | 调用方式 |
|----|---------|------|---------|
| 局内流 | `_rng_state` | 卡牌效果概率/抽洗/结算/Boss试炼/异彩判定 | `rng_randf/randi_range/randi/randf_range/shuffle` |
| 事件流 | `_event_state` | 事件池选取/事件效果发放 | `event_randf/randi_range/shuffle` |
| 商店流 | `_shop_state` | 商店货架/特价/盲盒/卡包内容 | `shop_randf/randi_range/shuffle` |
| 表现流 | `_cosmetic_rng` | UID/盲盒花纹/夸夸 | `cosmetic_randi_range/randf_range` |

`set_seed(s)` 用盐值打散派生机制流：
```gdscript
_event_state = _derive_stream_seed(s, 0x1E7A)
_shop_state  = _derive_stream_seed(s, 0x50C2)
```
`_derive_stream_seed` 与 `_step` 均含 `state == 0` → 时间回退保护（随机局也用时间，行为一致）。

### 流归属原则（关键）
- **局外内容**（事件/商店/卡包）必须走独立流 → 局内玩家操作（抽牌次数、随机触发）不推进局外流，同种子下商店/事件/卡包内容恒定
- **表现层**必须走 `cosmetic_*` → 不占种子流、也不要求复现
- 卡包内容（`open_gacha_pack` + `_roll_variant`）归**商店流**；事件决定"给什么包"用事件流，包内容用商店流
- 开局牌组抽取、Boss试炼、过关奖励（effect_5001 龙的财宝 / effect_5002 预制魔法）归**局内流**（关卡进程内容）；若要不随局内操作漂移需再拆"奖励流"

### GDScript 方法遮蔽规则
Autoload 类定义的同名方法优先于全局内置函数——仅在**该类内部**生效。外部脚本需显式写 `GameData.rng_xxx()` / `event_xxx()` / `shop_xxx()` 才能使用对应种子流。

### 已废弃命名
旧 `randf/randi/randi_range/randf_range` 已全部改名，**不要再调用**（方法不存在 → 返回 Variant → `:=` 类型推断报错）。所有机制随机必须走 4 流之一，表现层走 `cosmetic_*`。

### 接入范围
- **局内流**：GameData.gd 内部 + effectScript/wildCardScript 全部卡牌效果 + TagSystem + PlayField 洗牌
- **事件流**：event_page.gd 全部、EventConfig.gd 事件池、RoundMenu.gd 开局事件
- **商店流**：shop_page.gd 全部、blind_box.gd（除花纹）、GameData.open_gacha_pack/_roll_variant
- **表现流**：magic_copy_card UID、blind_box 花纹、effect_5004 夸夸、tips/音效/震动/动画等
- **故意未接入**：纯表现层（tips_label 喝彩/动画、audio_manager 音效、camera_2d 震动、flying_label 弧线、LoadMask 提示、GameData 胜利彩带）


## 标签系统架构（2026-08-08 重构）

### TagSystem Autoload

所有标签行为在  集中注册，流程节点通过  轮询。



### 调度接口

| 方法 | 用途 | 返回值 |
|------|------|--------|
|  | 单卡事件轮询 | void，通过 ctx 传结果 |
|  | 批量轮询 | void |
|  | 短路查询 | bool |
|  | 最值查询 | Variant |
|  | 累加查询 | int |

### 消费点迁移

迁移前：has_tag 散落在 PlayField/GameData/card_container/card_data 5 个文件
迁移后：所有标签行为在 TagSystem._register_all() 一处注册

### 新增标签 CHECKLIST

1. : TagType 枚举 + TAG_PRIORITY_ORDER + TAG_COLORS
2. : _register_all() 注册 handler
3. 流程节点：dispatch_one 调用（如 PlayField._fly_out_cards_and_clear_slots 的 BOUNCE）
4. : 两个 TAG_DESCS 块添加条目
5. : 对应魔法配置（如有）

## 魔法预览系统（2026-08-08 重构）

### 旧架构

select_card._preview_effect() 用 match _magic_id 硬编码 11 个 case。加新魔法需改 select_card.gd。

### 新架构

BaseMagic 提供 virtual ，每个魔法脚本覆写。
select_card 加载脚本 → 实例化 → 调用 preview_effect(target) → queue_free()。



consumes_slot() 替代硬编码豁免列表（）。

## 卡牌预升级（2026-08-08）

- GameData.UPGRADE_CHANCE := {0: 0.40, 1: 0.25, 2: 0.10}（C/B/A，品质越低概率越高）
- open_gacha_pack 中摇完品质+变种后判定：
- 使用 card_data.quality（实际品质）而非 selected_quality（可能降级）

## 卡牌统计账簿（2026-08-08）

- card_ledger: { card_id: { levels, uses, best_score } }
- flush_level_to_ledger(level_score)：每关结算调用，取 stats.card_usage 增量
- _last_committed 防重复提交，随 _reset_stats 清空
- 耐久性：每关写入，不等到局末

## 特价区架构（2026-08-08）

- on_sale_slots = [GoodsPack, GoodsMagic, GoodsWildCard]（3 个预放组件）
- _refresh_on_sale：按 category 权重选出 1 个，只显示该组件
- 购买路由：三个独立 handler（_on_sale_pack_purchased / _on_sale_magic_purchased / _on_sale_wild_purchased）
- GoodsMagic/GoodsWildCard 新增 discount_rate 字段，refresh_state 显示折后价
- 当前 category 写死为 0（只出卡包），后续王牌效果解锁魔法/王牌类别
## 标签系统架构（2026-08-08 重构）

### TagSystem Autoload

所有标签行为在 `TagSystem._register_all()` 集中注册，流程节点通过 `dispatch_one(event, card, ctx)` 轮询。

事件型标签将结果写入 ctx 字典，调用方读取 ctx 决定后续行为：

```
register(T.RETAIN, "on_end_round", func(_card, ctx): ctx["retain"] = true)
register(T.BOUNCE, "on_after_resolve", func(_card, ctx): ctx["bounce"] = true)
```

查询型标签通过 query_any / query_max / query_sum 同步返回值：

```
register(T.CHAMELEON, "query_has_color", func(_card, _target): return true)
register(T.SUPERBODY, "query_effective_level", func(_card): return 3)
```

### 调度接口

| 方法 | 用途 | 返回值 |
|------|------|--------|
| dispatch_one(event, card, ctx) | 单卡事件轮询 | void，通过 ctx 传结果 |
| dispatch(event, cards, ctx) | 批量轮询 | void |
| query_any(method, card, default, extra_args) | 短路查询 | bool |
| query_max(method, card, default) | 最值查询 | Variant |
| query_sum(method, card, default) | 累加查询 | int |

### 消费点迁移

- 迁移前：has_tag 散落在 PlayField/GameData/card_container/card_data 5 个文件 8 个消费点
- 迁移后：所有标签行为在 TagSystem._register_all() 一处注册
- 新增标签 CHECKLIST：card_data 枚举 → TagSystem 注册 handler → 流程节点 dispatch → CardUI TAG_DESCS → MagicConfig 魔法配置（如需）

### TagType 编号

- 掩码→顺序：1,2,4,8,16,32,64,128 → 1,2,3,4,5,6,7,8,9,10,11,12,13
- Array[int] 存储 + t in tags 查找，掩码无意义
- 所有引用走枚举名（TAG_DESCS/TAG_TEXTURES/EventConfig），改值自动跟随

## 魔法预览系统（2026-08-08 重构）

### 旧架构

select_card._preview_effect() 用 match _magic_id 硬编码 11 个 case。加新魔法需改 select_card.gd。

### 新架构

BaseMagic 提供 virtual `preview_effect(_target: CardData)`，每个魔法脚本覆写。
select_card 加载脚本 → 实例化 → 调用 preview_effect(target) → queue_free()。

consumes_slot() 替代硬编码豁免列表（`_magic_id != 20001 and ...`）。

## 卡牌预升级（2026-08-08）

- GameData.UPGRADE_CHANCE := {0: 0.40, 1: 0.25, 2: 0.10}（C/B/A）
- 品质越低概率越高，BASE 包 ~52% 至少 1 张升级
- open_gacha_pack 中摇完品质+变种后判定
- 使用 card_data.quality（实际品质）而非 selected_quality

## 卡牌统计账簿（2026-08-08）

- card_ledger: { card_id: { levels, uses, best_score } }
- flush_level_to_ledger(level_score)：每关结算调用，取 stats.card_usage 增量
- _last_committed 防重复提交，随 _reset_stats 清空
- 每关写入保证耐久性（崩溃不丢数据）
- card_data.increment_effect_count(delta)：激发标签在此 +1 额外计数

## 特价区架构（2026-08-08）

- on_sale_slots = [GoodsPack, GoodsMagic, GoodsWildCard]（3 个预放组件）
- _refresh_on_sale：按 category 选出 1 个，只显示该组件，其余隐藏
- 购买路由：三个独立 handler（_on_sale_pack_purchased / _on_sale_magic_purchased / _on_sale_wild_purchased）
- GoodsMagic/GoodsWildCard 新增 discount_rate 字段，refresh_state 显示折后价（仅折后价，不含划线和百分比标记）
- 当前 category 写死为 0（只出卡包），后续王牌效果解锁魔法/王牌类别
- 特价魔法候选池排除 90001 + SHOP_MAGIC_BLACKLIST
- 特价王牌候选池仅 B/A 品质

## OPCard 标签显示（2026-08-08 修复）

- 标签改为静态构建：在 _update_static_ui() 中紧跟 EffectLabel.text 设置后一次性拼接
- 删除 _append_tags_to_effect_label() 追加/清洗模式（split("。") 无法正确清理 \n 后的标签行）
- 格式：同排用 `、` 分隔，无 `【】`，末尾起一行
- _state_slot_enter 中强制重置 hover_Rect.visible=false + _change_scale(normal_vec)

## 批量王牌效果串行同步（2026-09-01 新增）

### 问题
批量型王牌（连锁引爆 1001 / 你先别急 L9001 / 红灯 L9005 / 警戒线 L9006 / 蓝色忧郁 L9007）对多张运算牌的效果原本一次性全部生效，但飘字逐张 0.6s 间隔 → 效果动画与飘字脱节。

### 模式：apply_card_effects_serial（BaseWildEffect.gd 静态方法）
```gdscript
static func apply_card_effects_serial(cards: Array, apply: Callable) -> void
```
- `apply: Callable(card: Card) -> String`：同步修改该卡数据并返回飘字文本（空串 = 无变化不飘）
- 收集窗口内（`SettlementSequencer.is_collecting()`）逐张提交一个 `AnimationRequest`（execute 改数据 / animate 播动画），由 `play_phase_requests` 的 gap 间隔驱动天然串行
- 非收集窗口回退为同步 `_apply_and_show_card_effect`

### 关键实现细节
1. **数据改动推迟进 request 的 execute**：`_execute` 循环不再同步改数据。这样 `PlayField._make_wild_fired_callback`（on_fired）比对 before_snap 发现「未变化」→ 批量 `_update_ui` 自动 no-op，无需改它。
2. **飘字文本跨 lambda 传递必须用数组持有者**：
   ```gdscript
   var tip_holder: Array = [""]
   func(): tip_holder[0] = str(apply.call(card))   # execute
   func(): _apply_and_show_card_effect(card, tip_holder[0])  # animate
   ```
   局部标量变量跨 execute/animate 两个 lambda 写读不可靠（animate 读到空 → 飘字消失）；数组是引用类型，`[0]` 原地改必然同步。
3. **末位牌数字滚动被掐断**：`_update_ui`（CardUI.gd）**无条件** `_number_tween.kill()`。批量串行时末位牌滚动刚启动，`_refresh_slots_ui()` 立即掐断 → 标签冻在旧值。修复：animate 里 `await tree.create_timer(0.4).timeout`（滚动≈0.35s）等播完再返回；`gap_after` 收紧 0.2s（合计≈0.6s 贴合原飘字节奏）。
4. **阶段超时**：批量逐个播放较慢，`_apply_pre_settle_mods` 与 `_execute_wild_card_phase` 的 `play_phase_requests()` 放宽为 `play_phase_requests(20.0)`（5s 默认会跳掉后段卡的数据变更）。

### 王牌结算重复飘名抑制（同批修复）
- `SettlementSequencer` 新增 `current_phase: int`（emit_phase 记录、play_phase_requests 结束清空为 -1）
- `BaseWildEffect.show_self_tip`：`current_phase == Phase.WILD_CARD && text == get_wild_name()` → return（名字已在阶段开始由 PlayField 显示过）。自定义飘字（如「初始底数+N」）不受影响。
- `PlayField._execute_wild_card_phase` 阶段开始的名字显示：
  - 位置用 `effect.global_position`（卡面中心）而非 `wild_node.global_position`（节点原点，结算前是 0,0）
  - 配置用 `FileManager.get_wild_config(id)`（兼容试炼王牌 9xxx，`wild_id_mapping` 不含 9xxx）
  - 颜色经 `WildCardUI.COLOR_NAMES.get(key, "white")` 映射（bbcode 需要 "red" 非 "R"）


## 试炼王牌调优笔记（2026-09-02）

> 触发器语义速查：**「王牌效果结算时」= ON_SETTLE_STEP**；**「游戏开始时/开局」= ON_GAME_START**（见规范5，已统一派发）。开局惩罚/加成一律加 once-guard。

- **三色试炼（L9005/9006/9007）范式**：底数增益挂 `ON_SETTLE_STEP`（王牌结算，`apply_card_effects_serial` 对同色牌逐张 +1 飘字）；同色牌结算/弃置抬目标分挂 `ON_CARD_RESOLVED`/`ON_DISCARD_CARD`，分支内 `goal*=1.05/1.08` + `GameData.battle_info_changed.emit()` 刷面板 + 飘「目标分数 +X%」。条件判断放 `_can_trigger`（同色判定统一 `GameData.has_color`），未命中静默、不误播 activate 动画（L9001 是 ON_SETTLE_STEP 逐张改底数的先例）。
- **L9004 东山再起**：ON_SETTLE_STEP `goal*=1.15`（逐结算复利，desc 目标分+15%）；ON_CARD_RESOLVED 末尾牌重触发不变。
- **L9008 奖励关**：惩罚 ON_GAME_START 一次性回合-1（once-guard）；奖励 ON_SETTLE_STEP 每剩余1费用→1$（`remaining=cost.max-cost.spent`，`cost.spent`=场上牌总费用，清场才归零，故结算瞬间量到本回合未用额度）。
- **L9009 卡牌大师**：抽牌 15% 改色成功时 `show_self_tip("卡名 变X", 新色bbcode)`。
- **L9011 量子纠缠**：改费用在 `CardData.cost_shift` 记方向（-1/+1/0），CardUI CostLabel 按方向着色（变低 #4da6ff 蓝 / 变高 #ff6666 红 / 0 白或 LIMITed 蓝）；`GameData.on_card_to_discard` 里 `cost_shift=0` 弃牌恢复。改色值只需动 CardUI 那两行。
- **L9013 资产税**：`ON_SETTLE_STEP`，每王牌结算 `goal += goal*money*0.02`（desc 与 code 2% 已对齐；money 随利息增长会复利放大税基）。
- **L9014 倾囊相授**（最简单版）：ON_GAME_START 一次性 `max_round += 1`（奖励回合）+ `goal*=1+0.25*owned_wild_cards.size()`（每王牌目标分+25%，含试炼自身），once-guard。
- **L9015 傲慢**：N 惰性初始化（同 9002，`slot_n` 0-based、`get_current_description` 显示「目标牌位:N+1」）；ON_SETTLE_STEP（王牌结算）把 N 号位牌底数翻倍（`turn_base_delta += base+delta`），**N 号位空则本回合跳过、N 不变**；生效后 N 随机改变；ON_GAME_START 目标 ×2。
- **L9016 嫉妒**：改为 ON_SETTLE_STEP **自行处理**（参照 9002 移除链路：弃牌 + on_card_to_discard + 清 settlement_queue + 槽位置空 + 右飞离屏），找场上最高/最低底数牌，最高牌底数永久加到最低牌上；原 PlayField `discard_highest` stage_mod 分支已删。
- **L9020/9021 统一迁到 ON_SETTLE_STEP**（stage_mod 在 SETTLE_END 消费，WILD_CARD 设置仍早于消费，安全）：9020 color_score=0.20（另 ON_GAME_START 目标×2）、9021 score_mult=1.5+cost_per_empty=1（王牌结算飘「得分×1.5·下回合费用-N」）。
- **收尾乘分类（color_score/score_mult）的显示坑**：math 动画已把 InfoBoard 刷到乘前终值，settle_end 再乘大 current_score 时**必须 `battle_info_changed.emit()`** 刷新，否则界面数字不变像「没生效」（通关判定已用乘后值）。此 emit 统一加在 `PlayField._apply_after_settle_stage_mods` 末尾。效果是否真生效可先用两行探针（设 flag / 消费分支各 print）快速定位。
- **L9019 贪婪**：迁 ON_SETTLE_STEP（直接 `base_card.val += money`、`money ×0.8`，飘字显示实际加值 added 而非扣后值）。
- **L9022 斗转星移**：N 存 `GameData.level_swap_n_value`（PlayField swap 用，非 wild_data），惰性初始化 + `get_current_description`「目标牌位:N」。**PlayField swap 段**：物理换位（`await _perform_settlement_swap`）后 N 重roll 保证 ≠ 旧值；结算顺序**强制按 `active_zone_index` 升序排序** = 换位后场上新左→右（勿依赖 swap 内部按队列索引配对，牌被移除/空槽会失配）。
