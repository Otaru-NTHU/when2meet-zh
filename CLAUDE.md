# CLAUDE.md — 約時間（when2meet-zh）

## 這個 repo 是什麼
無註冊、憑連結的中文約時間工具。需求與 API 合約在 `SPEC.md`，先讀它。

## 結構
- `docs/`：前端，純 Vanilla JS，**沒有建置步驟**，GitHub Pages 直接服務這個資料夾。
  - `config.js` 唯一的部署設定（`BACKEND: 'local'|'remote'`、`API_BASE`）
  - `app.js` 路由 + 三個畫面 + 時間格元件 `buildGrid()`
  - `backend-local.js` 示範模式的 localStorage 後端，**也是 API 合約的參考實作**
  - `holidays-tw.json` 台灣國定假日，每年加一次
- `backend-gas/Code.gs`：Google Apps Script 後端（試算表當 DB）
- `backend-fastapi/main.py`：FastAPI + SQLite 後端（stdlib sqlite3，無 ORM）

## 鐵則
1. **三個後端的邏輯必須一致**：改任何 action、驗證規則、錯誤訊息，三處一起改（local / GAS / FastAPI）。
2. 密碼雜湊固定 `sha256(salt + ':' + password)` hex；不要換演算法，否則 GAS ↔ SQLite 資料不能互搬。
3. 填寫者的 `slots` 永遠是 `allowed` 的子集合，由後端過濾，前端只是不讓點。
4. 前端不引入框架、不加打包工具；要保持「複製 docs/ 就能跑」。
5. 錯誤訊息一律中文，直接顯示給使用者。
6. GAS 的 `doPost` 已用 `LockService`；FastAPI 用 `_db_lock` 單執行緒寫入。不要移除。
7. 前端用 `Content-Type: text/plain` 送 JSON 是刻意的（避免 GAS 的 CORS 預檢），不要改成 `application/json`。

## 測試方式
- 示範模式：`cd docs && python3 -m http.server 8077` → 開 `http://localhost:8077/#/new`
- FastAPI：把 `config.js` 暫改 `BACKEND:'remote', API_BASE:'/api'`，`uvicorn main:app --port 8090`，開 `http://localhost:8090/`
- Playwright 可用：建立 → 加入 → 拖曳 `.cell:not(.blocked)` → 儲存 → `/admin` 檢查熱力圖；`page.on('dialog')` 要 accept。
- 測完記得把 `config.js` 改回 `local`（repo 預設值）。

## 風格
- 2 空格縮排；前端用 `el()` 建 DOM，不用 innerHTML 拼字串（除非靜態文字）。
- Python 用 type hints、`snake_case`；JS 用 `camelCase`；試算表欄位名與 JSON 欄位名相同。
- 提交訊息用中文或英文皆可，一句話說清楚改了什麼。

## 待辦 / 可能的下一步
- 匯出 .ics、活動過期自動清理、填寫者「勉強可以」第三級、QR code 分享。
