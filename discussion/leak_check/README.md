# CLIProxyAPI Token 洩漏風險檢查報告

## 報告概述

本報告針對 `router-for-me/CLIProxyAPI` 專案進行全面的 Token 洩漏安全稽核，分析專案是否存在「偷偷將使用者 Token 傳送至第三方」的機制或潛在風險路徑。

**稽核日期：** 2025 年  
**稽核版本：** `go.mod` 中 `module github.com/router-for-me/CLIProxyAPI/v6`  
**稽核人員：** AI 自動程式碼分析

---

## 整體結論

> **本專案不存在任何預設的主動 Token 外傳機制。**
> 
> 專案沒有遙測（telemetry）、分析（analytics）或任何「背景偷偷上傳資料」的程式碼。然而，存在若干設計層面的風險，在特定前提條件下可能導致 Token 間接洩漏。

---

## 風險等級摘要

| # | 風險項目 | 嚴重度 | 前提條件 | 詳細文件 |
|---|----------|--------|---------|---------|
| 1 | Management Panel HTML 動態從第三方 URL 下載並執行 | 🔴 HIGH | 預設啟用，需攻擊者控制 `cpamc.router-for.me` 或 GitHub releases | [02-management-html.md](02-management-html.md) |
| 2 | APICall 端點支援 `$TOKEN$` 注入至任意 URL | 🔴 HIGH | 需取得管理 Key（management key）| [03-api-call-endpoint.md](03-api-call-endpoint.md) |
| 3 | Request Log 將完整 Bearer Token 寫入本地日誌 | 🟡 MEDIUM | 需使用者主動啟用 `request-log: true` | [04-request-log.md](04-request-log.md) |
| 4 | Git / S3 Token Store 可設定同步至遠端 | 🟡 MEDIUM | 需使用者主動設定 `GITSTORE_GIT_URL` / `OBJECTSTORE_*` | [05-token-store.md](05-token-store.md) |
| 5 | 無主動遙測或分析機制 | 🟢 SAFE | — | 本文件 |
| 6 | AMP Proxy Header 正確清理 | 🟢 SAFE | — | 本文件 |
| 7 | Request Log Header 遮罩正確實作 | 🟢 SAFE | 前提：不啟用 `request-log: true` | [04-request-log.md](04-request-log.md) |

---

## 確認安全的機制

### ✅ 無遙測 / 分析 / 追蹤機制

透過搜尋以下關鍵字確認專案中**完全不存在**遙測回傳：

- `telemetry`, `analytics`, `beacon`, `track`
- `sentry`, `datadog`, `mixpanel`, `amplitude`
- 任何定期向第三方上傳使用資料的程式碼

### ✅ AMP Proxy 正確移除認證 Header

`internal/api/modules/amp/proxy.go` 在轉發請求至 `ampcode.com` 前，明確移除客戶端的 `Authorization`、`X-Api-Key`、`X-Goog-Api-Key` header，並注入正確的上游金鑰，不會將使用者 Token 傳遞至上游。

### ✅ Request Headers 的遮罩機制

`internal/util/provider.go` 中的 `MaskSensitiveHeaderValue()` 函式正確遮罩以下 header：
- `Authorization`
- `api-key`
- `token`
- `secret`

此遮罩套用於來自用戶端的 Request Headers 日誌記錄，確保日誌中不會出現明文 Token。（注意：此遮罩**不套用**於 `apiRequest` 段落，詳見 [04-request-log.md](04-request-log.md)）

---

## 修復建議摘要

詳細修復建議請參閱 [06-recommendations.md](06-recommendations.md)，重點摘要如下：

1. **【最優先】** 對 `management.html` 建立獨立於下載來源的 SHA256 或程式碼簽章驗證機制
2. **【建議】** 對 `APICall` 端點加入 URL 白名單或允許使用者明確設定可存取的域名範圍
3. **【建議】** 在 `writeAPISection` 中對 `apiRequest` 套用 Header 遮罩
4. **【資訊】** 在文件中清楚說明 `GITSTORE_GIT_URL` 和 `OBJECTSTORE_*` 設定的安全風險

---

## 文件索引

| 文件 | 說明 |
|------|------|
| [01-risk-summary.md](01-risk-summary.md) | 整體風險評估與攻擊路徑分析 |
| [02-management-html.md](02-management-html.md) | Management Panel HTML 動態下載詳細分析 |
| [03-api-call-endpoint.md](03-api-call-endpoint.md) | APICall 端點 `$TOKEN$` 注入路徑分析 |
| [04-request-log.md](04-request-log.md) | Request Log 中 Token 洩漏至本地日誌分析 |
| [05-token-store.md](05-token-store.md) | Git / S3 Token Store 配置風險分析 |
| [06-recommendations.md](06-recommendations.md) | 具體修復建議 |
