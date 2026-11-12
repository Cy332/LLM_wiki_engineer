---
title: "Kubernetes (K8s) 容器編排與雲端原生基建實務"
type: tool
domain: [infra, backend]
tags: [kubernetes, k8s, container-orchestration, service, ingress, hpa, probes, workload-identity]
sources:
  - "[[sources/2026-11-12-container-orchestration-k8s-johnliu]]"
  - "[[sources/2026-11-12-redis-hotkey-bigkey-titansoft]]"
updated: 2026-11-12
status: evolving
---

# Kubernetes (K8s) 容器編排與雲端原生基建實務

## 1. 核心解決什麼問題

當後端系統從單機 `Docker` / `docker-compose` 走向多台伺服器的 Production 環境時，單純的容器工具無法解決跨機器的高可用與維運難題：
- 容器在半夜崩潰時誰自動重啟它？某台實體機死機時誰把容器遷移到健康的機器？
- 50 台機器中哪台資源最空閒適合放新容器？
- 如何在 **0 秒停機（Zero-Downtime）** 的情況下完成新版本部署與退版？

**Kubernetes (K8s)** 作為業界標準的容器管理協調系統（Container Orchestration），透過**宣告式（Declarative）控制迴圈**將多台伺服器抽象為單一叢集（Cluster），自動化處理容器調度、服務發現、流量路由、彈性擴縮與自我修復。

> 關於何時該用 Docker Compose、Docker Swarm 或 Kubernetes，請見：[[comparisons/container-orchestration-compose-vs-swarm-vs-k8s|容器編排選型：Docker Compose vs Swarm vs Kubernetes]]。

---

## 2. 核心架構與運作機制

### 2.1 宣告式管理 (Declarative Desired State) 與調度三階段
工程師不再下達一步步的命令式（Imperative）指令，而是透過 YAML 提交**期望狀態（Desired State，如 `replicas: 3`）**，由 K8s Controller 持續比對「實際狀態」並自動補齊差額。
當 **Scheduler（調度器）** 決定新建 Pod 落腳哪台 Node 時，會執行三階段決策：
1. **過濾 (Filtering / Predicates)**：剔除資源不足（如剩餘記憶體小於容器宣告的 `Requests`）或不符合條件的節點。
2. **評分 (Scoring / Priorities)**：對候選節點打分數（例如優先將同一服務的副本分散至不同實體機以防單點故障，或選擇資源利用率最優的機器）。
3. **綁定 (Binding)**：將 Pod 指派給最高分的節點啟動。

---

### 2.2 網路拓撲：東西向 (`Service`) vs 南北向 (`Ingress`)

在 K8s 中，**「容器是消耗品（Cattle, not Pets），隨時會死掉重建，每次拿到的 Pod IP 都不同」**。因此絕不可在設定檔寫死 Pod IP：

| 流量方向 | K8s 核心物件 | 運作機制與解決痛點 |
| :--- | :--- | :--- |
| **東西向流量**<br/>*(叢集內微服務互叫)* | **Service (`ClusterIP`)** + **CoreDNS** | 為一組動態增減的 Pod 提供一個**固定不變的虛擬 IP (ClusterIP)** 與內部域名（如 `http://payment-service:8080`），底層自動對健康的 Pod 執行 Round-Robin 負載平衡。 |
| **南北向流量**<br/>*(外部使用者連入叢集)* | **Ingress** *(L7 Reverse Proxy，如 NGINX / Envoy / Traefik)* | 避免為每個微服務各自購買昂貴的雲端 Load Balancer（ALB/NLB）與重複管理憑證。全叢集只需**一個外部 Load Balancer 指向 Ingress**，由 Ingress 統一執行 **SSL/TLS 憑證卸載 (SSL Offloading)**，再依 URL 路徑（如 `/api/v1/orders`）以純 HTTP 轉發給內部 Service。 |

> 關於基礎設施層的 `Ingress` 與業務層 `API Gateway` 的差異與共存架構，請參閱：[[comparisons/ingress-vs-api-gateway|Ingress 與 API Gateway 的職責邊界與對比]]。

---

## 3. 實務最佳實踐：韌性、儲存與資安配置

### 3.1 健康檢查雙探針 (`Liveness Probe` vs `Readiness Probe`)

為支援自我修復與零停機滾動更新（Rolling Update），後端服務必須實作探針端點：

