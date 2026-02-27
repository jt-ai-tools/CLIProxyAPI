# 07 - Demo / 開發環境安全設定建議

> **適用情境：** 僅在本機（localhost）使用，不對外暴露服務，以 Demo 或開發測試為目的。

---

## 最小化設定（直接複製使用）

```yaml
# 綁定 localhost，不對外暴露
host: "127.0.0.1"
port: 8317

# API Key：本機用隨便設一個，但不要留空
api-keys:
  - "dev-local-only"

# 管理 API：Demo 不需要，直接停用
remote-management:
  secret-key: ""           # 空值 = 管理 API 全部回傳 404，最乾淨
  disable-control-panel: true  # 停用管理面板（避免自動下載外部 HTML）

# 關閉 request log（預設已關閉，確認一下）
# request-log: false  ← 預設值，不需要特別設定

# 關閉 debug log（避免敏感資訊出現在 stdout）
debug: false
```

---

## 核心原則與說明

### 1. 綁定 `127.0.0.1`，不要用 `0.0.0.0`

`config.yaml` 的 `host` 預設值為空字串，代表**監聽所有網路介面（包含對外）**。

```yaml
# ❌ 預設值（對外開放）
host: ""

# ✅ 本機 Demo 應改為
host: "127.0.0.1"
```

只要綁定在 `127.0.0.1`，同一台機器以外的任何裝置都無法連線，大多數的攻擊面自動消失。

---

### 2. 停用管理 API（`secret-key` 留空）

管理 API（`/v0/management/*`）提供包括「將 Token 傳送至任意 URL」的強大功能。Demo 用途完全不需要它。

```yaml
remote-management:
  secret-key: ""  # 空值 = 管理 API 全部回傳 404
```

這同時也消除了稽核報告中的 [高風險 APICall 端點](03-api-call-endpoint.md) 問題。

---

### 3. 停用控制面板（`disable-control-panel: true`）

管理面板 HTML 每 3 小時從外部 URL 自動下載（詳見 [02-management-html.md](02-management-html.md)）。Demo 不需要此功能。

```yaml
remote-management:
  disable-control-panel: true
```

---

### 4. 設定一個 API Key（不要留空）

即使只在 localhost 使用，設定 API Key 可防止同機器上的其他程序或指令碼意外呼叫到此服務。

```yaml
api-keys:
  - "dev-local-only"   # 本機用，隨意取名即可
```

---

### 5. 不要啟用 `request-log`

`request-log: true` 會將完整的 Bearer Token 寫入本地日誌（詳見 [04-request-log.md](04-request-log.md)）。預設為關閉，不要主動開啟它。

---

### 6. 不要設定遠端 Token Store

除非有多機器共用的需求，否則不要設定以下環境變數：

```bash
# 不要設定這些（Token 會同步至遠端）
# GITSTORE_GIT_URL=...
# OBJECTSTORE_ENDPOINT=...
```

---

## 快速檢查清單

開始 Demo 前，確認以下項目：

- [ ] `host: "127.0.0.1"` — 服務不對外暴露
- [ ] `remote-management.secret-key: ""` — 管理 API 已停用
- [ ] `remote-management.disable-control-panel: true` — 停用外部 HTML 下載
- [ ] `api-keys` 至少設定一個值（不為空）
- [ ] `debug: false` — 不在 stdout 噴出敏感資訊
- [ ] 沒有設定 `GITSTORE_GIT_URL` / `OBJECTSTORE_*` 環境變數
