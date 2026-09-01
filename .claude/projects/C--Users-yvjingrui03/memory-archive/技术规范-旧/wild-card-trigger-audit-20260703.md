---
name: wild-card-trigger-audit-20260703
description: 王牌触发器全面审计 — 统一 ON_SETTLE_STEP，信号驱动计数，永久累加
metadata: 
  node_type: memory
  type: project
  originSessionId: 7d05796f-d264-43de-902b-66b79c55bc49
---

## 触发器时机规范（2026-07-03 终版）

| 触发器 | 触发时机 | 是否有结算动画 |
|--------|----------|:--:|
| `game_start` | 进入关卡时 | - |
| `round_start` | 每回合开始 | - |
| `draw_card` | 每抽一张牌后 | - |
| `discard_card` | 每次弃牌后 | - |
| `shuffle` / `shuffle_in` | 洗牌时 | - |
| `after_place` | 放置/移动/移除槽位牌后 | - |
| **`settle_step`** | **王牌结算阶段（CARD_EFFECT→WILD_CARD）** | **有 play 动画** |
| `after_settle` | 结算完成后（MATH_REVEAL→SETTLE_END） | 无动画 |
| `wild_activate` | 主动王牌：回合初充能，点击区牌触发 | activate 动画 |

- 全部 `ON_PRE_SETTLE` → 改为 `ON_SETTLE_STEP`，统一在王牌结算阶段播动画
- 不生效王牌直接跳过，无 `inoperative` 动画

## 信号驱动计数模式

需要「卡牌效果实际生效」才计数的王牌，用 `card_effect_triggered(color, card_node)` 信号：
- `_init()` 中连接信号（effect 节点不在场景树，`_ready()` 不触发）
- 信号在 `op_card.do_effect()` 的 `effect_success == true` 时发射
- 计数器存入 `wild_data.data["key"]`，**跨回合永久累加**，不取余不重置

### 已应用此模式的王牌：
- 1002 子母弹：红牌效果生效→RNG→`card_node.do_effect()` 再触发效果 + `play_wild_activate_anim(1002)`
- 1004 永恒火焰：红牌效果生效→`red_accumulated++`，结算时 `bonus = total/3`，计数器永不重置
