# 晨間簡報網頁版（morning-briefing）

固定網址的晨報網頁。版型寫一次、之後幾乎不動；每天只更新 `data/latest.json` 一個檔案。

- Repo：`Eric-Tai-5269/morning-briefing`
- 部署後網址：`https://eric-tai-5269.github.io/morning-briefing/`

## 檔案結構

```
index.html          版型 + CSS + 渲染邏輯（讀 config 與 data 後渲染五張卡）
config/site.json    站台設定、Notion 表單選項、觀察清單（75 檔，換股票才改）
data/latest.json    今天的內容 ← Make 每天覆寫這個
data/2026-08-01.json 當日存檔（每天新增一個）
data/archive.json   { "dates": [...] } 日期選單靠它
preview.html        單檔預覽版（資料內嵌），電腦上直接雙擊可看，不需部署、不用上傳也行
robots.txt          Disallow（擋搜尋引擎）
```

> ⚠️ `index.html` 需透過 GitHub Pages 或本機伺服器開啟才讀得到資料。**電腦上直接雙擊請開 `preview.html`**（已內嵌當日資料）。

## Step 1 — 部署（用網頁上傳，不用終端機）

1. 打開你的 repo：`https://github.com/Eric-Tai-5269/morning-briefing`
2. 按 **Add file → Upload files**，把這個資料夾裡的**所有檔案與 `config/`、`data/` 兩個子資料夾**一起拖進去（拖整個資料夾內容，不要拖最外層那層）。
3. 下方 **Commit changes**。
4. 左上 **Settings → Pages**：Source 選 **Deploy from a branch**、Branch 選 **main**、資料夾選 **/(root)**，Save。
5. 等 1–2 分鐘，網址 `https://eric-tai-5269.github.io/morning-briefing/` 就會上線。

替代方案（都不用終端機）：Netlify Drop 拖資料夾、Cloudflare Pages Direct Upload。

## Step 2 — 改 Make 場景 A（5338818），改成推 JSON 而非夾附件

目前場景 A 是把整份 HTML 硬寫進 Gmail。要改成：每天把 `data/latest.json` 推上 GitHub、Gmail 只送「摘要＋固定網址按鈕」。

新流程（HTTP 模組）：

1. **GET** `https://api.github.com/repos/Eric-Tai-5269/morning-briefing/contents/data/latest.json`
   - Header `Authorization: Bearer <PAT>`、`Accept: application/vnd.github+json`
   - 取回 `sha`（第一次檔案已存在會成功；若不存在會 404，要設定「允許錯誤狀態繼續」）
2. **PUT** 同一路徑，body：`{ "message": "update latest", "content": <base64(JSON)>, "sha": <上一步的 sha> }`
3. **PUT** `data/{{formatDate(now;"YYYY-MM-DD")}}.json`（新的當日存檔，不帶 sha）
4. **PUT** `data/archive.json`（先 GET 拿 sha，把新日期加進 `dates` 再 PUT）
5. Gmail 送摘要信 + 一顆連到 `https://eric-tai-5269.github.io/morning-briefing/` 的按鈕（不再夾附件）

- **PAT**：用 fine-grained personal access token，只授權這個 repo 的 **Contents: Read and write**。到 GitHub → Settings → Developer settings → Fine-grained tokens 建立。**Token 是密碼，只貼進 Make 的 HTTP 連線設定，別貼進聊天或 commit。**
- 排程：從 on-demand 改成 **daily 07:00（Asia/Taipei）**。
- 成本約 6–8 ops/天，免費額度 1000/月，足夠。

> 改場景前務必先 `scenarios_get(5338818)` 讀出目前 blueprint 再改（update 是整份覆蓋），並用 `app-modules_list` 確認 HTTP 模組的正確識別碼，不要猜。

## Step 3 — 讓 Make 自己抓資料（最大工程，之後做）

目前資料仍靠 Claude 產生。要全自動需接：Notion（今日事件／我的任務，場景 B 5347658 已有 Notion 連線可複用）、Gmail 成大信箱（**尚未在 Make 連線**）、天氣 API、股價 API、新聞 RSS；信件重要性判斷與新聞摘要走 Claude API 模組。

## Step 4 — 隱私

GitHub Pages 一律公開（即使 repo 設 private）。現況：`robots.txt` + `<meta noindex>` 只擋搜尋引擎。要真的鎖，用 **Cloudflare Pages + Cloudflare Access**（免費 50 人）綁 Google 登入。
另外 `config/site.json` 的 `calendarHook` 會公開，任何人都能往你的 Notion 塞資料；解法是在場景 B 加 filter 檢查 `&k=<密碼>`，或行事曆按鈕只留在 Email。

## 資料格式（`data/latest.json`）

| key | 內容 |
| --- | --- |
| `weather` | `{icon, high, low, note}` |
| `schedule` | `{available, message, tasks:[{title,time,done,pri,cat}]}` |
| `mail` | `{window, footnote, items:[{level(red/amber/green), tag, title, deadline, kind, detail, calendar}]}`；`calendar` = `{title,date,cat,pri,label}`，`null` 表不顯示按鈕 |
| `news` | `{window, groups:[{label, items:[{en,zh,summary,source,url,spotify?}]}]}` |
| `market` | `{asOf, note, highlight, indices:[{market,name,value,change,note}], movers:[{name,ticker,change,price,reason}], quotes:{代號:漲跌%}, disclaimer, historyNote}` |

`config/site.json` 的 `watchlist`：`{TW:{分類:[[名稱,代號,描述,風險,參考價],...]}, US:{...}}`。堆疊卡的漲跌用 `market.quotes[代號]`；sparkline 依 `參考價 + quotes` 生成、目前為示意，接股價 API 後可改為真實走勢。

## Step 5 — 週報（之後）

直接讀七天份 `data/YYYY-MM-DD.json` 彙整即可，不用另建 Data Store。
