---
name: suanle-project
description: Godot 4.6 卡牌游戏《算了》— 主项目，位于 E:/简单的算术魔法
metadata: 
  node_type: memory
  type: project
  originSessionId: 0654740a-3317-451c-b1da-b62233160d2a
---

《算了》（引擎内名称「简单算术魔法」）是一款数学运算核心的 DBG Roguelite 策略卡牌游戏，灵感来自 Balatro。

**项目路径**：`E:\简单的算术魔法`（2026-06-06 从 Documents 迁移到 E 盘）

**核心玩法**：玩家将运算牌拖放到 5 个槽位，点击结算后从左到右依次计算（+/-/×/÷）。3 个盲注为一个单元，共 18 关，之后开启无尽模式。

**Why**：用户正在从旧版本（算了吧？plus）重构/迁移到新版本（算了），旧版本是参考原型。

**How to apply**：处理此项目时，优先参考旧案 [[suanleba-plus-reference]] 的实现思路，但遵循新项目的代码风格。卡牌数据以 FileManager.gd 为准（已与 xlsx 同步）。项目位于 E 盘而非 Documents。设计决策参考 [[design-decisions-20260602]] 和 [[design-decisions-20260610]]。

### 权威数据源

- **FileManager.gd** 是卡牌配置的唯一代码权威源
- 桌面 `卡牌.xlsx` 是设计参考表
- `游戏设计文档.md` 是设计说明文档

### 设计方向

- 四色流派：红(×乘法爆发)、黄(+永久累积)、蓝(÷符号变换，暂缓)、绿(- DOT，暂缓) + 白(通用润滑)
- 颜色不是职业——可自由混搭，通过效果联动（颜色条件）做软引导
- 超模连锁是刻意设计的爽感来源
- 卡包在商店购买、买后当场三选一
- 王牌仅在商店产出，随机展示供选购
- **RoundMenu 保留**作为回合间隙场景主控
- 盲注分级：中小盲注失败不结束，Boss 关失败游戏结束
- 费用系统：8→12 封顶
- **已设计但未实现**：运算牌异彩（炫彩/错版/闪光/限定/折损，已实现）、王牌镀层（待实现）、魔法牌系统、卡牌升级系统 — 详见 [[design-decisions-20260610]]

### 卡牌配置现状（2026-06-10）

- 红色卡牌：17 张（10001-10017），**17 个效果全部实现**
- 黄色卡牌：15 张（20001-20015），**仅淬炼实现**，其余 14 个空壳
- 白色卡牌：4 张（50001-50004），**仅模仿者实现**，催化剂/置换标注待重新设计
- 蓝色卡牌：15 张（30001-30015），设计完成待实现（2026-06-17），弃置触发 + 运算符转化流派，详见 [[blue-deck-design-20260617]]
- 绿色卡牌：0 张
- 王牌：12 张（0001-2006），**12 个效果全部实现**
- 卡包类型：RED_BASE / RED_ADVANCED / YELLOW_BASE / YELLOW_ADVANCED
- 新手套牌：仅 red_starter（10 张红）

### 效果系统

- **BaseEffect**：`apply(slot_cards, index)` → 修改卡牌属性→返回 true/false→`on_effect_triggered(ctx)` 副作用
- **描述格式**：`{key}` 占位符，`GameData.format_named_params()` 按 level 替换，level>1 金色高亮
- **多语言**：FileManager 有 Lang 枚举和 card_desc_i18n 字典接口

### 王牌效果系统

- `WildData` (Resource)，`GameData.owned_wild_cards` 为唯一数据源
- `trigger_wild_effects(event_type, context)` 遍历分发
- `_wild_effect_cache` 按 card_id 缓存效果 Node
- `WildSlot` 管理王牌 UI，PlayField 和 RoundMenu 各有实例
- 12 张王牌效果全部独立脚本实现，含状态管理 (`_state` Dictionary)

### 牌库永久化（2026-06-10 实现）

- `draw_pile` 是永久牌库，`_reset_state()` 不再清空
- `consolidate_permanent_deck()` 游戏结束时收集所有非临时卡牌→draw_pile
- `reset_for_new_game()` 从永久牌库加载，首次为空时从 initial_decks 创建
- `_is_temporary()` 接口预留
- `card_container.gd` 读取 `GameData.draw_pile` 展示牌库

### 三选一抽卡动画（2026-06-10 重构）

- 纯 Tween 驱动：背面降下（0.1s 间隔）→ draw 掀开（0.1s 间隔）→ tips/skip 渐显
- 选中卡牌飞向 Desk 按钮，其余 draw_back 翻面→飞出
- Desk 按钮开包期间禁用+移出屏幕，SELECTING 阶段回归
- DrawCardPage 有 `enter_page()`/`exit_page()`（into/out），选卡完成播 `draw_finish`
- 选中卡→draw_pile，未选中/跳过→丢弃

### ShowCard 信号（2026-06-10 实现）

- 新增 `signal card_clicked(card: ShowCard)`，左键点击发射
- 三选一界面：`card_clicked.bind(i)` 处理选择
- 牌库界面：`card_clicked` → 显示详情页
- 影子：视口中心偏移（与 OPCard 一致）

### RoundMenu 导航（2026-06-10 实现）

- `last_screen` 追踪上一页面
- `_switch_to` 防重复（`target == now_screen: return`）
- Desk 按钮：仓库已打开→返回 last_screen，防 warehouse→warehouse 死循环
- 页面切换：exit_page() → await animation_finished → enter_page()

### 数据流

```
FileManager（静态配置）
    ↓
GameData（运行时状态：draw_pile永久牌库/hand/discard_pile/card_slots,
         owned_wild_cards + _wild_effect_cache）
    ↓
PlayField（UI交互层：Card节点拖拽/槽位/结算动画 + 王牌钩子点）
    ↓
RoundMenu（回合间隙：InfoPage/ShopPage/DrawCardPage/CardWarehouse/WildSlot）
```

### 初始牌组

- `FileManager.initial_decks`，当前仅 `deck_id = "red_starter"`
- 10 张：收尾专家×1 + 星火×3 + 头号种子×2 + 引燃×3 + 二次利用×1

### 关卡配置

- 18关 = 6单元 × 3盲注（Small/Medium/Boss）
- 难度：入门500→进阶2500→考验10K→深化40K→巅峰150K→终局600K→最终Boss 150万
- 全关卡 5 回合，Boss debuff 预留字段待设计
