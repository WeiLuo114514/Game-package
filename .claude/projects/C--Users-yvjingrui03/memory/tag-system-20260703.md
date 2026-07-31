---
name: tag-system-20260703
description: 标签系统：互斥→位掩码并存，TagType 枚举改为 1/2/4/8 位掩码值
metadata: 
  node_type: memory
  type: project
  originSessionId: 7d05796f-d264-43de-902b-66b79c55bc49
---

## 2026-07-03 重构

### 枚举变更
```gdscript
enum TagType { NONE = 0, EXCLUSIVE = 1, TEMPORARY = 2, CONSUME = 4, WEALTH = 8 }
```
优先级：专属 > 财富 > 临时 > 消耗

### 工具方法
- `CardData.has_tag(t: int) -> bool` — `return (tag & t) != 0`
- `CardData.add_tag(t: int) -> void` — `tag |= t`

### 写入点（~13 处全改）
- 所有 `target.tag = TagType.X` → `target.add_tag(TagType.X)`
- `copy.tag = NONE` (清空) 保持不变

### 读取点（~16 处全改）
- 所有 `data.tag == TagType.X` → `data.has_tag(TagType.X)`
- 魔法互斥检查改为 `not target.has_tag(EXCLUSIVE)` 防止覆盖

### 生命周期路由
- `on_card_to_discard`: TEMPORARY 直接销毁，CONSUME 进消耗堆，可并存
- `_is_temporary()`: 桩代码修复为 `return _card.has_tag(TEMPORARY)`
- PlayField 手牌超限: `has_tag` 替代 `==`

### CardUI 显示
- 描述：遍历 TAG_PRIORITY_ORDER，有则追加彩色标签文本
- 颜色：专属=orange, 财富=gold, 临时=gray, 消耗=tomato
- 专属标签贴图保留，其余纯文字
- 专属标题橙色保留
