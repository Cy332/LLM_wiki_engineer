---
title: "來源摘要：Medallion Architecture (獎章分層架構) 與 Multi-Hop 資料管線實作"
type: source
domain: [data-eng]
tags: [medallion-architecture, data-lakehouse, elt, kafka, airflow, minio, databricks]
sources:
  - "raw/articles/2026-11-12-medallion-architecture-databricks-pipeline.md"
updated: 2026-11-12
status: evergreen
---

# 來源摘要：Medallion Architecture (獎章分層架構) 與 Multi-Hop 資料管線實作

- **作者**：Databricks Official Glossary & Prabakar S
- **發布日期**：2024-09-06 (Databricks 定義發表於 2022-03-09)
- **原始檔案**：[`raw/articles/2026-11-12-medallion-architecture-databricks-pipeline.md`](../../raw/articles/2026-11-12-medallion-architecture-databricks-pipeline.md)
- **原文連結**：[Databricks Glossary](https://www.databricks.com/glossary/medallion-architecture) | [Medium 實作教學](https://medium.com/@prabakar193/medallion-architecture-a-step-by-step-guide-to-multi-hop-data-processing-563e26e8bf7a)

---

## 核心主旨

本份原始資料結合了 Databricks 官方對 **[[concepts/medallion-architecture|Medallion Architecture (獎章架構 / Multi-hop Architecture)]]** 的標準定義，以及使用 `MinIO (S3)`、`Apache Kafka` 與 `Apache Airflow` 搭建事件驅動三層資料管線的端到端實作。核心精神是在 Data Lakehouse 中透過 **Bronze ➔ Silver ➔ Gold** 三個階段，漸進式提升資料的結構化程度與品質。

---

## 重點提煉

1. **三層資料架構定義 (Bronze / Silver / Gold)**：
   - **Bronze Layer（青銅 / 原始層）**：原封不動（As-is）接收外部來源系統的原始資料，並附加載入時間、Process ID 等 Metadata。重點在於快速擷取（CDC）、保留歷史冷儲存、完整資料血緣（Lineage）與稽核能力，確保隨時可從頭重算（Reprocessing）而無須重新拉取外部來源。
   - **Silver Layer（白銀 / 清洗企業視圖層）**：將 Bronze 資料進行比對、合併、去重與「剛剛好（Just-enough）」的清洗，建立全企業統一的核心業務實體視圖（Enterprise View，如主客戶表、去重交易表）。通常遵循 **ELT** 範式（優先確保資料快速入湖），資料模型偏向 3NF 或寫入優化的 Data Vault，供資料科學家、分析師進行自助式分析與 ML 特徵工程。
   - **Gold Layer（黃金 / 應用聚合層）**：針對特定業務專案或報表需求進行最終商業邏輯轉換與品質檢核。採用高度反正規化（Denormalized）、讀取優化且較少 Join 的模型（如 Kimball 星狀綱要 Star Schema 或 Inmon Data Marts），直接服務 BI 儀表板、高管報表與推薦系統。
2. **Event-Driven Multi-Hop 實作管線 (MinIO + Kafka + Airflow)**：
   - **儲存層**：在 MinIO (S3 相容物件儲存) 建立 `bronze`、`silver`、`gold` 三個 Bucket。
   - **事件觸發 (Lambda Notification)**：設定 MinIO Bucket Event，當有檔案上傳（`PUT`）至 `bronze` 或 `silver` 時，自動發送事件訊息至對應的 Kafka Topic（`bronze_layer_data` 與 `silver_layer_data`）。
   - **編排與轉換 (Airflow DAGs)**：
     - Silver DAG 監聽 `bronze_layer_data`，拉取原始 Parquet 檔清洗後寫入 `silver` Bucket。
     - Gold DAG 監聽 `silver_layer_data`，拉取清洗後的 Parquet 檔，依日期進行業務聚合並輸出至 `gold` Bucket 的日期分區目錄中。
