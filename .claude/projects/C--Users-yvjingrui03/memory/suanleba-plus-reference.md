---
name: suanleba-plus-reference
description: Godot 4.6 旧版参考项目《算了吧？plus》，位于 Documents/算了吧？plus
metadata: 
  node_type: memory
  type: project
  originSessionId: 9e74e843-3fb5-489c-814c-307ca232792e
---

**Why**：用户要求修改新项目（[[suanle-project]]）时参考此旧案。旧案功能更完整（有完整王牌系统、结算动画、卡包抽卡等），新项目是精简重构版本。

**How to apply**：实现新功能时，先在此旧案中找到对应的实现代码（英文目录 `code/`），理解其逻辑，然后适配到新项目（中文目录 `代码/`）。

**项目路径**：`C:\Users\yvjingrui03\Documents\算了吧？plus`

**关键差异**：
| 方面 | 旧案 (算了吧？plus) | 新案 (算了) |
|------|---------------------|------------|
| 核心类 | `PlayUI` | `PlayField` |
| 卡牌类 | `Card` (card.gd) | `Card` (op_card.gd) |
| 目录 | 英文 (`code/`, `screen/`) | 中文 (`代码/`, `场景/`) |
| 结算 | 有动画 (light_ball, settlement_number) | 已移植逐卡动画 |
| 王牌系统 | 完整 (hands, slots, 描述面板) | 部分实现 |
| 分数字段 | `total_score`, `initial_number` | `current_score`, `base_number` |
| 费用 | 动态 (按回合分阶段) | 固定 max_cost=9 |
| 手牌排序 | 有 sort_hand_cards() | 已移植 sort_hand_by_value/color |

**已从旧案移植的功能**：
- 逐卡结算动画 + 卡牌飞出屏幕
- 结算时手牌下降动画
- 手牌排序（按底数、按颜色）
- 信息面板颜色方案
- SettlementLabel 结算分数显示
- `seltting` 标志位阻止结算中交互
