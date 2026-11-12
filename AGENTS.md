# AGENTS.md — 後端與資料工程實務 LLM Wiki 規範 (Schema)

本文件是這座「後端、資料工程、雲端基建與跨職能協作實務知識庫」的核心操作規範（Schema）。
身為維護此知識庫的 LLM Agent，你必須嚴格遵守本文件的架構、格式與工作流程。

---

## 1. 核心理念與角色定位

- **持久化與複利累積 (Compounding Artifact)**：這不是一次性的 RAG 問答，而是一座隨時間不斷演進、交叉引用（Cross-referenced）的知識庫。每次消化新資料或回答深度問題，都要將知識編譯並整合進現有的 Wiki 網路中。
- **分工模式**：
  - **人類 (User)**：負責提供原始素材（文章、踩坑筆記、程式碼片段、架構想法）、決定關注焦點、在 Ingest 時確認方向、並透過 Obsidian 閱讀與探索。
  - **LLM (Agent)**：負責將使用者提供的素材歸檔至 `raw/`，並全權維護 `wiki/` 下的所有 Markdown 檔案（摘要、拆解概念、更新工具頁面、建立雙向連結、標記衝突、維護索引與日誌）。
- **實務導向 (Pragmatism First)**：拒絕空泛的教科書定義。所有內容必須聚焦於工程實務：**Trade-offs（取捨）**、**適用與不適用場景**、**常見坑點 (Gotchas / Anti-patterns)**、**配置與程式碼實例**、以及 **跨角色協作邊界**。

---

## 2. 目錄結構與三層架構

```text
.
├── AGENTS.md                  # 本規範文件 (Schema)
├── raw/                       # 【不可變原始層】所有外部輸入與原始素材（存入後 LLM 只讀不改）
│   ├── articles/              # 技術文章、部落格、官方文件、演講或論文節錄
│   ├── notes/                 # 使用者的實務隨手記、會議紀錄、ADR 草稿、線上事故原始紀錄
│   ├── snippets/              # 原始程式碼片段、Config 範本 (K8s/Terraform/SQL 等)
│   └── assets/                # 圖片、架構圖附件
└── wiki/                      # 【LLM 維護層】結構化、雙向連結的實務知識庫
    ├── index.md               # 全局內容目錄（依知識型態與領域分類，每次異動必更新）
    ├── log.md                 # 時間軸操作日誌（Append-only，紀錄每次 ingest / query / lint）
    ├── concepts/              # 核心概念、演算法與架構模式 (如：Consistent Hashing, Outbox Pattern, Medallion Architecture)
    ├── tools/                 # 技術棧與工具實務 (如：Kafka, PostgreSQL, Spark, Airflow, Kubernetes, Git)
    ├── playbooks/             # 實戰 SOP、規範、Git 版本控管、跨職能協作 (PM/FE/BE/Data/SRE/QA 職責分工與交付標準)
    ├── postmortems/           # 線上事故覆盤、效能瓶頸排查、真實踩坑案例 (Troubleshooting & Cases)
    ├── comparisons/           # 技術選型與對比分析 (如：Kafka vs RabbitMQ,バッチ vs ストリーミング)
    ├── presentations/         # 基於 Wiki 知識編譯之互動式簡報與教學投影片 (.html / Marp)
    └── sources/               # 對應 `raw/` 每份原始資料的結構化摘要頁 (Source Summaries)
```

### 原始資料自動歸檔規則 (`raw/`)
當使用者直接在對話中貼上文章、筆記、程式碼，或提供檔案時，LLM 應先依其性質決定放入 `raw/` 的哪個子目錄，並以 `YYYY-MM-DD-<kebab-case-slug>.md`（或對應副檔名）命名存檔：
- 外部技術文章 / 官方文件 ➔ `raw/articles/YYYY-MM-DD-<slug>.md`
- 個人實務筆記 / 踩坑紀錄 / 團隊規範草稿 ➔ `raw/notes/YYYY-MM-DD-<slug>.md`
- 純程式碼 / SQL / YAML / Config 範本 ➔ `raw/snippets/YYYY-MM-DD-<slug>.md`
- 圖片與架構圖 ➔ `raw/assets/`

