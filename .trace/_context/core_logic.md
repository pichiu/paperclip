# 核心領域邏輯

## Paperclip 的「心臟」

Paperclip 最核心的非瑣碎邏輯是 **Heartbeat Execution Engine**（`server/src/services/heartbeat.ts`）。此檔案是超過 9,000 行的單體式 service，負責管理代理程式的完整生命週期。

---

## 1. Heartbeat Execution Engine

### 核心模式：DB-backed Queue + Atomic Execution Lock

```
agentWakeupRequests 資料表
    │  (喚醒請求時插入)
    ▼
heartbeatRuns 資料表 (status: queued)
    │  (由 timer/event 驅動清空)
    ▼
execute() 附 issues 的 atomic lock
    │
    ▼
heartbeatRuns (status: running → completed/failed)
```

**為何使用 DB-backed？**：即使在 ephemeral VM 環境中佇列也不會遺失，因此採用 DB 持久化佇列而非 in-memory 佇列。

### 樂觀鎖（Issue Checkout）

`server/src/services/issues.ts` 的 `checkout()` 方法使用 `expectedStatuses` 進行樂觀鎖，防止雙重 checkout。透過 atomic 的 `UPDATE WHERE status = ANY(expectedStatuses)`，防止多個代理程式同時取得同一個任務。

### Session 連續性（`agent_task_sessions` 資料表）

跨 heartbeat 維持代理程式 session 的機制：
- `taskKey`：唯一識別代理程式與任務組合的 key
- `sessionId`：保存 Claude Code 的 session ID 等資訊
- 下一次 heartbeat 重複使用 `sessionId`，以繼承前次的 context

### Context Snapshot 建構（`buildPaperclipWakePayload`）

於 `server/src/services/heartbeat.ts:1789` 建構傳遞給代理程式的「喚醒 payload」：

```typescript
{
  // 公司・專案・goal 資訊
  company: { id, name, goal },
  project: { id, name, description },
  
  // 目前任務
  issue: {
    id, identifier, title, description, status,
    parentIssue,      // 父任務的連鎖（goal alignment 的依據）
    goalContext,      // goal 系譜
  },
  
  // 組織 context
  agentConfig,       // 代理程式的 role/title/capabilities
  skills,            // 注入的 skill markdown（Paperclip Skill 等）
  
  // 執行環境
  workspace,         // CWD、git worktree 路徑等
  secrets,           // 已展開的 secret
  
  // 前次執行摘要（繼續執行時）
  previousRunSummary,
}
```

這是傳遞給代理程式的 prompt「本文」。

---

## 2. Budget & Cost System

### Budget Policy 的層級結構

`server/src/services/budgets.ts`：

```
公司層級 budget policy
    ├── 專案層級 budget policy
    │       └── 代理程式層級 budget policy
    └── 代理程式層級 budget policy（直屬公司）
```

各 policy 包含：
- `windowKind`：`monthly` 或 `lifetime`
- `hardStopAmount`：超過此金額強制停止代理程式
- `warningAmount`：警告閾值

`getInvocationBlock()` 在代理程式開始執行前確認所有層級的 policy，若需封鎖則回傳 `{ blocked: true, reason, scopeType, scopeId }`。

### 成本事件記錄

以下列粒度記錄至 `cost_events` 資料表：
- company / agent / project / goal / issue / provider / model / billing_type

這使得細粒度的成本分析成為可能。

---

## 3. Issue Tree Control System

`server/src/services/issue-tree-control.ts` 管理的樹狀層級執行控制：

### Issue Tree Hold（暫停/停止）

暫停整個 Issue 樹狀結構（父子關係）的功能：
- `issue_tree_holds` 資料表：hold 的定義
- `issue_tree_hold_members` 資料表：哪些 Issue 被納入 hold

代理程式在 checkout 新任務之前會確認 hold 狀態，若樹狀結構處於 hold 中則拒絕 checkout。

### Issue Execution Policy

`server/src/services/issue-execution-policy.ts`：

每個 Issue 都有 `executionPolicy` JSONB 欄位，包含：
- `mode`：`default` | `manual_only` | `blocked`
- `stages`：執行階段的定義（例如：外部服務監控）
- `monitor`：已排程的外部服務檢查設定

---

## 4. Plugin System

### 架構

Plugin 以 **out-of-process Worker** 方式運作（`server/src/services/plugin-worker-manager.ts`）：

```
Paperclip Server
    │
    ├── PluginWorkerManager
    │       └── Worker Process（Node.js child_process）
    │               ├── Plugin SDK (@paperclipai/plugin-sdk)
    │               └── Plugin Code（npm package）
    │
    └── Host Services（IPC via plugin-host-services.ts）
            ├── DB 存取（公司範圍限制）
            ├── Storage 存取
            ├── Job Scheduling
            └── Tool Registration
```

### Plugin 生命週期（`plugin-lifecycle.ts`）

狀態機：

```
installed → ready → disabled
    │           │       │
    │           ├── error
    │           └── upgrade_pending
    └── uninstalled
```

- `installed` → `ready`：Plugin 的 worker process 啟動
- `ready` → `disabled`/`error`：優雅地關閉 process

### Capability Gate

`plugin-capability-validator.ts` 驗證 plugin manifest 中宣告的 `capabilities`，若 host IPC 呼叫嘗試使用宣告範圍外的能力則予以封鎖。

---

## 5. Governance & Approval System

`server/src/services/approvals.ts`：

需要審核的動作：
- 新代理程式招募（`agent.hire`）
- CEO 的初始策略提案
- 設定變更（revision 管理）

審核流程：
1. 代理程式或系統建立審核請求（`approvals` 資料表）
2. Board 使用者呼叫 `/api/approvals/:id/approve` 或 `/api/approvals/:id/reject`
3. 核准後執行對應動作（例如：將代理程式 `status` 改為 `active`）

透過 `agent_config_revisions` 資料表管理代理程式設定的變更歷史，可進行回復。

---

## 6. Recovery System

`server/src/services/recovery/` 目錄：

偵測並修復孤立執行（伺服器重新啟動後仍停留在 `running` 狀態的執行記錄）：
- `reconcilePersistedRuntimeServicesOnStartup()`：啟動時確認執行中的記錄
- 將滯留的執行記錄轉換為 `failed` 狀態，並自動產生 Recovery Issue
- Watchdog 監控執行進度，偵測一定時間內無回應的執行記錄

---

## 重要的 Abstraction 模式

| 模式 | 實作位置 | 用途 |
|--------|--------|------|
| **Adapter Pattern** | `server/src/adapters/registry.ts` | 將不同代理程式 runtime 統一為一致介面 |
| **Service Layer** | `server/src/services/` | 隔離商業邏輯與路由處理器 |
| **Activity Log** | `server/src/services/activity-log.ts` | 所有變更操作的稽核日誌 |
| **Live Events** | `server/src/realtime/` | 透過 WebSocket 即時推送更新至 UI |
| **Optimistic Lock** | `issueService.checkout()` | 防止雙重 checkout |
| **State Machine** | `plugin-lifecycle.ts` | Plugin 生命週期管理 |
