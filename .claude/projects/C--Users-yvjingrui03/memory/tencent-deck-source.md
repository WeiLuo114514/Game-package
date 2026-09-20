---
name: tencent-deck-source
description: 腾讯比赛《作品介绍 Deck》的源文件位置与重新生成流程（Playwright + 本地 http.server）
metadata: 
  node_type: memory
  type: reference
  originSessionId: 7a6ceb63-2615-4574-9e58-529f96e84ee1
---

《算术魔法》腾讯比赛「作品介绍 Deck」的可编辑源文件位于
`C:\Users\yvjingrui03\Desktop\算数魔法\腾讯游戏创作大赛\源文件\`（`index.html` + `img/`），
产出的 PDF 在其上一级目录的 `算术魔法-游戏介绍.pdf`。

**Why:** 需要重新生成时，直接改 `index.html` 再渲染即可；`img/` 里是已降采样到 1920×1080 的截图副本，
改版不受原始素材变动影响。

**How to apply:**
- 渲染方式：在 `源文件/` 起一个本地服务（`python -m http.server 8766 --bind 127.0.0.1`），
  再用 Playwright 的 `page.pdf({format:'A4', printBackground:true, displayHeaderFooter:true,
  margin:{top:'13mm',bottom:'15mm',left:'14mm',right:'14mm'}})` 输出。
  **不能用 `file://`**，Playwright 会拦截。
- 排版约定：每个 `.page` 是固定高 262mm 的 flex 容器 + `page-break-after`，一节一页；
  加内容时要确认页数不变且没有内容被 `overflow:hidden` 截掉。
- 校验方式：PyMuPDF (`fitz`) 逐页 `get_text()`，可检查空白页与目录页码是否与实际章节页一致。
- 素材坑：宣传素材根目录 `E:\卡牌素材\销售` 已改名为 `E:\卡牌素材\宣发`，
  旧 HTML 里的相对路径全部失效；新截图在 `宣发\新截图\`（2560×1440）。

相关：[[tencent-indie-competition-20260915]]
