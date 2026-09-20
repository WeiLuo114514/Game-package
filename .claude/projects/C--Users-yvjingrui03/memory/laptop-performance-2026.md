---
name: laptop-performance-2026
description: 用户游戏本(HP OMEN i9-13900HX+RTX 4060)散热诊断——GPU 热降频；2026-09-20 清灰换硅脂+相变片；双物理盘布局与 git 备份拓扑
metadata: 
  node_type: memory
  type: project
  originSessionId: 465fec14-b134-4771-8216-575efb299267
---

2026-08-29 用 nvidia-smi 遥测采样确诊：RTX 4060 Laptop GPU **持续热降频**（满载 89°C 撞墙，时钟从 2535MHz 掉到 ~1850MHz，功耗被压到 ~70W）。用户自换硅脂清灰，曾因 CPU 硅脂不足爆卡，故怀疑本次是 **GPU 核心硅脂没涂好**。用户已计划去线下店重涂硅脂+清灰。

同期现象：三角洲整局 GPU 99% 占用但帧率锁死 90-100；游戏内存 14.5-15/16GB 接近上限（用户暂不升级）。瓦洛兰特偶发抽风掉帧待修后观察。2026-08-29 已彻底清理抖音开机自启（计划任务卸载+2 服务禁用）。

**Why:** 用户是游戏策划/开发者，游戏性能问题可能反复出现，且主力游戏已从三角洲转为瓦洛兰特。

## 磁盘布局与 git 备份拓扑（2026-09-20 实测）

- **Disk 0** = 镁光 MTFDKBA1T0TFH（954G）：`C:`（516G，系统+记忆库）+ `D:`（436G）
- **Disk 1** = 梵想 S790 1TB（932G）：只有 `E:` —— **Arithmetrick 项目在这里**
- 两块**独立物理盘** ⇒ 把项目副本放 C:、记忆库放 E: 才构成有效异盘备份；单盘内的 `git commit` 不算备份（.git 与数据同盘同命）

**远程拓扑（两仓库历史无共同祖先，分支名叫法不一致，别推错）：**

| 本地 | 远程 | 分支 |
|---|---|---|
| `E:\简单的算术魔法` | `WeiLuo114514/Arithmetrick` | `master` |
| `C:\Users\yvjingrui03` | `WeiLuo114514/Game-package` | 本地 `master` → 远程 **`memory`**（upstream 已配 `origin/memory`） |

- Game-package 的远程 `master`/`main` 是**空树**（原 LFS 指针文件已被最后提交删光），与记忆库历史无关，**推记忆库绝不能 force 到 master**
- **坑**：本地 `master` ↔ 远程 `memory` 不同名，而 `push.default=simple` 在名字不匹配时会**静默拒绝**裸 `git push`。记忆库必须显式写 `git push origin master:memory`
- 国内直连 github.com:443 不通，push 前需先开代理（2026-09-20 已验证：开代理后正常）

**How to apply:** 用户再提游戏掉帧/性能时，先调出此诊断；换硅脂后可复跑 `C:\Users\yvjingrui03\fps_diag.ps1`（保留中）对比 GPU 时钟是否回到 ~2500MHz。用户提「备份」时，先确认目标是异地（push 到远程）还是异盘（拷到另一块盘）——两者防的是不同的故障。相关：[[gaming-taste]]、[[suanle-project]]
