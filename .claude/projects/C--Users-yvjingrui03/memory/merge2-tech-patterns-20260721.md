---
name: merge2-tech-patterns-20260721
description: 二合爬塔RPG技术规范 — Autoload架构/输入系统/棋盘刷新/六元素效果/进度条Tween
metadata: 
  node_type: memory
  type: project
  originSessionId: 7d31385e-0275-4fad-9f80-d06fec28d61c
---

# 二合爬塔RPG 技术规范

## Autoload 架构

- **BoardManager**: 5×6 二维数组 grid[row][col]，null=空格。棋子为 Dictionary {element: int, level: int}
- **GameManager**: 玩家/敌人属性、回合状态、伤害公式。信号驱动 UI 更新
- 视图层（chess_board/UI/enemy）通过信号订阅 Autoload，不直接写数据

## 输入系统

- 棋盘 Chess Control 设 `mouse_filter = STOP`，棋子设 `MOUSE_FILTER_IGNORE`
- 使用 `_input(event)` 全局捕获鼠标事件（非 `_gui_input`），避免棋子边界限制
- `set_process_input(true)` 确保 Control 也能收到 `_input`
- 点击/拖拽判定：`_press_time` + `_press_pos`，松开时 `elapsed < 200 && dist < 80` = 点击
- `_process` 中持续更新拖拽棋子位置：`piece.position = mouse - size/2`
- 棋子命中检测 (`_piece_at`) 必须跳过 `_dragging_piece`，否则拖拽棋子自己盖住目标永远找不到
- 点击选中后`_dragging_piece.position = _drag_origin`弹回原位，不重建视图
- 点击触发动作用 `_selected_grid_pos` (Vector2i) 而非节点引用，视图重建后引用不会失效

## 棋盘视图刷新

- MVP 阶段全量重建：`queue_free` 所有子节点 → 遍历 grid 重新 instantiate
- `@onready` 变量在 `add_child()` 之后才初始化，`setup()` 必须放到入树之后调用
- `node.size = Vector2(180, 180)` 必须显式设置（`custom_minimum_size` 只在容器内生效）

## 进度条分层 Tween

- 顶层 ProgressBar（亮色）：瞬间更新 `.value`
- 底层 ProgressBar（暗色）：`create_tween()` → 延迟 0.3s → `tween_property(value, target, 0.4)` EASE_OUT QUAD
- BottomEnergy 显示预览值 `max(0, enemy_countdown - 1)` 而非当前值
- shield_bar 为单层条，无 Tween

## 六元素效果

| 元素 | 释放 | 合成附加 |
|------|------|------|
| 元 | 攻击系数 × 弱点修正 | 伤害 ×50%（非 25%） |
| 火 | 搭档系数 × 火倍率(lv) × 弱点修正 | 自身伤害 × 火倍率 |
| 水 | extend_countdown(lv) | extend_countdown(1) |
| 风 | extend_countdown(1 + lv/2) | extend_countdown(1) |
| 电 | 护盾 = 系数 × lv × 2 | 护盾 = 系数 × lv × 1 |
| 草 | 回复 = 系数 × lv × 2 | 回复 = 系数 × lv × 1 |

火倍率表: [0.75, 1.0, 1.25, 1.5, 1.75, 2.0, 2.25, 2.5, 2.75, 3.0]

## 伤害公式（v2 重设计）

- `攻击值 = BASE_DAMAGE[level] + ATK_RATIO[elem] × player_atk`
- BASE_DAMAGE: [5,10,18,30,48,72,105,150,210,290]（非线性）
- ATK_RATIO: 元 0.8 > 风/火/电/草 0.5 > 水 0.4
- 除法公式: `伤害 = max(1, floor(攻击值 × 3 / (3 + DEF)))`，DEF_K = 3.0
- 敌人→玩家同理: `max(1, floor(enemy_atk × 3 / (3 + player_def)))`
- 弱点 ×1.5, 抗性 ×0.5；火倍率 [0.75, 1.0, 1.25...3.0]

## 回合系统

- 每次操作 `tick_countdown()` → `enemy_countdown -= 1`
- 水和风在动作处理中提前 `extend_countdown(amount)` 加倒计时
- 倒计时归零 → `end_player_turn()` → 护盾衰减 30% → 敌人攻击 → 重置倒计时
- 护盾上限 = `player_max_hp × 2`
- **无 AP 系统**，仅用倒计时控制节奏

## Godot 4 类型注意事项

- Dictionary 返回类型不能含 null → 字段缺省用 `{}` 或去掉返回类型标注
- `custom_minimum_size` 仅作用于容器内的 Control，独立节点需手动 `node.size = Vector2(...)`
- `@onready` 在节点入树前不初始化，必须在 `add_child()` 之后才能访问
