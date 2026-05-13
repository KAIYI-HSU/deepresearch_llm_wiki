# LLM Wiki 引用（Citation）機制完整技術分析

> 分析對象：`nashsu/llm_wiki` v0.4.7
> 範圍：聊天回應的檢索 → 引用追蹤 → 持久化 → UI 呈現的完整生命週期
> 文件版本：2026-05-13

> 📚 **關聯文件**：
> - [`repo-analysis-zh-TW.html`](repo-analysis-zh-TW.html) — 整體架構、技術棧、功能總覽（請先閱讀）
> - [`README.md`](README.md) — 專案功能列表與安裝
> - [`llm-wiki.md`](llm-wiki.md) — Karpathy 原始設計模式
>
> 本文件聚焦 **citation 機制**的深度剖析，不重複架構描述；
> 若要了解四訊號相關性、兩階段攝入、Deep Research 流程等背景，請參閱上方關聯文件。

---

## 目錄

1. [Context (Chunk) 儲存管理](#1-context-chunk-儲存管理)
2. [去重複機制](#2-去重複機制)
3. [LLM 產生回應的 Citation 呈現格式](#3-llm-產生回應的-citation-呈現格式)
4. [Citation 相關的 LLM Prompt 內容](#4-citation-相關的-llm-prompt-內容)
5. [目前 Citation 機制的優缺點](#5-目前-citation-機制的優缺點)
6. [如何做到「精準 Citation」](#6-如何做到精準-citation)

---

## 1. Context (Chunk) 儲存管理

### 1.1 整體儲存模型

本專案的「可被引用內容」分為三層儲存：

| 層級 | 形式 | 位置 | 用途 |
|---|---|---|---|
| 原始層 | 文件檔 | `raw/sources/*` | 不可變的來源（PDF/DOCX/MD/網頁剪輯） |
| Wiki 層 | Markdown 頁 | `wiki/<type>/<slug>.md` | LLM 產出的結構化頁面，**引用單位** |
| 向量層 | LanceDB chunks | `.llm-wiki/lancedb/wiki_chunks_v2` | 語意檢索的最小單位（chunk） |

關鍵設計：**引用的最終單位是 Wiki 頁面，不是 chunk**。
chunks 僅作為「召回」候選人，最終 LLM 看到的、引用的、UI 顯示的都是完整 Wiki 頁。

### 1.2 Chunk 結構與切分

Chunks 由 `src/lib/text-chunker.ts` 的 `chunkMarkdown()` 產生，遵循 Markdown 感知的 Recursive Character Text Splitter 策略：

```typescript
interface Chunk {
  index: number          // 在頁面中的順序（0-based）
  text: string           // chunk 內容（不含 frontmatter、不含 heading 前綴）
  headingPath: string    // 標題麵包屑，例如 "## Intro > ### Usage"
  charStart: number      // 原始字串的字元 offset
  charEnd: number
  oversized: boolean     // 超過 maxChars 上限的標記
}
```

預設參數：
- `targetChars: 1000`（目標）
- `maxChars: 1500`（硬上限）
- `minChars: 200`（合併過短的相鄰 chunks）
- `overlapChars: 200`（相鄰 chunks 重疊 200 字元，避免語意斷裂）

**切分優先順序（6 級）：**
1. Heading 區段（`##` / `###` / `####`）
2. 段落邊界（`\n\n`）
3. 換行（`\n`）
4. 句末標點（`. 。! ！? ？; ；`）
5. 空白（` 　\t`）
6. 硬切（最後手段）

**保護規則：**
- ☑ **絕不切開 fenced code block**（```...``` 視為原子單元）
- ☑ **絕不切開 markdown 表格**（連續 `|` 開頭視為原子單元）
- ☑ **切分前 YAML frontmatter 會被剝離**，不污染 embedding

### 1.3 LanceDB v2 Schema（per-chunk 儲存）

每個 chunk 在 LanceDB 中佔一列，定義於 `src-tauri/src/commands/vectorstore.rs:332-348`：

```rust
fn make_schema_v2(dim: i32) -> Arc<Schema> {
    Arc::new(Schema::new(vec![
        Field::new("chunk_id",     DataType::Utf8,  false),  // "{page_id}#{chunk_index}"
        Field::new("page_id",      DataType::Utf8,  false),
        Field::new("chunk_index",  DataType::UInt32, false),
        Field::new("chunk_text",   DataType::Utf8,  false),
        Field::new("heading_path", DataType::Utf8,  false),
        Field::new("vector",
            DataType::FixedSizeList(Float32, dim), false),
    ]))
}
```

`chunk_id` 永遠由後端自動生成為 `{page_id}#{chunk_index}`，前端不能自行決定，避免 ID 注入攻擊。

### 1.4 Chunk 寫入策略：Delete-then-Add（非真正 upsert）

`vector_upsert_chunks` 的語意是「**先刪除整個 page_id 的所有 chunks，再批次寫入新的**」：

```rust
table.delete(&format!("page_id = '{}'", page_id)).await
table.add(data).execute().await
```

**理由（程式註解直翻）：**
> Chunk indexes may shift when a page is re-ingested (content shrinks /
> grows / re-splits), so we never try to match-and-update by chunk_id —
> the clean-slate replace is both simpler and always correct.

這個 clean-slate 策略確保：
- ✅ 不會殘留舊版 chunks 污染搜尋結果
- ✅ 不會因 chunk index 偏移造成「同一段內容對應兩個 chunk_id」的幻影
- ⚠️ 代價：每次更新都是全量替換，沒有增量

### 1.5 Embedding 文字組成（重要！）

寫入 LanceDB 的不是 chunk 原文，而是「**enriched 版本**」(`src/lib/embedding.ts:248-257`)：

```typescript
function enrichChunkForEmbedding(pageTitle, chunk): string {
  const parts = []
  if (pageTitle.trim()) parts.push(pageTitle.trim())
  if (chunk.headingPath.trim()) parts.push(chunk.headingPath.trim())
  parts.push(chunk.text.trim())
  return parts.join("\n\n")
}
```

亦即 embed 的內容是：
```
<頁面標題>

## A > ### B    ← heading path 麵包屑

<chunk 正文>
```

**這個設計直接決定了引用準確度**：短的 chunk 若沒有標題上下文，embedding 會發散；加入 heading path 後，「Mixture of Experts」相關的 300 字段落能準確被相關 query 召回。

### 1.6 Page-Level Score Aggregation（chunk → page）

LLM 不直接看 chunk，而是看完整 page。聚合邏輯在 `src/lib/embedding.ts:420-438`：

```typescript
// 1. 從 LanceDB 取 top-K × 3 個 chunks
// 2. 依 page_id 分組
// 3. 每組計算 blended score：
const top  = chunks[0].score                          // 最強 chunk 分數
const tail = chunks.slice(1).reduce((s, c) => s + c.score, 0)
const blended = top + Math.min(tail * 0.3, Math.max(0, 1 - top))
//                              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                              其餘 chunks 的貢獻被 capped 在 1.0 - top 以下
```

**設計考量**：避免「多個弱 chunks」壓過「單一強 chunk」。如果一個頁面只有一個高分 chunk（top=0.9），其他都是低分（tail=2.0），blended 不會爆到 0.9 + 0.6 = 1.5，而是被 cap 在 0.9 + 0.1 = 1.0。

---

## 2. 去重複機制

本專案存在 **5 個不同層級的去重**，分別對應不同階段：

### 2.1 來源檔層級：SHA256 增量快取

`src/lib/ingest-cache.ts` 在每次 ingest 前計算來源檔的 SHA256，若 hash 與上次相同則整段 ingest 略過：

```
raw/source.pdf  → SHA256(content) → 比對 .llm-wiki/ingest-cache.json
  └ 相同 → 整段流程 skip（含兩階段 LLM 呼叫、embedding）
  └ 不同 → 重新跑全部
```

避免重複攝入造成的：
- LLM 重複呼叫成本
- Wiki 內容被覆寫
- LanceDB chunks 被白白 delete-then-add

### 2.2 Chunk 層級：自動清理 stale chunks

如 §1.4 所述，page 重新攝入時所有舊 chunks 被 `table.delete(page_id = '...')` 整批刪除後重建。
這個機制天然防止「同一段內容存兩份」。

### 2.3 搜尋結果層級：RRF（Reciprocal Rank Fusion）

主聊天的 token search 與 vector search 兩個獨立排名清單合併時，用 RRF（K=60）融合，定義於 `src/lib/search.ts:38-53`：

```typescript
// fused(p) = Σ over lists L of  1 / (K + rank_L(p))
//
// K=60 是 Cormack et al. (SIGIR 2009) 的標準常數。
const RRF_K = 60

for (const r of results) {
  const tRank = tokenRank.get(normalizePath(r.path))
  const vRank = vectorRank.get(getFileStem(r.path))
  let rrf = 0
  if (tRank !== undefined) rrf += 1 / (RRF_K + tRank)
  if (vRank !== undefined) rrf += 1 / (RRF_K + vRank)
  r.score = rrf
}
```

**RRF 隱含的去重**：兩個清單若都包含同一個 page，會自然「加權合併」而不是「列兩次」。
重複只會強化分數，不會產生重複結果。

### 2.4 Frontmatter 層級：sources / tags / related 的 Union Merge

當 LLM 重新攝入時，產出的新頁面 frontmatter 必須與既有頁面**聯集合併**而非覆寫，避免「先前 source 被靜默刪除」。實作於 `src/lib/sources-merge.ts:145-170`：

```typescript
export function mergeArrayFieldsIntoContent(
  newContent, existingContent,
  fields: readonly string[],  // 例如 ["sources", "tags", "related"]
): string {
  // 對每個欄位：
  //   oldValues = parseFrontmatterArray(existingContent, field)
  //   newValues = parseFrontmatterArray(result, field)
  //   merged    = mergeLists(oldValues, newValues)  // case-insensitive dedup
  //   result    = writeFrontmatterArray(result, field, merged)
}
```

**這是引用準確度的關鍵保障**：一個 entity 頁可能被 5 個來源共同提及，frontmatter 的 `sources: [...]` 必須累積這 5 個來源，不能因第 6 次攝入只看到一個來源就把前 5 個沖掉。

### 2.5 Deep Research 層級：URL Set 去重

Deep Research 的 3 個 search queries 各取 5 筆結果，用 `Set<string>` 以 URL 去重（`src/lib/deep-research.ts:73-91`）：

```typescript
const allResults: WebSearchResult[] = []
const seenUrls = new Set<string>()

for (const query of queries) {
  const results = await webSearch(query, searchConfig, 5)
  for (const r of results) {
    if (!seenUrls.has(r.url)) {
      seenUrls.add(r.url)
      allResults.push(r)
    }
  }
}
```

---

## 3. LLM 產生回應的 Citation 呈現格式

本專案有**兩條獨立的引用呈現路徑**，依使用情境不同：

### 3.1 主聊天路徑（Wiki 內部問答）

**LLM 看到的內容**（system prompt + user message）：

```
## Wiki Pages

### [1] Constitutional AI
Path: wiki/concepts/constitutional-ai.md

<完整頁面內容>

---

### [2] Anthropic
Path: wiki/entities/anthropic.md

<完整頁面內容>

---

### [3] RLHF
...
```

**LLM 回應要求的格式**（嚴格定義於 system prompt）：

```
正文中使用 [1], [2], [3] 標註引用
正文中使用 [[wikilink]] 連結到 wiki 頁面
回應最後加上隱藏註解：
<!-- cited: 1, 3, 5 -->
```

實際範例輸出：
```markdown
Anthropic 提出的 Constitutional AI [1] 採用了一套規則導向的訓練方法，
與傳統 [[rlhf]] 不同 [2]。其核心精神是讓模型自我修正 [1][3]...

<!-- cited: 1, 2, 3 -->
```

### 3.2 Deep Research 路徑（網路研究）

**LLM 看到的內容**：

```
## Web Search Results

[1] **Constitutional AI: Harmlessness from AI Feedback** (arxiv.org)
<完整內容>

[2] **Anthropic releases Claude 3** (anthropic.com)
<完整內容>

...
```

**LLM 回應要求的格式**：
```
正文中使用 [N] 標註外部來源
正文中使用 [[wikilink]] 連結到既有 wiki 頁面
（不需要 <!-- cited: --> 註解）
```

**程式自動產生的 References 章節**（不是 LLM 寫的）：

```typescript
// deep-research.ts:173-175
const references = webResults
  .map((r, i) => `${i + 1}. [${r.title}](${r.url}) — ${r.source}`)
  .join("\n")
```

最終存到磁碟的研究頁：

```markdown
---
type: query
title: "Research: Constitutional AI"
created: 2026-05-13
origin: deep-research
tags: [research]
---

# Research: Constitutional AI

<LLM 合成的本文，含 [N] 與 [[wikilink]]>

## References

1. [Constitutional AI: Harmlessness...](https://arxiv.org/abs/2212.08073) — arxiv.org
2. [Anthropic releases Claude 3](https://www.anthropic.com/...) — anthropic.com
```

### 3.3 兩種格式的對照表

| 維度 | 主聊天 | Deep Research |
|---|---|---|
| 引用編號來源 | 內部 wiki 頁面（[1]..[N]） | 外部網路結果（[1]..[N]） |
| 隱藏的「cited」註解 | 有：`<!-- cited: 1,3,5 -->` | 無 |
| Wikilink 用途 | 連結到引用以外的相關頁 | 將新研究焊回既有圖譜 |
| References 區塊 | 無（編號即引用） | 有（程式產生） |
| 持久化欄位 | `message.references[]`（chat-store） | 直接寫入 markdown |

### 3.4 Citation 解析與 UI 呈現

主聊天訊息底下的「Cited References Panel」由 `extractCitedPages()` 解析（`chat-message.tsx:550-572`），採**三層 fallback**：

```typescript
// 優先：隱藏註解
const citedMatch = text.match(/<!--\s*cited:\s*(.+?)\s*-->/)
if (citedMatch) {
  const numbers = citedMatch[1].split(",").map(n => parseInt(n))
  return numbers.map(n => lastQueryPages[n - 1])
}

// Fallback 1：正文裡的 [1][2] 等編號標記
const numberRefs = text.match(/\[(\d+)\]/g)
if (numberRefs) {
  const numbers = [...new Set(numberRefs.map(r => parseInt(r.slice(1, -1))))]
  return numbers.map(n => lastQueryPages[n - 1])
}

// Fallback 2：[[wikilink]] — 用於持久化訊息已遺失 lastQueryPages 時
const wikilinks = text.match(/\[\[([^\]|]+?)(?:\|[^\]]+?)?\]\]/g)
if (wikilinks) { /* 用 wiki 子目錄常規路徑解析 */ }
```

訊息物件本身也持久化引用（`src/stores/chat-store.ts:22`）：

```typescript
interface DisplayMessage {
  id: string
  role: "user" | "assistant" | "system"
  content: string
  timestamp: number
  conversationId: string
  references?: MessageReference[]  // 在串流結束時被寫入
}
```

`finalizeStream(content, references)` 在 LLM 回應完成時把當前查詢的 pages 直接存入訊息物件，避免「重啟後 lastQueryPages 已遺失」的問題。

---

## 4. Citation 相關的 LLM Prompt 內容

### 4.1 主聊天 System Prompt（`chat-panel.tsx:311-341`）

```
You are a knowledgeable wiki assistant. Answer questions based on the
wiki content provided below.

## Rules
- Answer based ONLY on the numbered wiki pages provided below.
- If the provided pages don't contain enough information, say so honestly.
- Use [[wikilink]] syntax to reference wiki pages.
- When citing information, use the page number in brackets, e.g. [1], [2].
- At the VERY END of your response, add a hidden comment listing which
  page numbers you used:
  <!-- cited: 1, 3, 5 -->

Use markdown formatting for clarity.

## Wiki Purpose
<purpose.md 內容>

## Wiki Index
<wiki/index.md 內容，依 query relevance 修剪過>

## Page List
[1] Constitutional AI (wiki/concepts/constitutional-ai.md)
[2] Anthropic (wiki/entities/anthropic.md)
...

## Wiki Pages

### [1] Constitutional AI
Path: wiki/concepts/constitutional-ai.md

<完整頁面內容>
...

---

## ⚠️ MANDATORY OUTPUT LANGUAGE: zh-TW

You MUST write your entire response in **zh-TW**.
The wiki content above may be in a different language, but this is
IRRELEVANT to your output language. Ignore the language of the wiki
content. Write in zh-TW only.
```

**注意三個關鍵指令：**

1. **"Answer based ONLY on the numbered wiki pages"** — 嚴格禁止 LLM 引用未提供的內容
2. **"At the VERY END... <!-- cited: 1, 3, 5 -->"** — 強制末尾隱藏註解，給 UI 解析用
3. **語言指令在 prompt 最後重複強調** — Wiki 內容可能是英文，回應必須繁中

### 4.2 Deep Research Synthesis Prompt（`deep-research.ts:114-133`）

```
You are a research assistant. Synthesize the web search results into a
comprehensive wiki page.

<language directive>

## Cross-referencing (IMPORTANT)
- The wiki already has existing pages listed in the Wiki Index below.
- When your synthesis mentions an entity or concept that exists in the
  wiki, ALWAYS use [[wikilink]] syntax to link to it.
- For example, if the wiki has an entity 'anthropic', write [[anthropic]]
  when mentioning it.
- This is critical for connecting new research to existing knowledge in
  the graph.

## Writing Rules
- Organize into clear sections with headings
- Cite web sources using [N] notation
- Note contradictions or gaps
- Suggest additional sources worth finding
- Neutral, encyclopedic tone

## Existing Wiki Index (link to these pages with [[wikilink]])
<wiki/index.md 內容>
```

注意這裡的 Cross-referencing 是 **大寫 IMPORTANT** 標記為「critical」，明確指示 LLM「研究結果必須焊回現有圖譜」。

### 4.3 Ingest Step 2 Generation Prompt（`ingest.ts:1006-1100`）

兩階段攝入的「生成」階段對 frontmatter 的格式有極為嚴格的 prompt 約束：

```
## Frontmatter Rules (CRITICAL — parser is strict)

Every page begins with a YAML frontmatter block. Format rules:

1. The VERY FIRST line of the file MUST be exactly `---`.
2. Each frontmatter line is a `key: value` pair on its own line.
3. The frontmatter ends with another `---` line on its own.
5. Arrays use the standard YAML inline form `[a, b, c]`.
   Wikilinks belong in the BODY only — never write `related: [[a]], [[b]]`
   (invalid YAML); write `related: [a, b]` with bare slugs.

Required fields and types:
  • type     — one of: source | entity | concept | comparison | query | synthesis
  • title    — string
  • created  — date in YYYY-MM-DD form
  • updated  — same as created
  • tags     — array of bare strings
  • related  — array of bare wiki page slugs (no `wiki/`, no `.md`, no `[[…]]`)
  • sources  — array of source filenames; MUST include "<source_file>".
```

此 prompt 透過「明確命令 + 反例提示」雙管齊下，讓 LLM 不會把 wikilink 寫進 frontmatter（這是 YAML 解析錯誤的最常見來源）。

### 4.4 三個 Prompt 的 Citation 策略對照

| Prompt | Citation 編號 | Wikilink | 持久化標記 | 強制程度 |
|---|---|---|---|---|
| 主聊天 | `[N]` for 內部 wiki 頁 | 補充用 | `<!-- cited: ... -->` | **強制** |
| Deep Research | `[N]` for 外部網路 | **強制** | `## References` 章節 | **強制 wikilink** |
| Ingest 生成 | 無 | 補充用 | `sources: [...]` frontmatter | **強制 sources 欄位** |

---

## 5. 目前 Citation 機制的優缺點

### 5.1 優點

#### ✅ 多層備援的解析機制
`extractCitedPages()` 採用「隱藏註解 → [N] 編號 → [[wikilink]]」三層 fallback，
即使 LLM 漏寫了 `<!-- cited: -->` 或訊息被持久化後 `lastQueryPages` 已失效，
仍能從 markdown 內容反向推斷引用來源。

#### ✅ References 由程式產生，杜絕 URL 幻覺
Deep Research 的 References 章節由 `webResults.map(...)` 自動產生，
**LLM 不負責寫 URL**，徹底避免 hallucinated citation 這個業界常見問題。

#### ✅ 引用會反向焊回圖譜
透過 `sources: []` frontmatter 的 union merge，研究結果不只是「掛在訊息上的標籤」，
而是真正成為實體頁的來源證據，並驅動 4-signal relevance 中最重的 ×4.0 source-overlap 邊。

#### ✅ Chunk 召回但頁面引用
向量索引切到 chunk 粒度做語意召回，但給 LLM 看的是完整頁面、引用單位是頁面。
避免了「LLM 引用了一個破碎 chunk，使用者點進去發現找不到對應段落」的常見尷尬。

#### ✅ 訊息層持久化
`DisplayMessage.references[]` 隨訊息存入聊天紀錄，重啟後依然可解析顯示。

#### ✅ Stale chunk 自動清除
Page 重新攝入時 LanceDB 採 delete-then-add 整批替換，
避免「同一段內容多版本並存」造成的引用混亂。

### 5.2 缺點

#### ❌ Citation 粒度太粗：頁面層，無法定位段落
LLM 引用「[1]」意指整個頁面，但頁面可能有數千字。使用者點進去後仍需自己找段落。
雖然 LanceDB 有 chunk_text + heading_path，但這個資訊**沒被傳遞到 LLM 的引用語意中**。

#### ❌ 多來源引用容易遺漏
若一個觀點來自頁面 [1][3][5] 三個頁面，LLM 可能只寫 `[1]` 而漏掉 [3][5]。
目前 prompt 沒有強制「列出所有支持來源」的指令，也沒有後驗證機制。

#### ❌ `<!-- cited: -->` 註解信任 LLM 自報
LLM 可能：
- 漏寫該註解
- 列了沒實際引用的編號
- 編了超出範圍的編號（雖然有 filter 但會靜默丟棄）

#### ❌ Wikilink 引用無法驗證對應頁面存在
LLM 寫 `[[non-existent-entity]]` 時不會被即時擋下，只能等 Lint 階段檢出。
這對於 Deep Research 寫進 wiki 的內容尤其有風險（可能引入幻覺實體）。

#### ❌ 引用不區分「主要支持」與「輔助參考」
所有引用同等對待，但實際上同一段論述可能有「強支持來源」+「弱關聯來源」。
目前格式 `[1][3]` 無法表達這個強度差異。

#### ❌ 無法引用到非 markdown 內容
Wiki 中的圖片（透過 vision caption 處理）雖然被嵌入到 markdown，
但**引用粒度仍是頁面，不到圖片**。LLM 若是基於某張圖的描述回答，使用者很難回溯。

#### ❌ Chunk-to-page 聚合丟失了「最相關段落」資訊
雖然 `searchByEmbedding` 內部回傳了 `matchedChunks`（前 3 個最強 chunk），
但目前主聊天 pipeline 沒消費這個資訊，
LLM 看到的還是「整頁全文」，沒有「重點段落標記」。

#### ❌ 引用 source 的 deduplication 是案例敏感的
`mergeLists` 是 case-insensitive 去重（first-seen casing 勝），
但檔名 `Foo.pdf` vs `foo.pdf` 在不同檔案系統表現不一，可能造成 macOS（不分大小寫）vs Linux（分大小寫）的不一致。

### 5.3 總結評估表

| 面向 | 表現 | 評等 |
|---|---|---|
| URL 幻覺防護 | 程式產生 References，完全杜絕 | A+ |
| 引用持久化 | 訊息層 + frontmatter 雙重備援 | A |
| 解析穩健性 | 三層 fallback | A |
| 引用粒度 | 頁面層，缺少段落級定位 | C |
| 多來源完整性 | 依賴 LLM 自覺，無強制 | C |
| 後驗證 | 無 dead-wikilink 檢測 | C- |
| 圖片引用 | 不支援 | D |
| 強弱來源區分 | 不支援 | D |

---

## 6. 如何做到「精準 Citation」

> 「精準引用」 = ① 不遺漏多來源 + ② 不引用幻覺內容 + ③ 引用粒度可追溯。

以下提出針對性的改進設計，分為「可在現有架構漸進加入的改良」與「需要架構性調整的方案」兩類。

### 6.1 漸進式改進（不破壞現有 API）

#### 改進 1：強制 LLM 列出「所有」支持來源

修改主聊天 system prompt 加入：

```diff
+ ## Multi-source citation rule (CRITICAL)
+ - When a claim is supported by MULTIPLE pages, you MUST cite ALL of them
+   in the order of strongest to weakest support.
+ - Example: "Constitutional AI uses RLHF [1][3][5]" — list all 3 even if [1]
+   is the primary source.
+ - If you cannot find supporting pages for a claim, do NOT make the claim.
+   State explicitly: "The provided pages do not cover this."
```

#### 改進 2：把 chunk 層級的「段落定位」傳遞給 LLM

利用 `searchByEmbedding` 已回傳的 `matchedChunks`，在組裝 system prompt 時加入：

```
### [1] Constitutional AI
Path: wiki/concepts/constitutional-ai.md
Most relevant section: "## Training Process > ### Self-Critique"

<完整頁面內容>
```

並修改 prompt：

```
- When citing [N], if the cited page has a "Most relevant section" hint,
  prefer to also include that section path: e.g. [1: Training Process].
```

UI 端解析 `[N: Section]` 後可直接 deep-link 到對應 anchor。

#### 改進 3：Dead-wikilink 即時偵測

在 `finalizeStream` 或 `cleanedSynthesis` 寫入磁碟前，掃描所有 `[[wikilink]]`，
比對 wiki 子目錄中的真實檔案，產生：

```typescript
interface CitationDiagnostics {
  unresolvedWikilinks: string[]   // LLM 引用了不存在的頁面
  numericOutOfRange: number[]     // [N] 編號超過實際提供頁面數
  missingCitations: string[]      // 出現了 [[wikilink]] 但無對應 [N]
}
```

UI 在訊息底下用警示 chip 顯示，使用者一眼可見「LLM 引用了 X 個不存在的頁面」。

#### 改進 4：引用強度分級

讓 LLM 用 `[N!]`（強支持）/ `[N?]`（推測 / 部分相關）區分：

```
Constitutional AI 是 Anthropic 提出 [1!] 的訓練方法 [2!]，
可能受到 [3?] 的啟發。
```

UI 端對應顯示「★」/「☆」icon。

### 6.2 架構性改進

#### 方案 A：Chunk-Level Citation（段落級引用）

**目標**：LLM 引用單位由「頁面」改為「(頁面, heading-path)」對。

實作：
1. 修改 `searchByEmbedding` 不做 page-level aggregation，直接回傳 chunks
2. System prompt 提供 chunks 而非整頁：
   ```
   ## Chunks
   ### [1] from "Constitutional AI" > "## Training" > "### Self-Critique"
   <chunk_text>

   ### [2] from "Constitutional AI" > "## Background"
   <chunk_text>
   ```
3. UI 端 `extractCitedPages` 改為 `extractCitedChunks`，直接 deep-link 到 heading

**權衡**：上下文長度可能膨脹（同一頁面被切多個引用），但引用粒度顯著提升。

#### 方案 B：兩階段 Generation with Citation Verification

**目標**：在 LLM 生成完成後，跑第二次 LLM 驗證引用一致性。

```
Step 1: LLM 產出回應與 [1][2] 引用
Step 2: 驗證 LLM 接收 (回應正文, 所有來源頁) → 輸出
  {
    "verified": [
      { "claim": "...", "supportingPages": [1, 3], "originalCitation": [1] },
      { "claim": "...", "supportingPages": [], "issue": "no_support" }
    ]
  }
Step 3: 把驗證結果合併入訊息 (引用補全 / 警告幻覺)
```

**權衡**：double LLM 成本（與本專案的兩階段 ingest 相同精神），但能達到「不遺漏多來源」的目標。

#### 方案 C：Self-Citation by Embedding Similarity

**目標**：對 LLM 產出的每個句子，事後用 embedding 找其最相似的 chunks，自動補完引用。

```
for each sentence S in LLM_response:
  S_emb = fetchEmbedding(S)
  top_chunks = vector_search(S_emb, k=3, filter: page_id IN cited_pages)
  if max_similarity > threshold:
    append [N: heading_path] to S
```

**權衡**：每句一個 embedding 呼叫成本不小（一段回應可能 20 句），
但完全 deterministic、不靠 LLM 自報。可作為**驗證層**而非強制層。

### 6.3 推薦的實作優先順序

按「對引用品質提升 / 實作成本」排序：

| 優先級 | 改進 | 預期效益 | 預估工作量 |
|---|---|---|---|
| P0 | 改進 3：dead-wikilink 即時偵測 | 高（杜絕幻覺鏈接） | 1–2 天 |
| P0 | 改進 1：強制多來源引用 prompt | 中高（降低遺漏） | 0.5 天 |
| P1 | 改進 2：matchedChunks 段落提示 | 高（粒度提升） | 2–3 天 |
| P2 | 改進 4：引用強度分級 | 中（UX 改善） | 1 天 |
| P3 | 方案 A：Chunk-level citation | 極高（粒度顯著提升） | 1–2 週（破壞 API） |
| P3 | 方案 B：兩階段驗證 | 極高（最高品質） | 1 週（雙倍成本） |
| P4 | 方案 C：Embedding 自動補引 | 中（驗證層） | 1 週 |

### 6.4 最小可行改進（一週內可上線）

若資源有限，建議**先做 P0 兩項 + 改進 2 的子集**：

1. 在 `extractCitedPages` 後新增 `validateCitations()` 函式（dead-wikilink + out-of-range）
2. 在主聊天 system prompt 末段加入「多來源強制列出」段落
3. 在 system prompt 的 `### [N] Title` 區塊加入 `Most relevant section: <heading_path>` 行，
   並指示 LLM 「優先在 `[N]` 後標註 section」

預估這套組合可將引用準確度提升明顯，且不破壞現有的訊息持久化結構。

---

## 結語

本專案的引用機制設計遵循「**LLM 負責創造、程式負責驗證**」的分工：

- LLM 寫的 `[N]` 編號 → 程式對應到實際 page 物件
- LLM 寫的 `[[wikilink]]` → 程式（autoIngest 後）建立 4-signal 圖譜邊
- LLM 寫的 References URL → 不存在；URL 由 webResults 自動產生

這個分工的精神值得肯定，但目前實作止於「**頁面級引用 + 文字級驗證**」。
若要進一步達到學術級的「精準引用」，需要：
1. **段落級的 chunk citation**（方案 A）
2. **後驗證的引用一致性檢查**（方案 B）
3. **embedding 補強的自動補引**（方案 C）

三條路徑彼此互補，可依專案資源逐步落實。

---

> 📌 本文件以 `nashsu/llm_wiki` v0.4.7 為分析對象。
> 所有程式碼引用皆來自實際 source code（位置標於每段註解）。
