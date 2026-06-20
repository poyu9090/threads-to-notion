---
name: threads-to-notion
description: >
  Threads 聊天室訊息整理：把使用者轉發到 Threads 聊天室（私訊 DM）的貼文整理好，寫進 Notion 待讀資料庫。
  運作方式不使用 Threads 官方 API，而是透過 Claude in Chrome 讀取「已登入的 Threads 網頁版」聊天室，
  抓出每則被分享的貼文連結、擷取內文，產生摘要與「讀後建議心得」、分類主題，去重後寫入當月的 Notion 資料庫。
  當使用者說「整理 Threads」「整理 Threads 聊天室訊息」「更新 Threads 待讀」「把 Threads 聊天室／轉發的貼文整理到 Notion」
  「跑一下 Threads 靈感筆記」「我轉到 Threads 聊天室的東西幫我整理」等，即使沒有明講 Notion 或 Skill 名稱，都應該使用這個 Skill。
  這個 Skill 專門處理「Threads 聊天室/私訊裡轉貼的貼文 → Notion」這條流程。
---

# Threads 聊天室訊息整理

把使用者平常在 Threads 看到、隨手轉發到「聊天室（私訊）」的貼文，整理成可以之後再看的清單，寫進 Notion。
**不使用 Threads API**：靠 Claude in Chrome 讀已登入的 Threads 網頁版。

## 核心理念

使用者的痛點是「看到有興趣的就轉發，但之後不知道要看什麼」。所以這個 Skill 的價值有兩層：
1. **整理**：把零散的轉貼變成結構化、可掃視的清單。
2. **判斷**：每則給一句**摘要**（這是什麼）＋一句**建議心得**（值不值得看、怎麼用、重點是什麼），
   讓使用者一眼決定「要不要點進去看」。建議心得是這個 Skill 真正的加值，不要敷衍。

## 前置需求（缺一就先處理，不要硬跑）

- **Claude in Chrome** 擴充功能已連線（工具名為 `mcp__Claude_in_Chrome__*`）。
  若工具是延遲載入，先用 ToolSearch 查 `Claude_in_Chrome` 載入 `navigate` / `get_page_text` / `read_page` / `find` / `javascript_tool`。
- 該 Chrome 已**登入 threads.com**。沒登入時不要嘗試繞過，直接請使用者登入後再繼續。
- **Notion MCP** 已連線（`mcp__*__notion-*`）。先用 ToolSearch 載入 `notion-create-pages` / `notion-create-database` / `notion-fetch` / `notion-create-view`。

## 設定檔

所有狀態存在這個 Skill 目錄下：
- `config.json` — Threads 聊天室網址、Notion 父頁（`parentPageId`；亦相容舊的 `yearPageId`/`monthPages` 月份結構）、各月資料庫 ID、主題清單。
- `state/processed.json` — 已處理過的貼文連結（**去重的依據，跨月共用**）。

每次開始前先讀 `config.json`。若 `threadsChatUrl` 為空，先做 **首次設定**（見下）。

## 流程

### 步驟 0 — 首次設定（config 還沒設好時，缺哪項就做哪項）

**(A) Notion 要放哪**（`monthDatabases` 為空，且 `monthPages` 與 `parentPageId` 都沒有時）
1. 問使用者：「要把 Threads 待讀整理放進 Notion 哪一頁底下？」請他給一個 Notion 頁面網址/ID；
   或由 Skill 用 `notion-create-pages` 幫他在工作區建一個 `Threads 待讀` 容器頁。
2. 把該頁 ID 寫進 `config.json` 的 `notion.parentPageId`。之後每月資料庫都建在它底下。

**(B) 聊天室是哪一個**（`threadsChatUrl` 為空時）
1. `navigate` 到 `https://www.threads.com/messages`（或 config 的 `threadsInboxUrl`）。
2. 若導向登入頁 → 請使用者登入，等他完成。
3. 用 `get_page_text` 列出收件匣的對話，問使用者「你都轉發到哪一個聊天室？」
4. 進入該對話，把網址寫回 `config.json` 的 `threadsChatUrl`，之後就不再問。

### 步驟 1 — 開啟聊天室、收集被分享的貼文連結

