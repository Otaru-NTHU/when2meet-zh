# Google Apps Script 後端

完整部署步驟見根目錄 `README.md` 第 1 節。重點：

1. 新試算表 → 擴充功能 → Apps Script → 貼上 `Code.gs` 與 `appsscript.json`
2. 指令碼屬性加 `SPREADSHEET_ID` 與 `FRONTEND_URL`
3. 執行一次 `setup()`（建工作表 + 每日摘要觸發器）
4. 部署為網頁應用程式：執行身分「我」、存取權「所有人」
5. 把 `/exec` 網址填進前端 `docs/config.js` 的 `API_BASE`

修改程式後需「管理部署作業 → 編輯 → 新版本」才會生效。

## 資料表

- `events`：每個活動一列。`allowed`、`finalSlots` 為 JSON 字串；`pwHash = SHA256(salt + ':' + 密碼)`
- `participants`：每位填寫者一列，`slots` 為 JSON 字串
- `changes`：通知方式為「每日摘要」的異動佇列，寄出後 `sent = TRUE`

可以直接在試算表裡刪掉整列來刪除活動或填寫者。
