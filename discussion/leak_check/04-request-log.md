# 04 - Request Log 中 Token 洩漏至本地日誌分析

## 風險等級：🟡 MEDIUM（需主動啟用 `request-log: true`）

---

## 概述

CLIProxyAPI 提供可選的請求日誌功能（`request-log: true`）。日誌系統對來自用戶端的 Request Headers 有正確的遮罩保護，但對於「實際送往上游 AI API 的請求」（`apiRequest`）則**直接寫入日誌，不做任何遮罩**。這意味著當啟用 `request-log` 時，含有完整 `Authorization: Bearer <token>` 的上游請求 Header 會被明文寫入本地日誌檔案。

---

## 相關程式碼

**檔案：** `internal/logging/request_logger.go`

### 日誌寫入邏輯

```go
func (l *FileRequestLogger) writeNonStreamingLog(
    w io.Writer,
    // ...
    requestHeaders map[string][]string,  // 來自用戶端的 Headers（有遮罩）
    // ...
    apiRequest []byte,                   // 實際送往上游的請求（無遮罩）
    // ...
) error {
    // ✅ 用戶端 Headers：透過 writeRequestInfoWithBody 寫入，有遮罩保護
    if errWrite := writeRequestInfoWithBody(w, url, method, requestHeaders, ...); errWrite != nil {
        return errWrite
    }
    
    // ❌ 上游 API 請求：透過 writeAPISection 寫入，無遮罩
    if errWrite := writeAPISection(w, "=== API REQUEST ===\n", "=== API REQUEST", apiRequest, time.Time{}); errWrite != nil {
        return errWrite
    }
    // ...
}
```

### 用戶端 Headers 的遮罩（正確保護的部分）

```go
// writeRequestInfoWithBody 中：
for key, values := range headers {
    for _, value := range values {
        masked := util.MaskSensitiveHeaderValue(key, value)  // ✅ 有遮罩
        if _, errWrite := io.WriteString(w, fmt.Sprintf("%s: %s\n", key, masked)); errWrite != nil {
            return errWrite
        }
    }
}
```

### 上游 API 請求的直接寫入（未遮罩的部分）

```go
// writeAPISection 中：
if _, errWrite := w.Write(payload); errWrite != nil {  // ❌ 直接寫入，無遮罩
    return errWrite
}
```

---

## apiRequest 的內容說明

`apiRequest` 是透過 `c.Get("API_REQUEST")` 從 Gin context 取得的，由各 Auth handler 在轉發上游 API 請求前設定（`internal/api/middleware/response_writer.go`）。

其內容通常為轉送至上游 AI 服務的完整 HTTP 請求，包含：

```
=== API REQUEST ===
POST https://api.openai.com/v1/chat/completions HTTP/1.1
Authorization: Bearer sk-proj-xxxxxxxx...（完整 Token，明文）
Content-Type: application/json
...
```

---

## 遮罩函式分析

**檔案：** `internal/util/provider.go`

```go
func MaskSensitiveHeaderValue(key, value string) string {
    lk := strings.ToLower(key)
    // 遮罩的 header 名稱：包含 authorization, api-key, token, secret
    if strings.Contains(lk, "authorization") ||
        strings.Contains(lk, "api-key") ||
        strings.Contains(lk, "token") ||
        strings.Contains(lk, "secret") {
        return maskValue(value)
    }
    return value
}
```

此函式本身實作正確，問題在於 `writeAPISection` 根本**未呼叫**此函式。

---

## 影響範圍

| 項目 | 說明 |
|------|------|
| 前提條件 | 使用者需在設定檔中明確設定 `request-log: true` |
| 洩漏位置 | 本地日誌目錄（非傳送至遠端） |
| 洩漏內容 | 送往上游 AI API 的完整 Bearer Token |
| 風險情境 | 日誌檔案被惡意程式讀取、同機器其他使用者讀取、日誌被意外包含在備份中上傳 |

---

## 防護建議

**對使用者：**
- 若非必要，不要啟用 `request-log: true`
- 若需要啟用，確保日誌目錄僅有執行 CLIProxyAPI 的使用者可讀取（`chmod 700`）

**對開發者（修復建議）：**

在 `writeAPISection` 呼叫前，先對 `apiRequest` 的 Headers 進行遮罩處理。或在 `writeAPISection` 函式中加入 Header 遮罩邏輯：

```go
// 建議修復（示意）：
maskedAPIRequest := maskAPIRequestHeaders(apiRequest)
if errWrite := writeAPISection(w, "=== API REQUEST ===\n", "=== API REQUEST", maskedAPIRequest, time.Time{}); errWrite != nil {
    return errWrite
}
```
