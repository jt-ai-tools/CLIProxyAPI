# 06 - 具體修復建議

本文件針對安全稽核中發現的各項風險，提供具體的修復建議與緩解措施，依優先級排列。

---

## 🔴 優先級 1：Management Panel HTML 安全強化

### 問題

`management.html` 從 `cpamc.router-for.me`（fallback）和 GitHub releases 下載，SHA256 驗證的雜湊值來自同一來源，無法防範源頭被篡改的情境。

### 建議方案一：內嵌固定 Hash（最強防護）

在二進位檔案建構時，將受信任版本的 `management.html` SHA256 雜湊值直接編譯進程式碼（類似 Subresource Integrity）：

```go
// buildinfo/buildinfo.go 或獨立檔案
const TrustedManagementHTMLHash = "sha256:abc123..." // 建構時由 CI/CD 注入

// updater.go 中驗證
if downloadedHash != buildinfo.TrustedManagementHTMLHash {
    log.Warnf("management.html hash mismatch! Expected %s, got %s", 
        buildinfo.TrustedManagementHTMLHash, downloadedHash)
    // 拒絕使用此版本或至少提示使用者
}
```

### 建議方案二：對 Fallback URL 同樣進行 Hash 驗證

目前 `ensureFallbackManagementHTML` 函式下載後不做任何 Hash 驗證，應加入：

```go
func ensureFallbackManagementHTML(ctx context.Context, client *http.Client, localPath string) bool {
    data, downloadedHash, err := downloadAsset(ctx, client, defaultManagementFallbackURL)
    // ...
    // 加入：與已知安全的 Hash 做比對，或記錄警告但繼續
    log.Warnf("Using fallback management HTML (hash=%s) - review this file", downloadedHash)
    // ...
}
```

### 緩解方案（立即可用）

**使用本地靜態檔案，完全繞過自動下載：**

```yaml
# config.yaml 設定
remote-management:
  disable-control-panel: true  # 停用管理面板（最保守）
```

或設定環境變數使用本地已審查的 HTML 檔：

```bash
export MANAGEMENT_STATIC_PATH=/path/to/your/trusted/management.html
```

---

## 🔴 優先級 2：APICall 端點 URL 白名單

### 問題

`POST /v0/management/api-call` 允許向任意 URL 傳送含有 `$TOKEN$` 的請求，可造成 SSRF 及 Token 外洩。

### 建議方案：加入 URL 白名單設定

在設定檔中加入允許的目標域名清單：

```yaml
# config.yaml 建議新增
remote-management:
  api-call-allowed-hosts:
    - "api.openai.com"
    - "api.anthropic.com"
    - "generativelanguage.googleapis.com"
    # 不設定則使用預設：僅允許已知 AI API 域名
```

在 `api_tools.go` 中驗證：

```go
// 檢查 URL 是否在允許清單中
if !isAllowedHost(parsedURL.Host, cfg.AllowedHosts) {
    c.JSON(http.StatusForbidden, gin.H{"error": "target host not in allowed list"})
    return
}
```

### 緩解方案（立即可用）

確保管理 API 端點僅能從本地存取（`127.0.0.1`），並設定強固的管理 Key（建議 32 字元以上隨機字元）。

---

## 🟡 優先級 3：Request Log 中的 apiRequest 遮罩

### 問題

`writeAPISection` 函式直接將 `apiRequest`（含完整 `Authorization: Bearer <token>` header）寫入日誌，未套用任何遮罩。

### 建議修復

在 `internal/logging/request_logger.go` 的 `writeNonStreamingLog` 函式中，寫入 `apiRequest` 前先對 Header 進行遮罩：

```go
// 修改前：
if errWrite := writeAPISection(w, "=== API REQUEST ===\n", "=== API REQUEST", apiRequest, time.Time{}); errWrite != nil {
    return errWrite
}

// 修改後：
maskedAPIRequest := maskAPIRequestSensitiveHeaders(apiRequest)
if errWrite := writeAPISection(w, "=== API REQUEST ===\n", "=== API REQUEST", maskedAPIRequest, time.Time{}); errWrite != nil {
    return errWrite
}
```

需要新增的輔助函式（示意）：

```go
// maskAPIRequestSensitiveHeaders 對 HTTP 請求原始數據中的敏感 Header 進行遮罩
func maskAPIRequestSensitiveHeaders(rawRequest []byte) []byte {
    // 逐行處理，格式為 "Header-Name: value"
    lines := bytes.Split(rawRequest, []byte("\n"))
    for i, line := range lines {
        if colonIdx := bytes.IndexByte(line, ':'); colonIdx > 0 {
            headerName := string(bytes.TrimSpace(line[:colonIdx]))
            headerValue := string(bytes.TrimSpace(line[colonIdx+1:]))
            masked := util.MaskSensitiveHeaderValue(headerName, headerValue)
            lines[i] = []byte(fmt.Sprintf("%s: %s", headerName, masked))
        }
    }
    return bytes.Join(lines, []byte("\n"))
}
```

---

## 🟡 優先級 4：文件改善

### 問題

遠端 Token Store 設定（`GITSTORE_GIT_URL`、`OBJECTSTORE_*`）的安全風險在目前文件中說明不足，使用者可能在不了解風險的情況下錯誤設定。

### 建議

在 `README.md` 和相關文件中明確加入安全警示：

```markdown
> ⚠️ **安全警示**：`GITSTORE_GIT_URL` / `OBJECTSTORE_ENDPOINT` 環境變數設定後，
> 所有認證 Token 將自動同步至指定的遠端儲存。請確認目標 URL 指向您完全控制
> 且安全的服務，**切勿**將這些環境變數設定為不受信任的第三方服務。
```

---

## 整體安全加固建議

| 措施 | 類型 | 說明 |
|------|------|------|
| 停用管理面板（若不需要） | 設定 | `disable-control-panel: true` |
| 使用本地 management.html | 設定 | `MANAGEMENT_STATIC_PATH=/path/to/local/file` |
| 設定強固管理 Key | 設定 | 使用 32 字元以上隨機字元 |
| 限制管理 API 存取 | 網路 | 確保管理端點只在 `127.0.0.1` 監聽 |
| 不啟用 request-log | 設定 | 預設已關閉，不需主動設定 |
| 謹慎設定遠端 Token Store | 設定 | 確認 URL 指向受信任的服務 |
| 定期稽核 management.html | 維護 | 若有啟用管理面板，定期審查下載的 HTML |
