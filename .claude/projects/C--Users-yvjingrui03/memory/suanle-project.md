---
name: suanle-project
description: Arithmetrick（引擎内名「简单算术魔法」）项目现状 — 四色运算DBG Roguelite，位于 E:/简单的算术魔法，常规王牌36张+关卡王牌22张全完成，Steam审核通过，腾讯比赛9/15参赛
metadata: 
  node_type: memory
  type: project
  originSessionId: 3e6dbb5a-e1a4-4d08-a426-f215000e1f2d
---

# Arithmetrick（引擎内名「简单算术魔法」）— 项目现状

数学运算核心的 DBG Roguelite 策略卡牌游戏，灵感来自 Balatro。

**项目路径**：`E:\简单的算术魔法`（2026-06-06 迁移到 E 盘）

**核心玩法**：运算牌拖放到槽位，点击结算后从左到右依次计算（+/-/×/÷）。3 盲注=1 单元，共 18 关，之后无尽模式。

**Why**：从旧版「算了吧？plus」重构而来，旧版为参考原型（[[suanleba-plus-reference]]）。

**How to apply**：处理此项目时，优先看 `E:\简单的算术魔法\CLAUDE.md`（项目级规范）和 [[tech-patterns-effects-triggers]]（效果/王牌修改规范）。卡牌数据以 FileManager.gd 为准（已与 xlsx 同步）。

## 卡牌状态（全完成）

- **红色** 17 张全完成（10001-10017，效果全实现）
- **黄色** 17 张全完成（概率流派重做完毕）
- **蓝色** 17 张全完成（含 30017 还魂）
- **绿色** 15 张全完成
- **白色** 1 张：模仿者（无色空壳）
- **常规王牌 36 张全完成**（红黄蓝绿各 9 张，9xxx 体系）
- **关卡王牌（试炼）22 张全完成**（L9001-L9022）
- **魔法全完成**

## 权威数据源

- **FileManager.gd** — 卡牌/王牌/魔法配置唯一代码权威源（WildConfig/LevelWildConfig/CardConfig）
- 桌面 `卡牌.xlsx` / `王牌.xlsx` — 设计参考表
- `游戏设计文档.md` / `经济平衡.md` — 设计说明

## 核心架构

- `GameData.gd` — Autoload 单例，全局状态+信号+王牌/魔法管理+4条独立RNG流
- `PlayField.gd` — 游戏主界面，结算编排（Phase 驱动），代码量最大
- `FileManager.gd` — 配置查询入口；`CardUI.gd`/`op_card.gd` — 卡牌基类/运算牌
- `WildCardUI.gd`/`WildSlot.gd` — 王牌 UI/槽位；`BaseEffect.gd`/`BaseWildEffect.gd` — 效果基类
- `SettlementSequencer.gd` — 结算动画序列器（AnimationRequest + Phase 阶段驱动）
- `TriggerDefines.gd` — 触发器常量（含 ON_MATH_BEGIN 等）
- **效果/王牌修改规范 → [[tech-patterns-effects-triggers]]**

## 关键系统（2026-09 现状）

- **结算流程**：CARD_EFFECT→SETTLE_BEGIN→WILD_CARD→MATH_BEGIN→MATH_REVEAL→SETTLE_END（详见 tech-patterns）
- **RNG 四流**：局内/事件/商店/表现独立种子流（8/31 拆分）
- **随机流归属**：局外内容(事件/商店/卡包)走独立流，表现层走 cosmetic_*（8/31）
- **标签系统**：TagSystem Autoload 集中注册，has_color/get_effective_level 统一入口
- **王牌批量串行**：apply_card_effects_serial（飘字+动画同步，8/28-9/1）
- **王牌重触发**：bonus_requeue 插队 + ON_CARD_RESOLVED is_bonus 一牌一次判定
- **图鉴页 + 历史记录系统**（JSON 持久化 50 局，8/28）
- **收藏系统**：collected_deck 持久化 + 专属卡 uid 兜底（8/30）
- **系统参数**：settle_tempo（节奏倍率）/ double_click_window（双击窗口）可调
- **Steam 商店页审核通过**（7/24 税务审核、8/9 等待→已通过）

## 近期工作（9/1 → 9/8）

- **王牌 bug 修复 + 效果优化**：
  - 9/08（多会话主线收尾）：随波逐流自增益计数；折損重做「免费弃置」+弃牌费用阶梯解耦(fold_fee_discard_count)；潮涌等对局内销毁王牌补刷新(remove_wild_card→player_stats_changed)；SelectCard 事件选牌确认+预览修复；新局容量复位(生效槽/王牌槽/魔法槽→init_game_from_deck)；商店颜色偏好落盘 settings.cfg；满级卡预览按钮隐藏；EndingPage 收藏面板残留修复；战场魔法僵尸回调修复；L9013 资产税 0.5%/上限20%；**存档 slot 目录缺失修复**（详见 [[tech-patterns-effects-triggers]] 系统坑位段）
  - 9/01：子母弹重触发队列化、9002今天不行重写、9003反方向的钟迁到ON_MATH_BEGIN、9004东山再起双触发器、盲盒卡包跨reset存活
  - 9/02：L9004 目标分+15% 惩罚重做；三色试炼 L9005/9006/9007 底数增益迁 ON_SETTLE_STEP + 同色牌结算抬目标分（飘字+刷面板）；L9008 回合-1+费用换钱；L9009 改色飘字；L9011 费用字体蓝/红着色（cost_shift）；L9013 触发改王牌结算；L9014 定稿（开局回合+1 + 每王牌目标分+25%）
  - **系统级：`ON_GAME_START` 升格为通用开局触发器**——`GameData.trigger_wild_game_start()` 每关战斗开始（PlayField 进入）对 owned 王牌（普通+试炼统一）派发，不再依赖特殊入口；`givewild` 9xxx 路由到 force_boss_trial。详见 [[tech-patterns-effects-triggers]] 规范5
- **腾讯独立游戏比赛 9/15 截止**，用 Arithmetrick 参赛 [[tencent-indie-competition-20260915]]

## 经济/关卡参考

- 经济平衡：`经济平衡.md`；18关=6单元×3盲注，Boss 失败游戏结束
- 系统参数：生效槽位4（可扩展）/手牌6/费用8/回合3~4/王牌5/魔法3（详见 tech-patterns）

## 相关记忆

- [[tech-patterns-effects-triggers]] — 效果/王牌修改规范（权威）
- [[suanleba-plus-reference]] — 旧案参考原型
- [[tencent-indie-competition-20260915]] — 腾讯比赛
- 设计文档（世界观/情感化/绿蓝牌/底牌/经济/结算序列器/术语表）见 MEMORY.md 索引
