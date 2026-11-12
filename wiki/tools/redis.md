---
title: "Redis 分散式快取與記憶體資料庫實務"
type: tool
domain: [database, backend, troubleshooting]
tags: [redis, redis-cluster, cache, hotkey, bigkey, lfu, lru]
sources:
  - "[[sources/2026-11-12-cache-strategies-johnliu]]"
  - "[[sources/2026-11-12-redis-hotkey-bigkey-titansoft]]"
updated: 2026-11-12
status: evolving
---

# Redis 分散式快取與記憶體資料庫實務

## 1. 核心解決什麼問題

`Redis` 是業界最主流的開源 In-Memory 資料結構儲存系統，常作為**分散式快取（Distributed Cache）**、分散式鎖（Distributed Lock）與訊息佇列使用。
- **極致讀寫速度**：將資料存於 RAM 中，單次操作在 `ns` ~ `us` 等級（比關聯式資料庫快上萬倍）。
- **多實例狀態共享**：讓無狀態（Stateless）的多台後端 API Pod 共享同一份快取資料與 Session 狀態。
- **高可用與水平擴展**：透過 Master-Slave 複製、Sentinel 與 **Redis Cluster（16,384 個 Hash Slots 分片）** 提供跨節點容錯與容量擴充。

> 關於快取讀寫模式（Cache-Aside / Write-Through / Write-Back）與穿透/擊穿/雪崩防禦，請先閱讀：[[concepts/caching-strategies-and-pitfalls|快取策略與三大災難防禦]]。

---

## 2. 核心架構與運作機制

### 2.1 單執行緒模型 (Single-Threaded Event Loop) 的雙面刃
- **優勢**：Redis 的核心命令執行採用單執行緒搭配 I/O Multiplexing（多路復用），避免了多執行緒切換（Context Switch）與鎖競爭的開銷，且保證單一命令或 Lua Script 的原子性（Atomicity）。
- **致命限制（Trade-off）**：**因為命令是排隊單執行緒執行的，只要有一個慢指令（如對百萬筆資料執行 `KEYS *`、`HGETALL`，或讀取/刪除一個 20MB 的 `BigKey`）卡住主執行緒，後面成千上萬個正常請求都會被阻塞超時，甚至導致節點無法回應心跳而觸發 Cluster Failover！**

### 2.2 記憶體淘汰策略 (`maxmemory-policy`) 與 LFU 計數器
當 Redis 記憶體達到 `maxmemory` 上限時，會依據設定的策略逐出（Evict）Key：
- `noeviction`：預設值，記憶體滿時直接拒絕寫入（回傳 OOM 錯誤）。
- `allkeys-lru` / `volatile-lru`：淘汰最久未被使用的 Key。
- `allkeys-lfu` / `volatile-lfu`（**推薦用於精準識別熱點**）：
  - Redis 使用 **8-bit 的 LFU Counter（數值範圍 `0 ~ 255`）** 記錄每個 Key 的訪問熱度。
  - 計數器並非每次訪問都 `+1`，而是依據對數機率（由 `lfu-log-factor` 控制，預設為 `10`）遞增，並隨時間衰減。當 Counter 達到上限 **`255`** 時，通常代表該 Key 已累積約**百萬次級別**的高頻訪問！

---

## 3. 實務最佳實踐：HotKey 與 BigKey 偵測與治理

根據 [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|鈦坦科技 Redis 災難排查案例]]，生產環境必須定期巡檢並防範 **HotKey** 與 **BigKey**：

### 3.1 量化定義標準（經驗法則）

