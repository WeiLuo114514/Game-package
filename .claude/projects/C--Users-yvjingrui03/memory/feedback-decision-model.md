---
name: feedback-decision-model
description: 推荐决策模型修正：社交/联机因素权重应高于单机恢复逻辑
metadata: 
  node_type: memory
  type: feedback
  originSessionId: fe036b42-1c6a-437a-80d0-b5969a783f17
---

当推荐游戏或娱乐活动时，社交/联机因素的权重要高于纯单机精力管理分析。

**Why:** 2026-07-29，用户在多益 AI 面后问我该玩三角洲还是 STS2。我基于精力管理模型推荐三角洲（即时反馈回蓝），但用户最终和同学联机 STS2 打转牌华丽收场，社交乐趣完爆了单机恢复逻辑。

**How to apply:** 做推荐决策时，多问一句「有人一起吗」，联机/社交因素应在决策树上排在精力管理之前。三个支柱（FPS/策略/ACT）都有联机模式的可能，不要默认用户会 solo。
