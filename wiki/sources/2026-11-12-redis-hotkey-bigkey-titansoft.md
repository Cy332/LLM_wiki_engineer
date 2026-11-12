---
title: "來源摘要：Redis HotKey 與 BigKey 造成的一系列災難 (鈦坦科技)"
type: source
domain: [troubleshooting, database, infra]
tags: [redis, hotkey, bigkey, redis-cluster, k8s, hpa, oom, failover, titansoft]
sources:
  - "raw/articles/2026-11-12-redis-hotkey-bigkey-titansoft.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：Redis HotKey 與 BigKey 造成的一系列災難 (鈦坦科技)

- **作者**：Felix / 新加坡商鈦坦科技 (Titansoft)
- **發布日期**：2023-01-16
- **原始檔案**：[`raw/articles/2026-11-12-redis-hotkey-bigkey-titansoft.md`](../../raw/articles/2026-11-12-redis-hotkey-bigkey-titansoft.md)
- **原文連結**：[Medium 原文](https://medium.com/titansoft/d0d9d7a392ba)

---

## 核心主旨

本文為新加坡商鈦坦科技（Titansoft）的真實線上故障排查分享。團隊從「[[tools/kubernetes|K8s]] Client Pod 擴容到頂後降不下來、CPU 忽高忽低」的表象出發，一路追查到 [[tools/redis|Redis Cluster]] 單一節點網路流量異常、OOM 與 Master-Slave 連環 Failover，最終揪出同時身兼 **HotKey（百萬級訪問）與 BigKey（20MB 大小）** 的致命設定檔快取，並完整記錄了兩階段止血與根治過程（完整覆盤見 [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|事故覆盤：Redis HotKey 與 20MB BigKey 引發叢集 Failover 與 K8s HPA 失靈]]）。

---

## 重點提煉

1. **HotKey 與 BigKey 的實務量化基準**：
   - **HotKey**：例如總 QPS 10,000 中單一 Key 佔 3,000；或單節點 100Mbps 頻寬中 80Mbps 被單一 Hash Key 的 `HGETALL` 佔據。
   - **BigKey**：String > 5MB；List 元素 > 20,000 個；ZSET/Hash 成員 > 10,000 個；或 Hash 總體積達 100MB。
2. **第一階段排查與止血（針對 HotKey）**：
   - 在 `maxmemory-policy: allkeys-lfu` 下使用 `redis-cli --hotkeys` 掃描，發現 LFU 8-bit counter 達上限 `255`（約代表百萬次訪問）的固定 Key。
   - **Workaround**：透過 `Redis-Insight` 手動將該 Slot 遷移至較閒置的 Node（**注意：不可將 Master 的 Slot 搬空，否則會自動降級為 Slave**），並為 Cluster 擴充 2 組 Master-Slave 分散浮動 Key。
   - **結果**：解決了 OOM 與 Key 被逐出（Eviction）的問題，但尖峰時段單一節點仍因頻寬塞爆而持續發生 Master-Slave 互踢 Failover。
3. **第二階段排查與根治（揪出 20MB 的 BigKey）**：
   - 使用 `redis-cli --bigkeys` 掃描，發現該忙碌節點上有一個 **20MB 的 BigKey（且同時是每 5 秒被大量 Client 輪詢的 HotKey）**。
   - **BigKey 的七大危害**：阻塞 Cluster 數據遷移、記憶體與頻寬嚴重傾斜、觸發 OOM 逐出其他正常 Key、阻塞單執行緒造成大量慢查詢、刪除或轉移時卡死觸發 Failover、塞爆網卡頻寬連帶拖慢 Client 端導致 K8s HPA 誤判。
   - **根治方案**：短期先將 Client 輪詢週期從 5 秒拉長至 1 分鐘，並把該 Node 其他 Slot 搬走讓其專職服務該 BigKey；長期則修改程式邏輯，**將 20MB 的 BigKey 拆分為多個小 Key**，徹底消除單點頻寬與阻塞瓶頸。
