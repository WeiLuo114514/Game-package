# 记忆索引

> 本文件只做索引（每文件一行），具体内容按需在对应窗口「读取 XX 记忆」加载。
> 使用方式：Arithmetrick 项目 → 读 suanle-project + tech-patterns；求职 → 读 daily-life 月度 + interview-*；日常 → 读 daily-life 月度。

## Arithmetrick 项目（核心）

- [项目现状](suanle-project.md) — 四色运算DBG，36常规王牌+22关卡王牌全完成，Steam过审，腾讯比赛9/15参赛，当前王牌修复主线
- [效果/王牌技术规范](tech-patterns-effects-triggers.md) — 王牌修改规范(判定放_can_trigger/bonus_requeue/is_bonus一牌一次/ON_MATH_BEGIN/惰性初始化/清队列/pending跨reset)+EventEffect全表/标签/等级/飘字/批量串行/飘名抑制
- [旧案参考原型](suanleba-plus-reference.md) — 「算了吧？plus」参考原型

## Arithmetrick 设计文档

- [世界观设定](world-setting-20260815.md) — 数字空间推演历史，运算牌=历史节点/王牌=推演规则/四色=四幕剧
- [卡牌情感化系统](card-emotional-system-20260619.md) — ID系统/X品质/专属魔法/回忆录/收藏柜
- [卡牌养成情感价值](card-nurturing-emotional-value-20260705.md) — 永久牌库+印记DIY+UID=所有权依恋感，核心差异化
- [绿色套牌设计](green-deck-design-20260627.md) + [绿牌复盘](green-deck-retrospective-20260628.md)
- [蓝色套牌设计](blue-deck-design-20260617.md) — 15张÷弃置转化流
- [底牌系统](base-card-design-20260715.md) — 红色底牌×3，主动技能+冷却+替换继承
- [经济平衡](economic-balance-20260702.md) — 全魔法定价/异彩重做/盲盒/利息
- [王牌重设计](ace-redesign-20260622.md) — 27常规+22关卡王牌 9xxx 体系
- [设计决策 6/2](design-decisions-20260602.md) / [6/10](design-decisions-20260610.md) / [7/5](design-decisions-20260705.md)
- [设计哲学/面试叙事](design-philosophy-20260710.md) — 迭代驱动设计/风险筹码/30秒pitch
- [项目里程碑](milestones-20260606.md) — M3-M5 规划
- [发布前规划](release-plan-20260627.md) — 三色打磨发布/社区共创路线
- [商店颜色过滤](shop-color-filter-20260627.md) — 颜色按钮循环切换权重倾斜
- [结算序列器设计](settlement-sequencer-design.md) + [实施决策](settlement-sequencer-decisions-20260612.md)
- [名词释义表](terminology-glossary-20260620.md) — 运算卡/王牌/异彩/镀层/槽位/操作动词
- [教程系统](tutorial-system-20260715.md) — TutorialOverlay 步骤状态机+输入门控

## 参赛 / 备用项目

- [腾讯独立游戏比赛](tencent-indie-competition-20260915.md) — 9/15截止，用Arithmetrick参赛
- [元素反应复刻](elemental-reaction-project.md) + [反应规则](elemental-reaction-rules.md) — 腾讯开局大作业，Godot 4.6，16/16反应
- [米哈游3D解谜构想](mihoyo-3d-puzzle-20260710.md) — 冲刺+回溯双机制
- [二合爬塔RPG](merge2-tower-rpg-design.md) + [技术规范](merge2-tech-patterns-20260721.md) — 畅游笔试备用方案
- [Flambé体验笔记](flambe-notes-20260719.md) — 畅游笔试第三款游戏

## 求职

- [日常9月·offer决策](daily-life-2026-09.md) — 畅游offer到(15k/14-16薪/公积金12%)接受决策+腾讯比赛9/15倒计时
- [日常8月·求职主线](daily-life-2026-08.md) — 畅游四轮全过+测评通过→9/6已拿offer（含类别索引）
- [日常7月·求职全流程](daily-life-202607.md) — 投递→一面→笔试→二面→转北京（含类别索引）
- [面试复盘 7/24](interview-reflection-20260724.md) — 二面抽象→具体切换/示弱补救
- [面试复盘 8/13](interview-reflection-20260813.md) — 三面芦宁数值面/文档短板/估算锚定
- [面试复盘 8/18](interview-reflection-20260818.md) — 复试湛蕾业务深面/教学式反馈
- [复试前夜准备](interview-prep-20260818.md) — 三面复盘深化/战前清单

## 人际关系 / 自我认知

- [关系边界确立](relationship-boundary-20260720.md) — 单向兼容识别/三原则
- [亲密关系模式](intimacy-patterns-mate-selection-20260721.md) — 幻想驱动/择偶四维/价值等式
- [推荐决策模型修正](feedback-decision-model.md) — 社交/联机因素权重优先
- [深度对话 8/1](deep-conversation-20260801.md) — 博弈论/自由意志/ENFJ/多益复盘

## 用户画像 / 偏好

- [用户偏好](user-prefs.md) — 策划开发者/Godot 4.6/中文命名/成本策略
- [游戏口味](gaming-taste.md) — 五维认知框架/Portal封顶/The Witness退款

## 学习 / 论文

- [Unity学习计划](unity-learning.md) — 3D项目转Unity/团结引擎
- [AI/ML学习](ai-ml-learning.md) — LoRA微调NPC对话方向
- [毕业论文进度](thesis-progress-20260808.md) — LLM叙事评估+Prompt优化

## 其他

- [电脑性能](laptop-performance-2026.md) — GPU热降频待重涂硅脂/主力游戏转瓦洛兰特
- [成本策略](cost-strategy-20260815.md) — DeepSeek 8/17涨价应对

---
## 归档（已清理，备份在 memory-archive/）

- 开发日志 49 个（6/19-9/8，最新 dev-log-20260908）→ `memory-archive/开发日志/`
- 零散规范 9 个（tag-system/color-judgment/wild-trigger-audit 等）→ `memory-archive/技术规范-旧/`
- 逐日日常记录 10 个 → 已整合进 daily-life-2026-08.md
