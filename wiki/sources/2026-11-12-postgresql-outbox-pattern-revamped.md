---
title: "來源摘要：PostgreSQL + Outbox Pattern Revamped — Part 1"
type: source
domain: [database, backend, troubleshooting]
tags: [postgresql, outbox-pattern, partitioning, indexing, mvcc, vacuum]
sources:
  - "raw/articles/2026-11-12-postgresql-outbox-pattern-revamped.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：PostgreSQL + Outbox Pattern Revamped — Part 1

- **作者**：Mohammad S. Dousti (`msdousti`)
- **發布日期**：2025-06-08
- **原始檔案**：[`raw/articles/2026-11-12-postgresql-outbox-pattern-revamped.md`](../../raw/articles/2026-11-12-postgresql-outbox-pattern-revamped.md)
- **原文連結**：[DEV Community 原文](https://dev.to/msdousti/postgresql-outbox-pattern-revamped-part-1-3lai)

---

## 核心主旨

本文深入剖析在分散式系統中使用 [[tools/postgresql|PostgreSQL]] 實作 [[concepts/transactional-outbox-pattern|Transactional Outbox Pattern]] 時最常見的 **6 大線上事故 (Incidents)**。作者不僅給出正確的欄位型別與 Partial Index 設計，更透過壓測實驗揭露高併發讀寫下 PostgreSQL Visibility Map 失效導致 **Index Only Scan 退化出近億次 Heap Fetches（延遲暴增 5 個數量級至 18.5 秒）** 的致命問題，並提出基於 **`PARTITION BY LIST (published_at)` 分割表** 的革新架構。

---

## 重點提煉

1. **為何有時不用 Debezium (CDC) 而選擇自己實作 Polling Outbox？**
   - 避免維護 Debezium / Kafka Connect 的額外基建複雜度。
   - 目標 Message Broker 未必是 Kafka。
   - 避免 PostgreSQL Logical Replication Slot 在下游異常時卡住 WAL 清理導致磁碟爆滿，或阻礙大版本升級。
2. **六大線上事故與最佳實踐**：
   - **Incident 1（依賴本地時鐘 `ORDER BY created_at`）**：多 Pod 間存在毫秒級 Clock Drift，導致事件發送順序與實際發生順序不一致。**解法**：改用資料庫單調遞增序號 `id INT8 GENERATED ALWAYS AS IDENTITY` 並 `ORDER BY id`。
   - **Incident 2（錯誤的資料型別）**：主鍵誤用 `INT4` 會在 21 億筆時溢位當機（必用 `INT8`）；時間必用 `TIMESTAMPTZ`；`payload` 若用 `JSONB` 雖可建索引，但解析較慢且**遇到 `\x00` (Unicode NULL) 會直接拋錯拒絕寫入**，純轉發建議用 `JSON` 或 `TEXT`。
   - **Incident 3（錯誤的索引設計）**：針對 `WHERE published_at IS NULL ORDER BY id LIMIT 10`，必須建立 `CREATE INDEX idx1 ON outbox (id) WHERE published_at IS NULL`。切勿把索引建在 `(published_at)` 欄位上（全為 NULL 的 B-Tree 毫無排序作用）。
   - **Incident 4（單次抓取過多訊息）**：Polling 交易包含網路 I/O（發送至 Broker），若 `LIMIT` 過大（如 > 1,000）會形成長交易阻塞 `autovacuum`。**建議每次抓取 ~100 筆（絕不超過 1,000 筆）**，並依業務 Key（如 `user_id`）分組平行批次發送。
   - **Incident 5（Poison Pill 毒丸訊息）**：單筆格式錯誤（如違反 Schema Registry）會導致整批 Transaction 無限 Rollback 重試並卡死佇列。需以分組 Try-Catch 隔離並觸發告警。
   - **Incident 6（高併發下 Index Visibility Checking 與 Heap Fetch 災難）**：當高頻 Insert 與 Update (`published_at = NOW()`) 同時進行時，`idx1` 累積大量 Dead Tuples，且 Postgres 14+ 預設 `INDEX_CLEANUP AUTO` 常跳過索引清理，導致 `Index Only Scan` 必須回表檢查近億筆 Heap Tuples，耗時從 0.13ms 暴增至 18,553ms！
3. **終極解法：List Partitioned Outbox Table**：
   - 將 `outbox` 表依 `PARTITION BY LIST (published_at)` 拆為 `outbox_unpublished` (`FOR VALUES IN (NULL)`) 與 `outbox_published` (`DEFAULT`) 兩個分區，且**只在 `outbox_unpublished` 建立 `(id)` 索引**。
   - **關鍵鐵則**：執行 `UPDATE` 標記已發送時，**WHERE 子句必須帶上分區鍵 `AND published_at IS NULL`**，才能觸發 Partition Pruning，否則會對龐大的 `outbox_published` 進行全表掃描（Seq Scan）！
   - **成效**：更新後 Postgres 自動將該列從 `unpublished` 移至 `published` 分區，高併發查詢穩定維持在 1~3ms，且歷史已發送資料可直接透過 `TRUNCATE outbox_published` 瞬間清空，零膨脹、零 Vacuum 負擔。