1. `navigate` 到 `threadsChatUrl`。若出現登入頁，請使用者登入後再繼續。
2. **往上滾載入歷史**：Threads 是虛擬滾動，只渲染畫面內的訊息，要邊滾邊抓。
   具體做法見 `references/threads-scraping.md`。重點：反覆把聊天容器捲到最上、等載入、再抓，
   直到「再滾也沒有新連結」或「抓到的連結都已在 `processed.json`」為止。
3. 用 `javascript_tool` 抓出所有指向 Threads 貼文的連結（`/@帳號/post/代碼`），
   **正規化**成 `https://www.threads.com/@handle/post/CODE`（去掉查詢參數），並去除批次內重複。
   純文字訊息、非貼文的連結都略過。

### 步驟 2 — 去重

讀 `state/processed.json`，只留下**沒處理過**的連結。若全部都處理過 → 回報「沒有新貼文」並結束，不要空寫。
（檔案不存在時——例如剛從 Git clone 下來——視為空清單 `{"processedUrls": []}`，第一次寫入時再建立。）

### 步驟 3 — 擷取每則新貼文內容

對每個新連結：`navigate` 進去，用 `get_page_text` / `read_page` 取得 **作者 @帳號、貼文內文、發文日期**。
貼文被刪/私密/讀不到時，不要卡住——記為「（原文無法讀取）」，仍寫入該筆並於建議心得註明，連結保留讓使用者自己點。

### 步驟 4 — 產生摘要、建議心得、分類

每則產出四個欄位，**繁體中文（台灣用語）**，風格參考下方範例（精簡、不灌水）：

- **標題**：一句話的鉤子，約 10–20 字。
- **摘要**：1–2 句講清楚「這是什麼」。客觀陳述，不要在這裡下判斷。內容不足時照實說（例如「未說明細節，需點進原文」）。
- **建議心得**：1 句你的加值判斷——值不值得看／適合誰／重點或行動。這是使用者決定「要不要看」的依據，要具體。
- **主題**：從清單挑**一個**最貼近的：
  `生活靈感/情感`、`美食/旅遊`、`娛樂/推薦`、`日常/幽默`、`資安警示`、`科技/AI`、`職場/工作`、`理財/投資`、`學習/知識`、`購物/好物`。
  都不合適時挑最接近的，並可向使用者建議新增分類。

**風格範例（對齊使用者既有的「靈感筆記」）：**

```
標題：租鑽戒求婚，把選擇權還給對方
摘要：朋友求婚時租一只真鑽戒指，成功後再帶女友回店挑要戴一輩子的婚戒，租金全額折抵。
建議心得：很巧的求婚點子，重點在「不替對方決定」的心意；想求婚或寫情感類內容可收藏。
主題：生活靈感/情感
```

### 步驟 5 — 寫入當月 Notion 資料庫

1. 用**今天日期**算出當月 key `YYYY-MM`（例如 `2026-06`）。
2. 查 `config.json` 的 `notion.monthDatabases[key]`：
   - 有 → 用它的 `dataSourceId`。
   - 沒有 → 依 `references/notion-write.md` 決定父頁（`monthPages` → `parentPageId` → 步驟 0 首次設定），
     在父頁底下新建 `Threads 待讀 YYYY-MM` 資料庫（含「依主題」看板檢視），並把新 ID 寫回 `config.json`。
3. 用 `notion-create-pages`，parent 用該 `data_source_id`，每則一筆，properties：
   `標題`、`摘要`、`建議心得`、`連結`、`主題`。可把原文全文放進頁面內文備查。
   一次呼叫可建多筆（最多 100），照欄位 schema 填。
   **`看過`（checkbox）與 `備註`（text）是使用者欄位，寫入時一律留空、不要覆蓋。**
   **`加入日期`（created_time）由 Notion 自動帶入建立時間，Skill 不需也不能填。**

### 步驟 6 — 更新狀態並回報

1. 把這次寫入的連結加進 `state/processed.json`。
2. 向使用者回報：新增 N 筆、略過 M 筆（重複）、依主題分組的條列，並附當月資料庫連結。

## 重要原則

- **去重以正規化後的貼文連結為準**，跨月共用 `processed.json`，避免同一則被重複寫入。
- **絕不改用 Threads API 或非官方端點**；讀不到就回報，請使用者協助（登入、捲動、確認聊天室）。
- 寫入前先確認沒有空摘要、沒有重複連結；寧可少寫，不要亂寫進使用者的 Notion。
- 對使用者帳號要溫和操作：不發訊息、不點讚、不改任何 Threads 內容，只**讀取**。