> **鐵則**：一旦檔案寫入 `raw/`，即視為 **Immutable（不可變的 Source of Truth）**，後續絕對不可修改或刪除 `raw/` 內的舊檔案。

---

## 3. Wiki 頁面規格與寫作規範

### 3.1 檔案命名與連結格式 (Obsidian Style)
- **檔名**：統一使用英文小寫 `kebab-case.md`（例如：`kafka-consumer-rebalance.md`、`git-branching-strategy.md`）。只有 `sources/` 與 `postmortems/` 若具時間屬性可加上日期前綴 `YYYY-MM-DD-<slug>.md`。
- **內部連結**：全面採用 Obsidian 雙向連結語法 `[[folder/file-slug|顯示名稱]]` 或 `[[file-slug|顯示名稱]]`，積極在內文中串聯相關的概念、工具、Playbook 與踩坑案例，讓 Graph View 形成高密度的知識網路。

### 3.2 YAML Frontmatter 標準規格
每一篇 `wiki/` 下的內容頁面（除了 `index.md` 與 `log.md`）都**必須**包含以下 YAML Frontmatter（支援 Obsidian Dataview 查詢）：

```yaml
---
title: "頁面清楚的繁體中文標題"
type: concept | tool | playbook | postmortem | comparison | source
domain: [backend, data-eng]
tags: [kafka, message-queue, high-concurrency]
sources:
  - "[[sources/2026-11-12-example-source]]"
updated: 2026-11-12
status: evergreen | evolving | draft
---
```

**`domain` 允許的標準值（可複選）**：
- `backend`：後端系統架構、高併發、微服務、API 設計
- `data-eng`：資料工程、ETL/ELT、Batch/Streaming 管線、Data Lakehouse
- `database`：SQL/NoSQL 資料庫、索引調優、分片 (Sharding)、快取策略
- `infra`：雲端架構 (AWS/GCP)、K8s、Docker、Terraform、CI/CD、可觀測性
- `troubleshooting`：線上事故、效能排查、Postmortem 覆盤
- `workflow`：PM / Frontend / Backend / Data / DevOps(SRE) / QA 跨職能協作、Git 控管規範、工程文化

### 3.3 語言與內容結構要求
- **語言**：以**繁體中文（台灣工程社群慣用語）**撰寫，核心專有名詞與技術術語保留英文（例如：`Idempotency`、`Backpressure`、`Write-Ahead Log (WAL)`、`Consumer Group`）。
- **各類頁面必備結構**：
  - **`concepts/` & `tools/`**：核心解決什麼問題 ➔ 架構/運作原理 ➔ **Trade-offs 與限制** ➔ **實務最佳實踐 & 配置/程式碼範例** ➔ **常見坑點 (Gotchas)** ➔ 關聯頁面。
  - **`playbooks/`**：目標與適用情境 ➔ **各角色職責分工 (RACI / PM, FE, BE, Data, SRE, QA 邊界)** ➔ 標準操作流程 (SOP / Git 指令 / Checklist) ➔ 例外處理。
  - **`postmortems/`**：事故/問題背景與影響 ➔ 現象與指標異常 ➔ 排查過程 (Troubleshooting Trail) ➔ 根本原因 (Root Cause) ➔ 短期止血與長期改善 (Action Items)。
  - **`comparisons/`**：選型背景 ➔ 多維度對比表 (效能、維運成本、生態系、一致性等) ➔ **決策樹 (什麼場景選 A，什麼場景選 B)**。
- **觀點演進與衝突標記**：當新資料與既有 Wiki 內容產生矛盾，或實務做法隨版本演進而改變時，請使用 GitHub Alert 語法明確標示，切勿靜默覆蓋脈絡：
  > [!WARNING] 實務演進 / 觀點衝突
  > 過去在 [[...]] 提到做法 A，但根據最新來源 [[...]]，在高併發場景下做法 A 會導致...，建議改採做法 B。

---

## 4. 三大標準工作流 (Operations)

### 4.1 Ingest（半自動消化新資料流程）
當使用者提供新文章、筆記、程式碼或口頭分享實務經驗時，嚴格執行**兩階段半自動工作流**：

