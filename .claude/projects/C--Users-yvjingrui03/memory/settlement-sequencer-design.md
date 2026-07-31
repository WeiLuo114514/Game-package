---
name: settlement-sequencer-design
description: 结算动画序列器架构设计 — 触发器阶段划分 + AnimationRequest 统一规范 + GameData 总控流程
metadata: 
  node_type: memory
  type: project
  originSessionId: 05dab61f-f6f9-4b20-9b50-165572f1cac1
---

## 设计目标

将当前硬编码的结算动画管线重构为**触发器阶段驱动 + 动画请求异步播放**的可插拔架构。新增触发器或动画时不需改核心代码。

## 核心概念

### Trigger Phase（阶段枚举）

结算流程拆成可枚举的阶段，每个阶段之间可 `await` 暂停：

```
SETTLE_BEGIN        ← 结算开始
WILD_CARD           ← 单个王牌结算（循环，每张王牌一个阶段）
CARD_EFFECT         ← 单张卡牌效果展示（循环，每张牌一个阶段）
MATH_REVEAL         ← 单张卡牌数学运算展示（循环，每张牌一个阶段）
SETTLE_END          ← 结算收尾（ON_AFTER_SETTLE + 标签淡出 + 卡牌飞出 + 手牌归位）
```

### AnimationRequest（统一动画规范）

所有节点提交的动画请求遵循同一接口：

```gdscript
class AnimationRequest:
    var name: String           # 调试用
    var priority: int = 0      # 同阶段内排序
    var execute: Callable      # 数据修改（同步，可选，大多数情况已在 on_effect_triggered 中完成）
    var animate: Callable      # 动画播放（可 await），返回 bool 表示是否实际播放了动画
```

### 信号 + 请求机制

1. GameData 进入新阶段 → `emit_phase("CARD_EFFECT", ctx, requests_array)`
2. 注册了该阶段的节点收到信号 → 检查条件 → 符合则向 `requests_array` 追加 `AnimationRequest`
3. GameData 收集完毕 → 按 priority 排序 → 依次 `await req.animate.call()`
4. 全部播放完毕 → GameData 进入下一阶段

GameData 在 emit 和播放之间的窗口内**阻塞等待**，保证数据一致性。

### 完整时间线

```
Phase: SETTLE_BEGIN
  ├─ emit → 王牌1 提交请求 → await animate
  ├─ emit → 王牌2 提交请求 → await animate
  └─ ...

Phase: CARD_EFFECT ×N（左→右）
  ├─ Card[0]: execute → emit → 效果请求 → await animate
  ├─ Card[1]: execute → emit → 效果请求 → await animate
  └─ ...

Phase: MATH_REVEAL ×N（左→右）
  ├─ Card[0]: play_result_ani → 运算展示 → await
  ├─ Card[1]: play_result_ani → 运算展示 → await
  └─ ...

Phase: SETTLE_END
  ├─ emit → ON_AFTER_SETTLE 效果
  ├─ 标签淡出 → 卡牌飞出 → 槽位清理
  └─ 手牌归位
```

### 数据流

每一步：execute（改数据）→ animate（播动画）→ await → 下一步基于最新数据继续。动画展示的是当前步执行后的**阶段性结果**。

### 效果脚本改动

以金色种子为例：

```gdscript
func on_effect_triggered(_ctx):
    var money = Effect_params.get("money", 1)
    GameData.player_money += money
    GameData.submit_request(AnimationRequest.new(
        "金色种子", 0,
        func(): pass,
        func(): await play_coin_fly(money); InfoBoard.update_info(); return true
    ))
```

## 重要纠正

**`settlement_effect()` ≠ 结算流程。** `settlement_effect()` 是牌位变更时的**实时预览**（玩家拖放卡牌到槽位后立即调用），内部的 card_reset + do_effect 是为了展示效果预览。正式结算（`_perform_settlement_animated`）不包含 card_reset 阶段，卡牌效果也不重新执行。

**王牌 ON_AFTER_PLACE 在 `settlement_effect()` 中触发**，不是结算流程的一部分。`_perform_settlement_animated` 中只有 ON_SETTLE_STEP（王牌动画）和 ON_AFTER_SETTLE（结算后）。

## 实施策略（2026-06-12 全部完成 ✅）

1. ✅ `SettlementSequencer` Autoload 单例（AnimationRequest 类 + Phase 枚举 + emit_phase 方法）
2. ✅ 改造 `_perform_settlement_animated` 为 Phase 驱动
3. ✅ 逐个迁移效果脚本的动画注册
4. ✅ 黄色卡牌的 on_effect_triggered bug 随迁移一并修复
