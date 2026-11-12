---
title: "Transactional Outbox Pattern (交易收發箱模式)"
type: concept
domain: [backend, database, data-eng]
tags: [outbox-pattern, distributed-transactions, dual-write, postgresql, kafka, event-driven, debezium]
sources:
  - "[[sources/2026-11-12-postgresql-outbox-pattern-revamped]]"
updated: 2026-11-12
status: evolving
---

# Transactional Outbox Pattern (交易收發箱模式)

## 1. 核心解決什麼問題：雙寫問題 (The Dual-Write Problem)

在微服務與事件驅動架構（Event-Driven Architecture）中，後端服務常需要在處理業務時**「更新本地資料庫（如 [[tools/postgresql|PostgreSQL]]）」**並同時**「發送事件訊息到 Message Broker（如 Kafka、RabbitMQ）」**。

若直接在程式碼中先寫 DB 再呼叫 Broker API（或反之），會面臨致命的 **Dual-Write Problem（雙寫不一致）**：
- 若 DB Commit 成功，但網路瞬斷或 Broker 當機，訊息將遺失（下游服務狀態永遠不一致）。
- 若先發送訊息成功，但隨後 DB Commit 失敗 Rollback，下游服務已經收到幽靈事件。

**Transactional Outbox Pattern** 的核心解法是：將要發送的訊息作為一筆資料，與業務資料放在**同一個本地資料庫 Transaction** 中寫入一張 `outbox` 表（保證 ACID 原子性），再由獨立的背景程序（Publisher / Relay）從 `outbox` 表讀取未發送訊息並推送至 Message Broker，成功後標記為已發送。

---

## 2. 架構流派對比：Polling Publisher vs Log-Based CDC (Debezium)

實作 Outbox Relay 主要有兩種架構流派：

| 維度 | 1. 輪詢發布 (Polling Publisher) | 2. 日誌擷取 CDC (如 Debezium + Kafka Connect) |
| :--- | :--- | :--- |
| **運作原理** | 應用程式或排程 Worker 定期執行 `SELECT ... FROM outbox WHERE published_at IS NULL` 抓取並發送。 | 透過解析資料庫底層交易日誌（如 PostgreSQL 的 `WAL`）即時捕捉 `outbox` 表的 `INSERT` 事件並推入 Kafka。 |
| **優勢** | - **架構單純**：不需額外維運 Debezium / Kafka Connect 叢集。<br/>- **Broker 無關**：可自由發送到任何目標（RabbitMQ、SNS/SQS、gRPC、Kafka）。<br/>- **無 Replication Slot 風險**：不會因下游卡死導致 PostgreSQL Logical Replication Slot 積壓 WAL 撐爆硬碟，也不會阻礙 DB 大版本升級。 | - **低延遲**：接近即時串流，無輪詢間隔。<br/>- **零查詢負載**：不需對 DB 執行高頻 `SELECT` 與 `UPDATE`。 |
| **劣勢 / 挑戰** | 在高吞吐量下，高頻 `INSERT` + `UPDATE` 極易引發 [[tools/postgresql|PostgreSQL]] **索引膨脹 (Index Bloat)** 與 **Visibility Map / Heap Fetch 效能崩潰**（需搭配 Partitioning 解決）。 | 維運成本高，強綁定 Kafka 生態系，且需嚴密監控 PostgreSQL WAL Replication Slot 延遲。 |

---

## 3. 實務最佳實踐與六大線上事故防禦 (PostgreSQL 實戰)

根據 [[sources/2026-11-12-postgresql-outbox-pattern-revamped|PostgreSQL + Outbox Pattern Revamped]] 的生產環境覆盤，自行實作 Polling Outbox 必須嚴格規避以下六大坑點：

### 3.1 坑點 1：嚴禁使用 `ORDER BY created_at` 排序事件
- **事故原因**：在分散式系統中，不同 K8s Pod 的本地時鐘存在微小的 **Clock Drift（時鐘偏移，哪怕只有幾毫秒）**。若由各 Pod 產生 `created_at` 並依此排序拉取，會導致事件發送順序與真實因果順序錯亂。
- **正確做法**：永遠依賴資料庫單調遞增的 Sequence（`id INT8 GENERATED ALWAYS AS IDENTITY`），查詢時使用 `ORDER BY id`。

### 3.2 坑點 2：欄位資料型別選擇 (`INT8` / `TIMESTAMPTZ` / `JSON` vs `JSONB`)
- **`id` 必用 `INT8` (`bigint`)**：切勿使用 `INT4` (`integer`，上限僅 21 億筆，高頻事件表極易溢位造成全站寫入當機)。
- **`payload` 優先用 `JSON` 或 `TEXT`，慎用 `JSONB`**：
  - 雖然 `JSONB` 支援內部欄位索引，但 Outbox 只需原封不動轉發 Payload，不需要查詢 JSON 內部。
  - `JSONB` 寫入時需要額外解析轉換（較慢且較佔空間），更致命的是 **PostgreSQL 的 `JSONB` 不允許包含 `\x00` (Unicode NULL 字元)**，一旦序列化器產出 `\x00` 就會直接拋出 `ERROR: invalid input syntax for type json` 導致業務 Transaction 失敗！