1. **階段一：原始歸檔與對齊提案 (Archive & Propose)**
   - 若使用者直接貼上內容，先依照「原始資料自動歸檔規則」將原始內容完整存入 `raw/<category>/YYYY-MM-DD-<slug>.md`。
   - 閱讀該份原始資料，並讀取 `wiki/index.md` 了解現有知識庫全貌。
   - **暫停寫入 `wiki/`**，先向使用者提出「Ingest 摘要與更新計畫」：
     1. **核心實務提煉**：列出 3~5 點該資料最關鍵的技術重點、Trade-offs 或踩坑結論。
     2. **預計變更清單**：
        - 新增哪篇 `wiki/sources/` 摘要頁？
        - 預計**新增**哪些概念/工具/Playbook/Postmortem 頁面？
        - 預計**更新**哪些既有頁面（補充哪個段落、或標註哪個衝突）？
     3. **提問與延伸**：若有模糊處或可結合使用者自身團隊現況的地方，提出 1~2 個簡短問題。
2. **階段二：編譯寫入與索引更新 (Compile & Index)**
   - 待使用者確認（或微調方向）後，一次完成所有檔案的建立與修改：
     - 建立 `wiki/sources/YYYY-MM-DD-<slug>.md`。
     - 建立或更新所有目標頁面（確保雙向連結 `[[...]]` 與 YAML Frontmatter 完整）。
     - 更新 `wiki/index.md`，將新頁面加入對應分類並更新一語道破的簡介。
     - 在 `wiki/log.md` 尾端追加一筆格式化的 `ingest` 紀錄。

### 4.2 Query（查詢與知識沉澱流程）
當使用者針對後端、資料工程、Git 或跨部門協作提問時：
1. 先讀取 `wiki/index.md`，定位相關的 Wiki 頁面並深入閱讀。
2. 綜合既有 Wiki 內容（必要時搭配外部知識，但須標明哪些來自知識庫、哪些是一般補充）給出結構化回答，並附上 `[[頁面連結]]` 引用。
3. **知識回寫 (File Back)**：如果這次問答產出了極具價值的成果（例如：完整的技術選型對比表、新的架構設計、跨職能 SOP、或是排查 Checklist），在回答結尾主動詢問使用者：「**這份分析/整理是否要歸檔進 Wiki（例如存入 `wiki/comparisons/...` 或 `wiki/playbooks/...`）？**」若使用者同意，即將其寫入 Wiki 並更新 `index.md` 與 `log.md`。

### 4.3 Lint（知識庫健檢流程）
當使用者要求進行 `lint` 或健檢時，全面掃描 `wiki/` 並回報：
1. **紅字連結 / 待建頁面 (Missing Pages)**：在內文中被 `[[...]]` 引用，但實際上還沒建立檔案的重要概念或工具。
2. **孤兒頁面 (Orphan Pages)**：沒有任何其他頁面連結過來的孤立頁面。
3. **矛盾與過時資訊 (Stale / Contradictions)**：不同頁面之間說法不一致、或缺乏 `sources` 支撐的論點。
4. **知識缺口建議 (Knowledge Gaps)**：根據目前 `index.md` 的分佈，建議下一個值得補強的主題或 Playbook。
5. 完成修復後，在 `wiki/log.md` 追加一筆 `lint` 紀錄。

---

## 5. 索引 (`index.md`) 與日誌 (`log.md`) 維護格式

### `wiki/index.md` 格式規範
保持清晰分類，每個條目包含雙向連結、領域標籤與一句話實務重點摘要：
```markdown
- [[tools/kafka|Apache Kafka]] `[backend, data-eng]` — 高吞吐分散式事件串流平台：涵蓋 Partition 設計、Rebalance 機制與 Exactly-once 實務。
```

### `wiki/log.md` 格式規範
採 **Append-only（由舊到新或固定附加於檔案末端）**，標題必須使用固定前綴以便透過 `grep "^## \[" wiki/log.md` 解析：
```markdown
## [YYYY-MM-DD] ingest | 資料標題或主題名稱
- **Raw Source**: `raw/articles/YYYY-MM-DD-slug.md`
- **新增頁面**: [[sources/YYYY-MM-DD-slug]], [[concepts/example]]
- **更新頁面**: [[tools/kafka]]
- **摘要**: 簡述本次整合的核心知識點。
```
（操作類型包含：`init`、`ingest`、`query`、`lint`）
