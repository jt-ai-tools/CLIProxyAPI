# Amp 路由模組：模型映射與智慧路由策略

> 回到 [README](README.md) | 上一篇：[05-proxy-architecture.md](05-proxy-architecture.md) | 下一篇：[07-chatgpt-pro-setup.md](07-chatgpt-pro-setup.md)

---

## 1. Amp 模組是什麼？

**Amp**（`ampcode.com`）是一個 AI 開發工具平台，其 CLI（`amp`）向 API 發出類似 OpenAI / Anthropic / Gemini 標準格式的請求。

`AmpModule` 的職責是：

1. **提供者別名路由**（Provider Alias Routes）：將 Amp CLI 的特殊路徑格式轉換為 CLIProxyAPI 的標準路徑。
2. **模型映射**（Model Mapping）：當請求的模型本地沒有 Provider 時，映射到另一個可用的模型。
3. **智慧路由降級**（Fallback）：如果本地和映射都找不到 Provider，則轉發到 `ampcode.com`（消耗 Amp Credits）。
4. **管理路由代理**（Management Proxy）：代理 `/api/user`、`/api/threads` 等 Amp 管理 API 到上游。

---

## 2. 路由決策流程

```
Amp CLI 請求（例：POST /api/provider/openai/v1/chat/completions）
      │
      ▼
┌─────────────────────────────────────────────────────┐
│  FallbackHandler.Wrap(handler)                      │
│                                                     │
│  1. 解析請求中的 model 名稱                          │
│  2. 查詢本地是否有 Provider（util.GetProviderName）  │
│                                                     │
│  if 本地有 Provider：                               │
│      → RouteTypeLocalProvider                       │
│        直接交給本地 handler 處理（免費）              │
│                                                     │
│  else if 有匹配的 ModelMapping：                    │
│      → RouteTypeModelMapping                        │
│        重寫 model 名稱，交給本地 handler（免費）     │
│                                                     │
│  else if 有 Amp upstream 可用：                     │
│      → RouteTypeAmpCredits                          │
│        轉發請求到 ampcode.com（消耗 Amp Credits）    │
│                                                     │
│  else：                                             │
│      → RouteTypeNoProvider                          │
│        回傳 503                                     │
└─────────────────────────────────────────────────────┘
```

---

## 3. Provider 別名路由

**檔案**：[`internal/api/modules/amp/routes.go`](../../internal/api/modules/amp/routes.go)

Amp CLI 使用非標準路徑格式：

| Amp CLI 路徑格式 | 等效的標準路徑 |
|-----------------|--------------|
| `/api/provider/openai/v1/chat/completions` | `/v1/chat/completions` |
| `/api/provider/anthropic/v1/messages` | `/v1/messages` |
| `/publishers/google/models/<model>:streamGenerateContent` | `/models/<model>:streamGenerateContent` |

CLIProxyAPI 透過路由別名，直接將這些路徑對應到已有的 Handler，不重複實作邏輯。

---

## 4. 模型映射機制

**檔案**：[`internal/api/modules/amp/model_mapping.go`](../../internal/api/modules/amp/model_mapping.go)

### 設定範例

```yaml
ampcode:
  model-mappings:
    # 精確匹配：將 claude-opus-4.5 映射到 claude-sonnet-4
    - from: "claude-opus-4.5"
      to: "claude-sonnet-4"

    # 正規表達式匹配
    - from: "gpt-4.*"
      to: "gemini-2.5-pro"
      regex: true

  # 強制優先使用 model-mappings（即使本地有 Provider）
  force-model-mappings: false
```

### 映射優先順序

```
1. 精確匹配（exact match，不區分大小寫）
2. 正規表達式匹配（regex，依設定順序評估）
3. 無匹配 → 回傳 ""（使用降級策略）
```

### Thinking Suffix 保留

模型名稱中可能包含思考預算後綴，例如 `gemini-2.5-pro(8192)`：

```go
// 解析後綴
requestResult := thinking.ParseSuffix("claude-sonnet-4(16000)")
// requestResult.ModelName = "claude-sonnet-4"
// requestResult.HasSuffix = true
// requestResult.BudgetValue = 16000

// 映射後保留後綴（除非 to 本身已有後綴）
```

---

## 5. Gemini Bridge Handler

**檔案**：[`internal/api/modules/amp/gemini_bridge.go`](../../internal/api/modules/amp/gemini_bridge.go)

Amp CLI 呼叫 Gemini 時使用 Google 特有路徑格式：

```
/publishers/google/models/gemini-2.5-pro:streamGenerateContent
```

`createGeminiBridgeHandler` 重寫 URL 參數為：

```
:action = "gemini-2.5-pro:streamGenerateContent"
```

這樣標準的 Gemini Handler（期望 `:action` 參數）就能正常處理。

---

## 6. Amp API Key 的多來源讀取

**檔案**：[`internal/api/modules/amp/secret.go`](../../internal/api/modules/amp/secret.go)

```go
// MultiSourceSecret 實作了三層來源優先順序：
// 1. 設定檔中的 upstream-api-key（最高優先）
// 2. 環境變數 AMP_API_KEY
// 3. Amp 本機 secrets 檔案（~/.local/share/amp/secrets.json）
```

### Amp 本機 secrets 格式

```json
{
    "apiKey@https://ampcode.com/": "amp_xxxxxxxxxx"
}
```

此格式**與 Amp CLI 共用 secrets 檔案**，因此使用者只需登入一次 Amp，CLIProxyAPI 就能自動讀取 Key。

### 多客戶端 Key 映射

進階使用：不同的本地 `api-key` 可以映射到不同的 Amp 上游 Key：

```yaml
ampcode:
  upstream-api-keys:
    - upstream-api-key: "amp_team_key_xxx"
      api-keys:
        - "local-key-for-alice"
        - "local-key-for-bob"
    - upstream-api-key: "amp_personal_key_yyy"
      api-keys:
        - "local-key-for-charlie"
```

---

## 7. 管理路由安全設計

**檔案**：[`internal/api/modules/amp/routes.go`](../../internal/api/modules/amp/routes.go)

管理路由（`/api/user`, `/api/threads` 等）有額外的安全保護：

```go
// 1. 停用 CORS（防止瀏覽器 cross-origin 攻擊）
noCORSMiddleware()

// 2. 可選的 localhost-only 限制
localhostOnlyMiddleware()
// 使用 RemoteAddr 而非 X-Forwarded-For，防止 Header 偽造

// 3. 驗證本地 API Key
authMiddleware()
```

### Localhost 限制設定

```yaml
ampcode:
  restrict-management-to-localhost: true
```

---

## 8. 熱重載支援

`AmpModule` 實作了 `OnConfigUpdated()` 方法，支援以下項目的熱重載（無需重啟）：

| 設定項 | 效果 |
|--------|------|
| `model-mappings` | 即時更新模型映射規則 |
| `upstream-api-key` | 即時更換 Amp 上游 Key |
| `upstream-url` | 即時切換上游 URL |
| `restrict-management-to-localhost` | 即時開關 localhost 限制 |

---

## 相關文件

- [05-proxy-architecture.md](05-proxy-architecture.md) — Proxy 詳細架構
- [07-chatgpt-pro-setup.md](07-chatgpt-pro-setup.md) — 設定 ChatGPT Pro Proxy
