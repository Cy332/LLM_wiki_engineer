# Wiki 演進日誌 (Changelog)

> 本檔案為 Append-only 時間軸紀錄。每筆紀錄標題格式固定為 `## [YYYY-MM-DD] <action> | <title>`，方便使用 `grep "^## \[" wiki/log.md` 快速檢索。

## [2026-11-12] init | 初始化後端與資料工程實務 LLM Wiki
- **操作內容**: 建立 [`AGENTS.md`](../AGENTS.md) 核心規範、`raw/` 四大原始素材分類目錄（`articles`, `notes`, `snippets`, `assets`）與 `wiki/` 六大知識分類目錄（`concepts`, `tools`, `playbooks`, `postmortems`, `comparisons`, `sources`）。
- **領域設定**: 涵蓋 `backend`、`data-eng`、`database`、`infra`、`troubleshooting`、`workflow`（跨職能協作與 Git 控管）。

## [2026-11-12] ingest | 後端工程師的第二堂課 (1) 進階 Git
- **Raw Source**: `raw/articles/2026-11-12-advanced-git-johnliu.md`
- **新增頁面**: [[sources/2026-11-12-advanced-git-johnliu]], [[tools/git]], [[playbooks/git-version-control-and-history-management]]
- **更新頁面**: [[index]]
- **摘要**: 消化 Johnliutw 進階 Git 文章，建立 Git 工具實務頁（補充 `switch`/`restore` 新語法與 `.gitignore` 坑點）與 Git 歷史管理 Playbook（包含 Atomic Commit、歷史修改決策樹與 `reset` `--soft`/`--mixed`/`--hard` 三模式對照）。

## [2026-11-12] ingest | 批次消化六大領域核心技術與實務文章 (Cache, K8s, Medallion, Outbox, Redis 災難, PM/QA 協作)
- **Raw Sources**:
  - `raw/articles/2026-11-12-cache-strategies-johnliu.md`
  - `raw/articles/2026-11-12-container-orchestration-k8s-johnliu.md`
  - `raw/articles/2026-11-12-medallion-architecture-databricks-pipeline.md`
  - `raw/articles/2026-11-12-postgresql-outbox-pattern-revamped.md`
  - `raw/articles/2026-11-12-redis-hotkey-bigkey-titansoft.md`
  - `raw/articles/2026-11-12-how-to-work-with-qa-3pmlab.md`
- **新增頁面**:
  - **Sources (6)**: [[sources/2026-11-12-cache-strategies-johnliu]], [[sources/2026-11-12-container-orchestration-k8s-johnliu]], [[sources/2026-11-12-medallion-architecture-databricks-pipeline]], [[sources/2026-11-12-postgresql-outbox-pattern-revamped]], [[sources/2026-11-12-redis-hotkey-bigkey-titansoft]], [[sources/2026-11-12-how-to-work-with-qa-3pmlab]]
  - **Concepts (3)**: [[concepts/caching-strategies-and-pitfalls]], [[concepts/transactional-outbox-pattern]], [[concepts/medallion-architecture]]
  - **Tools (3)**: [[tools/redis]], [[tools/postgresql]], [[tools/kubernetes]]
  - **Playbooks (1)**: [[playbooks/cross-functional-collaboration-pm-eng-qa]]
  - **Postmortems (1)**: [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover]]
  - **Comparisons (2)**: [[comparisons/container-orchestration-compose-vs-swarm-vs-k8s]], [[comparisons/ingress-vs-api-gateway]]
- **更新頁面**: [[index]]
- **摘要**: 完成知識庫六大領域 (`backend`, `data-eng`, `database`, `infra`, `troubleshooting`, `workflow`) 的基石建置，共編譯 16 個新頁面並建立跨領域雙向連結（如串聯 Redis 20MB BigKey 事故與 K8s HPA 縮容失靈、PostgreSQL MVCC Visibility Map 與 Outbox 分區表、以及 Git 分支與 PM/Eng/QA 的 Feature Flags 協作流程）。

## [2026-11-12] query | 編譯《新手後端工程師實務須知》互動式 HTML 簡報投影片
- **輸出檔案**: `wiki/presentations/backend-onboarding-slides.html`
- **引用頁面**: [[tools/git]], [[playbooks/git-version-control-and-history-management]], [[playbooks/cross-functional-collaboration-pm-eng-qa]], [[concepts/caching-strategies-and-pitfalls]], [[tools/redis]], [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover]], [[tools/postgresql]], [[concepts/transactional-outbox-pattern]], [[tools/kubernetes]], [[comparisons/container-orchestration-compose-vs-swarm-vs-k8s]], [[comparisons/ingress-vs-api-gateway]]
- **更新頁面**: [[index]], `AGENTS.md`
- **摘要**: 綜合知識庫五大模組編譯 12 頁互動式 HTML 投影片（支援鍵盤左右翻頁、章節跳轉、深淺色主題切換與 Speaker Notes 講者備忘錄），並於 `wiki/presentations/` 歸檔。

## [2026-11-12] query | 以五階段 Graph 流程打造《新手後端與資料工程實務須知》雙模式圖譜簡報（獨立保留雙版本）
- **輸出檔案**: `wiki/presentations/backend-onboarding-graph-slides.html`（並完整保留第一版 `wiki/presentations/backend-onboarding-slides.html`）
- **新增納編節點**: 擴充至 14 個知識節點（新增 `N14 Medallion 資料湖倉三層事件管線`），涵蓋全 6 大領域。
- **更新頁面**: [[index]]
- **摘要**: 依循 5 階段 Graph 製程與檢查點（Macro-Graph 拓撲 ➔ Micro-Graph 逐頁圖表規格 ➔ 雙模式互動引擎 ➔ 視覺圖表注入 ➔ 驗收歸檔），打造左側「互動知識星系圖」+ 右側「全視覺化流程圖投影片」的雙模式簡報系統，並與第一版經典章節簡報分開並存。
