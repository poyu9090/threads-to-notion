# Notion 寫入細節

## 資料庫 schema（每月一個）

每個月在「對應月份頁」底下一個資料庫，標題 `Threads 待讀 YYYY-MM`，欄位：

| 欄位 | 型別 | 說明 |
|------|------|------|
| 標題 | title | 一句話鉤子 |
| 摘要 | text | 這是什麼（1–2 句） |
| 建議心得 | text | 值不值得看／重點／行動（1 句） |
| 連結 | url | 正規化後的貼文網址（去重依據） |
| 主題 | select | 分類擇一 |
| 看過 | checkbox | **使用者自己勾**，Skill 不要動 |
| 備註 | text | **使用者手動輸入**，Skill 不要動 |

> `看過` 與 `備註` 是使用者欄位：建立資料庫時要有，但寫入貼文時**留空**，不要覆蓋使用者已填的內容。

DDL（用 `notion-create-database`，parent 為當月月份頁的 `page_id`）：

```sql
CREATE TABLE ("標題" TITLE, "摘要" RICH_TEXT, "建議心得" RICH_TEXT, "連結" URL,
  "主題" SELECT('生活靈感/情感':pink, '美食/旅遊':orange, '娛樂/推薦':purple,
  '日常/幽默':yellow, '資安警示':red, '科技/AI':blue, '職場/工作':gray,
  '理財/投資':green, '學習/知識':brown, '購物/好物':default),
  "看過" CHECKBOX, "備註" RICH_TEXT)
```

建完後再加一個依主題分群的看板檢視（`notion-create-view`，`type: board`，`GROUP BY "主題"`），
方便重現「依分類掃視」的閱讀體驗。把回傳的 `databaseId` 與 `dataSourceId` 寫回 `config.json`。

## 找/建當月月份頁

1. 先看 `config.json` 的 `notion.monthPages[<月份數字>]` 有沒有現成頁 ID。
2. 沒有的話，`notion-fetch` 年份頁（`yearPageId`）找名為「一月～十二月」的子頁；
   仍找不到就在年份頁底下新建一個該月份頁，並把 ID 寫回 config。

## 寫入資料列

用 `notion-create-pages`，`parent` 用 `{ "type": "data_source_id", "data_source_id": "<該月 dataSourceId>" }`，
一次最多 100 筆。每筆的 `properties`：

```json
{
  "標題": "租鑽戒求婚，把選擇權還給對方",
  "摘要": "朋友求婚時租一只真鑽戒指，成功後再帶女友回店挑要戴一輩子的婚戒，租金全額折抵。",
  "建議心得": "很巧的求婚點子，重點在不替對方決定的心意；想求婚或寫情感內容可收藏。",
  "連結": "https://www.threads.com/@rndcomtw/post/DYnycqUH1aX",
  "主題": "生活靈感/情感"
}
```

可選：把原文全文放進該頁 `content`（Notion Markdown）備查，但標題不要重複放進內文。

## 注意

- 欄位名稱要跟 schema 完全一致（繁體中文）。`連結` 不是英文 "url"，照一般欄位填即可。
- `主題` 的值必須是九個選項之一，否則會建立新選項（通常不想要）。
- 寫入前再對一次 `processed.json` 去重，避免重複列。
