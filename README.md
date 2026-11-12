# 後端與資料工程實務 LLM Wiki & 互動式教學簡報

本專案為基於 **LLM Wiki (Compounding Knowledge Artifact)** 架構建立之「後端、資料工程、雲端基建與跨職能協作實務知識庫」，並利用知識庫內容編譯出兩套互動式教學型 HTML 簡報（展示線性工作流 v1 與五階段 Graph 圖譜工作流 v2 的演進對比）。

---

## 🚀 線上互動教學簡報（GitHub Pages 入口）

1. **[📊 教學簡報 v1：經典 12 頁線性章節教學版 (`backend-onboarding-slides.html`)](./wiki/presentations/backend-onboarding-slides.html)**
   - **特色**：適合初學者循序漸進閱讀，包含 5 大核心模組、左側可收合章節目錄、頂部進度條、深淺色主題切換（`D` 鍵）與底部白話講者備忘錄（`N` 鍵）。
2. **[🌌 教學簡報 v2：14 節點知識星系圖譜雙模式版 (`backend-onboarding-graph-slides.html`)](./wiki/presentations/backend-onboarding-graph-slides.html)**
   - **特色**：經反思將檢查點前移（Shift-Left）並升級為五階段 Graph 工作流後產出。左側為 **Macro-Graph 互動知識星系圖**（高亮跨領域因果關係，如 Redis BigKey 如何跨領域導致 K8s HPA 擴縮容失靈），右側為 **Micro-Graph 全視覺化架構流程圖投影片**，支援 `Split` / `G` / `S` 三種檢視模式。

---

## 📚 知識庫與作業報告快速導覽

- **[📝 作業完整報告 (`homework-report.md`)](./homework-report.md)**：包含主題介紹、Wiki 與簡報對應表、Loop / Graph 流程圖與設計說明、實際執行 Gate Check 驗收紀錄、以及兩大真實踩坑反思。
- **[🗂️ LLM Wiki 全局索引 (`wiki/index.md`)](./wiki/index.md)**：依 6 大知識型態分類的 19 篇結構化 Wiki 頁面總覽。
- **[📜 時間軸操作日誌 (`wiki/log.md`)](./wiki/log.md)**：完整紀錄 `init`、`ingest`、`query` 與雙版本簡報編譯歷程。
- **[⚙️ 知識庫規範 Schema (`AGENTS.md`)](./AGENTS.md)**：定義三層架構、Obsidian 雙向連結、YAML Frontmatter 與兩階段半自動 Ingest 流程。

---

## 📂 專案目錄結構

```text
.
├── AGENTS.md                  # 知識庫核心規範 (Schema)
├── README.md                  # 專案首頁與快速連結導覽
├── homework-report.md         # 作業完整說明報告 (Markdown 版)
├── raw/                       # 【不可變原始層】7 篇外部收集之技術專文與事故紀錄
│   └── articles/
└── wiki/                      # 【LLM 維護層】結構化、雙向連結的實務知識庫
    ├── index.md               # 全局內容目錄
    ├── log.md                 # 時間軸操作日誌
    ├── concepts/              # 核心概念與架構模式 (Cache, Outbox, Medallion)
    ├── tools/                 # 技術棧底層原理與調優 (Git, Redis, PostgreSQL, K8s)
    ├── playbooks/             # 實戰 SOP 與跨職能協作規範 (Git 規範, PM/Eng/QA 協作)
    ├── postmortems/           # 真實線上事故覆盤 (Redis 20MB BigKey 連環爆)
    ├── comparisons/           # 技術選型對比與決策樹 (Docker/Swarm/K8s, Ingress/Gateway)
    ├── presentations/         # 互動式教學型 HTML 簡報 (v1 線性版 & v2 圖譜雙模式版)
    └── sources/               # 對應 raw/ 7 篇原始資料的結構化摘要頁
```
