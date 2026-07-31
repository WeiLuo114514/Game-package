---
name: infopage-anim-state-20260702
description: InfoPage 动画状态机规范 — AnimState 五状态/统一关卡检查/守卫规则/外部调用约定
metadata: 
  node_type: memory
  type: reference
  originSessionId: 6b9e87a8-cad4-4cf0-8104-ca01b028ceb2
---

## 状态机

```
enum AnimState { IDLE, ENTERING, EXITING, PLAYING_NEW_UNIT, PLAYING_QUITE_UNIT }
```

| 状态 | 触发 | 动画 | 完成后 |
|---|---|---|---|
| IDLE | 默认 | — | — |
| ENTERING | `enter_page()` → `_play_into_chain()` | "into" | → IDLE → `_check_unit_state()` |
| EXITING | `exit_page()` | "out" | `visible=false` → IDLE |
| PLAYING_NEW_UNIT | `_play_new_unit()` | "newUnit" | 启用按钮 → IDLE |
| PLAYING_QUITE_UNIT | `_play_quite_then_new()` | "quiteUnit" | → PLAYING_NEW_UNIT → IDLE |

## 统一关卡检查 `_check_unit_state()`

在 "into" 动画结束后调用，比较 `_current_unit_start`（当前已显示单元的起始关卡 1/4/7/10/13/16）与当前关卡 `GameData.now_game_level`：

- `_current_unit_start == 0` → 首次进入，播放 `newUnit`
- `new_unit_start != _current_unit_start` → 跨单元（BOSS 后），播放 `quiteUnit` → `newUnit`
- 同单元 → 仅 `_setup_panels()` + `_set_nav_buttons_enabled(true)`

## 协程守卫规则

**`_play_into_chain()` 和 `_play_new_unit()` 在 `await` 之后必须检查 `anim_state`：**

```gdscript
await animation_player.animation_finished
if anim_state != AnimState.ENTERING:  # 或 PLAYING_NEW_UNIT
    return  # 被 exit_page 中断
```

原因：`exit_page()` 可能在协程 await 期间被 `_switch_to` 调用，它会设置 `anim_state = EXITING` 并播放 "out" 打断当前动画。协程醒来后必须检测到此中断并中止。

## 外部调用约定

**RoundMenu._on_load_finish**：
```gdscript
if now_screen == info_page and info_page.anim_state == InfoPage.AnimState.IDLE:
    info_page.on_load_finish_check()
```

**RoundMenu._switch_to**：
- 调用 `exit_page()` 后 `await` 其 `animation_finished`
- 调用 `enter_page()` 启动后台协程，不等待
- 立即设置 `now_screen = target`

**_on_desk_pressed 守卫**：
```gdscript
if info_page.anim_state != InfoPage.AnimState.IDLE:
    return  # "newUnit" 动画的 Desk:disabled track 会误触发 pressed
```

**_set_nav_buttons_enabled**：
- 仅控制 `shop_button` 和 `next_game_button`
- Desk 由 "newUnit"/"quitUnit" 动画的 `disabled` track 控制，不在此处干预
