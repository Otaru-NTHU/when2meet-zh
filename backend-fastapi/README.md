# FastAPI + SQLite 後端

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env            # 選填：SMTP、FRONTEND_URL、DIGEST_HOUR、DB_PATH
uvicorn main:app --host 0.0.0.0 --port 8000
```

- `POST /api` 單一端點，body 為 JSON `{action, ...}`；合約見根目錄 `SPEC.md`。
- 會自動把 `../docs` 當前端服務在 `/`。前端 `config.js` 設 `BACKEND: 'remote'`、`API_BASE: '/api'`。
- 資料庫：`data/meet.sqlite3`（WAL 模式）。備份就是複製這個檔；不要用 Dropbox / Google Drive 同步它。
- 每日摘要：背景執行緒在 `DIGEST_HOUR` 整點（伺服器本地時間）寄出。
- 密碼儲存：`sha256(salt + ':' + password)`，與 GAS 版相同，資料可互相搬移。

## systemd 範例

```ini
[Unit]
Description=yueshijian
After=network.target

[Service]
WorkingDirectory=/opt/when2meet-zh/backend-fastapi
ExecStart=/opt/when2meet-zh/backend-fastapi/.venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always

[Install]
WantedBy=multi-user.target
```
前面再用 nginx / Caddy 反代並加 HTTPS。