| 問題類型 | 判定基準（滿足其一即須警戒） | 潛在系統危害 |
| :--- | :--- | :--- |
| **HotKey**<br/>*(熱點鍵)* | 1. 總 QPS 10,000 中，單一 Key 佔了 **3,000+ QPS**。<br/>2. 單節點 100Mbps 頻寬中，**80Mbps** 被某個 Hash Key 的 `HGETALL` 佔用。 | 單一節點 CPU 飆高、Cluster 分片負載嚴重傾斜、一旦過期或節點掛掉將直接擊穿後端資料庫。 |
| **BigKey**<br/>*(大鍵值)* | 1. **String** 型別 Value 大小 **> 5MB**（超過 10KB 即建議注意）。<br/>2. **List / Hash / Set / ZSET** 成員數量 **> 10,000 ~ 20,000 個**，或總體積達 **100MB**。 | 塞爆網卡 Output 頻寬、阻塞單執行緒引發慢查詢雪崩、Cluster 遷移 Slot 超時失敗、觸發 OOM 逐出其他正常 Key、拖慢 Client 反序列化導致 K8s HPA 失靈。 |

### 3.2 內建 CLI 掃描指令

```bash
# 1. 掃描 HotKey（前提：Redis 的 maxmemory-policy 必須設定為 allkeys-lfu 或 volatile-lfu）
$ redis-cli -h <host> -p <port> -a <password> --hotkeys

# 2. 掃描 BigKey（會透過 SCAN 抽樣統計各資料型別中最大的 Key，對線上影響較小，但仍建議在離峰或從節點執行）
$ redis-cli -h <host> -p <port> -a <password> --bigkeys
```

### 3.3 根治與緩解方案
1. **拆分 BigKey（最根本解法）**：
   - 將單一 20MB 的大 JSON String 拆分為多個小 Key，或改用 Hash 結構並透過 `HGET` / `HMGET` 局部讀取所需欄位（嚴禁無腦 `HGETALL`）。
   - 若集合元素過多（如 10 萬筆），按業務 ID 取模分片（如 `user_list:1`, `user_list:2`）。
2. **刪除 BigKey 時使用非同步指令**：
   - 在 Redis 4.0+ 中，刪除大型 Key 請務必使用 **`UNLINK <key>`** 取代 `DEL <key>`，讓記憶體回收在背景執行緒非同步進行，避免卡死主執行緒。
3. **降低 HotKey 集中度**：
   - 針對極少變動的固定設定類 HotKey（如全域 `xxxSetting`），配合少量的 **Client 端本地快取 (In-Memory Cache，設短 TTL 如 30~60 秒)**，直接將 Redis 請求量降維打擊。

---

## 4. 常見坑點 (Gotchas & Anti-patterns)

- **坑點 1：在 Redis Cluster 手動搬移 Slot 時把 Master 搬空**
  - **現象**：當某台 Master Node 負載過高，工程師使用 `Redis-Insight` 或 `redis-cli --cluster reshard` 想把上面的 Slot 全部搬去其他節點。
  - **致命後果**：**在 Redis Cluster 中，一旦某台 Master Node 上的 Hash Slot 被徹底搬空（剩 0 個 Slot），該 Master 會自動降級（Demote）變成別人的 Slave！** 務必至少保留少數 Slot（或只獨留問題 BigKey 的 Slot 在該節點隔離）。
- **坑點 2：以為擴充 Redis Cluster 節點就能解決固定名稱的 HotKey**
  - **原因**：Redis Cluster 是透過 `CRC16(key) % 16384` 決定 Key 落在哪個 Slot。如果是一個**固定名稱的 Key**（如 `global_config`），無論你把 Cluster 從 3 台擴充到 100 台 Master，這個 Key 永遠只會落在其中 **1 台** 機器上！擴充節點只能分散「浮動名稱 Key（如 `user_info_{id}`）」的分母。

---

## 5. 關聯頁面
- [[concepts/caching-strategies-and-pitfalls|快取策略與三大災難防禦 (Caching Strategies & Pitfalls)]]
- [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|事故覆盤：Redis HotKey 與 20MB BigKey 引發叢集 Failover 與 K8s HPA 失靈]]
- [[sources/2026-11-12-cache-strategies-johnliu|來源摘要：後端工程師的第二堂課 (18) 資料加速: Cache]]
- [[sources/2026-11-12-redis-hotkey-bigkey-titansoft|來源摘要：Redis HotKey 與 BigKey 造成的一系列災難 (鈦坦科技)]]
