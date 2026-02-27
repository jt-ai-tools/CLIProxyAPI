# Device Code Flow（無瀏覽器環境）

> 回到 [README](README.md) | 上一篇：[02-oauth-login-flow.md](02-oauth-login-flow.md) | 下一篇：[04-token-management.md](04-token-management.md)

---

## 1. 什麼是 Device Code Flow？

Device Code Flow（RFC 8628）是針對 **無法直接開啟瀏覽器的裝置**（如 SSH 伺服器、CI/CD 環境、嵌入式系統）設計的 OAuth 流程。

使用指令：

```bash
cliproxy login codex --device
```

---

## 2. 流程圖

```
使用者（SSH 連線）         CLIProxyAPI              auth.openai.com
      │                       │                           │
      │  login codex --device │                           │
      │──────────────────────▶│                           │
      │                       │ 1. POST /deviceauth/usercode
      │                       │──────────────────────────▶│
      │                       │◀──────────────────────────│
      │                       │  { device_auth_id,        │
      │                       │    user_code: "ABCD-1234" │
      │                       │    interval: 5 }          │
      │                       │                           │
      │◀──────────────────────│                           │
      │  顯示：                │                           │
      │  Device URL: https://auth.openai.com/codex/device │
      │  Device Code: ABCD-1234                           │
      │                       │                           │
      │  （使用者在其他裝置    │                           │
      │   開啟瀏覽器輸入代碼） │                           │
      │                       │ 2. 輪詢 /deviceauth/token │
      │                       │──────────────────────────▶│
      │                       │   每 5 秒詢問一次          │
      │                       │◀──────────────────────────│
      │                       │  { authorization_code,    │
      │                       │    code_verifier,         │
      │                       │    code_challenge }       │
      │                       │                           │
      │                       │ 3. POST /oauth/token      │
      │                       │   (用 authorization_code) │
      │                       │──────────────────────────▶│
      │                       │◀──────────────────────────│
      │                       │  access_token             │
      │                       │  refresh_token            │
      │                       │  id_token                 │
      │◀──────────────────────│                           │
      │  "Authentication saved!"                          │
```

---

## 3. 實作細節

**檔案**：[`sdk/auth/codex_device.go`](../../sdk/auth/codex_device.go)

### 關鍵端點

| 端點 | URL |
|------|-----|
| 取得 User Code | `https://auth.openai.com/api/accounts/deviceauth/usercode` |
| 輪詢 Token | `https://auth.openai.com/api/accounts/deviceauth/token` |
| Token 交換端點 | `https://auth.openai.com/oauth/token` |
| Device 驗證頁面 | `https://auth.openai.com/codex/device` |
| Token Exchange Redirect URI | `https://auth.openai.com/deviceauth/callback` |

### Step 1：請求 Device User Code

```go
type codexDeviceUserCodeRequest struct {
    ClientID string `json:"client_id"`
}

// POST https://auth.openai.com/api/accounts/deviceauth/usercode
// Body: {"client_id": "app_EMoamEEZ73f0CkXaXp7hrann"}
```

回應：

```go
type codexDeviceUserCodeResponse struct {
    DeviceAuthID string          `json:"device_auth_id"`
    UserCode     string          `json:"user_code"`     // 例："ABCD-1234"
    Interval     json.RawMessage `json:"interval"`      // 輪詢間隔（秒）
}
```

### Step 2：輪詢等待使用者授權

```go
type codexDeviceTokenRequest struct {
    DeviceAuthID string `json:"device_auth_id"`
    UserCode     string `json:"user_code"`
}

// 每隔 interval 秒輪詢一次
// POST https://auth.openai.com/api/accounts/deviceauth/token
```

回應（當使用者完成授權後）：

```go
type codexDeviceTokenResponse struct {
    AuthorizationCode string `json:"authorization_code"`
    CodeVerifier      string `json:"code_verifier"`
    CodeChallenge     string `json:"code_challenge"`
}
```

> 與標準 PKCE 流程不同，Device Flow 中的 PKCE `code_verifier` 是由 **OpenAI 伺服器端生成** 並在 Token 回應中一併回傳，不是由本機生成。

### Step 3：用 authorization_code 交換 Token

```go
// 使用 Device Flow 特定的 redirect_uri
authSvc.ExchangeCodeForTokensWithRedirect(
    ctx,
    authCode,
    "https://auth.openai.com/deviceauth/callback",  // Device Flow 專用 redirect_uri
    &codex.PKCECodes{
        CodeVerifier:  tokenResp.CodeVerifier,
        CodeChallenge: tokenResp.CodeChallenge,
    },
)
```

---

## 4. 兩種流程的核心差異

| 面向 | 標準 PKCE 流程 | Device Code Flow |
|------|----------------|------------------|
| 觸發指令 | `login codex` | `login codex --device` |
| 瀏覽器需求 | 需要（可自動開啟） | 不需要（手動在另一裝置開） |
| PKCE 生成方 | 本機（CLIProxyAPI） | OpenAI 伺服器 |
| redirect_uri | `http://localhost:1455/auth/callback` | `https://auth.openai.com/deviceauth/callback` |
| 本地 Server | 需要啟動（埠 1455） | 不需要 |
| 等待機制 | 接收 HTTP Callback | 固定間隔輪詢 |
| 超時時間 | 5 分鐘 | 15 分鐘 |

---

## 5. Token Metadata 標記

Device Flow 登入完成後，儲存的 Token 會帶有特殊 Metadata：

```go
// internal/cmd/openai_device_login.go
authOpts := &sdkAuth.LoginOptions{
    Metadata: map[string]string{
        "codex_login_mode": "device",  // 標記此 Token 由 Device Flow 取得
    },
}
```

此 Metadata 會被序列化並儲存在 Token 檔案中，方便後續診斷。

---

## 6. 適用情境

- **SSH 遠端伺服器**：無法在伺服器端直接開啟瀏覽器
- **CI/CD 環境**：初始化一次性長效 Token
- **Docker 容器**：容器內無 GUI 環境
- **公司網路限制**：埠 1455 被防火牆封鎖

---

## 相關文件

- [02-oauth-login-flow.md](02-oauth-login-flow.md) — 標準 PKCE 流程
- [04-token-management.md](04-token-management.md) — Token 儲存與刷新機制