| 探針類型 | 核心問題 | 檢測失敗時 K8s 的反應 | 實務適用情境 |
| :--- | :--- | :--- | :--- |
| **Liveness Probe**<br/>*(存活探針)* | **「你還活著嗎？」** | **直接殺掉該容器並重新啟動一個新的。** | 程式發生 Deadlock 卡死、無限迴圈，Process 雖未退出但已無法回應任何請求。 |
| **Readiness Probe**<br/>*(就緒探針)* | **「你準備好接客了嗎？」** | **不殺掉容器**，而是**暫時將它從 Service 的負載平衡名單中剔除**，等恢復 `200 OK` 再導回流量。 | 容器剛啟動正在建立龐大的 DB Connection Pool、或正在從 [[tools/redis|Redis]] 載入數百 MB 快取預熱。 |

### 3.2 資源限制 (`Requests` vs `Limits`) 與防範「壞鄰居效應」
在同一台實體 Node 上會混部多個微服務容器，必須在 YAML 明確宣告 CPU 與 Memory 界線：
- **`requests`（保證下限）**：容器至少需要的資源量，**Scheduler 依此數值決定調度**。
- **`limits`（硬性上限）**：容器絕對不可超越的天花板。
- **壞鄰居效應 (Noisy Neighbor)**：若未設定 `limits`，某個發生 Memory Leak 的爛服務會吃光整台實體機的記憶體，觸發 Linux 作業系統的 **OOM Killer（Out-of-Memory Killer）** 無差別殺死同一台機器上的其他正常服務容器！

### 3.3 配置抽離 (`ConfigMap` / `Secret`) 與持久化儲存 (`PV`)
- **Twelve-Factor App 原則**：同一個 Docker Image 應不需重新 Build 即可部署至 Dev / Staging / Prod。非機密設定放 `ConfigMap`，敏感資訊（DB 密碼、API Key）放 `Secret`，於啟動時掛載為環境變數或檔案。
- **Persistent Volumes (PV)**：容器內部磁碟是暫時性的（重啟即還原至 Image 初始狀態）。需永久保存的檔案或有狀態資料必須透過 `PV / PVC` 掛載外部雲端磁碟。

### 3.4 零信任網路 (`NetworkPolicy`) 與雲端身分驗證 (`Workload Identity`)
1. **NetworkPolicy（微型防火牆）**：K8s 預設為扁平網路（所有 Pod 可互連）。必須配置 `NetworkPolicy` 實施最小權限（例如：只允許 `app=backend` 連線至 `mysql-service:3306`，阻斷 `app=frontend` 直連 DB，縮小駭客入侵後的爆炸半徑）。
2. **Workload Identity（取代永久 IAM 金鑰）**：
   - **Anti-pattern**：在雲端後台開一組永久有效的 IAM Access Key / Secret Key 塞進 K8s `Secret`（一旦外洩駭客即可在外部任意存取雲端資源）。
   - **最佳實踐**：使用雲端原生的 **Workload Identity (GKE) / IRSA (AWS EKS)**，將 K8s 內的 `ServiceAccount` 綁定雲端 IAM Role，由底層自動注入極短效期的動態 Token。

---

## 4. 常見坑點 (Gotchas & Anti-patterns)

- **坑點 1：未設 `Readiness Probe` 導致每次發版（Rolling Update）線上大量報錯**
  - **原因**：沒有設定 Readiness Probe 時，K8s 只要看到新容器的 Process 一啟動（哪怕 Spring Boot / Node.js 還在初始化、還沒連上 DB），就會立刻把線上真實流量灌進去並砍掉舊容器，導致用戶瞬間收到大量 `Connection Refused` 或 `502 Bad Gateway`。
- **坑點 2：下游元件（如 Redis BigKey）阻塞導致 K8s `HPA` 縮容失靈**
  - **真實案例**：在 [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|鈦坦科技事故案例]] 中，當後端服務高頻拉取 Redis 內的 20MB BigKey 時，網路阻塞與反序列化開銷會造成 Client Pod 的 CPU 使用率劇烈忽高忽低，導致 K8s **HPA (Horizontal Pod Autoscaler)** 擴展到最大 Pod 數後永遠無法觸發 Scale-down，造成嚴重的運算資源浪費。

---

## 5. 關聯頁面
- [[comparisons/container-orchestration-compose-vs-swarm-vs-k8s|容器編排選型：Docker Compose vs Swarm vs Kubernetes]]
- [[comparisons/ingress-vs-api-gateway|Ingress 與 API Gateway 的職責邊界與對比]]
- [[postmortems/2026-11-12-redis-hotkey-bigkey-cluster-failover|事故覆盤：Redis HotKey 與 20MB BigKey 引發叢集 Failover 與 K8s HPA 失靈]]
- [[sources/2026-11-12-container-orchestration-k8s-johnliu|來源摘要：後端工程師的第二堂課 (26) — 容器管理協調系統]]
