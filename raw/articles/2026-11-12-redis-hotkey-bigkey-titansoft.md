---
title: "Redis HotKey 與 BigKey 造成的一系列災難 | 2022 Titansoft Tech Days 精選議程"
author: "Felix / Tica Titansoft (新加坡商鈦坦科技)"
source_url: "https://medium.com/titansoft/d0d9d7a392ba"
published_at: 2023-01-16
ingested_at: 2026-11-12
---

# Redis HotKey 與 BigKey 造成的一系列災難 | 2022 Titansoft Tech Days 精選議程

> 原文連結：[Medium - 新加坡商鈦坦科技 Titansoft](https://medium.com/titansoft/d0d9d7a392ba)
> 發布日期：2023-01-16
> 講者：鈦坦科技 Product Developer — Felix

## 為什麼想分享這個主題？

發現在我們開發的產品服務在尖峰時段時，明明使用 Redis Cluster 卻只有一台的流量特別高，甚至發現 CPU high 以及 OOM 相繼引發 Fail Over 機制以及逐出鍵值的機制導致資料遺漏，可能會影響到 Client 端效能，導致 Client CPU 忽高忽低影響到 K8s HPA 判斷機制，進而影響 Pod scale-down 不下，想以自己實際體驗過的經驗和大家分享如何發現問題到嘗試解決的過程。

## 什麼是 HotKey?

當一個 Key 的訪問量過高時，高流量過於集中在某一台節點上或是單台 Redis 上時，以下是簡化的定義標準：
- ✅ Redis 目前總訪問量是 10000，而某個 Key 的工作量達到 3000，明顯此 Key 訪問量高於其他 Key。
- ✅ 集群模式中一個節點的總頻寬為 100Mbits/s，其中 80Mbits 是由於對某個 Hash 類型的 Key 執行 `HGETALL` 所佔用。
（主要定義條件應為該 Key 值是否對系統造成負面影響）

## 什麼是 BigKey?

會稱作 BigKey 的鍵值分為兩種，數據過大、列表或成員數量過多，以下只是簡化的定義標準：
- ✅ 一個 String 的 Key 值大小為 5MB。
- ✅ 一個 List 的 Key，列表裡有 20,000 個元素。
- ✅ 一個 ZSET 的 Hash Key，裡面有 10,000 個成員。
- ✅ 一個 Hash Key 大小有 100MB，成員數量僅 1000，但成員的數據過大。

## BigKey + HotKey = Double Kill…

### 現象與指標異常

1. **問題一**：Client 端的 K8s Pod Scale up 升至最高點後降不下來。
2. **問題二**：Client 端的 Pod CPU 平均使用量忽高忽低，導致 HPA 無法執行縮容策略。
3. **問題三**：檢查 Redis Cluster 時，發現只有一台 Master Node 的 Network Output 與忙碌程度相較於其他 Node 特別高。
4. **問題四**：Redis Cluster 發生 OOM & FailOver 事件（Redis OOM 會開始依照 Memory policy 策略逐出 Key，造成資料遺失）。

![Image 3](https://miro.medium.com/v2/resize:fit:700/1*qyNN_7XER-rT0EsbbWOBwQ.png)
![Image 5](https://miro.medium.com/v2/resize:fit:700/1*QHv9xzh2hxAjYS_CC5jF0w.png)

### HotKey 對服務造成的三大影響
1. **CPU High**：處理大量請求導致單執行緒 CPU 使用量飆高，其他請求受到影響產生一系列慢查詢，最終導致故障轉移 (Failover)。
2. **集群分片不均衡**：HotKey 所在的節點有大量訪問傾斜於此，導致其他節點資源浪費。
3. **Redis 服務崩潰**：超出 Redis 能承受的流量時，導致訪問請求直接指向後端資料庫，引發資料庫連鎖崩潰。

---

## 第一步嘗試 (Part 1)：排查與打散 HotKey

1. **如何找出 HotKey**：
   使用 `redis-cli` 內置參數（需確認 `maxmemory-policy` 為 `allkeys-lfu` 策略）：
   ```bash
   redis-cli -h <address> -p <port> -a <password> --hotkeys
   ```
   - **LFU Counter 解釋**：該欄位僅 8-bit（最大值 255）。當 counter 達到 255（預設 `lfu-log-factor` 為 10）時，代表約有百萬級別的訪問頻率。
2. **緊急 Workaround（手動遷移 Slot 與擴容）**：
   - 找出名稱固定的 HotKey，透過 `CLUSTER KEYSLOT` 與 `Redis-Insight` 工具，手動將較忙 Node 上的 HotKey Slot 遷移 (Reshard) 給較不忙的 Node。
   - **重要坑點**：**千萬不要把 Master 裡的 Slot 搬空，否則該 Master 會自動降級成為 Slave！**
   - 針對浮動名稱的 Key（如 `user_info_{id}`），將 Redis Cluster 增加 2 組 Master-Slave（增加分母），讓 Hash Slot 分佈更分散並降低 OOM 發生率。
3. **Part 1 結果**：
   OOM 不再發生，避免了 Key 被逐出導致資料遺失；但在週末尖峰時段，依然有一台 Node 特別忙碌並發生多次 Master-Slave 互相 FailOver！

---

## 第二步嘗試 (Part 2)：揪出 20MB 的 BigKey 並拆分

繼續深挖該台高流量 Node 後，發現上面存在一個 **20MB 大小的致命 BigKey（而且它同時也是 HotKey！）**。

### BigKey 對服務產生的七大影響
1. **數據遷移失敗**：Cluster Rebalance 時，過大的 BigKey 會阻塞其他 Key 的遷移，導致超時失敗。
2. **Memory 使用量不均衡**：單一節點記憶體遠高於其他節點。
3. **頻寬用量不均衡**：單一節點 Network Output 被塞爆。
4. **發生 OOM / 鍵值被逐出**：觸發 Maxmemory 逐出策略導致其他正常 Key 資料遺失。
5. **Client 端執行命令超時**：由於 Redis 核心為單執行緒，操作 BigKey 會阻塞後續所有命令，引發大量慢查詢。
6. **導致 Redis 啟動故障轉移 (Failover)**：對 BigKey 執行刪除（Redis 4.0.10 以下未使用 `UNLINK` 非同步刪除）或遷移時，主執行緒長時間卡死，被集群判定失聯而觸發 Failover。
7. **頻寬壅塞與 Client 端阻塞**：高頻讀取 20MB BigKey 塞爆網卡頻寬，連帶讓 Client 端反序列化與等待時間拉長，造成 Client Pod CPU 忽高忽低。

### 最終解決方案
1. **找出 BigKey**：
   ```bash
   redis-cli -h <address> -p <port> -a <password> --bigkeys
   ```
2. **短期止血 (Workaround)**：
   - 調降 Client 端 Schedule 輪詢該固定 Key 的頻率（例如從每 5 秒拿一次改為每 1 分鐘拿一次）。
   - 將該台 Node 上的其他正常 Slot 全部搬移到其他 4 台 Node，讓這台 Node 獨自承擔該 BigKey 的 Slot。
3. **長期根治 (Root Fix)**：
   - 修改應用程式架構，**將 20MB 的 BigKey 拆分為多個小 Key**，徹底消除單點網路頻寬與單執行緒阻塞，成功讓 Redis Cluster 負載均衡，K8s Client Pod 的 CPU 與 HPA 也恢復正常縮放。
