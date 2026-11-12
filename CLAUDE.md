# CLAUDE.md

本檔定義在此倉庫工作時必須遵守的架構約束與開發規範。功能與資料流的完整定義見 [SPEC.md](SPEC.md)；兩者衝突時以 SPEC.md 為準，並應同步修正本檔。

## 專案概述

Docs Workspace 是個人文件知識庫：Markdown 文件 CRUD、AI 非同步摘要與標籤、以自有文件為來源的 RAG 串流問答、對話中經核可的工具寫入，以及呈現 AI 管線延遲與檢索品質的透視面板。

技術選型：Next.js (App Router)、TypeScript、Tailwind CSS、Shadcn UI、Supabase (PostgreSQL + pgvector + Auth)、Drizzle ORM、Vercel AI SDK、Zod。

## 架構分層

```
進入點（Server Action / Route Handler）
        ↓
     Service（業務流程、交易邊界）
        ↓
  DAL（Drizzle） / AI（Prompt、工具、檢索）
```

- 相依方向單向向下，不得反向匯入或跨層呼叫。
- 進入點只做四件事：驗證身份、解析輸入、呼叫 Service、轉換回應。
- 只有 `src/server/dal/**` 可以匯入資料庫 client。
- 只有 `src/server/ai/**` 可以匯入 AI SDK 的模型呼叫。
- 同一項業務操作只有一條寫入路徑：UI 與工具呼叫共用同一個 Service 函式。

## 目錄慣例

| 路徑 | 內容 |
|------|------|
| `src/app/**` | 路由與頁面；預設為 Server Component |
| `src/features/<feature>/` | 該功能的 Client Components 與 Server Actions |
| `src/components/ui/` | Shadcn UI 元件，經 CLI 加入，不手動改寫結構 |
| `src/server/**` | 僅限伺服器的程式碼 |
| `src/lib/schemas/` | Zod 契約，前後端共用 |
| `src/lib/env/` | 環境變數驗證，`server.ts` 與 `client.ts` 分檔 |
| `drizzle/` | 產生的 SQL migration，納入版控 |

## TypeScript 與程式碼慣例

- `strict` 開啟；不使用 `any` 與非必要的型別斷言（`as`）。
- 型別由 Schema 推導（`z.infer`、Drizzle 的 `$inferSelect`），不另外手寫重複的 interface。
- 僅在需要互動或瀏覽器 API 時才標記 `"use client"`，並盡量下推至葉節點元件。
- 預期內的失敗（驗證錯誤、查無資料）以回傳值表達；例外只用於非預期狀況。
- 註解說明「為什麼」而非「做什麼」。

## 資料庫慣例

- Schema 變更流程：修改 `src/server/db/schema` → `drizzle-kit generate` → 檢視產生的 SQL → 套用。不對共用環境使用 `drizzle-kit push`。
- 已套用的 migration 不得修改，只能新增。
- 查詢一律明確選取欄位；列表查詢不取 `content`，任何查詢不取 `embedding`。
- 關聯資料以 JOIN 或聚合一次取回，不在迴圈中發出查詢。
- 分頁使用 keyset，不使用 `OFFSET`。
- 跨多表的寫入包在單一交易內。
- 新增查詢模式時以 `EXPLAIN ANALYZE` 確認使用到預期索引。

## AI 慣例

- 模型 ID 由環境變數提供，不寫死於程式碼。
- Prompt 集中於 `src/server/ai/prompts/`，以函式組裝，不散落在呼叫端。
- 需要機器可讀結果時使用結構化輸出並搭配 Schema，不以字串解析模型回應。
- 每次模型呼叫都要寫入一筆 `ai_traces`（延遲、Token 用量、檢索結果）。
- 代理迴圈必須設定步數上限。

## 提交規範

- Conventional Commits（`feat:`、`fix:`、`refactor:`、`docs:`、`chore:`）。
- 一個 commit 只做一件事；Schema 變更與其 migration 放在同一個 commit。
