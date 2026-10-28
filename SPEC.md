# SPEC — 約時間（when2meet-zh）

## 1. 目標

無註冊、憑連結存取的約時間工具（when2meet 的中文版），前端純靜態可放 GitHub Pages，
後端二選一：Google Apps Script + Google 試算表（零費用）或 FastAPI + SQLite（自架）。兩者共用同一 API 合約。

## 2. 角色與權限

| 角色 | 身分 | 能做的事 |
|---|---|---|
| 創辦人 | 活動 ID + 創辦人密碼（此活動專用） | 看完整結果與名單、改設定、改可選時段、刪填寫者、定案、寄定案通知、改密碼 |
| 填寫者 | 活動 ID + 自取名稱 + 自設密碼（此活動專用） | 填寫 / 修改自己的空檔、留 email；若創辦人開放則看熱力圖（名單與否由創辦人決定） |
| 路人 | 活動 ID | 看活動標題、日期、人數；若創辦人開放則看熱力圖 |

活動 ID 為 10 碼隨機字串（去掉易混淆字元），只有拿到連結的人找得到。

## 3. 時段模型

- `slotId = "<YYYY-MM-DD>|<key>"`
  - 逐小時模式：`key` = 兩位數小時 `00`…`23`，代表 `HH:00–HH+1:00`
  - 粗略模式：`key` ∈ `am`(08–12) / `noon`(12–13) / `pm`(13–18) / `eve`(18–22)，定義在 `docs/config.js` `COARSE_BLOCKS`
- 活動欄位 `allowed: slotId[]` = 創辦人開放的格子；填寫者的 `slots` 必為 `allowed` 的子集合（後端強制過濾）。
- 逐小時模式顯示範圍 `hourStart`/`hourEnd`（預設 8/22，可切 0/24）。範圍縮小時：前端列出受影響的已填寫格子並要求創辦人確認，後端依新的 `allowed` 移除範圍外的選擇並回傳 `strippedSlots` 數。
- 底色提示（不影響可否選擇）：週六日與 `docs/holidays-tw.json` 內的日期 → 淡紅；午休（12 時）與下班（<8 或 ≥18）→ 淡灰；未開放 → 斜線。假日優先於灰底。

## 4. 資料欄位

**event**：`id, title, creatorName, creatorEmail, salt, pwHash, mode('hour'|'coarse'), hourStart, hourEnd, dateStart, dateEnd, allowed[], showResults, showNames, notifyMode('none'|'instant'|'daily'), discordWebhook, finalSlots[], finalNote, createdAt, updatedAt`

**participant**：`eventId, name(活動內唯一), email, salt, pwHash, slots[], createdAt, updatedAt`

**change**（每日摘要佇列）：`eventId, name, action, at, sent`

密碼：`pwHash = hex(sha256(salt + ':' + password))`，salt 16 碼隨機。密碼最少 4 字。

驗證規則：標題 1–80 字；日期範圍 ≤ 62 天且起 ≤ 訖；`0 ≤ hourStart < hourEnd ≤ 24`；`allowed` 非空；`notifyMode ≠ none` 時 `creatorEmail` 必填；名稱 1–40 字。

## 5. API 合約

單一端點 `POST <API_BASE>`，body 為 JSON（`Content-Type: text/plain` 以避免 CORS 預檢），回應 JSON。
失敗一律 `{ok:false, error:"中文訊息"}`。

| action | 參數 | 回傳 |
|---|---|---|
| `createEvent` | event 欄位（不含 id/salt/pwHash）+ `password` | `{ok, id}` |
| `getEvent` | `id` | `{ok, event(公開欄位), participantCount, participants?}`，`participants` 僅在 `showResults` 時回傳，`showNames=false` 時名稱改為「參與者 N」 |
| `joinEvent` | `id, name, password, email?` | `{ok, participant:{name,email,slots}}`；名稱重複則錯誤 |
| `loginParticipant` | `id, name, password` | 同上 |
| `saveSlots` | `id, name, password, slots[], email?` | `{ok, participant, notified}`；slots 會被過濾成 allowed 子集合；觸發通知 |
| `adminLogin` | `id, password` | `{ok, event(含 creatorEmail/notifyMode/discordWebhook), participants:[{name,email,slots,createdAt,updatedAt}]}` |
| `adminUpdate` | `id, password, patch{任意 event 欄位, newPassword?}` | 同 adminLogin + `strippedSlots` |
| `adminDeleteParticipant` | `id, password, name` | 同 adminLogin |
| `adminNotifyFinal` | `id, password, message?` | `{ok, sent}`；寄給有留 email 的填寫者 |

公開欄位：`id, title, creatorName, mode, hourStart, hourEnd, dateStart, dateEnd, allowed, showResults, showNames, finalSlots, finalNote, createdAt, updatedAt`。

## 6. 通知

- `instant`：`joinEvent` / `saveSlots` 後立即寄 email 給 `creatorEmail`。
- `daily`：寫入 `changes`，由排程（GAS 時間觸發器 / FastAPI 背景執行緒）每天 `DIGEST_HOUR` 彙整寄出。
- `discordWebhook` 有值時，不論 notifyMode 一律同時推送。
- `adminNotifyFinal`：定案通知，寄給填寫者。

## 7. 前端畫面

- `#/new`：建立活動（三段：資訊 / 時間範圍與可選格子 / 顯示與通知）→ 成功頁顯示填寫連結與後台連結。
- `#/e/<id>`：填寫頁。未登入可看熱力圖（若開放）；登入（第一次填寫 / 修改我的選擇）後可拖曳塗選、疊看熱力圖、點名單標示某人空檔、儲存、留 email。
- `#/e/<id>/admin`：後台。熱力圖 + 誰有空 + 最多人時段 Top 8 + 名單（可刪）、設定、可選時段（含範圍切換與縮小警告）、定案與通知。
- 登入狀態存 `sessionStorage`（關分頁即清除），每次請求重送名稱 + 密碼，後端無 session。

## 8. 非目標（目前不做）

- 多時區、跨日時段、重複週期、匯出 ics、找回密碼、活動自動過期刪除。

## 9. 驗收清單

- [ ] 建立逐小時 / 粗略兩種活動並拖曳塗選
- [ ] 第二人用相同名稱加入被拒；正確密碼可登入修改
- [ ] showResults 關閉時填寫者看不到他人；showNames 關閉時只看到「參與者 N」
- [ ] 後台縮小可選範圍時跳出確認，確認後範圍外選擇被移除且 `strippedSlots` 正確
- [ ] 0–24 切回 8–22 時，有範圍外選擇則確認；無則直接切換
- [ ] instant / daily / Discord 通知各寄出一次
- [ ] 定案後填寫頁顯示 ★ 與定案通知
