# CLAUDE.md — 網站流量排名比較工具（traffic-rank-estimator）

單檔前端工具：使用者輸入一份網站名稱／網址清單 → 送給 LLM 估算各站「相對熱門程度」的趨勢指數（0-100，僅供參考，非真實流量數據，結算基準固定抓「目前日期往前 3 個月」）→ 多線折線圖比較 ＋ 全球/地區排名 KPI → 針對比較結果自由問答 → 可下載截圖或分享（含選填的姓名/組別/系所/學號）。無建置步驟、無框架、無 package.json，直接開啟 `index.html`（`file://`）或以靜態伺服器託管即可。

**此資料夾本身是獨立 git 儲存庫**，本地身分 `Mark Tsai <tsaimark@gmail.com>`，不受根目錄工作區規則約束（除語言等全域偏好）。

**100% 需要 API 金鑰才能運作，無規則式離線備援**（與工作區其他 BYOK 工具的關鍵差異）：沒有免費、CORS 友善的真實流量 API 可串接，估算的唯一來源就是 LLM 的訓練知識，因此沒有「無金鑰時退回規則式」這條路——`index.html` 與 `manual.html` 都在最上方明確告知使用者這一點，避免使用者以為跟其他工具一樣有免金鑰模式。

**不套用序號授權**（比照 `social-post-grader`／`coffee-ig-planner` 無授權慣例）。**無可攜式桌面版 exe**。

## 架構

單一 `index.html`：內嵌 `<style>` 與三個獨立 `<script>`（主程式 IIFE／PWA 安裝 IIFE／跑馬燈 IIFE），外部資源為 Chart.js CDN、html2canvas CDN（下載截圖用）與選用的 AI API fetch。

### 網站清單輸入解析（`parseSiteList`）

textarea 依換行與逗號同時切分、trim、去重（不分大小寫），上限 8 個（`MAX_SITES`），超過時 UI 提示只取前 8 個。清單草稿與下方「比較期間／地區／身分資訊」設定一起存 `localStorage`（key `trafficRankDraft`，值為 `{siteText, range, region, regionCustom, group, dept, studentId}` 物件；2026-09-08 由純字串改為物件，`loadDraft()` 回傳值也跟著變成物件，同日再加入 `group`/`dept`/`studentId` 三欄）。

### 比較期間、結算基準日與時間粒度（`resolveGranularity` / `periodCount` / `getAnchorDate` / `anchorLabel` / `fallbackPeriods`）

`#rangeSelect` 提供 6 個選項：近6個月／近12個月／近2年／近3年／近5年／近10年（value 為月數 6/12/24/36/60/120）。**期間越長，時間點單位自動變粗**（`resolveGranularity(months)`），避免資料點過多讓圖表難讀、也控制 token 成本：≤36 個月用「月」（最多 36 點）、≤60 個月（近5年）用「季」（20點）、>60 個月（近10年）用「年」（10點）。`periodCount(months, granularity)` 算實際要幾個時間點。

**結算基準日固定抓「目前日期往前 3 個月」，不是今天**（2026-09-08 使用者要求）：`getAnchorDate()` 回傳 `今天所在月份 - 3` 的月初日期，`anchorLabel(granularity)` 把這個日期依粒度轉成給 AI 看的文字（月 `2026年06月`／季 `2026年第2季`／年 `2026年`）。理由：太接近「現在」的資訊 AI 難以準確掌握（尤其 assistant 的知識截止日通常早於實際使用當下），固定往回退 3 個月當結算基準可以讓估算更可靠。`#rangeSelect` 選的「期間長度」是從這個基準點往前回推，**不是**從今天往前回推。`fallbackPeriods(months, granularity)` 在 AI 回傳的 `periods` 格式異常時於前端本機算出對應粒度、對應基準日的時間標籤（月 `YYYY-MM`／季 `YYYY-Qn`／年 `YYYY`）——這只是圖表座標軸文字，不涉及任何流量數字的臆造。**修改期間相關邏輯時，`getAnchorDate()` 只能有這一份實作，`fallbackPeriods()` 與 `buildEstimatePrompt()` 都要呼叫它、不要各自重算「今天」，否則兩處基準日會兜不起來。**

