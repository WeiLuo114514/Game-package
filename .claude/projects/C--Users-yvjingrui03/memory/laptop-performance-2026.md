---
name: laptop-performance-2026
description: 用户游戏本(HP OMEN i9-13900HX+RTX 4060)散热诊断——GPU 热降频待线下重涂硅脂；主力游戏已转为瓦洛兰特
metadata: 
  node_type: memory
  type: project
  originSessionId: 465fec14-b134-4771-8216-575efb299267
---

2026-08-29 用 nvidia-smi 遥测采样确诊：RTX 4060 Laptop GPU **持续热降频**（满载 89°C 撞墙，时钟从 2535MHz 掉到 ~1850MHz，功耗被压到 ~70W）。用户自换硅脂清灰，曾因 CPU 硅脂不足爆卡，故怀疑本次是 **GPU 核心硅脂没涂好**。用户已计划去线下店重涂硅脂+清灰。

同期现象：三角洲整局 GPU 99% 占用但帧率锁死 90-100；游戏内存 14.5-15/16GB 接近上限（用户暂不升级）。瓦洛兰特偶发抽风掉帧待修后观察。2026-08-29 已彻底清理抖音开机自启（计划任务卸载+2 服务禁用）。

**Why:** 用户是游戏策划/开发者，游戏性能问题可能反复出现，且主力游戏已从三角洲转为瓦洛兰特。
**How to apply:** 用户再提游戏掉帧/性能时，先调出此诊断；换硅脂后可复跑 `C:\Users\yvjingrui03\fps_diag.ps1`（保留中）对比 GPU 时钟是否回到 ~2500MHz。相关：[[gaming-taste]]
