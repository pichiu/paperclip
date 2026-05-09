# DISCOVERY_LOG.md — 探索過程中的發現、問題與技術債記錄

> 生成日：2026-05-05
> 對象 commit：main branch（探索時間點）
> ⚠️ 未驗證 = 本次 trace session 中無法直接確認的資訊

---

## 1. Web Search 發現摘要

### 專案定位與成長速度

Paperclip 於 2026 年 3 月 4 日公開，三週內 GitHub Stars 突破 30,000。外部媒體將其評為「多 Agent 公司的開源作業系統」（Towards AI）。行銷核心句為「If OpenClaw is an _employee_, Paperclip is the _company_」。

### Heartbeat pattern 的外部說明（MindStudio 文章）

> "The heartbeat pattern is a scheduling and context-management mechanism that wakes an AI agent at regular intervals, injects fresh context into its working memory, and allows it to operate without any human trigger."

程式碼中已確認：`server/src/services/heartbeat.ts`（超過 9,000 行）實作此 pattern。運作方式為 `agentWakeupRequests` → `heartbeatRuns` → adapter 執行的 DB-backed queue。

### 版本號慣例

Release 採用 `v{year}.{dayOfYear}.{patch}` 格式（例：`v2026.318.0`）。以日期為基礎的版本號，不採用 semver。

### 社群生態現況

- `https://github.com/gsxdsm/awesome-paperclip` 有社群 plugin 彙整
- ClipHub（一鍵公司範本配布）目前為 ⚪ COMING SOON 狀態
- `doc/CLIPHUB.md` 有設計規格；`docs/specs/cliphub-plan.md` 已部分 superseded

### ⚠️ 未驗證的外部資訊

- `paperclipai.net` 另一個網域存在，但與官方的關係不明
- `agencyenterprise/paperclip-ai` fork 存在，但活動狀況與目的不明
- Discord `#dev` channel 建議在提交 PR 前先討論，但實際回應速度與接受標準無法確認

---

## 2. 現有文件與程式碼的落差

### 2.1 `billing_code` / `request_depth` — 已實作但缺乏 UI

**文件記載**：`doc/SPEC.md` 提及 `billing_code`（費用歸屬碼）與 `request_depth`（agent 委派深度）。

**程式碼實際狀況**：
- `packages/db/src/schema/issues.ts:48-49` — DB schema 中已有 `requestDepth`、`billingCode` 欄位
- `packages/db/src/schema/cost_events.ts:19` — 有 `billing_code` 欄位
- `packages/db/src/schema/finance_events.ts:21` — 有 `billing_code` 欄位

**落差**：UI（`ui/src/`）中找不到 `billing_code` / `request_depth` 的顯示或編輯元件。⚠️ 未驗證 — 資料雖已累積，但在 Board 畫面可能無法查閱或使用。

### 2.2 Budget Delegation（cascading 委派）— 設計已明確，實作僅部分完成

**文件記載**：`doc/SPEC.md` §1（Board Governance）記載「CEO 可將 budget 委派給 agent，manager agent 也可再委派給下屬」。

**程式碼實際狀況**：
- `server/src/services/budgets.ts` 實作了 Company / Project / Agent 三層政策
- `getInvocationBlock()` 可跨層檢查

**落差**：SPEC 中的「cascading budget delegation」（CEO → 下屬的連鎖委派）是否完整實作 ⚠️ 未驗證。`doc/SPEC.md` 本身也明確記載「How this cascading budget delegation works in practice is TBD」，代表規格層面尚未確定。

### 2.3 `doc/SPEC.md` 的 V1 checklist — 部分仍為 DRAFT 狀態

`doc/SPEC.md` §9（Frontend/UI）與 §10（V1 Scope）仍帶有 `[DRAFT]` 標記。以 checklist 形式列出的 V1 Must Have 項目中，許多 `[ ]` 尚未勾選。這使文件的可信度存疑——實作可能已完成，但文件未跟進維護。

### 2.4 `doc/plans/` 的設計計畫文件 — 部分已 superseded

