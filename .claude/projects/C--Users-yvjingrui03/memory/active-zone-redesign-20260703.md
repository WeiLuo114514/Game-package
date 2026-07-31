---
name: active-zone-redesign-20260703
description: 生效区重构：5固定槽位→动态有序列表+可变容量 → 已实现
metadata: 
  node_type: memory
  type: project
  originSessionId: 7d05796f-d264-43de-902b-66b79c55bc49
---

## 实施结果（2026-07-03）

**全部完成 ✅**

### 架构
- `GameData.card_slots[5]` → `active_zone: Array[CardData]` 动态数组
- `active_zone_capacity` 变量（默认5，可被升级/王牌/魔法扩展）
- `set_active_zone_capacity()` 函数 + `active_zone_capacity_changed` 信号
- Slot1-5 Panel 节点全部删除，ActiveZone 改为整块 TextureRect 判定区

### CardContainer（新增）
- 文件：`代码/系统/ActiveCardContainer.gd`（~600行）
- 所有卡牌（手牌+生效区）统一放在 CardContainer 下，零 reparent
- 卡牌位置通过 `get_zone_card_position(index)` 计算（槽位式固定间距）
- 生效区插入判定：`get_slot_index_at_position(global_pos)` — 基于容量均分
- VSeparator 分隔线 → 升级为 SlotGuide.tscn 预制件（Label 数字 + TextureRect 高光）
- 手牌区域结算动画通过 `hand_y_offset` 偏移实现

### PlayField 精简
- 从 ~1700 行缩减到 ~970 行
- 卡牌交互/拖拽/焦点/排序/弃牌逻辑全部移至 ActiveCardContainer
- PlayField 保留：结算管线、状态机、游戏流程、王牌/魔法系统
- 通过 property 委托访问 card_container 的数组

### 关键修复
- op_card.gd: `table` → `card_container` + `play_field` 双引用（避免循环类型依赖）
- 循环引用修复：类型注解改用 `Node`/`Control` 基类
- `_replace_card_in_slot` 补 `release_card_focus()` 防卡死
- 硬编码 5/4 → `active_zone_capacity` / `_slot_cards.size()-1` 动态末位

### 技术规范
- **永远不要在同一会话中同时用 sed/Python 文本编辑 .tscn 文件** — 场景必须手动在 Godot 编辑器中修改
- **GDScript const 数组不能用枚举简写**，必须用全限定名或裸数值
- **`class_name` 循环引用**会导致解析死锁，用 `Node`/`Control` 基类类型注解代替
- **`_ready()` 不在场景树外的节点上调用** — 用 `_init()` 或首次 `_execute`/`_can_trigger` 中连接信号
