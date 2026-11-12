# 後端與資料工程實務 Wiki 索引 (Index)

> 本檔案為知識庫的全局目錄。每次 `ingest`、`query`（知識回寫）或 `lint` 後由 LLM 自動維護更新。
> **涵蓋領域 (`domain`)**：`backend` | `data-eng` | `database` | `infra` | `troubleshooting` | `workflow`

---

## 核心概念與架構模式 (`concepts/`)
*收錄系統設計、分散式系統、資料管線架構與演算法的核心原理與 Trade-offs。*

- [[concepts/caching-strategies-and-pitfalls|快取策略與三大災難防禦 (Caching Strategies & Pitfalls)]] `[backend, database]` — 涵蓋快取部署層級、`Cache-Aside` / `Write-Through` / `Write-Back` 讀寫模式、`LRU/LFU` 淘汰策略，以及快取穿透、擊穿、雪崩的實戰防禦。
- [[concepts/transactional-outbox-pattern|Transactional Outbox Pattern (交易收發箱模式)]] `[backend, database, data-eng]` — 解決微服務更新 DB 與發送訊息的雙寫不一致（Dual-Write），涵蓋 Polling vs CDC 對比、6 大線上事故防禦與 PostgreSQL `PARTITION BY LIST` 革新架構。
- [[concepts/medallion-architecture|Medallion Architecture (資料湖倉獎章分層架構)]] `[data-eng]` — Data Lakehouse 的 `Bronze` (原始) ➔ `Silver` (企業視圖) ➔ `Gold` (應用聚合) 多跳管線設計、ELT 思維與結合 S3/MinIO + Kafka + Airflow 的事件驅動實作。

---

## 技術棧與工具實務 (`tools/`)
*收錄資料庫、Message Queue、資料處理引擎、K8s/雲端元件與開發工具的配置、調優與常見坑點。*

- [[tools/git|Git 分散式版本控制系統實務]] `[workflow, backend]` — 涵蓋 `checkout` vs `switch/restore` 對照、`fetch` vs `pull` 機制、`stash` 暫存技巧、`log/reflog/blame` 追查三劍客與 `.gitignore` 安全規範。
- [[tools/redis|Redis 分散式快取與記憶體資料庫實務]] `[database, backend, troubleshooting]` — 單執行緒模型 Trade-offs、8-bit `LFU` 熱度計數機制、`--hotkeys` / `--bigkeys` 偵測治理與 Redis Cluster 搬移 Slot 的降級坑點。
- [[tools/postgresql|PostgreSQL 資料庫調優、索引與分區實務]] `[database, backend, troubleshooting]` — 高併發型別選擇（`INT8`、`TIMESTAMPTZ`、`JSONB` 的 `\x00` 坑點）、Partial Index 設計、MVCC Visibility Map 引起的 Heap Fetch 雪崩與分區裁剪（Partition Pruning）鐵則。
- [[tools/kubernetes|Kubernetes (K8s) 容器編排與雲端原生基建實務]] `[infra, backend]` — 宣告式架構與調度三階段、東西向 `Service` vs 南北向 `Ingress`、`Liveness/Readiness Probes`、`Requests/Limits` 防壞鄰居效應與 `Workload Identity` 零信任資安。

---

## 實戰指南、Git 與跨職能協作 (`playbooks/`)
*收錄 PM、Frontend、Backend、Data、DevOps/SRE 與 QA 的職責分工邊界、交付標準、Git 版本控管規範與標準作業流程 (SOP)。*

- [[playbooks/git-version-control-and-history-management|Git 版本控管與歷史紀錄管理實戰指南]] `[workflow, backend]` — 團隊版控實戰 SOP：Atomic Commit 原則、歷史修改決策樹（`--amend` / `reset` 三模式 / `revert` / `rebase`）、緊急插單 (`stash`) 與單點釋出 (`cherry-pick`) 流程。
- [[playbooks/cross-functional-collaboration-pm-eng-qa|跨職能研發協作指南：PM、前後端、QA 與 SRE 職責邊界與交付標準]] `[workflow, backend]` — 五大職能 RACI 品質把關矩陣、測試左移（QA 參與 PRD 審閱）、結構化測試維度清單、切小任務搭配 `Feature Flags` 小步交付與自動化回歸測試規範。

---

## 線上事故與踩坑排查 (`postmortems/`)
*收錄真實線上故障覆盤 (Postmortems)、效能瓶頸排查紀錄與 Troubleshooting 案例。*

- [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|事故覆盤：Redis HotKey 與 20MB BigKey 引發叢集 Failover 與 K8s HPA 失靈]] `[troubleshooting, database, infra, backend]` — 鈦坦科技真實案例：解析百萬級訪問的 20MB 固定 Key 如何塞爆 Redis Cluster 單點頻寬、觸發連環 Failover 並導致 K8s Client Pod CPU 震盪無法縮容。

