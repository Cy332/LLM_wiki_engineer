---
title: "後端工程師的第二堂課 (26) — 容器管理協調系統 (Container Orchestration)"
author: "Johnliutw (John Liu)"
source_url: "https://johnliutw.medium.com/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-26-%E5%AE%B9%E5%99%A8%E7%AE%A1%E7%90%86%E5%8D%94%E8%AA%BF%E7%B3%BB%E7%B5%B1-container-orchestration-f225d5fd1662"
published_at: 2026-06-28
ingested_at: 2026-11-12
---

# 後端工程師的第二堂課 (26) — 容器管理協調系統 (Container Orchestration)

> 原文連結：[Medium - Johnliutw](https://johnliutw.medium.com/%E5%BE%8C%E7%AB%AF%E5%B7%A5%E7%A8%8B%E5%B8%AB%E7%9A%84%E7%AC%AC%E4%BA%8C%E5%A0%82%E8%AA%B2-26-%E5%AE%B9%E5%99%A8%E7%AE%A1%E7%90%86%E5%8D%94%E8%AA%BF%E7%B3%BB%E7%B5%B1-container-orchestration-f225d5fd1662)
> 發布日期：2026-06-28

在我們充分的了解 Docker 後，我們知道 Docker 對於在不同的本地環境，快速的啟動開發具有很棒的相容性，但是有個最重要的問題 — 在 production 環境呢?

在單機環境下，我們習慣把 Docker 當成一個「輕量級的虛擬機」。一台伺服器跑幾個 Container，頂多用 `docker-compose` 串接資料庫與 API。但當使用人數增加，單台伺服器撐不住，我們需要將系統擴展到 3 台、10 台甚至 50 台伺服器時，維運上會有這些挑戰:

* 如果 Container 在非工作時間崩潰，誰來負責把它重新救活？
* 哪一台伺服器現在比較空閒？哪一台資源已經滿載，需要被救活？怎麼知道每台機器的狀態?
* 每次部署新版本，要怎麼做到 0 秒停機，避免使用者體驗變糟?

為了解決這些跨越多台機器的龐大管理難題，容器化協作系統（Container Orchestration System）是實務上的必要設計，其中最著名的就是 Kubernetes（K8s），而這些容器群，又會被叫做叢集 (cluster)。

![Image 1](https://miro.medium.com/v2/resize:fit:700/1*K9piNTmNtcRTg_eJFfpaWw.png)

## 容器化協作的七大核心要點

### 一、 容器程式管理與部署

過去我們寫 Bash Script 或敲打 Docker 指令，使用的是命令式（Imperative）思維。指令必須一步一步給，如果中間指令沒了，程式就會報錯當機。

而在容器化協作系統中，我們採用的是宣告式（Declarative）思維。你不再下達具體的動作指令，而是提交一份期望狀態（Desired State）的藍圖。

在實務上，這份藍圖通常是一份 YAML 檔。你會在裡面宣告：「我需要這個 API 跑 3 個 container，並且限制最多使用 512MB 的記憶體」。協作系統接收到藍圖後，底層的控制系統就會不斷比對「目前狀態」與「期望狀態」。如果某個 Container 意外掛掉，系統發現當前只剩 2 個，不符合期望的 3 個，它就會立刻自動在合適的機器上啟動一個新的 Container。

#### 關於資源調度（Scheduling）

當系統決定要啟動一個新的 container 時，通常會交由調度器（Scheduler）來決定這個容器要去哪台機器上生活，他的判斷順序通常是這樣:

1. **過濾**：調度器會先淘汰掉不符合條件的機器。例如這個容器宣告需要 2GB 的記憶體，那些只剩 1GB 記憶體的機器就會被剔除。
2. **評分**：在剩下的候選機器中，調度器會根據演算法打分數。例如，它會傾向將同一個服務的 3 個容器分散到不同的實體機器上（避免單點故障），或者優先放進資源利用率最優化的機器裡。
3. **綁定**：分數最高的機器勝出，容器正式在那台機器上啟動。

![Image 2](https://miro.medium.com/v2/resize:fit:700/0*XVKPKxhDjF0_Mmth.png)

透過這些機制，我們可以很輕易地實現「滾動更新」。通常系統會先啟動新的，確定沒問題後，再關掉舊的，逐步替換。

假設新容器所使用的新程式碼有嚴重的 Bug，這時容器協調機制通常能立刻暫停整個滾動更新的過程，由於此時線上依然保留著穩定的舊容器在服務客戶，我們就不會遇到任何大斷線的問題。

### 二、 服務發現與內部網路

搞定了容器的管理與部署，接下來我們會遇到第二個難題 — 容器網路管理。

在以前維護 VM 或實體機的時代，機器的 IP 就像是門牌號碼，一旦設定好，幾年都不會變，我們習慣把這些 IP 寫死在設定檔裡。

但在容器化協作系統中，「容器是消耗品，隨時會死掉，也隨時會重生」。既然容器會因為各種原因被銷毀與重建，它每次分配到的 IP 絕對不一樣。為了解決這個問題，協作系統通常會有服務發現（Service Discovery）與內部 DNS 機制。

系統會在容器之上，抽象出一層名為 Service 的物件。這個 Service 擁有一個內部固定不變的 IP（又叫 ClusterIP）；同時，系統內建的 DNS 伺服器，會自動將一個「人類可讀的域名」對應到這個不變的 Service IP 上，來方便人類閱讀管理，並保持 IP 彈性，例如 `http://payment-service:8080`。

#### 流量控制

在系統架構中，外部使用者連進來的流量通常稱為「南北向」流量，而容器之間在叢集內部的互相呼叫，稱為「東西向」流量。

而當容器間在執行東西向呼叫時，協作系統不僅會透過內部 DNS 確保兩者彼此聯絡，它還會負擔 Load Balancing 的工作。Service 層會持續監控背後有幾個健康的容器。當請求打過來時，底層網路會自動做 Round-Robin，將流量平均分散給背後所有的容器，完全不需要你在應用程式裡寫任何 Load Balancing 的邏輯。

### 三、 外部流量路由

![Image 3](https://miro.medium.com/v2/resize:fit:700/0*1FLV6IRdczeYukiS)

當我們解決了東西向流量的問題後，下一個面臨的挑戰就是：「外面的使用者，該怎麼連進來？」這個南北向流量問題。

在剛接觸容器時，許多工程師可能會直覺地，將每個需要對外開放的服務各自綁定一個公開 IP 或 Cloud Load Balancer。但這會導致以下後果：

1. 雲端供應商的每個獨立 Load Balancer 都是按小時計費的，如果有非常多服務，光是網路入口的費用就足以吃一輪。
2. 每個服務的 Domain 與 SSL/TLS 憑證都要分開設定與更新，極容易發生某個冷門容器憑證過期導致中斷的事故。

因此，容器化協作系統應用 Ingress 的概念。它本質上是一個運行在叢集最外層的高效能 Reverse Proxy（常見底層實作為 NGINX、Traefik 或 Envoy）。我們只需要向雲端供應商租用一個外部用的 Load Balancer 指向這個 Ingress，接著所有的外部路由邏輯，全部交由 Ingress 在叢集內部處理。

通常這個 Ingress 會具備 L7（應用層）的功能：

1. **路徑路由**，透過 YAML 定義路由規則：
   * 當使用者訪問 `example.com/api/v1/users` 時，請將流量精準轉發給內部的 `user-service`。
   * 當使用者訪問 `example.com/api/v1/orders` 時，請將流量轉發給 `order-service`。
   * 當使用者訪問 `example.com/` 時，轉發給前端的 `frontend-service`。
2. **SSL 憑證卸載**：SSL 加密通常需要消耗應用程式的 CPU 資源去處理加解密，而在協作系統中，我們採用憑證卸載。系統會將萬用字元憑證（eg: `*.example.com`）統一掛載到 Ingress 上。當外部使用者的 HTTPS 流量抵達 Ingress 時，Ingress 會在這裡負責解密。確定沒問題後，Ingress 到內部服務之間的連線，就直接走純文字的 HTTP。資源消耗變少，也不用逐一管理 SSL!

#### Ingress vs API Gateway

許多工程師會把協作系統的 Ingress 與業務層的 API Gateway 搞混：

![Image 4](https://miro.medium.com/v2/resize:fit:700/1*unD2v9_TxLTntbqOuk4axg.png)

在大型的架構中，這兩者經常是共存的：流量先經過 Ingress 進入叢集，接著 Ingress 將所有 API 相關流量轉發給內部的 API Gateway，Gateway 驗證完權限後，再把請求派發給底層的其他服務。

### 四、 擴展、可用性與韌性設計

分散式系統的殘酷現實是：硬體一定會壞，網路一定會斷。協作系統之所以被稱為韌性（Resilience）架構，在於它具備自我修復與彈性伸縮能力。

在協作系統中，我們依賴 HPA (Horizontal Pod Autoscaler，水平自動擴展) 機制。你可以宣告一系列規則：

* 監控 `order-service` 的所有相關容器
* 如果這群容器的平均 CPU 使用率超過 70%，就自動增加容器數量，最多增加到 20 個
* 如果 CPU 平均掉回 30% 以下時，自動縮減回最少 3 個

#### 健康檢查雙機制 (Liveness vs. Readiness)

![Image 5](https://miro.medium.com/v2/resize:fit:480/0*HrjXltrQzf3ZirxV.jpg)

要讓系統具備自我修復能力，協作系統必須先知道容器「健不健康」。為此，我們必須為應用程式準備兩支重要的 API 端點（通常是 `/health` 或 `/ready`），這被稱為探針（Probes）。

* **Liveness Probe（存活探針）->「你還活著嗎？」**：如果你的程式卡死了，或者資源快爆了導致無法處理任何請求，但 Process 卻沒有結束，這時 Liveness Probe 打過去會超時或收到 HTTP 500。連續幾次失敗後，協作系統的反應是：直接把這個容器殺掉，重新啟動一個新的。
* **Readiness Probe（就緒探針）->「你準備好了嗎？」**：有時容器正在建立龐大的資料庫連線池，或者正在從 Redis 載入數百 MB 的快取，如果 Readiness Probe 失敗，協作系統的反應是：不會殺掉它，而是暫時把它從 Service 的 Load Balancing 名單中剔除，直到它再次回應 200 OK，才把流量重新導向它。

如果你沒有設定 Probe 機制，當系統進行滾動更新，容器只要一啟動，協作系統就會認為它已經準備好了，立刻把線上使用者的流量全部灌進去。如果大型的應用程式還在熱機，根本沒有能力處理請求，結果就是，被分配到這個新容器的使用者，會瞬間面臨大量的 Connection Refused。

### 五、 設定與儲存管理

現代雲端原生架構聖經 Twelve-Factor App（十二要素應用程式），其中的一環就是將設定（Config）從程式碼中抽離。Docker Image 應該要是純淨、與環境無關的。同一個 Image，應該要能夠一路從開發、測試、被部署到正式環境，而不需要重新 Build。

為此，協作系統通常會設定兩種管理機制：
1. **ConfigMap (Config storement)：** 用來儲存非機密的配置，例如日誌層級、外部 API 的網址、分頁數量限制等。
2. **Secret (Secrets storement)：** 用來儲存敏感資訊，例如資料庫密碼、第三方服務的 API Key、TLS 憑證私鑰。

除了一般的 config 類型外，也要特別留意，容器的內部磁碟是暫時性的。為了解決需要永久保存資料的需求，例如資料庫的 Data 檔、使用者上傳的靜態資源，我們可以使用 PV（Persistent Volumes, 持久化儲存卷）。即使容器毀滅重啟，或者被調度到另一台實體機上，這個儲存卷都會自動跟隨過去重新掛載，確保資料不遺漏。

### 六、 安全性與權限控管

![Image 6](https://miro.medium.com/v2/resize:fit:700/0*CE1Ub4vssTti_m5A.jpeg)

在協作系統中，我們必須擁抱零信任架構，實踐最小權限原則。

在預設情況下，協作系統內部是一個巨大的扁平網路。我們需要設定網路策略 (NetworkPolicy)，它就像是附著在服務上的微型防火牆：
* 只有帶有 `app=frontend` 標籤的容器，可以連線到 `backend-service` 上的 8080 port。除此之外的任何流量全部阻擋
* 只有帶有 `app=backend` 標籤的容器，可以連線到 `mysql-service` 的 3306 port。

此外，系統對於「誰可以對叢集下指令」有著嚴格的 RBAC (Role-Based Access Control，角色存取控制)。

#### 容器的身分驗證 (Workload Identity)

一般開發者很常出現的一種安全漏洞：為了讓容器內的程式可以存取雲端，會在雲端後台申請一組永久有效的存取金鑰，然後把它當作 Config 或 Secret 塞進容器裡。一旦這組永久金鑰曝光，駭客就能用這組金鑰在外部為所欲為。

在現代雲端環境的整合中，我們通常使用 Workload Identity (工作負載身分) 機制，在協作系統內建立一個服務帳號（Service Account），並把它分配給這個容器，再向雲端供應商動態換取極短效期的 Token。

### 七、 系統監控、日誌與資源限制

容器協作系統中可觀測性（Observability）基礎設施，主要包含:
1. **集中式日誌**：部署日誌收集代理程式，自動監聽機器上所有容器標準輸出（stdout/stderr）的日誌，吐到遠端的集中式日誌平台（如 Elasticsearch）。
2. **指標監控與告警**：透過時間序列資料庫抓取指標（CPU、RPS、DB 連線池、P99 Latency）並設定告警規則。

#### 壞鄰居效應 (Noisy Neighbor)

在協作系統中，我們必須為每一個容器設定兩種資源宣告：
* **Requests**：容器保證「至少」需要的資源。調度器會用這個數值來決定把容器放在哪台實體機上。
* **Limits**：容器「絕對不可超越」的資源上限。底層作業系統嚴格執法限制。

如果沒有設定正確的 Limits，有些容器的資源用量可能會瘋狂暴增，直到把實體伺服器所有的資源全部吃光。此時作業系統為了自保，可能會觸發 OOM (Out of Memory) 殺手機制，無差別地把同一台伺服器上其他服務容器全部強制關閉。這就是所謂的「壞鄰居效應」。

## 容器協調系統該怎麼選呢?

1. **Docker Compose**：適合本地開發環境標準化與單機部署。致命傷是侷限在單一實體機內，無跨節點高可用性。
2. **Docker Swarm**：Docker 引擎內建的輕量級叢集工具，語法與 Compose 高度相容。適合中小型專案、沒有專職 SRE/DevOps 的小團隊。缺點是生態系組件支援度不如 K8s，且細緻權限與儲存網路管理較弱。
3. **Kubernetes (K8s)**：業界公有雲共同標準，為大規模、高複雜度分散式系統而生（支援 CRD, Operator）。代價是學習曲線陡峭、YAML 繁複、自建維護成本極高（建議使用雲端代管服務如 EKS / GKE）。