### 3.3 坑點 3：批次大小控制與平行分組發送 (Batch Size & Grouping)
- **避免長交易 (Long-running Transactions)**：Outbox Worker 的一個循環包含「開 Transaction ➔ `SELECT` N 筆 ➔ 網路 I/O 發送至 Broker ➔ `UPDATE` 標記已發送並 Commit」。若 N 太大（例如一次抓 5,000 筆），網路延遲會拉長 DB Transaction 時間，不僅阻塞 PostgreSQL `autovacuum`，一旦中途失敗還會導致整批 N 筆全部 Rollback 重發。
- **最佳實踐**：
  - 單次輪詢抓取 **`LIMIT 100` 左右（絕對不要超過 1,000 筆）**。
  - 在記憶體中將這 100 筆訊息依業務 Partition Key（如 `user_id`、`account_id`）分組：同一個 Group 內維持嚴格順序批次發送，不同 Group 之間則利用非同步協程（如 Kotlin Coroutines / Go Goroutines）平行發送以壓榨吞吐量。

### 3.4 坑點 4：毒丸訊息隔離 (Poison Pill Messages)
- **問題**：若某筆訊息因格式錯誤（例如違反 Kafka Schema Registry 的相容性規則）永遠無法發布成功，它會成為 **Poison Pill**，導致整個 Batch Transaction 無限失敗重試，卡死整條 Outbox 管線。
- **解法**：結合上述的分組機制，針對每個 Group 獨立 `try-catch`。當某個 Group 發生不可恢復錯誤時，捕獲異常並觸發即時告警（或記錄 `retry_count` 移入 Dead Letter Queue），讓同批次其他正常 Group 的訊息仍能順利更新 `published_at` 並 Commit。

---

## 4. 終極架構演進：以 List Partitioning 根治高併發 Heap Fetch 災難

> [!WARNING] 實務演進：為什麼單靠 Partial Index 在高吞吐量下仍會崩潰？
> 傳統教科書建議在單一 `outbox` 表建立部分索引：`CREATE INDEX idx1 ON outbox (id) WHERE published_at IS NULL;`。
> 然而在**高併發同時大量 Insert 與 Update (`published_at = NOW()`)** 的場景下，索引中剛被標記為已發送的列會變成 Dead Tuples。在 `autovacuum` 來得及清理並更新 **Visibility Map (VM)** 之前（且 PG 14+ 預設 `INDEX_CLEANUP AUTO` 常跳過索引清理），PostgreSQL 執行 `Index Only Scan` 時無法確認索引列是否對當前交易可見，被迫瘋狂回表（Heap Fetch）檢查近億筆資料！實驗證實這會讓查詢時間從 **0.13 ms 暴增至 18,553 ms（惡化 10 萬倍）**。

### 革新方案：按發送狀態進行分區 (`PARTITION BY LIST (published_at)`)

將 `outbox` 表物理拆分為「未發送 (`NULL`)」與「已發送 (`DEFAULT`)」兩個獨立分區表：

```sql
-- 1. 建立以 published_at 為分區鍵的主表
CREATE TABLE outbox (
    id INT8 GENERATED ALWAYS AS IDENTITY,
    payload JSON NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
) PARTITION BY LIST (published_at);

-- 2. 未發送分區（僅存放 published_at IS NULL 的活躍訊息）
CREATE TABLE outbox_unpublished 
    PARTITION OF outbox FOR VALUES IN (NULL);

-- 3. 已發送歷史分區（存放所有已標記時間戳的訊息，無須建立任何索引！）
CREATE TABLE outbox_published 
    PARTITION OF outbox DEFAULT;

-- 4. 僅在「未發送分區」上建立 id 索引
CREATE INDEX idx_outbox_unpublished_id ON outbox_unpublished (id);
```

### 標準輪詢與更新 SQL（必帶 Partition Key！）

當 Worker 發送完訊息、執行 `UPDATE` 時，**`WHERE` 子句必須同時包含 `AND published_at IS NULL`**：

```sql
WITH batch AS (
    SELECT id
    FROM outbox
    WHERE published_at IS NULL
    ORDER BY id
    LIMIT 100
)
UPDATE outbox
SET published_at = NOW()
WHERE id IN (SELECT id FROM batch)
  AND published_at IS NULL; -- 關鍵！觸發 Partition Pruning，避免掃描 outbox_published
```

**此架構的三大壓倒性優勢**：
1. **實體搬移取代原地滯留**：當 `published_at` 從 `NULL` 更新為 `NOW()` 時，PostgreSQL 底層會自動將該筆 Tuple 從 `outbox_unpublished` 刪除並插入 `outbox_published`，讓未發送分區永遠保持極小體積，高併發下查詢穩定在 **1~3 ms**。
2. **已發送分區零索引負擔**：`outbox_published` 完全不需要任何索引，純粹順序寫入（Append-only），對 Vacuum 極度友善。
3. **瞬間清理歷史資料**：要清理歷史已發送訊息時，不需執行產生大量 Dead Tuples 的 `DELETE`，只需定期執行 `TRUNCATE outbox_published;`，即可在毫秒內瞬間釋放硬碟空間！

---

## 5. 關聯頁面
- [[tools/postgresql|PostgreSQL 資料庫調優與實務]]
- [[sources/2026-11-12-postgresql-outbox-pattern-revamped|來源摘要：PostgreSQL + Outbox Pattern Revamped — Part 1]]
