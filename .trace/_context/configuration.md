# 設定と環境（Configuration）

## 設定の優先順位

```
環境変数 (最優先)
    │
    ▼
PAPERCLIP_ENV_FILE_PATH (.paperclip/.env または ~/.paperclip/.env)
    │
    ▼
CWD/.env
    │
    ▼
config.json (PAPERCLIP_CONFIG または ~/.paperclip/instances/default/config.json)
    │
    ▼
デフォルト値 (最低優先)
```

`server/src/config.ts:loadConfig()` が全設定を統合して `Config` オブジェクトを返す。

---

## 主要環境変数一覧

### サーバー・デプロイ

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `PORT` | `3100` | HTTP サーバーポート |
| `HOST` | `127.0.0.1` | バインドホスト |
| `PAPERCLIP_DEPLOYMENT_MODE` | `local_trusted` | `local_trusted` or `authenticated` |
| `PAPERCLIP_DEPLOYMENT_EXPOSURE` | `private` | `private` or `public` |
| `PAPERCLIP_BIND` | （HOST から推論） | `loopback` / `lan` / `tailnet` / `custom` |
| `SERVE_UI` | `false` | `true` でビルド済み UI を serve |
| `PAPERCLIP_HOME` | `~/.paperclip` | データホームディレクトリ |
| `PAPERCLIP_INSTANCE_ID` | `default` | インスタンス識別子（複数インスタンス共存用） |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | - | カンマ区切りの許可ホスト名 |
| `PAPERCLIP_PUBLIC_URL` | - | 公開 URL（認証コールバック用） |

### 認証

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `BETTER_AUTH_SECRET` | - | セッション HMAC シークレット（必須: `authenticated` モード） |
| `PAPERCLIP_AUTH_DISABLE_SIGN_UP` | `false` | サインアップ無効化 |
| `PAPERCLIP_AUTH_PUBLIC_BASE_URL` | - | 認証ベース URL |

### データベース

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `DATABASE_URL` | （未設定→embedded postgres） | 外部 Postgres 接続文字列 |
| `PAPERCLIP_DB_BACKUP_ENABLED` | `true` | 自動バックアップ有効/無効 |
| `PAPERCLIP_DB_BACKUP_INTERVAL_MINUTES` | `60` | バックアップ間隔（分） |
| `PAPERCLIP_DB_BACKUP_RETENTION_DAYS` | `30` | バックアップ保持日数 |
| `PAPERCLIP_DB_BACKUP_DIR` | `~/.paperclip/.../backups` | バックアップ保存先 |
| `PAPERCLIP_MIGRATION_AUTO_APPLY` | `true` (non-TTY) | マイグレーション自動適用 |

### シークレット管理

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `PAPERCLIP_SECRETS_PROVIDER` | `local_encrypted` | `local_encrypted` or `plain_text` |
| `PAPERCLIP_SECRETS_STRICT_MODE` | `false` | sensitive 変数の secret ref 強制 |
| `PAPERCLIP_SECRETS_MASTER_KEY` | - | 暗号化マスターキー（直接指定） |
| `PAPERCLIP_SECRETS_MASTER_KEY_FILE` | `~/.paperclip/.../secrets/master.key` | キーファイルパス |

### ストレージ

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `PAPERCLIP_STORAGE_PROVIDER` | `local_disk` | `local_disk` or `s3` |
| `PAPERCLIP_STORAGE_LOCAL_DIR` | `~/.paperclip/.../data/storage` | ローカルストレージパス |
| `PAPERCLIP_STORAGE_S3_BUCKET` | `paperclip` | S3 バケット名 |
| `PAPERCLIP_STORAGE_S3_REGION` | `us-east-1` | S3 リージョン |
| `PAPERCLIP_STORAGE_S3_ENDPOINT` | - | S3 互換エンドポイント URL |
| `PAPERCLIP_STORAGE_S3_FORCE_PATH_STYLE` | `false` | パススタイル URL 強制 |

### テレメトリ

| 変数名 | 説明 |
|-------|------|
| `PAPERCLIP_TELEMETRY_DISABLED=1` | テレメトリ無効化 |
| `DO_NOT_TRACK=1` | 標準的な無効化方法 |
| `CI=true` | CI 環境では自動無効 |

### エージェント・コスト制御

| 変数名 | デフォルト | 説明 |
|-------|----------|------|
| `PAPERCLIP_ENABLE_COMPANY_DELETION` | `local_trusted`: true | 会社削除の有効/無効 |
| `OPENCODE_ALLOW_ALL_MODELS` | - | OpenCode で全モデルを許可 |

### Worktree（ローカル開発）

| 変数名 | 説明 |
|-------|------|
| `PAPERCLIP_IN_WORKTREE=true` | worktree 内での実行を示す |
| `PAPERCLIP_WORKTREE_NAME` | worktree の表示名（UI バナー用） |
| `PAPERCLIP_WORKTREE_COLOR` | worktree の識別色（Hex） |

---

## config.json 構造

`~/.paperclip/instances/default/config.json` の主要セクション：

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

## データディレクトリ構造

```
PAPERCLIP_HOME/                        # デフォルト: ~/.paperclip
└── instances/
    └── PAPERCLIP_INSTANCE_ID/         # デフォルト: default
        ├── config.json                # インスタンス設定
        ├── db/                        # Embedded PostgreSQL データ
        ├── secrets/
        │   └── master.key             # ローカル暗号化マスターキー
        ├── data/
        │   ├── storage/               # ローカルファイルストレージ
        │   └── backups/               # DB バックアップ
        └── workspaces/
            └── <agent-id>/            # エージェントデフォルトワークスペース

~/.paperclip-worktrees/               # Worktree 用分離インスタンス
└── instances/
    └── <worktree-id>/
        ├── ...（同構造）
```

---

## Feature Flag / 実験的設定

`server/src/services/instance-settings.ts` が管理：

- `censorUsernameInLogs`: ログ内のユーザー名マスキング
- 実験的設定は `InstanceExperimentalSettings.tsx` ページから設定可能
- `PAPERCLIP_IN_WORKTREE` での guarded auto-restart 機能

---

## Secret 参照形式

エージェント環境変数でシークレットを参照する際：

```json
{
  "ANTHROPIC_API_KEY": {
    "type": "secret_ref",
    "secretId": "uuid-of-secret",
    "version": "latest"
  }
}
```

`strict` モードでは `SENSITIVE_ENV_KEY_RE` パターンにマッチするキー（`*_API_KEY`, `*_TOKEN`, `*_SECRET` 等）を plain text で設定しようとするとエラー。