### 地區排名 KPI（`#regionSelect`）

預設選項「台灣」＋常見地區（美國/日本/中國大陸/香港/東南亞）＋自訂＋「未指定（只看全球排名）」。`getResolvedRegion()` 讀值模式比照 `getResolvedModel()`（選 custom 才讀自訂輸入框）。全域排名估計一律會問，地區排名估計只有選了地區才會問，兩者都要求 AI 回傳「粗略量級」的一句話文字（例如「全球約前 500 名」），**刻意不要求裸數字**，避免呈現假精確度。

### AI 估算（`buildEstimatePrompt` / `callLLM` / `extractJsonObject`）

`AI_PROVIDERS`（Claude/OpenAI/Gemini/OpenRouter）、`MODEL_OPTIONS`（常用模型下拉+自訂）、`callLLM()`、`extractJsonObject()` 皆逐字沿用 `social-post-grader/index.html` 已驗證過的實作（Claude 需 `anthropic-dangerous-direct-browser-access` header、429/500/503/529 重試 3 次、180 秒逾時）。**與姊妹工具的差異**：`callLLM(provider, model, apiKey, promptOrMessages, onRetry)` 的第 4 個參數擴充為可接受字串（單發估算用）或 `messages` 陣列（Q&A 問答用），內部統一組成 `messages` 再依各服務商格式送出；Gemini 因原生角色名稱是 `model` 而非 `assistant`，在組 `contents` 時額外做一次角色轉換。

`buildEstimatePrompt(sites, months, granularity, region)` 明確告知模型「沒有即時/真實流量數據或排名存取權，以下數字與排名都是依訓練知識做的主觀估計」，並明確告知 `anchorLabel(granularity)` 算出的結算基準日（**不是現在的日期**、是刻意往回退 3 個月），要求以此為終點往前回推整段期間；依 `months`/`granularity` 動態算出要幾個時間點、依 `region` 是否有值決定要不要多問一個地區排名欄位，要求回傳嚴格 JSON：`{"periods":[...N個時間標籤],"sites":[{"name","indexSeries":[...N個0-100數字],"trendNote","globalRankEstimate","regionalRankEstimate"?},...],"summary"}`（欄位名 2026-09-08 由 `months`/`monthlyIndex` 改為粒度無關的 `periods`/`indexSeries`），並要求輸出順序與輸入清單順序一致（`name` 直接照抄輸入文字），讓驗證階段可以先用索引比對、索引失敗才退回名稱比對。

### 驗證（`validateAiComparison(parsed, requestedSites, months, granularity, region)`）

逐站檢查 `indexSeries` 是否為 `periodCount(months,granularity)` 個有限數字（`clampScore` 逐一夾在 0-100）；**單一站點格式異常時只略過該站並在結果卡片顯示提示**，不讓整批估算失敗（因為這裡沒有規則式 fallback 可退，比照 `social-post-grader` 的 `validateAiEvaluation` 逐項 fallback 精神，差別是這裡「fallback」是省略而非改用規則式）。`globalRankEstimate`/`regionalRankEstimate` 是自由文字，僅做長度裁切（120字），沒有格式驗證；`regionalRankEstimate` 只有 `region` 有值時才採用。整體 JSON 解析（`extractJsonObject`）失敗，或驗證後 0 個站點成功，則顯示錯誤訊息，使用者可直接重試。

### 折線圖與 KPI 卡片（Chart.js，`renderComparisonChart` / `renderResult`）

CDN `cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js`，`STATE.chartInstances{}` + 重繪前 `destroyChart()` 呼叫 `.destroy()` 的模式沿用 `資料儀表板/Dashboard/index.html`。`type:'line'`，x 軸為 `periods`（`ticks.autoSkip`+`maxTicksLimit:12` 避免長期間、多時間點時標籤擠爆），每個網站一條線，y 軸固定 `min:0,max:100`（指數本身就是正規化過的相對值，不像 Dashboard 用動態 `beginAtZero`）。所有顏色皆透過 `cssVar()` 在渲染當下讀取 CSS 變數，因此切換主題後只要重繪一次就會套用新色（見下方「多款主題」）。`renderResult()` 在每個網站的說明卡片內加一列 `.kpi-row`（🌍全球排名估計，選了地區時再加一塊📍地區排名估計），純文字量級估計，非可運算的數字。

