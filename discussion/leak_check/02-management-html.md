# 02 - Management Panel HTML 動態下載分析

## 風險等級：🔴 HIGH

---

## 概述

CLIProxyAPI 的管理面板 (`management.html`) 並非靜態打包在二進位檔案中，而是在**執行期間自動從外部 URL 下載並儲存在本地後提供服務**。此設計使管理面板的內容完全依賴外部伺服器，若外部伺服器被篡改，惡意 JavaScript 將在使用者瀏覽器中執行。

---

## 相關程式碼

**檔案：** `internal/managementasset/updater.go`

### 關鍵常數定義

```go
const (
    defaultManagementReleaseURL  = "https://api.github.com/repos/router-for-me/Cli-Proxy-API-Management-Center/releases/latest"
    defaultManagementFallbackURL = "https://cpamc.router-for.me/"
    managementAssetName          = "management.html"
    updateCheckInterval          = 3 * time.Hour  // 每 3 小時檢查一次更新
    managementSyncMinInterval    = 30 * time.Second
)
```

### 自動更新排程

```go
func StartAutoUpdater(ctx context.Context, configFilePath string) {
    // ...
    schedulerOnce.Do(func() {
        go runAutoUpdater(ctx)  // 背景 goroutine，永久執行
    })
}

func runAutoUpdater(ctx context.Context) {
    ticker := time.NewTicker(updateCheckInterval)  // 每 3 小時觸發
    defer ticker.Stop()

    runOnce()  // 啟動時立即執行一次

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            runOnce()  // 定期觸發
        }
    }
}
```

### Fallback 下載邏輯

當 GitHub releases 下載失敗且本地檔案不存在時，會從 `cpamc.router-for.me` fallback 下載：

```go
func ensureFallbackManagementHTML(ctx context.Context, client *http.Client, localPath string) bool {
    data, downloadedHash, err := downloadAsset(ctx, client, defaultManagementFallbackURL)
    // 直接從 https://cpamc.router-for.me/ 下載並存入本地
    if err = atomicWriteFile(localPath, data); err != nil { ... }
    log.Infof("management asset updated from fallback page successfully (hash=%s)", downloadedHash)
    return true
}
```

---

## SHA256 驗證機制分析

updater.go 確實有 SHA256 驗證機制：

```go
if remoteHash != "" && !strings.EqualFold(remoteHash, downloadedHash) {
    log.Warnf("remote digest mismatch for management asset: expected %s got %s", remoteHash, downloadedHash)
}
```

**但是，此驗證機制存在根本性的缺陷：**

- `remoteHash` 是從同一個 GitHub releases API 回應（JSON）中讀取的 `digest` 欄位
- 若攻擊者取得對 GitHub releases 的寫入權限，可同時替換惡意 `.html` 檔案 **和** 更新對應的 `digest` 值
- 因此，此驗證**無法防範源頭被篡改的供應鏈攻擊**
- 再者，Fallback URL (`cpamc.router-for.me`) 下載時**完全不做 Hash 驗證**

---

## 攻擊情境

### 情境 A：Fallback URL 被攻擊者控制

`cpamc.router-for.me` 是作者控制的自訂域名（非 GitHub 官方）。若此域名的 DNS、伺服器或 CDN 遭到攻擊：

1. CLIProxyAPI 在 GitHub releases 不可用時，從 `cpamc.router-for.me` 下載 HTML
2. 惡意 HTML 包含 JavaScript，偵測管理面板中儲存的認證資訊
3. 惡意 JS 呼叫 `POST /v0/management/api-call`，使用 `$TOKEN$` 將 Token 傳送至攻擊者伺服器

### 情境 B：GitHub Releases 帳號遭攻擊

若 `router-for-me` GitHub 帳號遭到入侵：

1. 攻擊者上傳惡意 `management.html` 至 `Cli-Proxy-API-Management-Center` releases
2. 同時更新 release 中的 `digest` 欄位以欺騙 Hash 驗證
3. 最多 3 小時後，所有使用預設 repository 設定的 CLIProxyAPI 實例自動更新至惡意版本

---

## 風險緩解現狀

| 緩解措施 | 現狀 |
|---------|------|
| SHA256 Hash 驗證 | ⚠️ 存在，但 hash 來源與下載來源相同，無法防範源頭篡改 |
| Fallback URL Hash 驗證 | ❌ 不存在 |
| 程式碼簽章驗證（Code Signing） | ❌ 不存在 |
| 允許停用自動更新 | ✅ `disable-control-panel: true` 可停用 |
| 允許自訂 repository | ✅ `panel_github_repository` 設定可覆寫 |
| 允許覆寫靜態檔案路徑 | ✅ `MANAGEMENT_STATIC_PATH` 環境變數可使用本地檔案 |

---

## 防護建議

1. **停用控制面板**（最保守）：在設定中設定 `disable-control-panel: true`
2. **使用自訂 repository**：將 `panel_github_repository` 設定為自己 Fork 的 repository，並親自審查程式碼後再 Release
3. **使用本地靜態檔案**：設定 `MANAGEMENT_STATIC_PATH` 指向自行審查過的本地 `management.html` 路徑，阻止所有自動下載
