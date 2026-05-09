# 外部整合（Integrations）

## 1. 代理程式 Runtime（最大的外部依賴）

### Claude Code（Anthropic）

- **Adapter**：`packages/adapters/claude-local/`
- **連線方式**：以子程序啟動本機 `claude` CLI
- **驗證**：`ANTHROPIC_API_KEY` 或 Claude Code session
- **Session 連續性**：以 `claude --resume <sessionId>` 從前次 session 繼續執行
- **模型**：claude-opus-4-7、claude-sonnet-4-6、claude-haiku-4-6 等

**錯誤處理**：
- `isClaudeTransientUpstreamError()` 偵測暫時性錯誤 → bounded retry（`BOUNDED_TRANSIENT_HEARTBEAT_RETRY_DELAYS_MS`）
- `isClaudeMaxTurnsResult()` → 以 max_turns_continuation 繼續至下一次 heartbeat
- `detectClaudeLoginRequired()` → 偵測需要登入的狀態並通知 Board

### OpenClaw Gateway

- **Adapter**：`packages/adapters/openclaw-gateway/`
- **連線方式**：WebSocket over HTTPS（`wss://...`）
- **驗證**：Device Identity（ECDH keypair）— 將 `deviceId` 與公鑰註冊至 OpenClaw
- **協定**：JSON-RPC 風格的請求/回應框架
- **啟動方式**：非 Fire-and-forget（Webhook），而是常時連線 WebSocket 等待 callback

**錯誤處理**：
- WebSocket 中斷時進行重連重試
- 有逾時設定（`request timeout`）
- 以 `pendingRequest` Map 追蹤請求的回應

### Codex / Gemini / OpenCode / Pi / Cursor

相同模式（CLI 子程序啟動）。各 adapter 解析 CLI 的 stdout/stderr 以取得 token 使用量與執行結果。

---

## 2. PostgreSQL（內嵌或外部）

- **套件**：`embedded-postgres@18.1.0-beta.16`（已修補）
- **連線**：透過 `DATABASE_URL` 環境變數可切換至外部 Postgres
- **ORM**：Drizzle ORM
- **Migration**：SQL 檔案位於 `packages/db/src/migrations/`

**連線生命週期**：
1. 在 `server/src/index.ts` 的 `startServer()` 中啟動實例（或建立外部連線）
2. 以 `applyPendingMigrations()` 套用尚未執行的 migration
3. 以 `createDb(url)` 建立 Drizzle ORM 實例

---

## 3. Better Auth（驗證）

- **套件**：`better-auth`
- **模式**：僅在 `authenticated` 部署模式下使用
- **提供功能**：session 管理、使用者註冊/登入、Cookie 驗證
- **設定**：`BETTER_AUTH_SECRET` 環境變數（HMAC secret）
- **端點**：`/api/auth/*` 以下的所有路徑

---

## 4. Object Storage（檔案附件・作業成果）

**`server/src/storage/`**：

| 提供者 | 設定 | 用途 |
|----------|-----|-----|
| `local_disk` | `PAPERCLIP_STORAGE_DIR` | 本機開發・自架伺服器 |
| `s3` | `STORAGE_S3_BUCKET` 等 | 正式環境 S3 相容儲存 |

`StorageService` 介面：
- `put(key, buffer, contentType)` — 上傳檔案
- `get(key)` — 取得檔案
- `delete(key)` — 刪除檔案
- `getSignedUrl(key, expiry)` — 產生簽署 URL

---

## 5. Telemetry（匿名使用統計）

- **`server/src/telemetry.ts`**
- **傳送內容**：匿名的使用模式（專案名稱已雜湊化）
- **排除項目**：個人資訊・Issue 內容・prompt・檔案路徑・secret
- **停用方式**：
  - `PAPERCLIP_TELEMETRY_DISABLED=1`
  - `DO_NOT_TRACK=1`
  - `CI=true`（自動停用）
  - 在 config 檔設定 `telemetry.enabled: false`

---

## 6. GitHub Integration

`server/src/services/github-fetch.ts`：
- 呼叫 GitHub API（取得 PR 資訊等）
- ⚠️ 主要用途尚未確認（內部文件明確指出 PR review 工具不在 Paperclip 的職責範圍內）

---

## 7. MCP Server

`packages/mcp-server/`：
- 將 Paperclip 的資料以 MCP（Model Context Protocol）server 形式公開
- 讓 Claude Code 等支援 MCP 的代理程式可直接將 Paperclip 的資料作為 tool 使用

---

## 8. Feedback / Trace Share

`server/src/services/feedback-share-client.ts`：
- 匿名回饋共享功能
- 排程間隔：5,000ms（`FEEDBACK_EXPORT_FLUSH_INTERVAL_MS`）

---

## 9. 錯誤處理模式

| 情境 | 處理方式 |
|---------|---------|
| Claude 暫時性錯誤 | bounded retry（指數退避） |
| Budget 超支 | 停止代理程式，通知 Board |
| 代理程式無回應 | Watchdog 偵測，自動建立 recovery issue |
| DB 連線失敗 | 拒絕伺服器啟動（fail-fast） |
| Plugin 崩潰 | 重新啟動 worker process（不影響其他部分） |
| Storage 錯誤 | 回傳錯誤回應（捕捉後記錄日誌） |
| Webhook 逾時 | 設定逾時後記錄錯誤 |
