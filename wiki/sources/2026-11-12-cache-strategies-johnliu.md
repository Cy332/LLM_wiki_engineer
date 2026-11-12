---
title: "來源摘要：後端工程師的第二堂課 (18) 資料加速: Cache"
type: source
domain: [backend, database]
tags: [cache, redis, performance, backend-course, johnliu]
sources:
  - "raw/articles/2026-11-12-cache-strategies-johnliu.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：後端工程師的第二堂課 (18) 資料加速: Cache

- **作者**：Johnliutw (John Liu)
- **發布日期**：2025-11-30
- **原始檔案**：[`raw/articles/2026-11-12-cache-strategies-johnliu.md`](../../raw/articles/2026-11-12-cache-strategies-johnliu.md)
- **原文連結**：[Medium 原文](https://medium.com/johnliu-%E7%9A%84%E8%BB%9F%E9%AB%94%E5%B7%A5%E7%A8%8B%E6%80%9D%E7%B6%AD/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-18-%E8%B3%87%E6%96%99%E5%8A%A0%E9%80%9F-cache-80d3cbfa69a6)

---

## 核心主旨

本文系統性剖析後端系統導入快取（Cache）的三大核心價值（利用儲存階層速度差加速、保護脆弱的後端資料庫連線與 CPU、節省雲端與第三方 API 成本），並從「局部性原理」出發，拆解快取設計的三大維度（地點、讀寫模式、淘汰策略）與三大經典快取災難的防禦方案。詳見 [[concepts/caching-strategies-and-pitfalls|快取策略與三大災難防禦 (Caching Strategies & Pitfalls)]] 與 [[tools/redis|Redis 分散式快取與記憶體資料庫實務]]。

---

## 重點提煉

1. **快取三大效益**：
   - **速度層級差**：資料庫查詢在 `ms` 等級，而 [[tools/redis|Redis]] 等記憶體快取在 `ns` ~ `us` 等級，相差可達萬倍。
   - **保護後端資源**：將熱門 API 的 1,000 RPS 攔截於快取層，使資料庫負載降至 0.1 RPS，避免耗盡 DB Connection Pool 影響寫入交易（如訂單、留言）。
   - **成本節省**：降低複雜 SQL（`JOIN` / `GROUP BY`）的 CPU 消耗、減少昂貴 RDS 規格需求、省下按次計費的第三方 API 呼叫費用。
2. **局部性原理 (Principle of Locality)**：
   - **時間局部性 (Temporal Locality)**：剛被存取的資料短時間內極可能再被存取（如熱門新聞）。
   - **空間局部性 (Spatial Locality)**：剛被存取資料附近的關聯資料也可能被存取。
3. **快取部署地點**：
   - **Client-Side (瀏覽器)**：透過 `Cache-Control`、`ETag` 快取靜態資源，零網路請求但伺服器無法強制客戶端立即更新。
   - **Edge (CDN)**：全球邊緣節點吸收巨量靜態與串流流量，降低跨國 Latency，但不適用高度動態個人化內容。
   - **Server-Side**：分為 **In-Memory Cache**（極速但無法跨機共享、重啟即失，僅適合 Config）與 **Distributed Cache**（如 Redis / Memcached，多機共享、高可用，僅多一次內部網路 I/O）。
4. **三大讀寫模式**：
   - **Cache-Aside**：最經典讀多寫少模式。讀取時先查 Cache，Miss 則查 DB 並回寫 Cache；寫入時**直接寫 DB 並「刪除」Cache**（刪除優於更新，避免連續寫入 10 次卻無讀取時產生 9 次無效更新計算）。
   - **Write-Through**：由快取層同步寫入 DB，保證強一致性（適合金融交易），但寫入延遲高。
   - **Write-Back (Write-Behind)**：先寫快取即回傳成功，再非同步批次寫入 DB。寫入吞吐量極高（適合 IoT、監控指標），但快取當機有掉資料風險。
5. **四大淘汰策略 (Eviction Policies)**：
   - `TTL`（到期自動刪除）、`LRU`（最近最少使用，基於時間局部性）、`LFU`（最不常使用，基於歷史熱門度，見 [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|鈦坦科技 Redis HotKey 排查案例]]）、`FIFO`（先進先出）。
6. **三大快取災難與防禦**：
   - **快取穿透 (Penetration)**：查詢快取與 DB 皆不存在的 ID（如惡意攻擊 `user_id = -1`）➔ **防禦**：將空結果 (`null`) 也快取並設短 TTL。
   - **快取擊穿 (Breakdown)**：單一超熱門 HotKey 過期瞬間湧入萬級併發打穿 DB ➔ **防禦**：使用互斥鎖（Mutex Lock），僅允許搶到鎖的請求查 DB 並重建快取。
   - **快取雪崩 (Avalanche)**：大量 Key 同時過期或 Redis 節點掛掉 ➔ **防禦**：TTL 加入隨機抖動值（Jitter，如 `3600 + random(0, 300)`）、建立高可用叢集與服務降級機制。
