# 約時間 — 開源的中文 when2meet

不用註冊、只要有連結就能填寫的約時間工具。任何人都能建立活動、設定可選的時段範圍，
被邀請的人用「自己取的名稱 + 這次活動專用的密碼」填寫並隨時修改；
創辦人有獨立後台，可以看交集熱力圖、看每個人的空檔、開關是否公開結果、定案並通知。

- 介面全中文；週末與台灣國定假日淡紅、午休與下班時間淡灰（僅提示，仍可選）
- 逐小時（預設 08–22，可切 00–24）或「上午 / 中午 / 下午 / 晚上」粗略模式
- 填寫者只能在創辦人開放的格子裡選；縮小範圍時會先警告再刪除範圍外的選擇
- 通知：每次異動即時 email、每日摘要 email、Discord webhook（三者皆免費）
- **不需要資料庫**：後端可用 Google Apps Script + Google 試算表（零費用、免伺服器），
  也可用 FastAPI + SQLite 自架。前端是純靜態網頁，放 GitHub Pages 即可。

```
when2meet-zh/
├── docs/               前端（GitHub Pages 根目錄；config.js 是唯一要改的檔）
├── backend-gas/        後端 A：Google Apps Script（Code.gs）
├── backend-fastapi/    後端 B：FastAPI + SQLite（main.py）
├── SPEC.md             需求與 API 合約
└── CLAUDE.md           給 Claude Code 的開發指引
```

---

## 0. 先試玩（不需要任何後端）

`docs/config.js` 預設 `BACKEND: 'local'`（示範模式），資料只存在你的瀏覽器裡。
直接把 `docs/` 放到任何靜態主機、或在本機跑 `python3 -m http.server` 打開 `docs/index.html` 就能看介面。

---

## 1. 部署方案 A：GitHub Pages + Google Apps Script（推薦，零費用）

### 1-1. 建立後端（約 5 分鐘）

1. 用你要部署的 Google 帳號到 <https://sheets.new> 建一份新的試算表，命名例如「約時間資料」。
2. 試算表上方選單 **擴充功能 → Apps Script**，會開啟一個綁定這份試算表的指令碼專案。
3. 把預設的 `Code.gs` 內容全部換成本 repo `backend-gas/Code.gs` 的內容。
4. 左側齒輪 **專案設定** → 勾選「在編輯器中顯示 appsscript.json 資訊清單檔案」，
   回到編輯器把 `appsscript.json` 換成本 repo `backend-gas/appsscript.json`（設定時區為 Asia/Taipei、允許匿名存取）。
5. 同一頁「專案設定」最下方 **指令碼屬性 → 新增屬性**：
   `FRONTEND_URL` = 你的前端網址（例如 `https://otaru-nthu.github.io/when2meet-zh/`），信件裡的連結會用到。
6. 回到編輯器，上方函式下拉選 `setup`，按 **執行**。第一次會要求授權（存取試算表、寄信、建立觸發器），全部允許。
   這一步會自動建立 `events / participants / changes` 三個工作表與每日摘要的時間觸發器。
7. 右上 **部署 → 新增部署作業** → 類型選 **網頁應用程式**：
   - 執行身分：**我**
   - 誰可以存取：**所有人**（Anyone，包含匿名）
   按部署，複製出現的 **網頁應用程式網址**（`https://script.google.com/macros/s/…/exec`）。

> 之後若修改 Code.gs，要再 **部署 → 管理部署作業 → 編輯 → 版本：新版本** 才會生效；網址不變。

### 1-2. 建立前端（GitHub Pages）

1. Fork 或上傳本 repo 到你的 GitHub。
2. 編輯 `docs/config.js`：
   ```js
   BACKEND: 'remote',
   API_BASE: 'https://script.google.com/macros/s/…/exec',   // 上一步的網址
   REPO_URL: 'https://github.com/Otaru-NTHU/when2meet-zh',      // 選填，頁尾連結
   ```
3. GitHub repo → **Settings → Pages** → Source 選 **Deploy from a branch**，Branch 選 `main`、資料夾選 **/docs**，儲存。
4. 一兩分鐘後 `https://otaru-nthu.github.io/when2meet-zh/` 就可以用了。

### 1-3. 關於 NTHU / Google Workspace 帳號

- Workspace 帳號寄信額度是每天 1,500 封（個人 Gmail 100 封），通知量綽綽有餘。
- 部分機構的 Workspace 會限制「網頁應用程式 → 誰可以存取 → 所有人」這個選項（只允許網域內）。
  若步驟 1-1 第 7 步找不到「所有人」選項，表示機構有限制：改用個人 Gmail 帳號重做 1-1 即可，前端不用動，只要換 `API_BASE`。
- 試算表就是你的資料庫，隨時可以打開看。密碼只存加鹽後的 SHA-256 雜湊，看不到明碼。

---

## 2. 部署方案 B：自架 FastAPI + SQLite

適合想把資料留在自己機器上、或不想依賴 Google 的情況。FastAPI 會同時服務前端，一個行程搞定。

```bash
git clone https://github.com/Otaru-NTHU/when2meet-zh.git
cd when2meet-zh/backend-fastapi
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # 填 SMTP（要寄信才需要）與 FRONTEND_URL
```

編輯 `../docs/config.js`：
```js
BACKEND: 'remote',
API_BASE: '/api',
```

啟動：
```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```
打開 `http://localhost:8000/` 即可。資料在 `backend-fastapi/data/meet.sqlite3`。

- 正式上線建議放在 nginx / Caddy 後面加 HTTPS（密碼是以明文經 HTTPS 傳送後在伺服器端雜湊，所以一定要 HTTPS）。
- 每日摘要由程式內的背景執行緒在 `DIGEST_HOUR` 整點寄出；沒設 SMTP 時通知只會印在終端機。
- 也可以前端放 GitHub Pages、後端自架：`API_BASE` 填完整網址（例如 `https://meet.example.com/api`），CORS 已開。
- 用 systemd / pm2 / Docker 跑都可以；`DB_PATH` 環境變數可改資料庫位置。

---

## 3. 通知方式一覽

| 方式 | 費用 | 設定 |
|---|---|---|
| 每次異動 email | 免費（GAS：Gmail 100/日、Workspace 1,500/日） | 建立活動時選「每次有人填寫 / 修改就寄 email」 |
| 每日摘要 email | 免費 | 選「每天寄一封摘要 email」；GAS 的 `setup()` 已建觸發器，FastAPI 內建排程 |
| Discord webhook | 免費、無上限 | Discord 頻道 → 編輯頻道 → 整合 → Webhook → 複製網址，貼到活動設定 |
| 定案通知 | 免費 | 後台「選擇定案時段 ★」→ 儲存 → 「寄送定案通知」寄給有留 email 的填寫者 |

LINE Notify 已於 2025 年停止服務，LINE Messaging API 免費額度每月僅 200 則，故未採用。

---

## 4. 每年要做的事

`docs/holidays-tw.json` 放台灣國定假日（含補假日），目前有 2026、2027。
每年人事行政總處公布隔年行事曆後，照格式加幾行即可。週六、日由程式自動判斷。

---

## 5. 常見調整

- 粗略模式的時段切法：`docs/config.js` 的 `COARSE_BLOCKS`。
- 午休 / 下班的灰底時段：`LUNCH_HOURS`、`WORK_START`、`WORK_END`。
- 配色：`docs/style.css` 最上方的 CSS 變數。
- 日期範圍上限（預設 62 天）：三個後端各有 `MAX_DAYS`。

## 授權

MIT
