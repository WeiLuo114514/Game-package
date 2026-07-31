---
name: tag-system-20260628
description: 卡牌标签系统 — TagType 枚举，专属/临时/消耗三标签互斥，2026-06-28 完成
metadata: 
  node_type: memory
  type: project
  originSessionId: ff04781d-6f43-4508-9f13-4122a2cdbe7b
---

## 标签系统 (TagType)

CardData 新增 `tag` 属性（`TagType` 枚举），与异彩（Variant）平行的独立维度。2026-06-28 实现完成。

### TagType 枚举（card_data.gd:6）

| 值 | 枚举 | 含义 | 行为 |
|----|------|------|------|
| 0 | NONE | 无标签 | 默认，正常进出弃牌堆 |
| 1 | EXCLUSIVE | 专属 | 魔法槽位=3（无视品质），不可被复制，可由专属魔法覆盖任意标签 |
| 2 | TEMPORARY | 临时 | 进弃牌堆时直接销毁（不进弃牌堆/消耗堆），手牌溢出时跳过 |
| 3 | CONSUME | 消耗 | 进弃牌堆时路由到消耗堆（consume_pile），不进弃牌堆 |

**Why**：绿色套牌的森·木循环需要「临时衍生卡」机制，焚身需要「用后即消耗」机制，专属魔法需要「不受品质限制的魔法槽」——三个需求收敛为一个标签系统。独立于异彩维度，避免枚举膨胀。

**How to apply**：标签互斥，一张牌只能有一个标签。专属魔法可以覆盖任何已有标签（包括临时/消耗→专属）。消耗堆是永久性的，不随回合重置。

### 赋值来源

| 标签 | 赋值位置 | 方式 |
|------|---------|------|
| EXCLUSIVE | `magic_exclusive.gd:10` | 专属魔法施加 |
| EXCLUSIVE | `select_card.gd:136` | 预览/三选一上下文 |
| TEMPORARY | `森.gd:14` | 森创造木时赋予 |
| TEMPORARY | `GameData.new_OPcard():472` | CardConfig extras["tag"] |
| CONSUME | `焚身.gd:6` | init_with_card 自赋予 |
| CONSUME | `CardConfig.gd:112` | 焚身的 extras["tag"] 配置 |

### 关键交互

- **销毁**：`GameData.on_card_to_discard()` — TEMPORARY 直接 erase，CONSUME 进 consume_pile
- **手牌溢出**：`PlayField.gd:723-726` — TEMPORARY 跳过不回收，CONSUME 进消耗堆
- **复制阻止**：`magic_copy_card.gd:14` — EXCLUSIVE 牌不可复制，副本 tag=NONE
- **魔法槽位**：`card_data.get_max_magic_slots()` — EXCLUSIVE 固定返回 3
- **魔法覆盖**：`magic_exclusive.gd:8-10` — 已 EXCLUSIVE 则跳过，否则覆盖任意标签为 EXCLUSIVE

### 与异彩的关系

标签和异彩是完全独立的两个维度：
- 异彩（Variant）：NORMAL/HOLO/MISPRINT/SHINY/LIMITED/DAMAGED，管结算行为修饰（概率双发/插队等）
- 标签（TagType）：NONE/EXCLUSIVE/TEMPORARY/CONSUME，管卡牌生命周期路由（去哪/怎么销毁/槽位数）
- 一张牌可以同时有异彩和标签，两者不互斥
