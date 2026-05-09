# Trace Metadata

## 分支資訊
- **Base Branch**: claude/generate-codebase-docs-JPMK3
- **Trace Branch**: claude/generate-codebase-docs-JPMK3

## 最後 Trace 資訊
- **Base Commit Hash**: d6bee62f02de058a4f12ac9a10dedf8c58c4b1d5
- **日期**: 2026-05-05
- **Trace 類型**: full
- **涵蓋範圍**: 全部（server / ui / cli / packages / adapters / plugins）

## 文件清單

| 文件 | 對應 Base Commit | 最後更新日期 |
|------|-----------------|-------------|
| INDEX.md | d6bee62 | 2026-05-05 |
| ARCHITECTURE.md | d6bee62 | 2026-05-05 |
| DATA_MODEL.md | d6bee62 | 2026-05-05 |
| API_SURFACE.md | d6bee62 | 2026-05-05 |
| DEV_GUIDE.md | d6bee62 | 2026-05-05 |
| CODEBASE_MAP.md | d6bee62 | 2026-05-05 |
| DISCOVERY_LOG.md | d6bee62 | 2026-05-05 |

## 變更歷程

| 日期 | 類型 | Base Commit 範圍 | 更新的文件 | 摘要 |
|------|------|-----------------|-----------|------|
| 2026-05-05 | full | initial..d6bee62 | 全部 | 初次 trace |

## Trace 範圍備註

專案規模：2,007 個檔案（TypeScript 1,149 + TSX 304 + 其他）

完整 trace 範圍：
- `server/` — Express API 伺服器（路由、服務、adapter、middleware）
- `ui/` — React 儀表板
- `cli/` — CLI 工具
- `packages/db/` — DB schema 與 migration
- `packages/shared/` — 共用型別與常數
- `packages/adapter-utils/` — Adapter 工具函式
- `packages/adapters/` — 所有 agent adapter
- `packages/plugins/sdk/` — Plugin SDK

部分 trace（檔案較多，僅涵蓋概覽）：
- `docs/` — Mintlify 公開文件（內容與 README / 現有文件多有重疊）
- `evals/` — LLM 評估腳本（promptfoo）
- `tests/e2e/` — E2E 測試實作細節
- `scripts/` — 建置與發布腳本細節