`doc/AGENTCOMPANIES_SPEC_INVENTORY.md` 明確記載 `docs/specs/cliphub-plan.md` 為「Earlier blueprint bundle plan; partially superseded」。`doc/plans/` 內的各計畫文件（例：`2026-02-16-module-system.md`）與當前實作的整合程度，需逐一確認。

### 2.5 `server/src/services/github-fetch.ts` 用途不明確

**程式碼實際狀況**：
- `server/src/services/company-portability.ts` 與 `server/src/services/company-skills.ts` 匯入了 `ghFetch`、`gitHubApiBase`、`resolveRawGitHubUrl`

**落差**：`doc/` 中沒有 GitHub 整合的詳細說明。推測是 Company Import/Export 時直接從 GitHub 取得 skill 或公司範本的功能，但 ⚠️ 未驗證。

---

## 3. TODO / FIXME / HACK 彙整

對整個程式碼庫（`server/src/`、`packages/`、`ui/src/`、`cli/src/`）進行搜尋後，明確的 `TODO/FIXME/HACK` 雖不多，但確認了以下重要項目：

| 位置 | 種類 | 內容 |
|------|------|------|
| `ui/src/adapters/runtime-json-fields.tsx:5` | TODO | `// TODO(issue-worktree-support): re-enable this UI once the workflow is ready to ship.` — Issue-scoped worktree 的 UI 仍處於停用狀態 |
| `ui/src/pages/AgentDetail.tsx:882` | TODO | `// } else if (activeView === "skills") { // TODO: bring back later` — AgentDetail 頁面的 skills 檢視停用中 |
| `cli/src/commands/client/company.ts:383` | TODO | `// TODO: replace this temporary claude_local fallback with adapter selection in the import TUI.` — Company import TUI 缺少 adapter 選擇，暫時 fallback 至 `claude_local` |
| `server/src/services/plugin-loader.ts:170` | 未實作旗標 | `Registry support is not yet implemented; this field is reserved.` — Plugin 遠端 registry 功能為保留欄位，尚未實作 |
| `server/src/services/plugin-loader.ts:1038` | 未實作旗標 | `"plugin-loader: remote registry discovery is not yet implemented"` — 執行時也會拋出錯誤 |
| `ui/src/pages/RoutineDetail.tsx:1010` | UI 旗標 | `{kind === "webhook" ? " — COMING SOON" : ""}` — Routine 的 webhook trigger 在 UI 顯示 COMING SOON |
| `ui/src/adapters/metadata.ts:26,40` | 設計旗標 | 透過 `comingSoon` 旗標停用 adapter 的機制存在 — 哪些 adapter 為 COMING SOON 需進一步確認 |

---

## 4. 未解答的疑問

### 架構上的疑問

**Q1. Plugin Remote Registry 的設計意圖**
`plugin-loader.ts` 的 type union 中含有 `"registry"` 類型，並帶有 `Registry support is not yet implemented` 的註解。不清楚是否預計與 ClipHub 整合，或為獨立的 plugin registry。

**Q2. GitHub 整合的全貌**
`github-fetch.ts` 被 Company Portability 與 Company Skills 使用，但認證流程（GitHub token 的管理位置）不明。Agent 操作 GitHub PR 時的認證路徑，與 Board 從 GitHub 匯入 skill 時的認證路徑可能不同。

**Q3. `embedded-postgres@18.1.0-beta.16` 的「已 patch」內容**
`recon.md` 記載為「已 patch」，但具體修改內容尚未確認。雖然建議正式環境使用外部 Postgres，但在正式環境使用 embedded 的 beta 版風險 ⚠️ 未驗證。

**Q4. `feedbackService` 的運作模式**
`server/src/index.ts` 啟動流程中有 `feedbackService(db)`，`integrations.md` 記載 `server/src/services/feedback-share-client.ts` 的 flush 間隔為 5000ms。feedback 資料的傳送目標與隱私邊界是否與 telemetry 同樣受到適當控制 ⚠️ 未驗證。

