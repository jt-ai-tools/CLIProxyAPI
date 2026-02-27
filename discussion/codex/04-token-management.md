# Token 儲存、刷新與生命週期管理

> 回到 [README](README.md) | 上一篇：[03-device-flow.md](03-device-flow.md) | 下一篇：[05-proxy-architecture.md](05-proxy-architecture.md)

---

## 1. Token 資料結構

**檔案**：[`internal/auth/codex/token.go`](../../internal/auth/codex/token.go)

### CodexTokenStorage（持久化格式）

```go
type CodexTokenStorage struct {
    IDToken      string         `json:"id_token"`       // JWT 身份令牌
    AccessToken  string         `json:"access_token"`   // API 呼叫用令牌
    RefreshToken string         `json:"refresh_token"`  // 用於自動刷新
    AccountID    string         `json:"account_id"`     // OpenAI 帳號 ID
    LastRefresh  string         `json:"last_refresh"`   // 上次刷新時間（RFC3339）
    Email        string         `json:"email"`          // 帳號 email
    Type         string         `json:"type"`           // 固定為 "codex"
    Expire       string         `json:"expired"`        // Token 過期時間（RFC3339）
    Metadata     map[string]any `json:"-"`              // 附加 metadata（不序列化）
}
```

### 儲存格式（JSON 檔案範例）

```json
{
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refresh_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "account_id": "user_abc123",
    "last_refresh": "2026-02-27T12:00:00Z",
    "email": "user@example.com",
    "type": "codex",
    "expired": "2026-02-28T12:00:00Z"
}
```

---

## 2. Token 儲存路徑

Token 被序列化為 JSON 後，存放在設定的 `auth-dir` 目錄下：

```
<auth-dir>/
└── codex_<email_hash>.json    # 每個帳號一個檔案
```

**目錄建立**：`os.MkdirAll(filepath.Dir(authFilePath), 0700)` — 權限為 `0700`（僅擁有者可讀）。

---

## 3. Token 取得後的建立流程

```
ExchangeCodeForTokens()
    │ 呼叫 OpenAI /oauth/token
    │ 取得 access_token, refresh_token, id_token
    │
    ▼
ParseJWTToken(id_token)
    │ 解析 JWT Payload（不驗證簽名）
    │ 取得 email, account_id
    │
    ▼
CodexAuthBundle {
    APIKey:      ""  (此版本未使用，Token 直接作為 Bearer 使用)
    TokenData:   CodexTokenData { ... }
    LastRefresh: time.Now()
}
    │
    ▼
CreateTokenStorage(bundle)  →  CodexTokenStorage
    │
    ▼
SaveTokenToFile(path)
    │ JSON 序列化
    │ 合併 Metadata（自訂欄位注入）
    ▼
儲存至 auth-dir/<filename>.json
```

---

## 4. Metadata 合併機制

`CodexTokenStorage` 支援透過 `SetMetadata()` 注入任意額外欄位，在序列化時這些欄位會被「平鋪」進 JSON 根層級：

```go
storage.SetMetadata(map[string]any{
    "codex_login_mode": "device",
    "custom_label":     "my-work-account",
})

// 儲存後的 JSON：
// {
//     "id_token": "...",
//     "access_token": "...",
//     "codex_login_mode": "device",    <-- 注入的欄位
//     "custom_label": "my-work-account"
// }
```

這樣設計讓後端可以在不修改核心結構的情況下，擴充儲存的 metadata。

---

## 5. Token 刷新機制

**檔案**：[`internal/auth/codex/openai_auth.go`](../../internal/auth/codex/openai_auth.go)

### 刷新請求

```http
POST https://auth.openai.com/oauth/token
Content-Type: application/x-www-form-urlencoded

client_id=app_EMoamEEZ73f0CkXaXp7hrann
&grant_type=refresh_token
&refresh_token=<current_refresh_token>
&scope=openid profile email
```

### 重試機制

```go
func (o *CodexAuth) RefreshTokensWithRetry(ctx context.Context, refreshToken string, maxRetries int) (*CodexTokenData, error) {
    for attempt := 0; attempt < maxRetries; attempt++ {
        if attempt > 0 {
            // 指數退避：第 N 次重試等待 N 秒
            time.After(time.Duration(attempt) * time.Second)
        }

        tokenData, err := o.RefreshTokens(ctx, refreshToken)
        if err == nil {
            return tokenData, nil
        }

        // 不可重試的錯誤：refresh_token_reused（Token 被重複使用）
        if isNonRetryableRefreshErr(err) {
            return nil, err
        }
        // 其他錯誤可重試
    }
    return nil, fmt.Errorf("failed after %d attempts", maxRetries)
}
```

### 不可重試情況

當 OpenAI 回傳錯誤訊息包含 `refresh_token_reused` 時，表示 Refresh Token 已被使用過一次，此時必須重新登入，無法自動刷新。這是 OpenAI 的 **旋轉式 Refresh Token** 安全機制。

---

## 6. Token 儲存更新

刷新成功後，更新現有儲存：

```go
func (o *CodexAuth) UpdateTokenStorage(storage *CodexTokenStorage, tokenData *CodexTokenData) {
    storage.IDToken      = tokenData.IDToken
    storage.AccessToken  = tokenData.AccessToken
    storage.RefreshToken = tokenData.RefreshToken    // 新的 refresh_token
    storage.AccountID    = tokenData.AccountID
    storage.LastRefresh  = time.Now().Format(time.RFC3339)
    storage.Email        = tokenData.Email
    storage.Expire       = tokenData.Expire
}
```

---

## 7. SDK 刷新排程

**檔案**：[`sdk/auth/codex.go`](../../sdk/auth/codex.go)

```go
func (a *CodexAuthenticator) RefreshLead() *time.Duration {
    return new(5 * 24 * time.Hour)  // 提前 5 天刷新
}
```

`RefreshLead()` 回傳的值告訴 SDK 的 Auth Manager，在 Token **過期前 5 天** 就主動刷新。這確保了服務不會因 Token 過期而中斷。

---

## 8. Token 生命週期時序

```
登入時刻
    │
    │  <── access_token 有效期（約 24 小時）
    │
    │                   過期前 5 天
    │                       │
    │                       ▼
    │               自動刷新（SDK RefreshLead）
    │                       │
    │                       │  <── 新 access_token 有效期
    │
    │  <── refresh_token 有效期（約 30 天）
    │
    │  refresh_token 到期 → 需要重新登入
    ▼
過期
```

---

## 9. 安全考量

| 安全措施 | 說明 |
|---------|------|
| 檔案權限 `0700` | Token 檔案目錄僅擁有者可存取 |
| PKCE `S256` | 防止授權碼被截取攻擊 |
| State 參數驗證 | 防止 CSRF 攻擊 |
| Refresh Token 旋轉 | 每次刷新後舊 Token 即失效 |
| Token 不傳送給下游工具 | Proxy 以 Token 替換 API Key，下游工具不持有 OpenAI Token |

---

## 相關文件

- [02-oauth-login-flow.md](02-oauth-login-flow.md) — Token 如何取得
- [05-proxy-architecture.md](05-proxy-architecture.md) — Token 如何用於 API 請求
