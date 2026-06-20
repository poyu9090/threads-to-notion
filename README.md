# Threads 聊天室訊息整理

把你在 Threads 看到、隨手轉發到「聊天室（私訊）」的貼文，自動整理成**摘要＋建議心得**，依主題分類、去重後寫進 **Notion** 的當月資料庫，方便之後再看。

**不使用 Threads API**——靠 Claude in Chrome 讀你「已登入的 Threads 網頁版」聊天室；資料全程在你本機處理，作者不會收到任何資料。

> ⚠️ **使用前請讀**：自動化讀取 Threads 可能違反 Meta 服務條款，帳號有被限制/停用的風險。本工具依「現狀（AS IS）」提供、不負擔保責任，請自行評估後使用（見 [LICENSE](LICENSE)）。

## 適用對象與前置需求

這是一個 **Claude Code 的 Skill**，不是獨立 App。你需要：

- **Claude Code**（或相容、支援 skills 的 agent 環境）
- **Claude in Chrome** 擴充功能已連線，且該 Chrome 已**登入 threads.com**
- **Notion** 帳號，且已在 Claude 連好 **Notion MCP**（要有寫入權限）

任一缺少，Skill 會停下並告訴你缺什麼，不會硬跑。

## 安裝

把這個 repo clone 到你的個人 skills 目錄：

```bash
# macOS / Linux
git clone https://github.com/poyu9090/threads-to-notion.git ~/.claude/skills/threads-to-notion
```

```powershell
# Windows (PowerShell)
git clone https://github.com/poyu9090/threads-to-notion.git "$env:USERPROFILE\.claude\skills\threads-to-notion"
```

接著複製設定範本（大多留空即可，第一次執行會引導你填）：

```bash
cp config.example.json config.json        # Windows: copy config.example.json config.json
```

## 第一次執行

在 Claude Code 跟它說「**整理 Threads**」。第一次跑時它會引導你：
1. 問「要把整理放進 Notion 哪一頁底下？」→ 給它一個 Notion 頁面網址，或請它幫你建一個容器頁（存成 `parentPageId`）。
2. 帶你到 Threads 收件匣，問「你都轉發到哪個聊天室？」→ 記住該對話（存成 `threadsChatUrl`）。

之後就不會再問。

## 日常使用

跟 Claude 說「**整理 Threads**」或「**更新 Threads 待讀**」即可。它會：
1. 開你的聊天室、往上捲載入歷史
2. 抓出所有轉貼的貼文連結
3. 比對去重（看過的會跳過）
4. 產生摘要／建議心得／主題，寫進**當月** Notion 資料庫（每月一個，自動建立）
5. 回報新增幾筆、跳過幾筆

`看過`（勾選）、`備註`（手填）兩欄給你自己用，Skill 不會去動；`加入日期`由 Notion 自動帶入。

## 你的資料放哪

- `config.json`（你的設定）與 `state/processed.json`（去重紀錄）**只存在你本機**，已被 `.gitignore` 擋住、不會上傳。
- 整理結果寫進**你自己的 Notion**，不經過任何第三方伺服器。

## 已知限制

- **吃 Threads 改版**：靠讀網頁畫面，Threads 改版時抓取規則可能要更新（細節見 `references/threads-scraping.md`）。
- **要登入**：遇到登入牆會停下請你登入，不會嘗試繞過。
- **語言/分類**：預設繁體中文、台灣用語；分類清單可在 `config.json` 的 `categories` 調整。

## 檔案結構

- `SKILL.md` — 主流程（Claude 觸發時讀）
- `references/` — 進階細節（`threads-scraping.md` 爬取、`notion-write.md` 寫入）
- `config.example.json` — 設定範本
- `config.json` — 你的設定（本機，不上傳）
- `state/processed.json` — 去重清單（本機，不上傳）

## 授權

MIT，見 [LICENSE](LICENSE)。
