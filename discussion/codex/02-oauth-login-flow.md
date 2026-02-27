# OAuth 2.0 + PKCE 登入流程深度分析

> 回到 [README](README.md) | 上一篇：[01-overview.md](01-overview.md) | 下一篇：[03-device-flow.md](03-device-flow.md)

---

## 1. 流程概覽

CLIProxyAPI 實作的是標準 **OAuth 2.0 Authorization Code Flow + PKCE（RFC 7636）** 流程，以冒用 OpenAI Codex CLI 的 Client ID 完成授權，取得 OpenAI Access Token。

```
使用者                 CLIProxyAPI               auth.openai.com
  │                       │                            │
  │  cliproxy login codex │                            │
  │──────────────────────▶│ 1. 生成 PKCE Codes         │
  │                       │    (code_verifier,         │
  │                       │     code_challenge)        │
  │                       │                            │
  │                       │ 2. 啟動本地 Callback Server │
  │                       │    http://localhost:1455   │
  │                       │                            │
  │                       │ 3. 建構授權 URL             │
  │◀──────────────────────│                            │
  │  開啟瀏覽器            │                            │
  │─────────────────────────────────────────────────▶│
  │  用 ChatGPT Pro 帳號登入                           │
  │◀─────────────────────────────────────────────────│
  │  重導向到 localhost:1455/auth/callback?code=...    │
  │                       │                            │
  │                       │ 4. Callback Server 接收 code
  │                       │ 5. 驗證 state 參數          │
  │                       │ 6. code + code_verifier    │
  │                       │    → POST /oauth/token    │
  │                       │──────────────────────────▶│
  │                       │◀──────────────────────────│
  │                       │  access_token             │
  │                       │  refresh_token            │
  │                       │  id_token (JWT)           │
  │                       │                            │
  │                       │ 7. 解析 JWT 取得            │
  │                       │    account_id, email       │
  │                       │ 8. 儲存 Token 至 auth-dir  │
  │◀──────────────────────│                            │
  │  "Authentication saved!"                          │
```

---

## 2. PKCE 安全機制

**檔案**：[`internal/auth/codex/pkce.go`](../../internal/auth/codex/pkce.go)

PKCE（Proof Key for Code Exchange）是防止授權碼攻擊的安全擴充。

### 實作細節

```go
// 生成 96 個隨機位元組，Base64 URL 編碼（無 padding）
func generateCodeVerifier() (string, error) {
    bytes := make([]byte, 96)
    rand.Read(bytes)
    return base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(bytes), nil
}

// 對 code_verifier 取 SHA-256，再 Base64 URL 編碼
func generateCodeChallenge(codeVerifier string) string {
    hash := sha256.Sum256([]byte(codeVerifier))
    return base64.URLEncoding.WithPadding(base64.NoPadding).EncodeToString(hash[:])
}
```

### 授權請求中的 PKCE 參數

| 參數 | 值 | 說明 |
|------|-----|------|
| `code_challenge` | SHA256(code_verifier) \| base64url | 傳送給授權伺服器 |
| `code_challenge_method` | `S256` | 使用 SHA-256 |
| `code_verifier` | 原始隨機字串 | Token 交換時證明身分 |

---

## 3. 授權 URL 構造

**檔案**：[`internal/auth/codex/openai_auth.go`](../../internal/auth/codex/openai_auth.go)

```go
params := url.Values{
    "client_id":                  {"app_EMoamEEZ73f0CkXaXp7hrann"},
    "response_type":              {"code"},
    "redirect_uri":               {"http://localhost:1455/auth/callback"},
    "scope":                      {"openid email profile offline_access"},
    "state":                      {state},     // CSRF 防護
    "code_challenge":             {pkceCodes.CodeChallenge},
    "code_challenge_method":      {"S256"},
    "prompt":                     {"login"},
    "id_token_add_organizations": {"true"},
    "codex_cli_simplified_flow":  {"true"},
}
```

**重要**：`codex_cli_simplified_flow=true` 這個參數告訴 OpenAI 授權伺服器，此次登入是 Codex CLI 的簡化流程，會略過部分額外的同意步驟。

---

## 4. 本地 Callback Server

**檔案**：[`internal/auth/codex/oauth_server.go`](../../internal/auth/codex/oauth_server.go)

### 運作原理

```go
type OAuthServer struct {
    server     *http.Server
    port       int          // 預設 1455
    resultChan chan *OAuthResult
    errorChan  chan error
}
```

