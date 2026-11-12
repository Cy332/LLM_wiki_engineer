---
title: "來源摘要：後端工程師的第二堂課 (26) — 容器管理協調系統 (Container Orchestration)"
type: source
domain: [infra, backend]
tags: [kubernetes, k8s, docker, docker-swarm, container-orchestration, johnliu]
sources:
  - "raw/articles/2026-11-12-container-orchestration-k8s-johnliu.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：後端工程師的第二堂課 (26) — 容器管理協調系統 (Container Orchestration)

- **作者**：Johnliutw (John Liu)
- **發布日期**：2026-06-28
- **原始檔案**：[`raw/articles/2026-11-12-container-orchestration-k8s-johnliu.md`](../../raw/articles/2026-11-12-container-orchestration-k8s-johnliu.md)
- **原文連結**：[Medium 原文](https://johnliutw.medium.com/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-26-%E5%AE%B9%E5%99%A8%E7%AE%A1%E7%90%86%E5%8D%94%E8%AA%BF%E7%B3%BB%E7%B5%B1-container-orchestration-f225d5fd1662)

---

## 核心主旨

本文闡述當系統從單機 Docker 邁向多機 Production 環境時，為何需要容器管理協調系統（如 [[tools/kubernetes|Kubernetes (K8s)]]）。文章從宣告式管理、網路路由、自動擴縮、探針健康檢查、儲存與配置分離、零信任安全到可觀測性，完整拆解容器編排的七大核心支柱，並提供 `Docker Compose`、`Docker Swarm` 與 `Kubernetes` 的選型對比（詳見 [[comparisons/container-orchestration-compose-vs-swarm-vs-k8s|容器編排選型：Docker Compose vs Swarm vs Kubernetes]]）。

---

## 重點提煉

1. **宣告式 (Declarative) 與調度器 (Scheduler)**：
   - 放棄傳統命令式腳本，改用 YAML 定義「期望狀態 (Desired State)」，由控制迴圈自動維持副本數與滾動更新（Rolling Update）。
   - Scheduler 透過「過濾 (Filtering) ➔ 評分 (Scoring) ➔ 綁定 (Binding)」三階段決定容器落腳的實體節點。
2. **東西向與南北向網路路由**：
   - **東西向（叢集內部呼叫）**：容器為消耗品、IP 隨時變動，故透過 `Service (ClusterIP)` 與內部 DNS 提供固定域名與自動 Round-Robin 負載平衡。
   - **南北向（外部流量進入）**：避免為每個微服務租用昂貴的雲端 Load Balancer，統一由 `Ingress`（L7 Reverse Proxy）負責路徑路由與 SSL/TLS 憑證卸載（SSL Offloading）。並釐清了 [[comparisons/ingress-vs-api-gateway|Ingress 與 API Gateway 的職責差異]]。
3. **韌性設計：HPA 與雙探針機制 (Probes)**：
   - **HPA (Horizontal Pod Autoscaler)**：依據 CPU 等指標自動擴縮容器數量。
   - **Liveness Probe（存活探針）**：檢查程式是否卡死（Deadlock），失敗則**殺掉容器重啟**。
   - **Readiness Probe（就緒探針）**：檢查程式是否完成熱機（如載入快取、建立連線池），失敗則**暫時從 Service 負載平衡名單剔除**，防止滾動更新時新容器被流量瞬間打爆。
4. **設定抽離與持久化儲存**：
   - 遵循 Twelve-Factor App，將映像檔與環境變數分離：非機密用 `ConfigMap`，機密用 `Secret`。
   - 容器內部磁碟為暫時性（Ephemeral），需透過 `Persistent Volumes (PV)` 跨節點掛載持久化資料。
5. **零信任資安與 Workload Identity**：
   - 透過 `NetworkPolicy` 限制容器間扁平網路的連線權限（如禁止前端容器直連資料庫 3306 port），並以 `RBAC` 管控叢集操作權限。
   - 嚴禁在容器內塞入雲端永久有效金鑰（Long-lived IAM Key），應採用 **Workload Identity** 綁定 Service Account 動態換取短效期 Token。
6. **可觀測性與防範「壞鄰居效應 (Noisy Neighbor)」**：
   - 收集集中式日誌（stdout/stderr 至 Elasticsearch）與時間序列指標（P99 Latency、錯誤率告警）。
   - 強制設定 `Requests`（調度保證下限）與 `Limits`（硬性上限），避免單一異常容器耗盡節點記憶體而觸發 OS OOM Killer 無差別殺死同機其他容器。
