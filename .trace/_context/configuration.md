# 設定與環境（Configuration）

## 設定的優先順序

```
環境變數（最優先）
    │
    ▼
PAPERCLIP_ENV_FILE_PATH（.paperclip/.env 或 ~/.paperclip/.env）
    │
    ▼
CWD/.env
    │
    ▼
config.json（PAPERCLIP_CONFIG 或 ~/.paperclip/instances/default/config.json）
    │
    ▼
預設值（最低優先）
```

`server/src/config.ts:loadConfig()` 整合所有設定並回傳 `Config` 物件。

---

## 主要環境變數清單

### 伺服器・部署

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `PORT` | `3100` | HTTP 伺服器埠號 |
| `HOST` | `127.0.0.1` | 綁定主機 |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | `local_trusted` 或 `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` 或 `public` |
| `PAPERCLIP_BIND` | （由 HOST 推斷） | `loopback` / `lan` / `tailnet` / `custom` |
| `SERVE_UI` | `false` | 設為 `true` 以提供已建置的 UI |
| `PAPERCLIP_HOME` | `~/.paperclip` | 資料主目錄 |
| `PAPERCLIP_INSTANCE_ID` | `default` | 執行個體識別子（多執行個體共存用） |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | - | 逗號分隔的允許主機名稱清單 |
| `PAPERCLIP_PUBLIC_URL` | - | 公開 URL（驗證 callback 用） |

### 驗證

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `BETTER_AUTH_SECRET` | - | Session HMAC secret（`authenticated` 模式必填） |
| `PAPERCLIP_AUTH_DISABLE_SIGN_UP` | `false` | 停用註冊功能 |
| `PAPERCLIP_AUTH_PUBLIC_BASE_URL` | - | 驗證基底 URL |

### 資料庫

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `DATABASE_URL` | （未設定 → embedded postgres） | 外部 Postgres 連線字串 |
| `PAPERCLIP_DB_BACKUP_ENABLED` | `true` | 自動備份啟用/停用 |
| `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES` | `60` | 備份間隔（分鐘） |
| `PAPERCLIP_DB_BACKUP_RETENTION_DAYS` | `30` | 備份保留天數 |
| `PAPERCLIP_DB_BACKUP_DIR` | `~/.paperclip/.../backups` | 備份儲存位置 |
| `PAPERCLIP_MIGRATION_AUTO_APPLY` | `true`（非 TTY 時） | 自動套用 migration |

### Secret 管理

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `PAPERCLIP_SECRETS_PROVIDER` | `local_encrypted` | `local_encrypted` 或 `plain_text` |
| `PAPERCLIP_SECRETS_STRICT_MODE` | `false` | 強制敏感變數使用 secret ref |
| `PAPERCLIP_SECRETS_MASTER_KEY` | - | 加密主金鑰（直接指定） |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | `~/.paperclip/.../secrets/master.key` | 金鑰檔路徑 |

### Storage

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `PAPERCLIP_STORAGE_PROVIDER` | `local_disk` | `local_disk` 或 `s3` |
| `PAPERCLIP_STORAGE_LOCAL_DIR` | `~/.paperclip/.../data/storage` | 本機 storage 路徑 |
| `PAPERCLIP_STORAGE_S3_BUCKET` | `paperclip` | S3 bucket 名稱 |
| `PAPERCLIP_STORAGE_S3_REGION` | `us-east-1` | S3 區域 |
| `PAPERCLIP_STORAGE_S3_ENDPOINT` | - | S3 相容端點 URL |
| `PAPERCLIP_STORAGE_S3_FORCE_PATH_STYLE` | `false` | 強制使用路徑風格 URL |

### Telemetry

| 變數名 | 說明 |
|-------|------|
| `PAPERCLIP_TELEMETRY_DISABLED=1` | 停用 telemetry |
| `DO_NOT_TRACK=1` | 標準停用方式 |
| `CI=true` | CI 環境自動停用 |

### 代理程式・成本控制

| 變數名 | 預設值 | 說明 |
|-------|----------|------|
| `PAPERCLIP_ENABLE_COMPANY_DELETION` | `local_trusted`: true | 公司刪除的啟用/停用 |
| `OPENCODE_ALLOW_ALL_MODELS` | - | OpenCode 中允許所有模型 |

### Worktree（本機開發）

| 變數名 | 說明 |
|-------|------|
| `PAPERCLIP_IN_WORKTREE=true` | 表示在 worktree 內執行 |
| `PAPERCLIP_WORKTREE_NAME` | worktree 的顯示名稱（UI banner 用） |
| `PAPERCLIP_WORKTREE_COLOR` | worktree 的識別顏色（Hex） |

---

## config.json 結構

`~/.paperclip/instances/default/config.json` 的主要區段：

```json
{
  "server": {
    "deploymentMode": "local_trusted",
    "exposure": "private",
    "bind": "loopback",
    "host": "127.0.0.1",
    "allowedHostnames": []
  },
  "database": {
    "mode": "embedded-postgres",
    "connectionString": "postgres://...",
    "backup": {
      "enabled": true,
      "intervalMinutes": 60,
      "retentionDays": 30
    }
  },
  "secrets": {
    "provider": "local_encrypted",
    "strictMode": false,
    "masterKeyFilePath": "~/.paperclip/.../secrets/master.key"
  },
  "storage": {
    "provider": "local_disk",
    "localDisk": { "baseDir": "~/.paperclip/.../data/storage" },
    "s3": { "bucket": "", "region": "", "endpoint": "" }
  },
  "auth": {
    "publicBaseUrl": "",
    "disableSignUp": false
  },
  "telemetry": {
    "enabled": true
  }
}
```

---

## 資料目錄結構

```
PAPERCLIP_HOME/                        # 預設：~/.paperclip
└── instances/
    └── PAPERCLIP_INSTANCE_ID/         # 預設：default
        ├── config.json                # 執行個體設定
        ├── db/                        # Embedded PostgreSQL 資料
        ├── secrets/
        │   └── master.key             # 本機加密主金鑰
        ├── data/
        │   ├── storage/               # 本機檔案 storage
        │   └── backups/               # DB 備份
        └── workspaces/
            └── <agent-id>/            # 代理程式預設 workspace

~/.paperclip-worktrees/               # Worktree 用的隔離執行個體
└── instances/
    └── <worktree-id>/
        ├── ...（相同結構）
```

---

## Feature Flag / 實驗性設定

由 `server/src/services/instance-settings.ts` 管理：

- `censorUsernameInLogs`：遮蔽日誌中的使用者名稱
- 實驗性設定可透過 `InstanceExperimentalSettings.tsx` 頁面進行設定
- `PAPERCLIP_IN_WORKTREE` 的受保護自動重新啟動功能

---

## Secret 參照格式

在代理程式環境變數中參照 secret 時：

```json
{
  "ANTHROPIC_API_KEY": {
    "type": "secret_ref",
    "secretId": "uuid-of-secret",
    "version": "latest"
  }
}
```

在 `strict` 模式下，若嘗試以 plain text 設定符合 `SENSITIVE_ENV_KEY_RE` 模式的 key（`*_API_KEY`、`*_TOKEN`、`*_SECRET` 等），將會回傳錯誤。
