# Proxy 伺服器架構與請求轉發機制

> 回到 [README](README.md) | 上一篇：[04-token-management.md](04-token-management.md) | 下一篇：[06-amp-module.md](06-amp-module.md)

---

## 1. 整體請求流程

```
下游 AI 工具（Cursor / Claude Code / Amp CLI）
      │
      │  POST /v1/chat/completions
      │  Authorization: Bearer <本地設定的 api-key>
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                 CLIProxyAPI (Gin HTTP Server)                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  中介層（Middleware）                                │   │
│  │  1. AuthMiddleware：驗證本地 api-key                 │   │
│  │  2. CORS 中介層                                     │   │
│  │  3. Request Logging                                 │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│  ┌───────────────────────▼─────────────────────────────┐   │
│  │  路由引擎                                            │   │
│  │  /v1/chat/completions → openai.Handler              │   │
│  │  /v1/messages         → claude.Handler              │   │
│  │  /v1/models           → models.Handler              │   │
│  └───────────────────────┬─────────────────────────────┘   │
│                          │                                  │
│  ┌───────────────────────▼─────────────────────────────┐   │
│  │  Access Manager（Provider 選擇器）                   │   │
│  │  - 讀取 auth-dir 中的 Codex Token                   │   │
│  │  - RoundRobin / FillFirst 策略選擇 Provider         │   │
│  │  - 自動刷新過期 Token                               │   │
│  └───────────────────────┬─────────────────────────────┘   │
└──────────────────────────┼──────────────────────────────────┘
                           │
              ┌────────────▼────────────┐
              │                         │
         ┌────▼─────┐           ┌───────▼──────┐
         │  OpenAI  │           │  ampcode.com  │
         │  API     │           │  (Amp Proxy)  │
         └──────────┘           └───────────────┘
```

---

## 2. 伺服器啟動

**檔案**：[`internal/api/server.go`](../../internal/api/server.go)

### 核心技術棧

| 元件 | 函式庫 |
|------|--------|
| HTTP 框架 | [Gin](https://github.com/gin-gonic/gin) |
| 反向 Proxy | Go 標準函式庫 `net/http/httputil.ReverseProxy` |
| 設定解析 | `gopkg.in/yaml.v3` |
| 日誌 | `github.com/sirupsen/logrus` |

### 支援的 AI Provider 路由

```go
// 路由在 server.go 中被依序注册
openai.Register(engine, baseHandler)    // OpenAI / GPT 相容路由
claude.Register(engine, baseHandler)    // Anthropic Claude 路由
gemini.Register(engine, baseHandler)    // Google Gemini 路由
ampModule.Register(moduleCtx)           // Amp 路由（含 proxy）
```

---

## 3. Codex Token 如何被注入請求

### Provider 的抽象設計

CLIProxyAPI 使用 `sdkaccess.Provider` 介面：

```
sdkaccess.Manager
    │
    ├── Provider 1: codex_user@example.com (CodexTokenStorage)
    ├── Provider 2: codex_work@company.com (CodexTokenStorage)
    └── Provider 3: gemini_key_xxx (GeminiKey)
```

每個 `Provider` 提供一個 `Get(ctx)` 方法，回傳可用的認證材料（Access Token / API Key）。

### 請求發出時

當下游工具呼叫 `/v1/chat/completions` 時：

1. `AuthMiddleware` 驗證本地 `api-key`（`Authorization: Bearer <本地key>`）
2. `Access Manager` 根據輪詢策略選擇可用的 Codex Provider
3. Provider 回傳 `access_token`（此即 OpenAI Access Token）
4. 請求帶著 `Authorization: Bearer <openai_access_token>` 被轉發到 OpenAI API

---

## 4. 多帳號 Provider 管理

**檔案**：[`internal/access/reconcile.go`](../../internal/access/reconcile.go)

```go
// ReconcileProviders 負責熱重載 Provider 列表
// 當設定更新時，只更新有變更的 Provider，避免影響進行中的請求
func ReconcileProviders(oldCfg, newCfg *config.Config, existing []sdkaccess.Provider) (
    result   []sdkaccess.Provider,
    added    []string,
    updated  []string,
    removed  []string,
    err      error,
) { ... }
```

### 設定格式（多帳號範例）

```yaml
# config.yaml

# 方式 1：手動 API Key
codex-api-key:
  - key: "sk-proj-xxxx"       # 帳號 1
  - key: "sk-proj-yyyy"       # 帳號 2

# 方式 2：OAuth Token（透過 login 指令取得後自動讀取）
auth-dir: "./auths"

# 方式 3：Amp 模式（透過 ampcode.com）
ampcode:
  upstream-api-key: "amp_xxxxxxx"
```

---

## 5. 路由策略

**設定**：

```yaml
routing:
  strategy: "round-robin"  # 或 "fill-first"
```

| 策略 | 行為 |
|------|------|
| `round-robin`（預設） | 輪流使用每個帳號 |
| `fill-first` | 優先用第一個帳號，額度用完才換下一個 |

---

## 6. 請求重試與冷卻

```yaml
request-retry: 3          # 請求失敗最多重試 3 次
max-retry-interval: 60    # 冷卻最大等待時間（秒）

quota-exceeded:
  switch-project: true     # 配額用完自動切換帳號

disable-cooling: false     # 啟用冷卻排程
```

當某一個 Provider 的配額被用完（429 Too Many Requests），Access Manager 會自動：
1. 將該 Provider 標記為「冷卻中」
2. 切換到下一個可用 Provider
3. 在冷卻期結束後自動恢復使用

---

## 7. 熱重載機制

`server.go` 監聽設定檔案變更，觸發時呼叫：

```go
func (s *Server) Reload(newCfg *config.Config) error {
    // 1. 比較新舊設定的 YAML 差異
    // 2. 只更新有變更的部分
    // 3. 更新 Access Manager（Provider 列表）
    // 4. 更新 Amp 模組設定
    // 5. 不重啟 HTTP 伺服器，零停機更新
}
```

---

## 8. 代理 Header 安全處理

在 Amp Proxy 模式中（`proxy.go`），CLIProxyAPI 特別處理了 Header：

```go
proxy.Director = func(req *http.Request) {
    // 1. 移除客戶端的 Authorization Header
    //    避免洩漏本地的認證 Key 給 ampcode.com
    req.Header.Del("Authorization")
    req.Header.Del("X-Api-Key")
    req.Header.Del("X-Goog-Api-Key")

    // 2. 移除 URL 中的認證參數（防止憑證洩漏）
    removeQueryValuesMatching(req, "key", clientKey)
    removeQueryValuesMatching(req, "auth_token", clientKey)

    // 3. 注入正確的上游 API Key
    if key, _ := secretSource.Get(req.Context()); key != "" {
        req.Header.Set("X-Api-Key", key)
        req.Header.Set("Authorization", "Bearer "+key)
    }
}
```

---

## 相關文件

- [04-token-management.md](04-token-management.md) — Token 如何取得與刷新
- [06-amp-module.md](06-amp-module.md) — Amp 模組深度分析
- [07-chatgpt-pro-setup.md](07-chatgpt-pro-setup.md) — 完整設定指南