### 多款主題／底色（`THEMES` / `applyTheme` / `initThemeSwitcher`，2026-09-08 新增）

使用者要求「底色要有多款可以選擇，一定要有淺色底」。4 款主題定義在 `<style>` 開頭的 `[data-theme="..."]` CSS 區塊 ＋ JS 端 `THEMES` 陣列（兩邊 id 要對應）：`midnight`（午夜藍，深色，預設）、`violet-night`（深紫夜，深色，僅換強調色）、`sky-light`（晴空淺色，**淺色**，符合「一定要有淺色底」的硬性需求）、`sand-light`（暖米淺色，淺色）。色值皆取自 `dataviz` skill `references/palette.md` 的**已驗證 light/dark 分類色盤**（8 色 `--series-1`~`--series-8` 依 light/dark 兩欄切換）與該檔的 light/dark 版面 token 表（chart surface／page plane／ink／gridline），不是憑感覺調的。

**CSS 變數採「leaf token 全部改寫、公式 token 只算一次」的架構**：每個主題只需覆寫 `--bg`/`--surface`/`--text`/`--accent`/`--warn`/`--danger`/`--series-1~8` 等少數 leaf token（`[data-theme="sky-light"], [data-theme="sand-light"]` 共用同一組淺色 leaf token，`sky-light`/`sand-light` 各自再覆寫 `--accent`），像 `--accent-soft`／`--bg-glow-1`／`--topbar-bg`／`--danger-soft-bg` 這類半透明衍生色一律寫成 `rgba(var(--accent-rgb),0.14)` 這種「引用 RGB 三元組 var 的公式」，定義一次放在 `:root`，不必每個主題重複寫。**新增元件若要用半透明色，一律走這套 `*-rgb` + 公式 token 的模式，不要再手刻 `rgba(57,135,229,...)` 這種寫死色值**，否則新色值不會隨主題切換。

`applyTheme(id, persist)`：設定 `<html data-theme="...">`、依 `persist` 參數決定要不要寫 `localStorage`（key `trafficRankTheme`）、同步 `<meta name="theme-color">` 的 content（手機瀏覽器工具列會跟著變色）、更新 `#themeSwitcher` 內按鈕的 `aria-pressed`、**若已有比較結果就呼叫 `renderResult()` 重繪**（圖表/KPI 的顏色是渲染當下讀 CSS var，不重繪不會換色）。`<head>` 內 `<style>` 之前有一段同步執行的 inline `<script>`：讀 `localStorage.trafficRankTheme`，沒存過就依 `prefers-color-scheme` 決定預設深/淺色，在 CSS 解析前就設好 `data-theme` 屬性，避免翻頁先閃一下預設深色再跳到使用者選的主題（flash of wrong theme）。主題選單 UI 是右上角 4 個圓形色塊（`.theme-swatch`，半深半淺的 `linear-gradient` 預覽該主題的底色+強調色），不是下拉選單。

### 身分資訊欄位（`#nameInput`/`#groupInput`/`#deptInput`/`#studentIdInput`）

選填的姓名／組別／系所／學號四個文字輸入框，放在輸入卡片**最下方**（「AI 估算設定」之後、「開始比較」按鈕之前，2026-09-08 使用者要求從卡片最上方移到最下方），與網站清單/比較設定一起存進同一份 `trafficRankDraft` 草稿。**只用於下載截圖／分享時顯示身分，完全不會送進 AI prompt**（不是估算邏輯的一部分，純粹是畫面/截圖用的標示文字）。`renderResultIdentity()` 把四欄組成一行「姓名：W　｜　組別：X　｜　系所：Y　｜　學號：Z」寫進 `#resultIdentity`（在 `#resultCard` 內、圖表上方），全部留空時顯示「（未填寫姓名／組別／系所／學號）」。四個輸入框各自綁 `input` 事件：草稿即時存檔，且**若 `#resultCard` 目前是顯示狀態就即時呼叫 `renderResultIdentity()` 重繪**——這樣使用者在看到比較結果後才補填學號，不必重新按一次「開始比較」，截圖就會是最新值。

