# 系統整體架構概覽

> 回到 [README](README.md)

---

## 1. 設計目標

CLIProxyAPI 的核心目標是：**讓使用者以 ChatGPT Pro / OpenAI 帳號提供的訂閱額度，透過標準 OpenAI-compatible API 格式，供任意 AI 工具（Cursor、Claude Code、Amp CLI 等）呼叫**。

它不是簡單地「轉發 API Key」，而是完整實作了：

- OpenAI OAuth 2.0 PKCE 登入流程（取得 Token）
- Token 自動刷新與持久化
- 多帳號、多 Provider 的輪詢與容錯
- 熱重載設定（不中斷服務更新設定）

---

## 2. 專案目錄結構（Codex 相關部分）

```
CLIProxyAPI/
├── internal/
│   ├── auth/codex/           # OAuth 核心：PKCE、Token 交換、JWT 解析
│   │   ├── openai_auth.go    # CodexAuth：生成授權 URL、交換 Token、刷新 Token
│   │   ├── oauth_server.go   # 本地 OAuth Callback Server（埠 1455）
│   │   ├── token.go          # CodexTokenStorage：Token 序列化與持久化
│   │   ├── jwt_parser.go     # JWT Claims 解析（讀取 email、account_id）
│   │   ├── pkce.go           # PKCE Code Verifier / Challenge 生成（RFC 7636）
│   │   ├── openai.go         # 資料結構定義（PKCECodes、CodexTokenData、CodexAuthBundle）
│   │   └── errors.go         # 認證錯誤類型定義
│   │
│   ├── api/
│   │   └── modules/amp/      # Amp 路由模組（含 Proxy 轉發）
│   │       ├── amp.go        # AmpModule：主模組、熱重載
│   │       ├── routes.go     # 路由註冊：管理路由、Provider 別名路由
│   │       ├── proxy.go      # ReverseProxy：請求轉發至 ampcode.com
│   │       ├── secret.go     # SecretSource：Amp API Key 多來源讀取
│   │       ├── model_mapping.go  # 模型名稱映射（exact / regex）
│   │       └── fallback_handlers.go  # FallbackHandler：本地 → ampcode.com 降級
│   │
│   ├── cmd/
│   │   ├── openai_login.go        # `cliproxy login codex` 指令
│   │   └── openai_device_login.go # `cliproxy login codex --device` 指令
│   │
│   └── config/config.go      # CodexKey、AmpCode 設定結構
│
├── sdk/
│   └── auth/
│       ├── codex.go          # CodexAuthenticator：SDK 協調層（PKCE 流程）
│       └── codex_device.go   # Device Code Flow 實作
└── ...
```

---

## 3. 程式分層架構

```
┌─────────────────────────────────────────────────────┐
│  Layer 4: CLI Entry Point                           │
│  internal/cmd/openai_login.go                       │
│  internal/cmd/openai_device_login.go                │
└───────────────────┬─────────────────────────────────┘
                    │ 呼叫
┌───────────────────▼─────────────────────────────────┐
│  Layer 3: SDK Auth Manager                          │
│  sdk/auth/manager.go                               │
│  sdk/auth/codex.go  (CodexAuthenticator)           │
│  sdk/auth/codex_device.go  (Device Flow)           │
└───────────────────┬─────────────────────────────────┘
                    │ 呼叫
┌───────────────────▼─────────────────────────────────┐
│  Layer 2: OAuth 核心                                │
│  internal/auth/codex/openai_auth.go  (CodexAuth)   │
│  internal/auth/codex/oauth_server.go (Local Server)│
│  internal/auth/codex/pkce.go  (PKCE)               │
│  internal/auth/codex/jwt_parser.go  (JWT)          │
└───────────────────┬─────────────────────────────────┘
                    │ HTTP 請求
┌───────────────────▼─────────────────────────────────┐
│  Layer 1: OpenAI Auth API                           │
│  https://auth.openai.com/oauth/authorize            │
│  https://auth.openai.com/oauth/token                │
└─────────────────────────────────────────────────────┘
```

---

## 4. 兩種運作模式

### 模式 A：Codex 直接 API Key 模式（`codex-api-key`）

使用者在 `config.yaml` 中手動填寫從 OpenAI 取得的 API Key：

```yaml
codex-api-key:
  - key: "sk-proj-xxxx"
```

適合已有固定 API Key 的情境。

### 模式 B：OAuth 登入模式（`login codex`）

使用者執行 `cliproxy login codex` 指令：
1. 本地啟動 Callback Server（埠 1455）
2. 開啟瀏覽器導向 `auth.openai.com/oauth/authorize`
3. 使用者用 ChatGPT Pro 帳號登入
4. 取得 Token，自動儲存至 `auth-dir`

呼叫 AI API 時，系統自動以此 Token 認證。詳見 [02-oauth-login-flow.md](02-oauth-login-flow.md)。

### 模式 C：Amp Proxy 模式（`ampcode` 設定）

系統作為 Amp CLI 的上游 Proxy，支援路由至 `ampcode.com` 並使用 Amp Credits。
詳見 [06-amp-module.md](06-amp-module.md)。

---

## 5. 關鍵常數

| 常數 | 值 | 說明 |
|------|-----|------|
| `AuthURL` | `https://auth.openai.com/oauth/authorize` | OAuth 授權端點 |
| `TokenURL` | `https://auth.openai.com/oauth/token` | Token 交換端點 |
| `ClientID` | `app_EMoamEEZ73f0CkXaXp7hrann` | Codex CLI 的 OAuth Client ID |
| `RedirectURI` | `http://localhost:1455/auth/callback` | 本地 Callback 端點 |

> 注意：`ClientID` 是 OpenAI Codex CLI 的公開 Client ID，這使得 CLIProxyAPI 能夠假冒 Codex CLI 完成授權流程。

---

## 相關文件

- [02-oauth-login-flow.md](02-oauth-login-flow.md) — 詳細的 OAuth PKCE 流程
- [03-device-flow.md](03-device-flow.md) — 無瀏覽器的 Device Code Flow
- [04-token-management.md](04-token-management.md) — Token 儲存與刷新
- [05-proxy-architecture.md](05-proxy-architecture.md) — Proxy 請求轉發
- [06-amp-module.md](06-amp-module.md) — Amp 模組路由策略
- [07-chatgpt-pro-setup.md](07-chatgpt-pro-setup.md) — 實作指南
