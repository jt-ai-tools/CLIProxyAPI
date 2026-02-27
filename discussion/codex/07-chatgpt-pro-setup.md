# 使用 ChatGPT Pro 帳號建立 Proxy 的實作指南

> 回到 [README](README.md) | 上一篇：[06-amp-module.md](06-amp-module.md)

---

## 前言

本文說明如何利用 **ChatGPT Pro 帳號**（或 ChatGPT Plus / OpenAI 帳號），透過 CLIProxyAPI 建立一個本地 API Proxy，讓 AI 工具（如 Cursor IDE、Claude Code、Amp CLI 等）可以使用你的訂閱額度。

---

## 方案 A：OAuth 登入方式（推薦）

此方式使用 OpenAI 官方 OAuth 流程，適合一般本地環境。

### 步驟 1：安裝與設定

```bash
# 下載或編譯 CLIProxyAPI
git clone https://github.com/router-for-me/CLIProxyAPI
cd CLIProxyAPI
go build -o cliproxy ./cmd/server
```

### 步驟 2：建立最小設定檔

```yaml
# config.yaml
port: 8080

# 設定本地 Proxy 的認證 Key（供 AI 工具連接用）
api-key:
  - "my-local-secret-key"

# Token 儲存目錄
auth-dir: "./auths"
```

### 步驟 3：以 ChatGPT Pro 帳號登入

```bash
# 啟動 OAuth 登入流程
./cliproxy login codex
```

此指令會：
1. 在本地啟動 Callback Server（埠 1455）
2. 自動開啟瀏覽器至 `https://auth.openai.com/oauth/authorize`
3. 使用你的 ChatGPT Pro 帳號登入並授權
4. 自動儲存 Token 至 `./auths/` 目錄

登入成功後你會看到：
```
Authentication saved to ./auths/codex_xxxxx.json
Codex authentication successful!
```

### 步驟 4：啟動 Proxy 伺服器

```bash
./cliproxy run --config config.yaml
```

### 步驟 5：設定你的 AI 工具

在 AI 工具（如 Cursor）的設定中，填入：

```
API Base URL: http://localhost:8080/v1
API Key: my-local-secret-key
```

---

## 方案 B：Device Code Flow（SSH / 伺服器環境）

適合無法直接開啟瀏覽器的環境。

### 步驟

```bash
# 在伺服器端執行
./cliproxy login codex --device
```

輸出類似：
```
Starting Codex device authentication...
Codex device URL: https://auth.openai.com/codex/device
Codex device code: ABCD-1234
```

在另一台裝置（手機或桌機）的瀏覽器中：
1. 開啟 `https://auth.openai.com/codex/device`
2. 輸入代碼 `ABCD-1234`
3. 用 ChatGPT Pro 帳號授權

授權後，伺服器端會自動取得 Token 並儲存。

---

## 方案 C：手動填寫 API Key（靜態方式）

如果你已經有 OpenAI API Key（從 `platform.openai.com` 取得），可以直接設定：

```yaml
# config.yaml
codex-api-key:
  - key: "sk-proj-xxxxxxxxxxxx"
    models:
      - "gpt-4o"
      - "gpt-4o-mini"
      - "o1"
      - "o3"
```

> **注意**：ChatGPT Pro 訂閱的帳號若要使用此方式，需確認你的帳號有 API 存取權限（API 帳單獨立於 ChatGPT Pro 訂閱）。

---

## 方案 D：Amp CLI 整合（使用 ampcode.com 的 ChatGPT Pro 額度）

如果你是 Amp CLI 的使用者，且 Amp 已連結你的 ChatGPT Pro 帳號：

### 步驟 1：登入 Amp

```bash
amp login
```

### 步驟 2：設定 CLIProxyAPI

```yaml
# config.yaml
port: 8080

api-key:
  - "my-local-secret-key"

ampcode:
  # Amp CLI 本地 secrets 會自動讀取（~/.local/share/amp/secrets.json）
  upstream-url: "https://ampcode.com"

  # 模型映射：讓本地無 Provider 的模型也能透過 Amp 使用
  model-mappings:
    - from: "claude-opus-4.5"
      to: "claude-sonnet-4"
```

