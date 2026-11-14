# CLAUDE.md

本檔定義在此倉庫工作時必須遵守的架構約束與開發規範。功能與資料流的完整定義見 [SPEC.md](SPEC.md)；兩者衝突時以 SPEC.md 為準，並應同步修正本檔。

## 專案概述

Rummage 是個人知識庫，第一版以 Markdown 文件為核心：文件 CRUD、AI 非同步摘要與標籤、以自有文件為來源的 RAG 串流問答、對話中經核可的工具寫入，以及呈現 AI 管線延遲與檢索品質的透視面板。

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

## 系統邊界與安全紅線

### 信任邊界

伺服器是唯一受信任的執行環境。以下來源一律視為不受信任，進入系統前必須經 Schema 驗證：

| 來源 | 說明 |
|------|------|
| 瀏覽器輸入 | 表單、JSON body、URL 參數、Header、Cookie 內容。Server Action 等同公開的 POST 端點，可被任意參數直接呼叫 |
| 模型輸出 | 文字、結構化物件、工具呼叫參數 |
| 檢索內容 | 放入 Prompt 的文件片段，可能挾帶提示注入 |

客戶端的驗證、禁用狀態與路由守衛僅為使用者體驗，不構成任何安全保證。

### 授權

- 使用者身份只取自伺服器端驗證簽章後的 Session（`supabase.auth.getUser()` / `getClaims()`），不信任未驗證的 Session 內容。
- `ownerId` 只能由伺服器端 Session 傳入 DAL；不得出現在任何輸入 Schema（表單、API body、URL、工具參數）中。
- DAL 中針對單一資源的讀取、更新、刪除，查詢條件必須同時包含 `id` 與 `owner_id`。
- 非擁有者存取與資源不存在回傳相同結果（404），不洩漏資源是否存在。
- 所有業務資料表啟用 RLS，政策為 `owner_id = auth.uid()`。RLS 為縱深防禦，不得以其取代 DAL 的擁有者條件——Drizzle 以資料庫高權限角色連線，不經 RLS。
- 不使用 Supabase service-role / secret key。

### 機密管理

- `NEXT_PUBLIC_` 前綴的變數會在建置時以字面值內嵌於客戶端 bundle，視同公開。只有「外洩後仍有其他防線」的值可使用此前綴（例：受 RLS 約束的 Supabase publishable key）。
- 資料庫連線字串、模型 API key、Worker 密鑰不得使用 `NEXT_PUBLIC_` 前綴。
- `src/server/**` 下每個模組以 `import 'server-only'` 開頭，使客戶端誤匯入時建置失敗。
- 環境變數只透過 `src/lib/env/server.ts` 與 `src/lib/env/client.ts` 存取，啟動時以 Zod 驗證；其他位置不得直接讀取 `process.env`。
- 機密不得寫入日誌、錯誤訊息、`ai_traces` 或回傳給客戶端。

### 輸入契約

- 每個進入點在呼叫 Service 前以 Zod `parse` / `safeParse` 驗證輸入；未通過驗證的資料不得往下傳遞。
- 寫入契約採白名單：以 `z.object` 宣告允許的欄位，未宣告的鍵在解析時剔除；不得使用 `.passthrough()` 或將原始輸入展開（`...input`）後寫入。
- 契約集中於 `src/lib/schemas/`，型別以 `z.infer` 推導；表單驗證、Server Action、工具 `inputSchema`、結構化輸出共用同一份契約。
- 契約同時約束長度與數量上限，作為成本與濫用防護的第一道關卡。

### AI 邊界

- 檢索內容以明確分隔標記包覆，並在系統 Prompt 中宣告為參考資料而非指令。
- 寫入型工具須經使用者在 UI 核可後才執行。
- 工具的操作範圍以當前 Session 使用者為界；工具內部呼叫與 UI 相同的 Service 函式，不另開寫入路徑。
- 代理迴圈設定步數上限；AI 端點設定輸入長度上限與每使用者請求頻率限制。
- 背景 Worker 端點以共享密鑰驗證呼叫來源。

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

## 測試慣例

- 新增或修改 DAL 函式時，同一變更內必須包含跨使用者授權測試：以另一使用者身份操作，斷言讀取查無資料、更新與刪除不影響任何資料列。
- 涉及資料庫行為（擁有者條件、外鍵、交易、RLS、併發領取工作）的測試，對本機 Supabase 的真實 PostgreSQL 執行，不以 mock 取代。
- 測試不得連線 Production 或 Preview 資料庫；測試設定只讀取本機資料庫連線字串。
- 測試不呼叫外部模型 API，以固定回應替換。
- 修改切塊參數、Top-K、相似度門檻、Embedding 模型或 Prompt 後，執行評估（`evals/`）並將結果納入同一變更；品質下降需在變更說明中解釋取捨。
- Bug 修正先加入能重現問題的失敗測試，再修正。

## 提交規範

- Conventional Commits（`feat:`、`fix:`、`refactor:`、`docs:`、`chore:`）。
- 一個 commit 只做一件事；Schema 變更與其 migration 放在同一個 commit。
