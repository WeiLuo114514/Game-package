---
name: tutorial-system-20260715
description: 新手教程系统架构 — TutorialOverlay 状态机 + GameData flag 输入门控 + op_card/PlayField 集成点
metadata: 
  node_type: memory
  type: reference
  originSessionId: bea81b18-d45a-4dd2-b6cb-0d0ebd4cf8ec
---

# 新手教程系统

> 实现日期：2026-07-15

## 架构

```
TutorialOverlay (Control, GameWorld/CanvasLayer)
  ├── 步骤状态机（6 步）
  ├── 输入权限管理（GameData flag）
  ├── 信号监听（slot_changed / SettlementButton.pressed / all_done）
  └── 文字提示（instruction_text_label）
```

**设计决策**：不在 overlay 层吞鼠标事件来限制操作，而是在 `op_card.gd` 和 `PlayField.gd` 中通过 `GameData.tutorial_allows_*` flag 做门控。因为拖拽需要完整事件链，overlay 拦截会导致拖拽失效。

## 步骤流程

```
WELCOME → PLACE_FIRST → PLACE_SECOND → SETTLE → WATCH → COMPLETE
   ↑点击        ↑放槽位0       ↑放槽位1      ↑结算      ↑动画完成   ↑自动
```

每步自动设置 mouse_filter：WELCOME=STOP（捕获点击），其余=IGNORE（事件穿透）。

## 集成点（3 处微小改动）

| 文件 | 改动 |
|------|------|
| `GameData.gd:111-113` | 3 个 flag：`is_tutorial`、`tutorial_allows_drag`、`tutorial_allows_settle` |
| `op_card.gd:278` | `_gui_input` 入口：`if GameData.is_tutorial and not GameData.tutorial_allows_drag: return` |
| `PlayField.gd:214` | `_on_settlement_button_clicked` 入口：`if GameData.is_tutorial and not GameData.tutorial_allows_settle: return` |

## 入口

`StartMenu.gd:120-124` — `_on_tutorial_pressed()`：
```gdscript
func _on_tutorial_pressed():
    var overlay := get_node_or_null("../CanvasLayer/TutorialOverlay") as TutorialOverlay
    if overlay:
        overlay.start_tutorial()
```

`TutorialOverlay.start_tutorial()` 设置 `is_tutorial=true` → `init_game_from_deck()` → `change_screen("PlayField")` → 监听 `load_finish` → `_begin_guidance()`。

## 退路

- **跳过**：`_on_skip_pressed()` → `_cleanup_tutorial()` → `reset_for_new_game()` → 回 StartMenu
- **完成**：`_finish_tutorial()` → 同上
- **`_exit_tree()`**：安全兜底，确保 flag 被清

## 视觉引导

TutorialOverlay 不再处理遮罩/高亮。引导动画由用户在场景中自行布置（例如指向槽位/结算按钮的箭头动画），通过 `_current_step` 判断当前步骤来控制显示。

## 注意事项

- `tutorial_allows_*` flag 不在 `_reset_state()` 中重置（教程模式标志不属于局内状态）
- 教程结束必须显式调用 `_cleanup_tutorial()` 清除所有 flag
- `SettlementButton.pressed` 信号支持多连接，TutorialOverlay 作为额外监听者与 PlayField 原有处理并存