啟動後監聽兩個端點：
- `/auth/callback` — 接收授權碼（code + state）
- `/success` — 顯示成功頁面給使用者

### 等待回呼（帶 Timeout）

```go
func (s *OAuthServer) WaitForCallback(timeout time.Duration) (*OAuthResult, error) {
    select {
    case result := <-s.resultChan:
        return result, nil
    case err := <-s.errorChan:
        return nil, err
    case <-time.After(timeout):  // 預設 5 分鐘
        return nil, fmt.Errorf("timeout waiting for OAuth callback")
    }
}
```

---

## 5. Token 交換

**檔案**：[`internal/auth/codex/openai_auth.go`](../../internal/auth/codex/openai_auth.go)，函式 `ExchangeCodeForTokens`

### 請求格式

```http
POST https://auth.openai.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&client_id=app_EMoamEEZ73f0CkXaXp7hrann
&code=<authorization_code>
&redirect_uri=http://localhost:1455/auth/callback
&code_verifier=<original_random_verifier>
```

### 回應格式

```json
{
    "access_token": "eyJ...",
    "refresh_token": "eyJ...",
    "id_token": "eyJ...",
    "token_type": "Bearer",
    "expires_in": 86400
}
```

---

## 6. JWT 解析取得使用者資訊

**檔案**：[`internal/auth/codex/jwt_parser.go`](../../internal/auth/codex/jwt_parser.go)

CLIProxyAPI 在不做簽名驗證的情況下，解析 `id_token` 取得：

```go
type JWTClaims struct {
    Email         string        `json:"email"`
    CodexAuthInfo CodexAuthInfo `json:"https://api.openai.com/auth"`
    Sub           string        `json:"sub"`
    ...
}

type CodexAuthInfo struct {
    ChatgptAccountID    string `json:"chatgpt_account_id"`
    ChatgptPlanType     string `json:"chatgpt_plan_type"`  // "pro" / "plus" / ...
    ChatgptUserID       string `json:"chatgpt_user_id"`
    ...
}
```

> 注意：JWT 簽名未被驗證，因為 Token 本身已經由 OpenAI 授權伺服器透過 HTTPS 傳遞，可視為可信來源。

---

## 7. SDK 協調層

**檔案**：[`sdk/auth/codex.go`](../../sdk/auth/codex.go)

```go
type CodexAuthenticator struct {
    CallbackPort int  // 預設 1455
}

func (a *CodexAuthenticator) Login(ctx context.Context, cfg *config.Config, opts *LoginOptions) (*coreauth.Auth, error) {
    // 1. 決定 Device Flow 還是標準 PKCE Flow
    if shouldUseCodexDeviceFlow(opts) {
        return a.loginWithDeviceFlow(ctx, cfg, opts)
    }

    // 2. 生成 PKCE Codes
    pkceCodes, _ := codex.GeneratePKCECodes()
    state, _ := misc.GenerateRandomState()

    // 3. 啟動 OAuth Callback Server
    oauthServer := codex.NewOAuthServer(callbackPort)
    oauthServer.Start()
    defer oauthServer.Stop(...)

    // 4. 建構授權 URL，開啟瀏覽器
    authSvc := codex.NewCodexAuth(cfg)
    authURL, _ := authSvc.GenerateAuthURL(state, pkceCodes)
    browser.OpenURL(authURL)

    // 5. 等待 Callback
    result := oauthServer.WaitForCallback(5 * time.Minute)

    // 6. 驗證 state
    // 7. 交換 Token
    authBundle, _ := authSvc.ExchangeCodeForTokens(ctx, result.Code, pkceCodes)

    // 8. 建立 Auth 記錄並回傳
    return a.buildAuthRecord(authSvc, authBundle)
}
```

---

## 8. 手動輸入 Callback URL（備援機制）

若使用者在無法自動開啟瀏覽器的環境（如 SSH 遠端），系統會在等待 15 秒後提示：

```
Paste the Codex callback URL (or press Enter to keep waiting):
```

使用者可手動從瀏覽器複製 `http://localhost:1455/auth/callback?code=...&state=...` 並貼上，系統會解析後繼續流程。

---

## 相關文件

- [03-device-flow.md](03-device-flow.md) — 無需瀏覽器的 Device Code 替代流程
- [04-token-management.md](04-token-management.md) — Token 儲存與自動刷新
