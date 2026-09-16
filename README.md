# spec-check：讓獨立 AI agent 在 spec 定稿前先找碴的 Claude Code skill

[English](README.en.md) · 繁體中文

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg) ![Claude Code skill](https://img.shields.io/badge/Claude_Code-skill-blue)

首次上傳 2026-09-15 · 最後同步 2026-09-15

spec-check 是一個 Claude Code skill，也能裝在 Codex CLI。你把一份 spec、設計文件或需求文件交給它，它派 3 到 5 個沒看過你對話的 AI agent 並行審查，最後交還一份修改計畫。spec 本身一個字都不會被動，要等你核准才逐項改。

## 三十秒版

- 3 到 5 個獨立 agent 並行審，每個最多 5 條發現，每條都要附章節行號，也要回答「不修會發生什麼具體失敗」。
- 其中一個 agent 固定負責砍項，專找第一版不需要的東西。
- 你只出現兩次：開頭確認素材，結尾裁決。
- 所有 agent 共用一份 context-pack（步驟 1 產的素材摘要），不各自重讀。作者實測過，單一 agent 自己重讀一輪約 10 萬 token。
- 產出是修改計畫，附 before 與 after 對照，不是改好的 spec。

## 什麼是 Claude Code skill？

skill 是一個資料夾，裡面有一份 `SKILL.md`，告訴 Claude 什麼情境該用它、用的時候照什麼步驟。放進 `~/.claude/skills/` 之後，Claude 會在符合描述的情境自動載入；你也可以打 `/spec-check` 手動叫它。這個 repo 就是那個資料夾，clone 下來就能用。Anthropic 的官方說明在 [Agent Skills 文件](https://docs.claude.com/en/docs/claude-code/skills)。

## spec-check 做什麼？一張圖

![spec-check 概念圖：左欄輸入 spec 與已拍板決策，中欄四步處理（盤點寫 context-pack、給你確認一次、3 到 5 個獨立 agent 並行審、彙整查證），右欄輸出修改計畫與稽核上下文，spec 本身不動](docs/spec-check-overview.svg)

左邊是你給它的東西，中間是它做的四步，右邊是你拿到的東西。琥珀色的方塊是你會出現的地方。

## 它解決什麼問題？

你寫好一份 spec，請 AI 幫忙看看。它回你 11 條建議，每一條都是「建議補上 X」，沒有一條說「這個可以刪」。你照單全收，spec 從 200 行變成 350 行，做到一半才發現有一半是過度設計。這是 spec-check 寫出來之前，作者在 2026 年 8 月實測的基準：11 條建議，零砍項。

另一種情況：你和 AI 討論了一小時，拍板了三個決策，然後請它審 spec。第一條發現是「建議重新考慮決策 A」，你十分鐘前才決定的。更麻煩的是 spec 裡有一段是 AI 自己加的，你還沒點頭，審查時它被當成既定事實跳過了。

spec-check 對這兩種失敗各放了一道硬性約束：必選一個砍項 agent；每條建議都要答「不修會怎樣」；已拍板的決策不准重審；你還沒核准的段落要單獨列出來。

## 跟直接請 Claude 看 spec 有什麼不同？

| 替代做法 | 它做的事 | spec-check 多做的事 | 少了會怎樣 |
|---|---|---|---|
| 直接問 AI「幫我看這份 spec」 | 回一串建議 | 必選砍項 agent；每條答「不修會怎樣」；已定決策不重審；產修改計畫、不改 spec | 建議全是加項；剛拍板的決策被重啟；spec 被直接改動、沒有裁決軌跡 |
| superpowers 的 brainstorming 自審（[obra/superpowers](https://github.com/obra/superpowers)） | 作者自己檢查 4 項、當場修 | 派沒看過對話的 agent 審；多面向 premortem（假設做完後專案失敗了，為什麼）；Claude 抽驗證據；裁決留給人 | 自己寫的 spec 看不到自己的盲點；沒有「做完後為何失敗」的推演 |
| superpowers 的 writing-plans 自審、requesting-code-review | 驗 plan 有沒有覆蓋 spec；審程式碼 diff | 質疑 spec 本身對不對 | 錯的 spec 被忠實地做出來 |

其他工具審的是「有沒有照 spec 做」。spec-check 審的是「spec 該不該這樣寫」。

## 怎麼安裝？

目錄名必須是 `spec-check`，和 `SKILL.md` 裡的 `name` 一致，否則 Claude 找不到。

Claude Code，裝在全域（所有專案都能用）：

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.claude/skills/spec-check
```

Claude Code，只裝在某個專案：

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git .claude/skills/spec-check
```

Codex CLI：

```bash
git clone https://github.com/wuchengwei1996/claude-skill-spec-check.git ~/.codex/skills/spec-check
```

裝好了沒？在 Claude Code 裡打 `/`，選單出現 `spec-check` 就成功；或直接說「檢查這份 spec」看它有沒有接手。更新用 `cd ~/.claude/skills/spec-check && git pull`。

## 怎麼觸發？

說「檢查這份 spec」「spec 體檢」「驗一下 spec」「對 spec 做 premortem」都會觸發，或直接打 `/spec-check`。一段典型的對話：

> **你**：/spec-check docs/specs/退款流程-design.md
>
> **Claude**：讀完了。素材我盤點到這些（列出清單）。現況我畫成這張 ASCII 流程圖，有讀錯嗎？這份 spec 偏流程類，建議勾「使用情境」和「資料與整合」兩個面向，外加必選的砍項。有沒有我漏掉的資料？
>
> **你**：沒漏，直接跑。
>
> **Claude**：（派出 3 個 agent，各自讀 context-pack 與 spec，幾分鐘後回報）
> 彙整完了，修改計畫落在 `docs/specs/2026-09-15-退款流程-design-修改計畫.md`。需要你裁決的有 2 項，排最前：
> 🔴 §3.2「退款逾七天自動核准」與 §5 驗收條件矛盾。不修的話，實作者會二選一，測試永遠有一邊紅。before 與 after 對照在計畫裡。
> 🔴 砍項：§6 的「多幣別退款」第一版沒有任何使用情境用到。不砍的話多 3 張表、2 條 API，全部沒人叫。
> 建議修的 🟡 有 3 項，備註 🟢 有 1 項。核准後我再逐項改 spec。

最後一句是重點：它不會自己動手改 spec，要等你點頭。

## 它內部怎麼運作？

對應上面那張圖的四步。

1. 盤點素材，寫成 context-pack。讀 spec，列出已定決策、spec 有寫但你還沒核准的段落、待定項、素材路徑，落成一份 `稽核上下文.md`。之後每個 agent 都讀這一份，不各自重讀所有素材。
2. 給你看一次，只等你一次。一則訊息給齊三件事：素材清單、一張 ASCII 現況圖、面向選單。現況圖的目的是讓你十秒內看出 Claude 有沒有讀錯 spec，讀錯就會派錯工。你可以說「直接跑」跳過。
3. 派 3 到 5 個獨立 agent 並行審。完整性審查一個（矛盾、模糊句、驗收寫不清），premortem 一到三個（假設照這份 spec 做完後專案失敗了，從你的面向解釋為什麼），砍項一個（第一版不需要什麼？刪掉會怎樣？）。每個 agent 只讀不寫，每條發現要引章節行號，最多 5 條。交辦範本在 `SKILL.md` 附錄 A。
4. 彙整與查證，Claude 親做。去重，需要你裁決的項目全數查證據，其餘至少抽一條，每條「建議新增」反問「第一版不加會怎樣」。最後落成修改計畫，需裁決的排最前，每項附 before 與 after 對照。格式在 `SKILL.md` 附錄 B。

## 什麼時候不該用它？

- 要審的是程式碼，不是 spec：用 code review 工具。
- 要決定專案該不該做：spec-check 假設你已經決定要做。
- 要產實作計畫：那是 plan 工具的事，spec-check 只審 spec 本身。
- spec 只有幾十行：直接請 Claude 看可能更快。這個 skill 的價值在 spec 大到一個人讀不完整的時候。

## 可以自訂什麼？

面向選單（`SKILL.md` 步驟 2 的技術可行性、使用情境、前端、後端、資料與整合、維運、成本時程）可依你的領域增刪。agent 數量的規模判斷（spec 少於 200 行用 3 個）可調。附錄 B 的修改計畫格式是作者的文件慣例，換成你的也行。模型路由（完整性審查用 opus、其餘用 sonnet）只是建議，改成你環境有的模型即可。

## 常見問題

它會改我的 spec 嗎？不會。產出是修改計畫，你核准後才逐項改，改完還會回讀驗證。

沒裝 superpowers 也能用嗎？能。description 提到 superpowers 只是為了說明「產 plan 不是它的事」。

為什麼要先寫 context-pack，直接派 agent 不行嗎？兩個原因：省 token（agent 共用一份摘要，不各自重讀），以及防止重審已定決策（context-pack 明列「已定」清單，agent 不得推翻）。

agent 用什麼模型？一定要 opus 嗎？不一定。附錄 A 的範本不綁模型。

跑一次要多久？取決於 spec 長度和 agent 數量。作者測一份 30 行的假 spec、3 個 agent，約 5 分鐘。

## 同作者的其他 skill

- [claude-skill-explain](https://github.com/wuchengwei1996/claude-skill-explain)：先找出你卡在哪一種理解缺口，再選說明方法；你說「還是不懂」時它會換方法，不會把同一段講得更長。
- [claude-skill-show](https://github.com/wuchengwei1996/claude-skill-show)：先讀懂資料，再決定用圖表、圖解還是表格，做出來之後自己回讀驗收。

## 授權與來源

MIT。skill 格式依 Anthropic 的 [Agent Skills 文件](https://docs.claude.com/en/docs/claude-code/skills)；比較表裡提到的 brainstorming 與 writing-plans 來自 [obra/superpowers](https://github.com/obra/superpowers)。
