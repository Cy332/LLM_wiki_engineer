---
title: "PostgreSQL 資料庫調優、索引與分區實務"
type: tool
domain: [database, backend, troubleshooting]
tags: [postgresql, indexing, partial-index, partitioning, mvcc, vacuum, data-types]
sources:
  - "[[sources/2026-11-12-postgresql-outbox-pattern-revamped]]"
updated: 2026-11-12
status: evolving
---

# PostgreSQL 資料庫調優、索引與分區實務

## 1. 核心解決什麼問題

`PostgreSQL` 是功能最完整的開源關聯式資料庫（RDBMS），廣泛用於核心交易系統（OLTP）、事件佇列（如 [[concepts/transactional-outbox-pattern|Transactional Outbox Pattern]]）與半結構化資料處理。然而，由於其底層採用 **MVCC（多版本併發控制，更新等同於標記舊 Tuple 死亡並插入新 Tuple）** 架構，在高頻更新與刪除的場景下，必須深入理解其型別特性、部分索引（Partial Index）、Visibility Map（可見性映射）與資料表分割（Table Partitioning），才能避免嚴重的效能雪崩。

---

## 2. Schema 與資料型別最佳實踐

根據 [[sources/2026-11-12-postgresql-outbox-pattern-revamped|PostgreSQL 生產環境踩坑經驗]]，在設計高吞吐量資料表時應遵循以下規範：

| 欄位性質 | 推薦型別與語法 | 嚴禁 / 慎用型別 | 實務原因與踩坑分析 |
| :--- | :--- | :--- | :--- |
| **自增主鍵 / 序號** | `INT8 GENERATED ALWAYS AS IDENTITY` | `INT4` (`integer` / `serial`) | `INT4` 上限僅約 21.4 億（$2^{31}-1$）。高頻寫入表一旦耗盡 Sequence 會直接引發全站寫入當機，線上無停機將 PK 從 `INT4` 改為 `INT8` 極度痛苦。 |
| **時間戳記** | `TIMESTAMPTZ` (`timestamp with time zone`) | `TIMESTAMP` (不含時區) | 跨時區微服務或雲端容器環境下，不含時區的時間戳極易造成邏輯錯亂。 |
| **JSON 資料** | 需查詢內部欄位用 `JSONB`<br/>純儲存/轉發用 `JSON` 或 `TEXT` | 無腦全部用 `JSONB` | 1. `JSONB` 寫入時需解析為二進位樹，寫入較慢且體積較大。<br/>2. **致命坑點**：**`JSONB` 嚴格禁止 `\x00` (Unicode NULL 字元)**，若上游序列化字串含有 `\x00` 會直接報錯 `invalid input syntax for type json`！ |

---

## 3. 索引調優：Partial Index (部分索引) 與設計誤區

當查詢條件固定只針對資料表中的一小部分活躍資料（例如未處理的任務：`WHERE published_at IS NULL ORDER BY id LIMIT 10`）時，應使用 **Partial Index（部分索引）**：

```sql
-- ✅ 正確：針對排序欄位 (id) 建索引，並用 WHERE 過濾出未發送的列
CREATE INDEX idx_outbox_unpublished ON outbox (id)
WHERE published_at IS NULL;
```

### 常見錯誤索引 Anti-patterns
1. **誤區 A：不建 Partial Index，直接依賴主鍵 `outbox_pkey (id)`**
   - **後果**：隨歷史資料累積（99.9% 的列 `published_at` 都有值），PostgreSQL 順著主鍵索引掃描時必須讀取並用 `Filter: (published_at IS NULL)` 丟棄成千上萬筆已處理的歷史紀錄，查詢越來越慢。
2. **誤區 B：把索引欄位建成 `(published_at)` 而非 `(id)`**
   ```sql
   -- ❌ 錯誤示範：毫無作用的索引
   CREATE INDEX idx_bad ON outbox (published_at) WHERE published_at IS NULL;
   ```
   - **後果**：這會在 B-Tree 裡塞滿一模一樣的 `NULL` 值，且完全無法協助 `ORDER BY id` 排序！

---

## 4. 深入底層：MVCC Visibility Map 與 Index Only Scan 的 Heap Fetch 災難

