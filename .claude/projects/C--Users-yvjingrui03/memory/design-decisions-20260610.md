---
name: design-decisions-20260610
description: 2026-06-10 设计决策 — 特殊运算牌变体、王牌镀层、魔法牌、卡牌升级、牌库永久化
metadata: 
  node_type: memory
  type: project
  originSessionId: 0654740a-3317-451c-b1da-b62233160d2a
---

**Why**：2026-06-08~10 多轮开发后，核心流程稳定，进入系统扩展阶段。本日确定了 4 个新系统的设计方案。

**How to apply**：以下系统均未实现，仅完成设计定稿。开发时以此为规范。

## 1. 牌库永久化 & 数据流重构（已实现）

- `draw_pile` 现在是永久牌库（游戏结束不重置）
- `_reset_state()` 不再清空 `draw_pile`
- 新增 `consolidate_permanent_deck()`：游戏结束时收集 draw_pile + hand + discard_pile + card_slots 所有非临时卡牌 → draw_pile
- `reset_for_new_game()`：从 draw_pile（永久牌库）加载，首次为空时从 initial_decks 创建
- 新增 `_is_temporary()` 接口（预留，后续实现标签系统）
- `card_container.gd` 读取 `GameData.draw_pile` 展示牌库
- `collected_deck` 仍保留为图鉴用途

## 2. 三选一抽卡动画重构（已实现）

- 旧 `select_card` 动画（AnimationPlayer 轨道 + Sprite2D 卡背拼接）已被替换
- 改为纯 Tween 驱动：背面降下（0.1s 间隔）→ draw 掀开（0.1s 间隔）→ tips/skip 渐显
- 选中卡牌飞向 Desk 按钮 → 其余 draw_back 翻面 → 向右飞出淡出
- Desk 按钮：开包期间禁用+移出屏幕，三选一阶段回归（允许查看牌库辅助决策）
- DrawCardPage 新增 `enter_page()`/`exit_page()`（into/out 动画），供 SELECTING 阶段进出牌库
- 选卡完成播放 `draw_finish` 退场
- 选中卡 → `GameData.draw_pile`，未选中/跳过 → 丢弃

## 3. ShowCard 信号统一（已实现）

- 新增 `signal card_clicked(card: ShowCard)`，左键点击发射
- 右键分支已注释
- `draw_Packcard_page` 通过 `card_clicked.bind(i)` 处理三选一
- `card_container` 通过 `card_clicked` 显示详情页
- 影子逻辑改为视口中心偏移（与 OPCard 一致）

## 4. RoundMenu 导航优化（已实现）

- 新增 `last_screen` 追踪上一页面
- Desk 按钮：仓库已打开→返回 `last_screen`，否则→打开仓库
- `_switch_to` 新增 `if target == now_screen: return` 防重复
- `_on_desk_pressed` 新增 `last_screen != card_warehouse` 防死循环

## 5. 特殊运算牌变体（设计定稿，未实现）

`CardData` 新增 `variant` 枚举字段（默认 NORMAL）：

| 变体 | 枚举值 | 效果 | 卡包概率 |
|------|--------|------|----------|
| 普通 | NORMAL | 无额外效果 | 默认 |
| 炫彩 | HOLO | 结算时 50% 概率效果触发两次 | 待定 |
| 错版 | MISPRINT | 结算时 40% 概率插队到结算队列前再结算一次 | 待定 |
| 闪光 | SHINY | 受到王牌增益时，增益效果翻倍 | 待定 |

**实现要点**：
- 炫彩/错版：`PlayField._perform_settlement_animated()` 结算循环中检查 variant
- 闪光：`BaseWildEffect._execute()` 对目标卡牌做参数计算时检查 `card_data.variant == SHINY`
- 卡包开出时按概率随机赋予 variant（`open_gacha_pack()` 中处理）

## 6. 王牌镀层（设计定稿，未实现）

`WildData` 新增 `coating` 枚举字段（默认 NONE）：

| 镀层 | 效果 | 风险 |
|------|------|------|
| 黄金 | 生效时 5% 概率获得 1$ | 无 |
| 镭射 | 持有时本局初始底数 +5 | 无 |
| 玻璃 | 效果翻倍 | 每回合开始 8% 概率销毁 |
| 幸运 | 商店刷新、卡包品质概率提升 | 无 |

**实现要点**：
- 黄金：`_on_executed()` 中 `randf() < 0.05` → `GameData.player_money += 1`
- 镭射：`ON_GAME_START` 时检查 → `GameData.initial_base_number += 5`
- 玻璃：`_execute()` 中参数×2；`ON_ROUND_START` 时 `randf() < 0.08` → `remove_wild_card()`
- 幸运：`open_gacha_pack()` 品质概率提升；`shop_page` 刷新时提高高品概率
- 镀层获取方式待定（建议：商店单独售卖镀层道具，可给任意王牌附加）

## 7. 魔法牌系统（设计定稿，未实现）

- 商店第三栏位（当前显示"功能尚未实装"）
- 魔法牌为一次性消耗品，购买后立即生效
- 可能的效果类型：货币增益、底数临时加成、手牌上限提升、回合数+1 等
- `goods_magic.gd` 需重写，`GoodsMagic` 场景需完善 UI
- 需新增 `MagicConfig.gd` 配置文件

## 8. 卡牌升级系统（设计定稿，未实现）

- 商店"卡牌升级"按钮（当前 5$，逻辑为空）
- 选中牌库中的一张卡牌，消耗货币提升等级（level+1）
- `CardConfig` 已有按等级的 effect_params 数组，升级后自动读取更高等级参数
- 升级界面：从牌库选卡 → 预览升级后属性 → 确认消费
- 费用公式建议：基础费 × 当前等级（等级越高越贵）

## 9. 现有实现进度快照（2026-06-10）

**已完成**：
- 核心玩法循环（draw→place→settle→end）
- OPCard 拖拽/槽位/手牌/弃牌管理
- 红色 17 个效果全部实现
- 王牌 12 个效果全部实现
- 商店（卡包 + 王牌购买）
- 开卡包 + 三选一翻牌动画
- 牌库仓库（网格 + 详情页）
- 18 关关卡配置
- 结算页面 + 奖励计算
- 调试控制台
- 页面切换动画（into/out）
- 牌库永久化数据流

**待实现**：
- 黄色 14/15 效果为空壳，蓝色 1/1 空壳，白色 3/4 空壳
- 绿色流派 0 卡牌 0 效果
- 仅 1 套新手牌组（红）
- Boss debuff 机制
- 魔法牌系统
- 卡牌升级系统
- 特殊运算牌变体
- 王牌镀层
- 音效/音乐
- 存档系统
- 新手教程