### 訪客計數器與跑馬燈（2026-09-08 跑馬燈由 stub 改為實作）

Footer 已加訪客次數計數器（`visitor-badge.laobi.icu`，`page_id=m255525.trafficrankestimator`，比照工作區共用慣例）。頂部跑馬燈**逐字沿用 `social-post-grader/index.html` 已驗證過的共用實作**（`MARQUEE_CHECK_URL` 是工作區多個工具共用的同一顆 Google Apps Script 端點，POST 空 `serial`，只取回傳的 `marquee` 陣列，忽略 `valid`/`reason`；20 分鐘刷新一次；`localStorage` 快取 key 改成本工具專屬的 `trafficRankMarquee`）。本頁有 sticky `.topbar`，所以版面整合比照 `social-post-grader` 用 `body.has-marquee .topbar{top:30px}`（CSS 早在初版就已就位），不是 `coffee-ig-planner` 那種無 sticky header 的簡化版本。

### 下載截圖／分享（`html2canvas` CDN，2026-09-08 新增）

CDN `cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js`。`#exportCard` 是獨立於 `#resultCard` 之外的一張卡片，放「📷 下載截圖」「📤 分享」兩顆按鈕——**按鈕故意不放在 `#resultCard` 裡面**，因為 `captureResultCanvas()` 是對 `#resultCard` 這個節點整個做 `html2canvas()`，按鈕若在裡面會被一起拍進截圖。`#resultCard` 內含 `#resultIdentity`（身分資訊）／圖表／KPI／說明，所以截圖天然就包含這些內容，不需要另外拼一張「匯出專用」的隱藏 DOM。

`downloadScreenshot()`：`html2canvas` 產生 canvas → `toBlob('image/png')` → `URL.createObjectURL` 建一個暫時的 `<a download>` 連結並模擬點擊觸發下載 → `revokeObjectURL` 回收。`shareScreenshot()`：同樣先產生 blob，包成 `File`，若瀏覽器支援 `navigator.share`＋`navigator.canShare({files:[...]})`（主要是手機瀏覽器，例如 Android Chrome／iOS Safari）就跳系統分享選單；不支援（多數桌面瀏覽器）就退回跟下載一樣的行為並提示「已改為下載圖片」。`buildExportFilename()` 檔名帶學號（若有填，去除非英數字元）＋日期戳記。兩個函式都會先檢查 `STATE.lastComparison` 存在才執行，避免使用者還沒比較就點按鈕。

**下載時間戳記（2026-09-08 使用者要求）**：`captureResultCanvas()` 在呼叫 `html2canvas` 之前先 `renderResultIdentity(true)`，把「按下下載/分享當下」的日期時間（`formatDownloadTimestamp()`，`YYYY-MM-DD HH:mm`）壓進「姓名」那一欄（`姓名：王小明（下載時間：2026-09-08 16:23）`；姓名沒填也會強制顯示「姓名：未填寫（下載時間：…）」，確保時間戳一定會出現），截圖完成後（`.finally()`，無論成功或失敗）呼叫 `renderResultIdentity()`（不帶參數）還原成一般畫面，**不會**讓使用者事後看到畫面上留著一個過期的下載時間。`renderResultIdentity(withTimestamp)` 第二個路徑（`withTimestamp` 為 falsy）行為與加這個功能前完全一致，姓名欄位空白時整段省略，不影響既有的即時更新邏輯（`initCompareSettings()` 的四個 input 監聽器呼叫的都是不帶參數的版本）。

### 響應式／手機可用性

`.compare-settings`／`.api-grid`／`.identity-fields` 在 `max-width:640px` 會從多欄改單欄；`.topbar` 允許 `flex-wrap`（品牌／主題色塊／操作手冊連結在窄螢幕會自動換行，不會被擠爆）；`.theme-swatch` 用 26px 圓形按鈕，在小螢幕上仍維持可點擊的觸控面積；`.kpi-row`／`.export-actions` 本來就用 `flex-wrap:wrap`。已用 Playwright 在 375×812 視窗實測無橫向溢出、topbar/footer 正確換行。