**Q5. `MAXIMIZER MODE` 的實作計畫**
`ROADMAP.md` 中列有「⚪ MAXIMIZER MODE」。「更積極的委派、更深入的執行跟進」的具體實作意象（例：並行執行數上限放寬、略過 approval gate、自主上報閾值變更）不明。

**Q6. `acpx_local` adapter 中「ACPX」的真實身份**
`packages/adapters/acpx-local/` 中存在 `acpx_local` adapter，但 `recon.md` 或外部文章均無說明。⚠️ 未驗證。

**Q7. `pi_local` adapter 中「Pi」agent 的真實身份**
同樣地，`packages/adapters/pi-local/` 中的 Pi agent 所指為何不明。是 Claude Pi 模型，還是獨立的 agent runtime？⚠️ 未驗證。

---

## 5. 已知技術債

### 5.1 `heartbeat.ts` 過度龐大（超過 9,000 行的單體服務）

**位置**：`server/src/services/heartbeat.ts`

**問題**：agent 的完整生命週期（排程、context 建構、adapter 呼叫、費用記錄、復原、watchdog）集中在單一檔案。超過 9,000 行明顯違反 SRP（單一職責原則），難以測試，變更風險高。

**影響範圍**：`data_flow.md` 中描述的 Issue Checkout → Agent Execution 完整流程均依賴此檔案。

### 5.2 Plugin Registry（遠端）未實作，但型別定義中已混入

**位置**：`server/src/services/plugin-loader.ts:110,170,1038`

```typescript
| "registry";  // future: remote plugin registry URL
```

type union 中含有未來用途的值，執行時會拋出錯誤。作為 dead code 殘留，會對未來的實作者造成混淆。與 ClipHub 設計（`doc/CLIPHUB.md`）的接入點已有設計，但尚未實作。

### 5.3 Issue Worktree UI 仍處於停用狀態

**位置**：`ui/src/adapters/runtime-json-fields.tsx:5`

帶有 `issue-worktree-support` 標籤的 UI 已停用。`doc/plans/2026-03-10-workspace-strategy-and-git-worktrees.md` 有設計計畫，但實作是否完成不明。後端的 `workspace_strategy = 'isolated'` 可能已可運作，但缺少 UI 端的設定操作。

### 5.4 AgentDetail 的 skills 檢視停用

**位置**：`ui/src/pages/AgentDetail.tsx:882`

Agent 詳細頁面的「skills」tab 已停用。Skills Manager 在 ROADMAP.md 中標示為 ✅ 完成，但每個 agent 的 skill 詳細顯示 UI 尚缺。

### 5.5 Company Import TUI 的 adapter 選擇省略

**位置**：`cli/src/commands/client/company.ts:383`

Company import CLI 的 TUI 暫時 fallback 至 `claude_local`。在支援多個 adapter 的生態系中，import 時以硬編碼指定 adapter，對 Codex、Gemini、OpenClaw 使用者而言是功能限制。

### 5.6 `doc/SPEC.md` 多處殘留 `[DRAFT]` 與 `TBD`

**位置**：`doc/SPEC.md` §1, §2, §9, §10

Board Governance 的 Approval Gates（「其他治理閘門動作 TBD」）、Budget Delegation（「TBD」）、Frontend Views（`[DRAFT]`）等，在實作推進後文件未同步更新。對新進參與者可能造成錯誤印象。

### 5.7 `embedded-postgres` 使用 beta 版本

**位置**：`packages/db/` 依賴 `embedded-postgres@18.1.0-beta.16`

作為本地開發用途是合理的，但 beta release 對正式環境穩定性存在風險。`doc/DEVELOPING.md` 與 `doc/DATABASE.md` 均明記「正式環境建議使用外部 Postgres」，但未閱讀部署指南的使用者可能在正式環境直接使用 beta embedded Postgres。

---

## 6. 需要進一步深入調查的區域

本次 trace 涵蓋不足的重要領域：

