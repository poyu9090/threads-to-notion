# Threads 網頁版聊天室抓取細節

Threads 是 React 應用、class 名稱是亂數且會隨改版變動，**不要寫死 class**。
靠「語意穩定的東西」定位：連結的 `href` 結構（`/@帳號/post/代碼`）幾乎不會變。

所有 JS 用 `mcp__Claude_in_Chrome__javascript_tool` 在分頁內執行。

## 1. 抓取被分享的貼文連結

```js
// 回傳目前 DOM 裡所有 Threads 貼文連結（已正規化、去重）
(() => {
  const re = /threads\.(?:net|com)\/@([^/?#]+)\/post\/([A-Za-z0-9_-]+)/;
  const set = new Set();
  for (const a of document.querySelectorAll('a[href*="/post/"]')) {
    const m = (a.href || '').match(re);
    if (m) set.add(`https://www.threads.com/@${m[1]}/post/${m[2]}`);
  }
  return [...set];
})();
```

## 2. 往上捲動載入歷史（虛擬滾動）

聊天室只渲染畫面內的訊息，舊訊息要捲上去才會載入，太遠的又會被移除，所以要**邊捲邊抓、累積去重**。

先找出真正可捲動的容器（不是 window）：

```js
(() => {
  const cand = [...document.querySelectorAll('div')].filter(d => {
    const s = getComputedStyle(d);
    return /(auto|scroll)/.test(s.overflowY) && d.scrollHeight > d.clientHeight + 200;
  }).sort((a, b) => b.scrollHeight - a.scrollHeight);
  const el = cand[0];
  if (el) el.scrollTop = 0;      // 捲到最上＝載入更舊的訊息
  return el ? el.scrollHeight : 0;
})();
```

**迴圈做法**（在程式邏輯層，不是一段 JS）：
1. 跑「抓連結」，把結果併入累積集合。
2. 跑「捲到最上」。
3. 稍等讓內容載入（再呼叫一次抓連結即可，不需 sleep 太久）。
4. 重複，直到滿足任一停止條件：
   - 連續 2~3 次捲動後累積集合**數量沒再增加**（到頂了）。
   - 新抓到的連結**都已存在 `processed.json`**（已經追上上次的進度，不必再往上）。
5. 設一個合理上限（例如最多捲 30 次）避免無限迴圈。

> 若 `javascript_tool` 捲動無效（容器抓錯），改用鍵盤：聚焦訊息區後送 `Home` / `PageUp`，
> 或用 `mcp__Claude_in_Chrome__computer` 在訊息區滾輪上捲，每次再抓一次連結。

## 3. 擷取單篇貼文內容

`navigate` 到正規化後的貼文網址，再用 `get_page_text`（拿乾淨純文字）或 `read_page`。
通常頁面最上方主貼文就是作者與內文；只取**主貼文**，不要把底下留言一起當成內容。

抓不到（刪文/私密/被擋）時，回傳空字串即可——上層流程會記為「原文無法讀取」。

## 4. 常見狀況

- **登入牆**：出現登入畫面就停下，請使用者登入，不要嘗試自動登入或繞過。
- **純文字訊息 / 非貼文連結**：略過，只處理符合 `/@帳號/post/代碼` 的連結。
- **轉推/引用**：以連結指向的那一篇為準。
- **同一則被轉發多次**：正規化後去重即可。
