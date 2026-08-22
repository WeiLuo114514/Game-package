---
name: cost-strategy-20260815
description: DeepSeek 8-17 涨价应对决策：继续用 V4 Pro 忍价，不折腾 harness，等实习降强度+华子卡带来降价后再上强度
metadata: 
  node_type: memory
  type: project
  originSessionId: 3ebc2c00-b91e-448f-ae7e-8a76ceaf44c0
---

# 模型成本策略（2026-08-15 决策，同日更新）

DeepSeek API 2026-08-17 起涨价（V4 Pro 缓存命中 0.025→0.15/0.30 元每百万，输出 6→13.5/27），用户按当前填充内容强度测算月账单 130→约 308 元。

**用户决策：默认切到 flash，Pro 按需切换（8-15 实操）。**
- 已通过 `/model Haiku` 切到 deepseek-v4-flash 并保存为新会话默认；`/model Opus` 可一键切回 pro
- `/model` 快速参数不认简写（"flash" 报 400），必须传档位名或完整模型名
- settings.json 第 12 行 env.CLAUDE_CODE_EFFORT_LEVEL=max 会压过 `/effort` 会话内选择，想降推理强度需删该 env
- DeepSeek harness（宣称缓存命中 99-100%）等待成熟后再试——现命中率已 96.9%，理论最多省 ~100 元/月，不划算

**Why:** 用户定位：DS 仍是最便宜一档；自己指令详细、返工少；架构与技术选型靠自己做，模型智商不是瓶颈，预算才是。若找到实习开发强度会降，账单自然缩水；预计三四个月后华子（华为）显卡供应到位带来降价，届时再搞新作或上强度。
**How to apply:** 后续涉及模型成本/换工具建议时，默认「flash 主力 + Pro 按需」立场；开发强度变化或调价新闻出现时提醒用户重估此决策。