| 區域 | 位置 | 原因 |
|------|------|------|
| **Evals 框架** | `evals/` | `doc/plans/2026-03-13-agent-evals-framework.md` 存在。有基於 promptfoo 的 LLM 評估基礎，但內容未確認 |
| **Recovery 系統詳情** | `server/src/services/recovery/` | Watchdog、孤立 run 偵測、自動產生 Recovery Issue 的具體閾值與實作未確認 |
| **Memory Landscape** | `doc/memory-landscape.md` | 記憶體/context 設計文件存在但未閱讀。與 ROADMAP 的「Memory/Knowledge」未實作項目相關 |
| **Execution Semantics** | `doc/execution-semantics.md` | 執行語意詳細規格。可能包含 heartbeat 間 Issue 狀態轉換的正式定義 |
| **Smart Model Routing** | `doc/plans/2026-04-06-smart-model-routing.md` | 2026 年 4 月提出的模型選擇自動化計畫，實作有無不明 |
| **VS Code Task Interoperability** | `doc/plans/2026-04-12-vscode-task-interoperability-plan.md` | VS Code 任務整合計畫存在，但內容與進度未確認 |
| **Agent OS Technical Report** | `doc/plans/2026-04-08-agent-os-technical-report.md` | 以 Agent OS 為題的最新技術報告。對理解架構未來方向具重要參考價值 |
| **MCP Server 詳情** | `packages/mcp-server/` | 將 Paperclip 資料以 MCP 形式公開的實作細節（公開工具與資源定義）未確認 |
| **Skills 目錄** | `skills/` | Agent skill 的實際 markdown 定義群。`skills/paperclip/SKILL.md` 的內容未確認 |
| **測試・Eval 涵蓋範圍** | `tests/`, `evals/` | E2E 測試（Playwright）與 promptfoo evals 的實際涵蓋範圍未確認 |
| **Untrusted PR Review** | `doc/UNTRUSTED-PR-REVIEW.md` | 從安全角度而言重要的 PR review 政策，尚未閱讀 |

---

## 7. 維護者・社群確認事項

建議在 Discord `#dev` channel 或 GitHub Issue 中確認的事項：

### 設計・Roadmap

1. **ClipHub / Plugin Remote Registry 整合計畫**：`doc/CLIPHUB.md` 已有設計，但 `plugin-loader.ts` 的 `"registry"` type 是否與其連動，或為獨立系統，需確認。
2. **MAXIMIZER MODE 的具體設計**：Roadmap 中有記載，但技術變更點（並行執行、approval gate 調整、治理邊界）不明。是核心變更還是以 plugin 實作？
3. **Budget Delegation（cascading）的實作優先度**：`doc/SPEC.md` §1 記載為「TBD」。CEO 向下級 agent 的 budget 委派預計在哪個 Milestone 實作？

### 技術上的疑慮

4. **`heartbeat.ts` 的重構計畫**：是否有計畫拆分超過 9,000 行的檔案？如有服務拆分方針（例：SchedulerService、AdapterOrchestratorService 等）的討論，希望提供參考連結。
5. **`acpx_local` / `pi_local` adapter 的正式名稱與對象 agent**：外部文件無說明，不清楚針對哪個 agent runtime。
6. **Issue Worktree UI 重新啟用時程**：`ui/src/adapters/runtime-json-fields.tsx` 的 `TODO(issue-worktree-support)` 預計何時啟用？後端是否已可運作？
7. **`embedded-postgres` beta 版本的遷移計畫**：是否等待 `18.1.0-beta.16` 的 stable release，還是考慮遷移至其他 embedded DB（如 PGlite）？

### 文件整備

8. **`doc/SPEC.md` 中 `[DRAFT]` 章節的更新**：§9、§10 的 DRAFT 標記預計何時移除？文件更新以反映當前實作狀態的優先度為何？
9. **`billing_code` / `request_depth` 的 UI 呈現**：DB 中已有資料，但 dashboard 無法查閱。是否計畫在未來的 Cost Dashboard 或 Billing Ledger 功能（`doc/plans/2026-03-14-billing-ledger-and-reporting.md`）中使用？

---

*本文件為 trace session 的知識整理用途，非官方技術文件。標記 ⚠️ 未驗證 的資訊請直接透過程式碼或文件確認。*