### 步驟 3：啟動

```bash
./cliproxy run --config config.yaml
```

---

## 多帳號負載均衡

可以同時登入多個 ChatGPT Pro 帳號，實現額度輪詢：

```bash
# 登入帳號 1
./cliproxy login codex

# 登入帳號 2（會自動儲存為不同 Token 檔案）
./cliproxy login codex
```

設定路由策略：

```yaml
routing:
  strategy: "round-robin"  # 輪流使用每個帳號

# 配額用完自動切換
quota-exceeded:
  switch-project: true

# 失敗自動重試
request-retry: 3
```

---

## 完整設定範例

```yaml
# config.yaml（ChatGPT Pro Proxy 完整範例）
port: 8080
host: "127.0.0.1"  # 只允許本機存取

# 本地認證 Key（供 AI 工具連接使用）
api-key:
  - "your-local-secret-key-here"

# OAuth Token 儲存目錄
auth-dir: "./auths"

# 路由策略
routing:
  strategy: "round-robin"

# 配額管理
quota-exceeded:
  switch-project: true

request-retry: 3
max-retry-interval: 60

# 使用統計
usage-statistics-enabled: true

# 日誌
debug: false
logging-to-file: true
```

---

## 支援的模型（透過 ChatGPT Pro OAuth）

登入後可用的模型取決於你的訂閱等級：

| 模型 | 類別 | ChatGPT Pro 可用 |
|------|------|-----------------|
| `gpt-4o` | 旗艦對話 | ✅ |
| `gpt-4o-mini` | 輕量對話 | ✅ |
| `o1` | 推理模型 | ✅（Pro） |
| `o3` | 推理模型 | ✅（Pro） |
| `o3-mini` | 輕量推理 | ✅ |
| `codex-*` | 程式碼生成 | ✅ |

---

## 常見問題

### Q：Port 1455 被佔用

```
Error: port 1455 is already in use
```

解決：先結束佔用 1455 的程序，或使用 `--callback-port` 參數：

```bash
./cliproxy login codex --callback-port 2455
```

### Q：Token 過期怎麼辦

CLIProxyAPI 會自動刷新 Token（提前 5 天）。如果刷新失敗，重新執行 `login` 指令即可。

### Q：如何驗證登入成功

```bash
# 查看已儲存的 Token 檔案
ls ./auths/

# 測試 API
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer my-local-secret-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

### Q：SSH 遠端伺服器如何登入

方法 1：使用 Device Flow（推薦）

```bash
./cliproxy login codex --device
```

方法 2：SSH Port Forwarding

```bash
# 在本機執行（將遠端 1455 轉發到本機）
ssh -R 1455:localhost:1455 user@remote-server

# 然後在遠端伺服器執行
./cliproxy login codex
```

---

## 架構關係圖

```
你的帳號流
┌────────────────────────────────────────────────────────────────┐
│  ChatGPT Pro 帳號                                              │
│       │                                                        │
│       │ OAuth 2.0 + PKCE                                       │
│       ▼                                                        │
│  auth.openai.com ────→ Access Token ─→ ./auths/codex_xxx.json │
└────────────────────────────────────────────────────────────────┘

API 請求流
┌────────────────────────────────────────────────────────────────┐
│  AI 工具（Cursor / Claude Code）                               │
│       │  Authorization: Bearer my-local-secret-key             │
│       ▼                                                        │
│  CLIProxyAPI (localhost:8080)                                  │
│       │  讀取 ./auths/codex_xxx.json                           │
│       │  Authorization: Bearer <openai_access_token>           │
│       ▼                                                        │
│  api.openai.com                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 相關文件

- [01-overview.md](01-overview.md) — 系統架構概覽
- [02-oauth-login-flow.md](02-oauth-login-flow.md) — OAuth 流程詳解
- [03-device-flow.md](03-device-flow.md) — 無瀏覽器登入方式
- [04-token-management.md](04-token-management.md) — Token 管理詳解
- [05-proxy-architecture.md](05-proxy-architecture.md) — Proxy 架構詳解
