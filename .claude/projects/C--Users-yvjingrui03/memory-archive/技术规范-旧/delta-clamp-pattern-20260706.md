---
name: delta-clamp-pattern-20260706
description: turn_base_delta 写入时 clamp 技术规范 — 解决 max(0) 只读保护导致的「负数沼泽」问题
metadata: 
  node_type: memory
  type: reference
  originSessionId: 348d7a9c-1e40-46be-9fd7-da2852ac64d3
---

## 底数 delta clamp 写入模式

**Why**：旧 `get_display_base_number()` 用 `max(0, ...)` 做纯展示层 clamp，底层 `turn_base_delta` 仍为负值。量子纠缠（±3 jitter）把 delta 打到 -3 后，虚火 +1 只是从 -2 加到 -1，`max(0, -1) = 0`，效果被负数淹没。

**How to apply**：所有通过 `turn_base_delta` 修改底数的效果自然受益，无需逐个调整。

### 实现（card_data.gd）

```gdscript
func get_display_base_number() -> int:
    # 写入时 clamp：发现负数就把 delta 修正到地板
    if base_number + turn_base_delta < 0:
        turn_base_delta = -base_number
    return floori((base_number + turn_base_delta) * turn_base_mult)
```

### 对比

| 场景 | 旧（只读 clamp） | 新（写入 clamp） |
|------|------------------|-------------------|
| jitter=-3 → 虚火+1 → 末位×2 | max(0, (1-2)*2)=0 | delta→-1(地板) → +1→0 → ×2→2 |
| 多次读取 | 每次隐藏负数 | 首次后 delta 已修正 |

### 保护层级

1. `get_display_base_number()` — delta 写入 clamp（单牌底数）
2. `GameData.stat.base.val` — `maxi(0, ...)`（全局初始底数）
3. `check_final_score()` — `maxf(0, ...)`（结算分数入口+每步）
4. `get_preview_score()` — `maxf(0, ...)`（预览分数入口）