### 4.1 為什麼 `Index Only Scan` 還是會去讀資料表 (Heap Fetch)？
在 PostgreSQL 中，B-Tree 索引節點**不包含交易可見性資訊（Transaction Visibility / `xmin`, `xmax`）**。
當執行 `Index Only Scan` 時，PostgreSQL 會先查閱一張在記憶體中極小的位元圖——**Visibility Map (VM)**：
- 若 VM 標示該資料頁（Page）上的所有 Tuple 對所有交易皆可見（All-Visible），則直接回傳索引值（`Heap Fetches: 0`，耗時僅 ~0.1 ms）。
- **若該資料頁近期發生過 `INSERT`、`UPDATE` 或 `DELETE`，且 `autovacuum` 尚未清理並更新 VM**，PostgreSQL 就**必須回表（Heap Fetch）去讀取實體資料頁**，逐筆檢查該列是否真的還沒被更新或刪除！

### 4.2 高併發讀寫下的效能崩潰
在類似队列或 [[concepts/transactional-outbox-pattern|Outbox 表]] 的場景中，一端每秒插入數千筆新資料，另一端每秒將數千筆資料 `UPDATE SET published_at = NOW()`：
- 剛被更新的列在 Partial Index 中變成 Dead Tuples。
- 從 **PostgreSQL 14** 開始，`VACUUM` 預設啟用 `INDEX_CLEANUP AUTO`，當可移除的索引項目未達門檻時會跳過索引清理（可手動用 `VACUUM (INDEX_CLEANUP ON) table_name;` 強制執行）。
- **災難結果**：`Index Only Scan` 被迫在大量剛被修改過、尚未 Vacuum 的頁面中進行高達 **數千萬次的 Heap Fetches**，導致單次 `LIMIT 1000` 查詢從 **0.13 ms 暴增至 18.5 秒**！

---

## 5. 進階實戰：Table Partitioning (分區表) 與 Partition Pruning 鐵則

為了徹底擺脫高頻更新表的 Dead Tuples、Heap Fetch 與 Index Bloat 問題，最佳實務是利用 **宣告式分區 (`PARTITION BY LIST` 或 `PARTITION BY RANGE`)** 將「活躍熱資料」與「歷史冷資料」物理隔離。

### 5.1 依狀態分區範例 (`PARTITION BY LIST`)
```sql
CREATE TABLE outbox (
    id INT8 GENERATED ALWAYS AS IDENTITY,
    payload JSON NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    published_at TIMESTAMPTZ
) PARTITION BY LIST (published_at);

CREATE TABLE outbox_unpublished PARTITION OF outbox FOR VALUES IN (NULL);
CREATE TABLE outbox_published   PARTITION OF outbox DEFAULT;

CREATE INDEX idx_unpublished_id ON outbox_unpublished (id);
```
- 當執行 `UPDATE outbox SET published_at = NOW() ...` 時，PostgreSQL 會在底層自動將該筆資料從 `outbox_unpublished` **DELETE**，並 **INSERT** 到 `outbox_published`。
- 歷史資料清理只需執行 `TRUNCATE outbox_published;`，完全不產生 Dead Tuples，瞬間釋放空間。

### 5.2 常見坑點：更新或查詢分區表時忘記帶 Partition Key
> [!CAUTION] 致命線上事故：缺少 Partition Pruning 導致全表掃描
> 在上述分區表中，因為 `outbox_published` 歷史分區沒有建立 `id` 索引（為了省空間與寫入效能），如果你執行以下 SQL：
> ```sql
> -- ❌ 嚴重錯誤：未包含分區鍵 published_at IS NULL
> UPDATE outbox SET published_at = NOW() WHERE id = 12345;
> ```
> PostgreSQL 的 Query Planner 無法在規劃期得知 `id = 12345` 位於哪個分區，因此會對累積了數千萬筆資料、且沒有索引的 `outbox_published` 執行 **Sequential Scan（全表掃描）**！
>
> **鐵則**：操作分區表時，`WHERE` 子句**永遠要明確帶上分區鍵條件**以觸發 **Partition Pruning（分區裁剪）**：
> ```sql
> -- ✅ 正確：Planner 直接排除 outbox_published，只用索引掃描 outbox_unpublished
> UPDATE outbox SET published_at = NOW() WHERE id = 12345 AND published_at IS NULL;
> ```

---

## 6. 關聯頁面
- [[concepts/transactional-outbox-pattern|Transactional Outbox Pattern (交易收發箱模式)]]
- [[sources/2026-11-12-postgresql-outbox-pattern-revamped|來源摘要：PostgreSQL + Outbox Pattern Revamped — Part 1]]