### Q&A 問答面板（多輪 `messages` 陣列）

比照 `AI人物顧問/mentors-panel/index.html` 的多輪對話模式：首次比較成功後 `seedChat()` 把比較結果 JSON 包成一則模擬的 user/assistant 對話塞進 `STATE.chatMessages`，之後每次提問都 `push` 新的 user/assistant 訊息、整包送給 `callLLM()`。**僅存於記憶體，不寫 localStorage**；換一批網站重新比較時（`seedChat` 被重新呼叫）會覆蓋重置。問答框在尚未完成一次比較前停用（`disabled`）。

## 隱私與警語

無自建後端、無資料上傳到本工具以外的伺服器（LLM API 除外）。API 金鑰只存瀏覽器 localStorage；網站清單草稿存 localStorage；Q&A 對話僅存記憶體，重新整理即消失。首頁、手冊皆明列：AI 估算結果非真實流量數據、僅供教學與參考、請勿作為商業決策依據。

## 指令

無建置/測試指令。`python -m http.server 8813 --directory 行銷內容工具/traffic-rank-estimator`（工作區行銷內容工具資料夾埠號連號 8812 之後的下一個空號；`.claude/launch.json` 已加入對應設定，用 Preview MCP `preview_start` 啟動，不要手動另開伺服器）。

驗證 AI 路徑不需要真實金鑰：可在瀏覽器 console 攔截 `window.fetch` 回傳假造的 provider 回應格式（內嵌合法的估算 JSON payload），確認 `callLLM → extractJsonObject → validateAiComparison → renderResult` 整條管線正確，包含刻意讓某一站的 `indexSeries` 格式異常、確認只有該站被略過而非整批失敗，以及切換不同「比較期間」（月/季/年粒度）與「地區排名」選項後 prompt 與驗證的點數/欄位是否正確跟著變動；也要驗證 Q&A 送出問題時 `STATE.chatMessages` 有正確帶入先前比較結果（含排名 KPI）的上下文。測完記得還原 `window.fetch`。

驗證主題切換：4 顆色塊都點過一輪，確認 `document.documentElement.getAttribute('data-theme')`、`localStorage.trafficRankTheme`、`<meta name="theme-color">` 的 content 都正確跟著換；已有比較結果時切換主題，確認 `STATE.chartInstances.trendChart.data.datasets[i].borderColor` 等有換成新主題的色值（不是重新整理後才生效）。

驗證結算基準日：呼叫 `window.__trafficRank.getAnchorDate()` 確認回傳「本機系統時間所在月份 - 3」；分別跑月/季/年三種粒度，確認 `buildEstimatePrompt()` 送出的 prompt 字串裡有出現正確的 `anchorLabel`（而不是「今天」），且 `fallbackPeriods()` 在格式異常時算出的標籤序列最後一個確實對應這個基準日，不是系統當下月份。

驗證身分資訊與截圖：填姓名/組別/系所/學號後跑一次比較，確認 `#resultIdentity` 內文正確組合；比較完成後才修改學號，確認 `#resultIdentity` 不必重新比較就即時更新；點「📷 下載截圖」（測試環境無法真的驗證檔案落地，可改為直接呼叫 `html2canvas($('resultCard'))` 確認 resolve 出一個 `HTMLCanvasElement` 且 `width`/`height` > 0）；用 `navigator.share`/`navigator.canShare` 皆為 `undefined` 的情境（大多數桌面瀏覽器）驗證「分享」按鈕會退回下載並跳出對應提示。

## 部署

尚未推公開 GitHub repo／GitHub Pages（比照使用者「實驗性新工具部署前先確認」的標準做法，發布前需先詢問使用者）。未來若要推廣，只差 `.github/workflows` 自動部署這一步（跑馬燈／訪客計數器／截圖分享皆已於 2026-09-08 完成）。
