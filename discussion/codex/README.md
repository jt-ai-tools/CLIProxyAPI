# CLIProxyAPI × Codex 架構分析

> 本系列文件詳細說明 CLIProxyAPI 如何透過 OpenAI OAuth 認證機制，讓使用者以 ChatGPT Pro 帳號建立本地 AI API Proxy。

---

## 目錄

| 文件 | 說明 |
|------|----|
| [01-overview.md](01-overview.md) | 系統整體架構概覽與核心設計哲學 |
| [02-oauth-login-flow.md](02-oauth-login-flow.md) | OAuth 2.0 + PKCE 登入流程深度分析 |
| [03-device-flow.md](03-device-flow.md) | Device Code Flow（無瀏覽器環境）流程分析 |
| [04-token-management.md](04-token-management.md) | Token 儲存、刷新與生命週期管理 |
| [05-proxy-architecture.md](05-proxy-architecture.md) | Proxy 伺服器架構與請求轉發機制 |
| [06-amp-module.md](06-amp-module.md) | Amp 路由模組：模型映射與智慧路由策略 |
| [07-chatgpt-pro-setup.md](07-chatgpt-pro-setup.md) | 使用 ChatGPT Pro 帳號建立 Proxy 的實作指南 |

---

## 快速概念圖

```
┌─────────────────────────────────────────────────────────────────┐
│                      CLIProxyAPI 全局視圖                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   你的 AI 工具 (Claude Code / Cursor / Amp CLI)                  │
│        │  OpenAI-compatible API 請求                             │
│        ▼                                                         │
│   ┌──────────────────────────────────┐                          │
│   │  CLIProxyAPI (本地 Proxy Server) │                          │
│   │  ┌─────────────┐ ┌────────────┐ │                          │
│   │  │ Auth 中介層 │ │ 路由引擎   │ │                          │
│   │  └─────────────┘ └────────────┘ │                          │
│   └──────────────────────────────────┘                          │
│        │                  │                                      │
│        ▼                  ▼                                      │
│   OpenAI API          ampcode.com                               │
│   (codex auth)        (Amp upstream)                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 核心關鍵點

1. **認證層**：使用 OpenAI 官方 OAuth 2.0 + PKCE 流程，取得與 ChatGPT Pro 帳號綁定的 Access Token。
2. **Token 轉換**：OAuth Access Token → OpenAI API Key，讓下游工具直接以 API Key 模式呼叫。
3. **智慧路由**：`AmpModule` 負責模型映射，可將找不到本地 Provider 的請求自動轉發至 `ampcode.com`。
4. **熱重載**：設定變更無需重啟服務即可生效。

---

## 與其他代碼路徑的關係

```
internal/auth/codex/      ← OAuth 認證核心邏輯
sdk/auth/codex.go         ← SDK 認證協調層
internal/cmd/openai_*.go  ← CLI 登入指令入口
internal/api/modules/amp/ ← Amp 路由模組（含 Proxy）
internal/config/          ← 設定結構（CodexKey, AmpCode…）
```
