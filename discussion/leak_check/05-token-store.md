# 05 - Git / S3 / PostgreSQL Token Store 配置風險分析

## 風險等級：🟡 MEDIUM（需使用者主動設定）

---

## 概述

CLIProxyAPI 支援三種選擇性的遠端 Token Store 後端，可將所有認證資訊（Token、API Key、Cookie 等）同步至遠端儲存系統：

1. **Git Store** — 透過 `GITSTORE_GIT_URL` 設定，將 Token 以 JSON 檔案形式 commit 並 push 至 Git repository
2. **Object Store (S3)** — 透過 `OBJECTSTORE_*` 環境變數設定，將 Token 上傳至 S3 相容的 Object Storage
3. **PostgreSQL Store** — 透過 PostgreSQL 連線字串設定，將 Token 同步至 PostgreSQL 資料庫

**重要：這些功能都需要使用者主動設定才會啟用，預設情況下不會同步至遠端。**

---

## Git Store 分析

**檔案：** `internal/store/gitstore.go`

### 設定方式

根據 `cmd/run.go` 和設定檔處理邏輯：
- 環境變數 `GITSTORE_GIT_URL` — Git remote URL
- 環境變數 `GITSTORE_GIT_USERNAME` — Git 認證使用者名稱
- 環境變數 `GITSTORE_GIT_TOKEN` — Git 認證 Token

### 同步行為

每次 `Save()` 呼叫都會自動 commit 並 push 至遠端：

```go
func (s *GitTokenStore) Save(_ context.Context, auth *cliproxyauth.Auth) (string, error) {
    // ... 將 Token 寫入本地檔案 ...
    
    // 自動 commit 並 push 至遠端 repository
    if errCommit := s.commitAndPushLocked(
        fmt.Sprintf("Update auth %s", messageID), relPath,
    ); errCommit != nil {
        return "", errCommit
    }
    return path, nil
}
```

Push 邏輯：
```go
if err = repo.Push(&git.PushOptions{Auth: s.gitAuth(), Force: true}); err != nil {
    // Force push 至遠端
}
```

### 風險場景

若 `GITSTORE_GIT_URL` 指向攻擊者控制的 Git repository：
- 每次 Token 更新（OAuth 登入、Token 刷新）都會自動 push 至攻擊者的 repository
- Token 以 JSON 格式儲存，包含所有 `metadata`（含 `access_token`、`refresh_token`）

---

## Object Store (S3) 分析

**檔案：** `internal/store/objectstore.go`

### 設定方式（ObjectStoreConfig）

```go
type ObjectStoreConfig struct {
    Endpoint  string  // S3 端點 (e.g., s3.amazonaws.com)
    Bucket    string  // Bucket 名稱
    AccessKey string  // S3 Access Key
    SecretKey string  // S3 Secret Key
    Region    string
    Prefix    string
    LocalRoot string  // 本地鏡像目錄
    UseSSL    bool
    PathStyle bool
}
```

### 同步行為

Token 會被上傳至 `{Prefix}/auths/{auth_id}.json`，且每次 Token 更新均會觸發上傳。

---

## PostgreSQL Store 分析

**檔案：** `internal/store/postgresstore.go`

PostgreSQL Store 的風險與 Git / S3 相似：若 PostgreSQL 連線字串指向攻擊者控制的資料庫伺服器，所有 Token 均會被同步寫入。

---

## 特別注意事項：GITSTORE_GIT_TOKEN 的雙重用途

在 `internal/managementasset/updater.go` 中，發現一個值得關注的行為：

```go
// 向 GitHub releases API 發送請求時，若 GITSTORE_GIT_TOKEN 存在，
// 會將其作為 Authorization Token 附加在請求中
gitURL := strings.ToLower(strings.TrimSpace(os.Getenv("GITSTORE_GIT_URL")))
if tok := strings.TrimSpace(os.Getenv("GITSTORE_GIT_TOKEN")); tok != "" && 
   strings.Contains(gitURL, "github.com") {
    req.Header.Set("Authorization", "Bearer "+tok)
}
```

此行為的目的是允許向私有 GitHub repository 的 releases API 進行認證存取，但這意味著 `GITSTORE_GIT_TOKEN` 會被附加在對 `api.github.com` 的請求中。這是合理且預期的行為，但使用者需了解此 Token 的用途。

---

## 風險摘要

| Token Store 類型 | 前提條件 | 洩漏目標 |
|-----------------|---------|---------|
| Git Store | `GITSTORE_GIT_URL` 指向攻擊者 | 攻擊者的 Git repository |
| S3 Object Store | `OBJECTSTORE_ENDPOINT` 指向攻擊者 | 攻擊者的 S3 |
| PostgreSQL Store | PostgreSQL DSN 指向攻擊者 | 攻擊者的資料庫 |

---

## 防護建議

1. **僅在必要時啟用遠端 Token Store**：若只有一個機器使用 CLIProxyAPI，不需要啟用任何遠端 Store
2. **確認 URL 安全性**：設定前確認 Git URL、S3 Endpoint、PostgreSQL DSN 指向受信任的服務
3. **使用私有 repository**：若使用 Git Store，確保 repository 設為 Private
4. **定期稽核**：若使用遠端 Store，定期確認儲存的 Token 資料是否如預期
