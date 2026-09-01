---
name: base-number-formula-20260704
description: 底数计算公式与术语规范 — (base + delta) × mult，翻倍≠+base
metadata:
  type: project
  originSessionId: 7d05796f-d264-43de-902b-66b79c55bc49
---

## 底数计算公式

```
get_display_base_number() = floori((base_number + turn_base_delta) × turn_base_mult)
```

### 参数

| 参数 | 层级 | 含义 | 初始值 |
|------|------|------|--------|
| `base_number` | 永久层 | 卡牌原始底数 | 卡配置值 |
| `turn_base_delta` | 回合层 | 底数加算修正 | 0 |
| `turn_base_mult` | 回合层 | 底数乘算修正 | 1.0 |

计算顺序：**先加后乘** — delta 加在 raw base 上，mult 最后统一乘。

### 术语规范

- **「底数+N」「底数增加N」**：操作 `turn_base_delta += N`，累加到加算通道
- **「底数翻倍」「底数×N」**：操作 `turn_base_mult *= N`，乘到乘算通道
- 两者不等价：翻倍会放大所有已累积的 delta 收益

**Why:** 白矮星/引火曾用 `turn_base_delta += get_display_base_number()` 实现翻倍，导致 display 值（已含 mult）被注入 delta 通道，对虚火等已改 mult 的卡产生错误结果。

**How to apply:** 新效果实现「翻倍」语义时用 `turn_base_mult *= N`，实现「加底数」语义时用 `turn_base_delta += N`。描述文本严格区分。
