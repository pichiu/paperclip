# Data Flow — 代表性使用案例：Issue 的 Checkout 與代理程式執行

## 使用案例概要

「代理程式將 Issue（任務）checkout 後，透過 heartbeat 實際執行作業」的完整資料流程。

---

## 整體流程圖

```
HTTP Client (Board UI / Agent)
    │
    ▼
actorMiddleware              # 驗證・actor 識別
    │
    ▼
POST /api/issues/:id/checkout
    │
    ▼
issueService.checkout()      # 附樂觀鎖的 Issue checkout
    │
    ▼ (assignee 為 agent 時)
heartbeatService.wakeup()    # 代理程式喚醒佇列
    │
    ▼
enqueueWakeup()              # 插入 agentWakeupRequests 資料表
    │
    ▼
heartbeat timer loop         # 定期清空佇列
    │
    ▼
budgetService.getInvocationBlock()   # budget 檢查
    │
    ▼
secretService.inject()       # secret 注入
    │
    ▼
getServerAdapter()           # adapter 選擇（claude_local 等）
    │
    ▼
adapter.execute()            # 實際代理程式執行
    │
    ▼
heartbeatRuns 資料表更新 (status: running → completed/failed)
    │
    ▼
costService.record()         # token 成本記錄
    │
    ▼
activity logActivity()       # 稽核日誌記錄
    │
    ▼
publishLiveEvent()           # 透過 WebSocket 推送至 Board UI
```

---

## Step 1：驗證・Actor 識別（`server/src/middleware/auth.ts`）

執行 `actorMiddleware`，識別發出請求的 actor：

| 來源 | Actor 型別 |
|------|-----------|
| `local_trusted` 模式 | 固定為 `board` 使用者（無需驗證） |
| `Authorization: Bearer <jwt>` | 代理程式短命 JWT（`agent` 型） |
| `Authorization: Bearer <api-key>` | Agent API Key（`agent` 型）或 Board API Key |
| Cookie session | Better Auth session（`user` 型） |

將 actor 資訊設定至 `req.actor` 物件。

---

## Step 2：Issue Checkout（`server/src/routes/issues.ts:2911`）

```
POST /api/issues/:id/checkout
Body: { agentId, expectedStatuses }
```

### 驗證
- `validate(checkoutIssueSchema)` — 以 Zod schema 進行輸入驗證
- `assertCompanyAccess(req, companyId)` — 確認 actor 是否可存取目標公司
- 確認專案未處於暫停狀態

### 附樂觀鎖的 checkout（`issueService.checkout()`）

位於 `server/src/services/issues.ts`：

```sql
UPDATE issues
SET 
  status = 'in_progress',
  checkout_run_id = :runId,
  execution_locked_at = NOW()
WHERE id = :issueId
  AND status = ANY(:expectedStatuses)  -- 樂觀鎖
RETURNING *
```

若 `expectedStatuses` 不符（其他代理程式已搶先取得），則回傳 `409 Conflict`。

### 活動日誌
`logActivity(db, { action: "issue.checked_out", ... })` → 插入 `activity_log` 資料表

---

## Step 3：代理程式喚醒（`enqueueWakeup`）

`server/src/services/heartbeat.ts:7834`

1. **Budget 封鎖檢查**：`budgets.getInvocationBlock()`
   - 確認代理程式/專案/公司層級的月結/終身 budget policy
   - 若被封鎖 → 以 `status: skipped` 記錄至 `agentWakeupRequests`，回傳 `409` 錯誤
2. **代理程式狀態檢查**：若 `agent.status` 為 `paused/terminated/pending_approval`，同樣略過
3. **在 DB transaction 內**：
   - 以 `status: queued` 將新執行記錄插入 `heartbeatRuns` 資料表
   - 記錄至 `agentWakeupRequests` 資料表
   - 更新 `issues` 的 `execution_run_id`（執行鎖定）

---

## Step 4：Heartbeat 排程器的佇列清空

`server/src/services/heartbeat.ts` 的內部迴圈定期（或收到喚醒請求時）執行：

1. 從 `heartbeatRuns` 取得 `status: queued` 的執行記錄
2. 再次以 `budgets.getInvocationBlock()` 確認 budget
3. 以 `secretService.inject()` 將代理程式設定中的 secret 參照展開為實際值
4. 以 `companySkillService.getSkillsForAgent()` 取得代理程式的 skill
5. 以 `realizeExecutionWorkspace()` 解析 workspace（git worktree、CWD 等）
6. 以 `buildPaperclipWakePayload()` 建構傳遞給代理程式的完整 context payload
7. 以 `getServerAdapter(adapterType).execute(context)` 呼叫實際的 adapter

---

## Step 5：Adapter 執行（`packages/adapters/claude-local/src/server/execute.ts`）

呼叫 `execute(context: AdapterExecutionContext)`：

1. **指令解析**：解析 `claude` CLI 的路徑
2. **Session 狀態**：從既有 session ID 繼續執行，或建立新 session
3. **Prompt bundle 建構**：將 `buildPaperclipWakePayload` 產生的 payload 傳入 `--prompt`
4. **環境變數注入**：secret、API key、workspace 路徑等
5. **子程序啟動**：`child_process.spawn("claude", args, { cwd, env })`
6. **stdout/stderr 串流處理**：
   - 即時將日誌插入 `heartbeatRunEvents` 資料表
   - 同時寫出至日誌檔（`run-log-store.ts`）
7. **完成後**：回傳 `AdapterExecutionResult`（exitCode、usageSummary 等）

---

## Step 6：執行完成處理

1. 將 `heartbeatRuns.status` 從 `running` 更新為 `completed` / `failed`
2. `costService.record()` — 將 token 使用量記錄至 `cost_events` 資料表
3. 清除 `issues.executionRunId`（解除執行鎖定）
4. `publishLiveEvent("heartbeat_run_updated", ...)` — 透過 WebSocket 通知 UI
5. `budgets.recordCost()` — 更新 budget policy 的累積成本，確認是否觸發強制停止

---

## 補充：代理程式回呼 Paperclip API 的反向流程

執行中的代理程式（例如 Claude Code）使用短命 JWT（`createLocalAgentJwt`）回呼 Paperclip API：

```
GET  /api/issues/:id/heartbeat-context  # 取得目前任務的 context
POST /api/issues/:id/comments           # 發佈留言
PATCH /api/issues/:id                   # 更新狀態
POST /api/issues/:id/children           # 建立子任務
POST /api/issues/:id/checkout           # 代理程式 checkout 任務
POST /api/issues/:id/release            # 釋放任務
```

此 JWT 由 `server/src/agent-auth-jwt.ts` 產生與驗證，有效期限短暫（僅限執行期間）。

---

## 資料轉換摘要

| 層 | 轉換內容 |
|---|---------|
| HTTP Routing | 路徑參數展開 + Zod 驗證 |
| Middleware | Actor 識別 → `req.actor` |
| Service 層 | 商業規則（狀態轉換・樂觀鎖）、透過 Drizzle ORM 操作 DB |
| Adapter 層 | Paperclip context → 代理程式 CLI 指令 |
| 代理程式執行 | Claude Code 接收 prompt，透過 Paperclip Skill 操作 API |
| 完成後 | 使用量・成本・日誌 → 持久化至 DB + WebSocket 推送 |
