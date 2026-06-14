# threads-to-notion

把你轉發到 Threads 聊天室（私訊）的貼文，自動整理成摘要＋建議心得，依主題分類、去重後寫進 Notion 當月資料庫。**不使用 Threads API**，靠 Claude in Chrome 讀已登入的網頁版。

## 前置需求
- Claude in Chrome 擴充功能已連線，且該 Chrome 已登入 threads.com
- Notion MCP 已連線

## 日常使用
跟 Claude 說「**整理 Threads**」或「**更新 Threads 待讀**」即可。它會：
1. 開你的聊天室、往上捲載入歷史
2. 抓出所有轉貼的貼文連結
3. 比對 `state/processed.json` 去重（看過的會跳過）
4. 產生摘要／建議心得／主題，寫進當月 Notion 資料庫
5. 回報新增幾筆、跳過幾筆

`看過`、`備註` 兩欄是你自己用的，Skill 不會去動。

## 分享給別人時要改什麼（重要）
這個 Skill 預設綁定原作者的環境，**直接分享別人不能用**，必須換掉以下個人資料：

1. **`config.json`**：裡面是原作者的 Notion 頁面 ID 與聊天室網址。
   - 請改用 `config.example.json` 當範本，填**你自己的** `threadsChatUrl`、`yearPageId`、`monthPages`。
   - `monthDatabases` 留空，第一次跑會自動建立並回填。
2. **`state/processed.json`**：清空成 `{ "processedUrls": [] }`，否則會誤判別人的貼文已處理過。
3. **各自的連線**：每個人要有自己的 Claude in Chrome（登入自己的 Threads）和自己的 Notion 連線。
4. **Notion 結構**：預設假設「年份頁 → 月份子頁」的結構。若你的 Notion 不是這樣，第一次跑時告訴 Claude 你想把資料庫放哪，讓它幫你建。
5. **隱私認知**：這個 Skill 會讀你的**私訊內容**並寫進 Notion，分享對象要清楚這點。

## 已知限制
- **看 DOM 吃改版**：Threads 網頁版改版時，抓取用的選擇器可能要更新（細節在 `references/threads-scraping.md`，已盡量只靠穩定的連結結構定位）。
- **要登入**：遇到登入牆會停下請你登入，不會嘗試繞過。
- **聊天室預覽通常夠用**：多數貼文在聊天室就有完整預覽文字；需要更完整內容（例如留言裡的連結）時才會另外點進原文。

## 檔案結構
- `SKILL.md` — 主流程
- `config.json` — 你的設定（個人資料，分享前要換）
- `config.example.json` — 空白範本
- `state/processed.json` — 去重清單（跨月共用）
- `references/threads-scraping.md` — 抓取細節
- `references/notion-write.md` — Notion 寫入細節