---

## 技術選型與對比 (`comparisons/`)
*收錄不同架構方案或工具的多維度比較與決策樹。*

- [[comparisons/container-orchestration-compose-vs-swarm-vs-k8s|容器編排選型：Docker Compose vs Docker Swarm vs Kubernetes (K8s)]] `[infra, backend]` — 從單機開發、無專職 SRE 的輕量叢集到公有雲代管 K8s (EKS/GKE) 的多維度對比與選型決策樹。
- [[comparisons/ingress-vs-api-gateway|Ingress 與 API Gateway 的職責邊界與架構對比]] `[infra, backend]` — 釐清基礎設施層 `K8s Ingress`（SSL 卸載、L7 路徑轉發）與業務應用層 `API Gateway`（認證授權、業務限流、計費）的差異與串聯共存模式。

---

## 互動式簡報與內訓教材 (`presentations/`)
*收錄由 Wiki 知識網路編譯而成的 HTML 互動式投影片與團隊 Onboarding 教材。*

- [`presentations/backend-onboarding-slides.html`](presentations/backend-onboarding-slides.html) `[workflow, backend, database, infra, troubleshooting]` — **《新手後端工程師實務須知》經典章節版 (12 頁互動式簡報播放器)**：涵蓋 Git 版控與救援、跨職能協作與 Feature Flags、快取與 Redis BigKey/HotKey 災難、PostgreSQL 與 Outbox 分區表、以及 K8s 上線必備認知（含側邊章節大綱與講者備忘錄）。
- [`presentations/backend-onboarding-graph-slides.html`](presentations/backend-onboarding-graph-slides.html) `[workflow, backend, database, infra, data-eng, troubleshooting]` — **《新手後端與資料工程實務須知》雙模式 Graph 星系圖譜版 (14 Nodes · 5 Clusters)**：左側內建可點擊跳轉的 **Macro-Graph 全局知識星系圖**（含跨領域因果連線），右側為 **Micro-Graph 全視覺化流程圖投影片**（含 Medallion 三層湖倉管線）。

---

## 原始資料摘要 (`sources/`)
*收錄對應 `raw/` 原始文章、筆記與程式碼片段的結構化摘要。*

- [[sources/2026-11-12-advanced-git-johnliu|來源摘要：後端工程師的第二堂課 (1) 進階 Git]] `[workflow, backend]` — Johnliutw 進階 Git 文章摘要，涵蓋多功能並行、遠端拉取、歷史修改與暫存管理，並附錄《後端工程師的第二堂課》全系列課綱。
- [[sources/2026-11-12-cache-strategies-johnliu|來源摘要：後端工程師的第二堂課 (18) 資料加速: Cache]] `[backend, database]` — Johnliutw 快取設計專文摘要，涵蓋局部性原理、部署層級、三大讀寫模式、淘汰策略與穿透/擊穿/雪崩防禦。
- [[sources/2026-11-12-container-orchestration-k8s-johnliu|來源摘要：後端工程師的第二堂課 (26) — 容器管理協調系統 (Container Orchestration)]] `[infra, backend]` — Johnliutw 容器編排專文摘要，拆解 K8s 七大核心支柱與 Compose / Swarm / K8s 選型。
- [[sources/2026-11-12-medallion-architecture-databricks-pipeline|來源摘要：Medallion Architecture (獎章分層架構) 與 Multi-Hop 資料管線實作]] `[data-eng]` — Databricks 官方 Bronze/Silver/Gold 定義結合 MinIO + Kafka + Airflow 的事件驅動管線實作摘要。
- [[sources/2026-11-12-postgresql-outbox-pattern-revamped|來源摘要：PostgreSQL + Outbox Pattern Revamped — Part 1]] `[database, backend, troubleshooting]` — Mohammad S. Dousti 剖析 PostgreSQL Outbox 六大線上事故、MVCC Heap Fetch 退化實驗與 List Partitioning 解法。
- [[sources/2026-11-12-redis-hotkey-bigkey-titansoft|來源摘要：Redis HotKey 與 BigKey 造成的一系列災難 (鈦坦科技)]] `[troubleshooting, database, infra]` — 新加坡商鈦坦科技 Felix 分享 Redis Cluster HotKey + 20MB BigKey 排查、Slot 遷移與拆分根治經驗。
- [[sources/2026-11-12-how-to-work-with-qa-3pmlab|來源摘要：【PM夥伴攻略】如何跟QA合作？ (3PM Lab)]] `[workflow]` — 3PM Lab Anne Hsiao 分享軟體開發三層測試分工、QA 參與 PRD 左移偵錯、Feature Flags 小步交付與測試 Buffer 排程心法。
