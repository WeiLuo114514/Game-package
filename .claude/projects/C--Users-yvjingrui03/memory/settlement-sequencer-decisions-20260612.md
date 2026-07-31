---
name: settlement-sequencer-decisions-20260612
description: 结算序列器实施前用户决策（已实施完毕 ✅）— Phase 划分 / Priority 规范 / 动画映射 / 效果清单 / 错误处理
metadata: 
  node_type: memory
  type: project
  originSessionId: 74b32d56-76bd-4074-8283-d7691ef56a0c
---

## 用户决策（2026-06-12）

**Why**：在 [[settlement-sequencer-design]] 架构设计定稿后，用户确认了 5 项实施前决策。

**How to apply**：以下为开发规范，实现时以此为准则。

### 1. Phase 终稿

```
SETTLE_BEGIN → WILD_CARD×N → CARD_EFFECT×N → MATH_BEGIN → MATH_REVEAL×N → SETTLE_END
```

- **SETTLE_BEGIN**：无标志性动画，用于卡牌【若生效】效果。**数据修改（execute）和动画（animate）在此阶段同步串行**——execute 改数据 → animate 播动画 → 下一步基于最新数据。`settlement_effect()` 仅做预览（apply 临时修改 + on_effect_triggered 提交 AnimationRequest），持久数据在 SETTLE_BEGIN 才真正提交。
- **MATH_BEGIN**：过渡阶段——手牌区下沉 + SettlementLabel 渐显。独立于 MATH_REVEAL。
- **CARD_EFFECT**：预留扩展点，当前无请求。

### 2. Priority 编号规范

| 范围 | 用途 |
|------|------|
| 0–99 | 默认卡牌效果动画 |
| 100–199 | 王牌介入动画 |
| 200–299 | UI 刷新动画（InfoBoard 等） |
| 900–999 | 收尾清理 |

### 3. 动画→Phase 映射

- 王牌逐个结算 + 粒子特效 → WILD_CARD
- 手牌区下沉 + SettlementLabel 渐显 → MATH_BEGIN
- 卡牌 play_result_ani + 数学运算 → MATH_REVEAL
- ON_AFTER_SETTLE + 标签淡出 + 卡牌飞出 + 手牌归位 → SETTLE_END

### 4. 需动画的效果清单

| ID | 名称 | 动画 |
|----|------|------|
| 10009 | 蒸腾作用 | InfoBoard 显示下回合手牌上限 +N |
| 20001 | 淬炼 | 自身底数变化 + play 动画 |
| 20003 | 金色种子 | InfoBoard 货币增加动画 |
| 20008 | 金色合唱团 | InfoBoard 初始底数更新动画 |
| 20009 | 麦浪 | InfoBoard 显示下回合点数 +N |
| 20010 | 投资 | 手牌区 play 动画 + 货币 +1 |

### 5. 错误处理策略

- 单动画最长 3 秒（标注规范，由 callback 内部控制超时）
- 动画失败：`push_error` + 跳过，不阻塞结算
- 无效请求：`push_warning`

### 6. 王牌触发时机规则（2026-06-20）

用户最终拍板：

- **默认（描述中无明确时机）**：王牌在 `WILD_CARD` 阶段，按王牌槽位顺序轮到自己时检查条件并生效。
- **描述中明确了生效时机**：王牌在对应时机点直接插队——监听目标 Phase 的 `phase_started` 信号 → 提交 AnimationRequest → execute 中插入 `settlement_queue`（复用现有 炫彩/再放送 的 `insert()` 模式）。

**Why**：统一规范，避免未来王牌设计时对「能不能在某时机触发」的反复讨论。架构上 `settlement_queue` 的 `while` + `insert()` 和 `Sequencer.phase_started` 信号已原生支持动态插队，无需重构。

**How to apply**：设计新王牌时，先判断描述是否含时机关键词（如「当…时」「结算时」「弃置时」），有则走插队路径，无则走默认 WILD_CARD 阶段。
