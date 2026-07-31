---
name: color-judgment-refactor-20260626
description: 颜色判定集中化重构 — 从等值判定到 has_color() 统一入口，支持多色判定
metadata: 
  node_type: memory
  type: project
  originSessionId: d8aae4a9-63c9-4bdd-a124-f6f3e887cc43
---

2026-06-26，为支持颜色魔法（给卡牌附加额外颜色判定），将全项目 50+ 处散落的 `.color == "R"` 判定重构为统一入口。

**Why**：颜色魔法需要一张牌同时被红蓝双色识别。原来的 `card_data.color == "R"` 等值判定无法支持。如果每加一个新维度就要改 50 个文件，维护成本不可控。

**How to apply**：所有 gameplay 颜色判定必须走 `GameData.has_color(card_data, target)` 或 `card_data.has_color(target)`。视觉/粒子/排序仍用 `card_data.color` 主色。

## 技术决策

- **保留 color 为主色，新增 extra_colors 数组**：如果直接改 color 为 Array，CardUI/手牌排序/粒子纹理/颜色追踪等 30+ 处非 gameply 引用全部要重构。渐进式路线——新增不影响旧逻辑。
- **has_color 放 GameData 而非 CardData**：50+ 处调用方不用关心判定实现细节。以后加「诅咒颜色」「临时颜色」等新维度，只改 has_color 内部。
- **与结算序列器同模式**：单一真相源，散落的 if-else 变成统一入口。

## extra_colors 赋值路径

- 颜色魔法（60001-60004）：`target.extra_colors.append(color)`
- 复制魔法（70001）：副本保留 extra_colors（duplicate 深拷贝）
- 传火（10011）：改主色 `c.card_data.color = "R"`，不影响 extra_colors
- 预览：select_card 里 deep copy extra_colors

## 互动关系

- 颜色魔法 + 试炼 Debuff：多色牌同时受多个颜色 Debuff 影响（收益/风险同步缩放）
- 颜色魔法 + 王牌：多色牌可触发多个颜色王牌的 _can_trigger
- 商店颜色倾向按钮：影响卡包类型权重，与颜色系统独立
