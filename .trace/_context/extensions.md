# 擴充點（Extension Points）

## 1. Adapter System（最重要的擴充點）

### 概要

若要支援新的代理程式 runtime，需建立 **Adapter 套件**並新增至 `packages/adapters/`。

### Adapter 介面（`packages/adapter-utils/src/types.ts`）

```typescript
interface ServerAdapterModule {
  execute(context: AdapterExecutionContext): Promise<AdapterExecutionResult>;
  testEnvironment(context): Promise<AdapterEnvironmentTestResult>;
  sessionCodec?: AdapterSessionCodec;  // session 狀態的序列化/反序列化
  listSkills?(context): Promise<AdapterSkillSnapshot>;
  syncSkills?(context): Promise<void>;
  listModels?(): Promise<AdapterModel[]>;
  getQuotaWindows?(context): Promise<QuotaWindow[]>;
}
```

`execute()` 是核心：啟動代理程式 runtime 並回傳 `AdapterExecutionResult`。

### 現有 Adapter 清單

| Adapter | 套件 | 啟動方式 |
|----------|---------|---------|
| `claude_local` | `@paperclipai/adapter-claude-local` | 以子程序啟動本機 Claude Code CLI |
| `codex_local` | `@paperclipai/adapter-codex-local` | 以子程序啟動本機 Codex CLI |
| `cursor_local` | `@paperclipai/adapter-cursor-local` | Cursor API/CLI bridge |
| `gemini_local` | `@paperclipai/adapter-gemini-local` | 啟動本機 Gemini CLI |
| `openclaw_gateway` | `@paperclipai/adapter-openclaw-gateway` | HTTP Webhook（Fire-and-forget） |
| `opencode_local` | `@paperclipai/adapter-opencode-local` | 啟動本機 OpenCode CLI |
| `pi_local` | `@paperclipai/adapter-pi-local` | 啟動本機 Pi CLI |
| `acpx_local` | `@paperclipai/adapter-acpx-local` | ACPX 本機執行 |

### Plugin-based External Adapter

如 `doc/adapter-plugin.md` 所述，也可透過 **plugin 提供 adapter** 而非直接在伺服器端實作。這讓外部廠商無需 fork Paperclip Core 即可發布自有 adapter。

---

## 2. Plugin System

### Plugin 建立方式

使用 `packages/plugins/sdk/src/index.ts` 的 `definePlugin()`：

```typescript
import { definePlugin, runWorker } from "@paperclipai/plugin-sdk";

const plugin = definePlugin({
  async setup(ctx) {
    // 事件監聽器註冊
    ctx.events.on("issue.created", async (event) => {
      // Issue 建立時的處理
    });
    
    // Job 註冊（由排程器呼叫）
    ctx.jobs.register("sync-data", async (job) => {
      // 背景 job 處理
    });
    
    // Tool 註冊（代理程式可呼叫的 MCP tool）
    ctx.tools.register("my-tool", async (params) => {
      // tool 實作
    });
    
    // Launcher 註冊（在 UI 新增選單項目）
    ctx.launchers.register("my-launcher", ...);
  },
  
  async onHealth() {
    return { status: "ok" };
  },
});

export default plugin;
runWorker(plugin, import.meta.url);
```

### Plugin 可使用的 Host Services

| Service | 說明 |
|---------|------|
| `ctx.events` | 訂閱 Paperclip 的 domain 事件 |
| `ctx.jobs` | Job 的註冊與排程 |
| `ctx.tools` | MCP tool 的註冊 |
| `ctx.state` | Plugin 狀態儲存（key-value） |
| `ctx.db` | 受限的 DB 存取（僅限 plugin schema） |
| `ctx.data` | 資料提供者的註冊 |
| `ctx.launchers` | UI Launcher 的註冊 |
| `ctx.logger` | 結構化日誌 |
| `ctx.secrets` | Secret 讀取 |

### Plugin Manifest（`PaperclipPluginManifestV1`）

`plugin.json` 或 `package.json` 的 `paperclip` 欄位：

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "version": "1.0.0",
  "capabilities": ["events", "jobs", "tools"],
  "worker": "./dist/worker.js",
  "ui": "./dist/ui.js",
  "apiRoutes": [...],
  "webhooks": [...],
  "jobs": [...],
  "settings": { "schema": {...} }
}
```

`capabilities` 中僅宣告的能力可被使用（未宣告時會拋出 CapabilityDeniedError）。

---

## 3. Skill System（代理程式 Skill）

### Skill 的運作機制

Skill 是 Markdown 文件，在代理程式啟動時以 prompt 形式注入。這讓我們無需修改程式碼，即可向代理程式傳授新知識或操作步驟。

**Paperclip Skill**（`skills/paperclip/SKILL.md`）：教導代理程式如何操作 Paperclip API 的 skill，建議注入至所有代理程式。

### Skill 的註冊

1. **公司 Skill**（`company_skills` 資料表）：特定公司的代理程式可使用的 skill 清單
2. **代理程式 Skill**：與個別代理程式關聯的 skill
3. **Adapter Skill**：由 adapter 管理的 skill（Claude Code 的情況為 skill 檔案同步）

### Skill 的探索與管理

- `GET /api/skills/index` — 可用 skill 清單
- `GET /api/skills/paperclip` — Paperclip heartbeat skill 的 Markdown
- 透過 adapter 的 `listSkills()` / `syncSkills()` 與 adapter 端的 skill 同步

---

## 4. Routine System（已排程的 Routine）

Routine 是「定期執行的任務定義」，依照排程自動建立 Issue 並啟動代理程式。

### Trigger 種類（`routines.ts`）

| Trigger | 說明 |
|--------|------|
| `cron` | 依 cron 運算式排程 |
| `webhook` | HTTP Webhook 觸發 |
| `api` | 由 Paperclip API 手動觸發 |

### Routine 變數

可在 routine 範本中嵌入變數：
- `{{BRANCH}}` — 目前的 git branch
- 自訂變數定義

---

## 5. Execution Workspace Policy（執行 Workspace）

在專案設定「執行 workspace policy」可控制代理程式的工作位置：

- **`none`**：無 workspace（使用預設工作目錄）
- **`shared`**：所有代理程式使用共享 workspace
- **`isolated`**：每個 Issue 自動建立 git worktree
- **`operator`**：由 operator 指定的 branch

設定 `workspaceStrategy.provisionCommand` 可在 workspace 建立後執行自訂指令（例如：`pnpm install`）。

---

## 6. Governance Hooks（治理勾點）

### Board Approval Gates

`server/src/services/approvals.ts` 的審核系統作為勾點發揮功能：

- 招募代理程式時強制要求 Board 審核
- `agent.hire_hook.ts`（`server/src/services/hire-hook.ts`）定義招募時的 callback

### Agent Execution Policy

在各 Issue 設定 `executionPolicy` 可自訂執行流程：
- `stages`：新增需要審核的階段
- `monitor`：定義外部服務的輪詢階段
