---
name: thesis-progress-20260808
description: 毕业论文执行进度 — LLM叙事评估实验数据收集与指标计算完成
metadata: 
  node_type: memory
  type: project
  originSessionId: 6719e2b4-6e91-4ffa-8d93-186076cfcf61
---

# 毕业论文执行进度 2026-08-08

## 代码产出（均在 `C:\pythonProject5\毕业论文\`）

| 文件 | 功能 |
|------|------|
| `experiment_materials.py` | 2×2×2 因子实验材料：A系统层/A1详细、6 NPC角色卡(B0/B1)、情境卡(C1) |
| `run_experiment.py` | DeepSeek API 批量调用，240 条全量跑完，138,245 tokens |
| `compute_metrics.py` | Distinct-2 + Self-BLEU 自动指标计算（后续加 Perplexity/BERTScore/Entity Grid）|
| `view_results.py` | 数据库快速查看（分组均值/因子分解/NPC对比）|
| `migrate_to_mysql.py` | SQLite→MySQL 迁移（密码问题失败，改用 CSV/Excel）|
| `generate_rating_sheets.py` | 人工评分表生成：48 条分层抽样，8 人每人 16-19 条 |
| `experiment_results.db` | SQLite 数据库，含全部 240 条生成+指标 |

## 初步发现

- **Distinct-2**：A 因子几乎无差异，B1（详细角色卡）略降多样性
- **Self-BLEU**：B 和 C 加细节后 5 次重复生成之间相似度明显升高（B0→B1: 0.177→0.2095; C0→C1: 0.175→0.2115）
- 即：prompt 约束越多，多次生成之间的多样性越低（约束-多样性权衡）
- 系统层（A）对多样性影响几乎为零，起作用的是角色和情境的细节量

## 下一步

1. 等待人工评分 Excel 收回 → 合并
2. 计算 Perplexity + BERTScore + Entity Grid 指标（需下载模型）
3. ANOVA + 交互效应分析 + 校标验证
4. 论文正文补数据章节

## API 成本备忘（2026-08-13）

- DeepSeek 调价 **8/17 生效**，改为峰谷定价：高峰（9-12 点、14-18 点）为空闲时段 2 倍
- V4-Pro 高峰输出 27 元/百万 tokens（旧 6 元，+350%）；空闲 13.5 元（+125%）
- 输出价涨幅最大 → 生成类批量任务冲击最大
- 后续如有补充批量调用，**全部挪到空闲时段跑**（晚间/清晨），直接省一半
