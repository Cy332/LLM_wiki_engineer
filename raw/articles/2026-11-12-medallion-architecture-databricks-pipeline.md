---
title: "What is Medallion Architecture? (Databricks Glossary & Multi-Hop Pipeline Implementation)"
author: "Databricks & Prabakar S"
source_url: "https://www.databricks.com/glossary/medallion-architecture"
secondary_url: "https://medium.com/@prabakar193/medallion-architecture-a-step-by-step-guide-to-multi-hop-data-processing-563e26e8bf7a"
published_at: 2024-09-06
ingested_at: 2026-11-12
---

# Part 1: What is Medallion Architecture? (Databricks Official Definition)

> Source: [Databricks Glossary - Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)

## What is a medallion architecture?

A **medallion architecture** is a data design pattern used to logically organize data in a lakehouse, with the goal of incrementally and progressively improving the structure and quality of data as it flows through each layer of the architecture (from Bronze ⇒ Silver ⇒ Gold layer tables). Medallion architectures are sometimes also referred to as "multi-hop" architectures.

## What is the Bronze layer in a medallion architecture?

The **Bronze layer** is where we land all the data from external source systems. The table structures in this layer correspond to the source system table structures "as-is," along with any additional metadata columns that capture the load date/time, process ID, etc. The focus in this layer is quick Change Data Capture and the ability to provide an historical archive of source (cold storage), data lineage, auditability, reprocessing if needed without rereading the data from the source system.

## What is the Silver layer in a medallion architecture?

In the **Silver layer** of the lakehouse, the data from the Bronze layer is matched, merged, conformed and cleansed ("just-enough") so that the Silver layer can provide an "Enterprise view" of all its key business entities, concepts and transactions. (e.g. master customers, stores, non-duplicated transactions and cross-reference tables).

The Silver layer brings the data from different sources into an Enterprise view and enables self-service analytics for ad-hoc reporting, advanced analytics and ML. It serves as a source for Departmental Analysts, Data Engineers and Data Scientists to further create projects and analysis to answer business problems via enterprise and departmental data projects in the Gold Layer.

In the lakehouse data engineering paradigm, typically the ELT methodology is followed vs. ETL - which means only minimal or "just-enough" transformations and data cleansing rules are applied while loading the Silver layer. Speed and agility to ingest and deliver the data in the data lake is prioritized, and a lot of project-specific complex transformations and business rules are applied while loading the data from the Silver to Gold layer. From a data modeling perspective, the Silver Layer has more 3rd-Normal Form like data models. Data Vault-like, write-performant data models can be used in this layer.

## What is the Gold layer in a medallion architecture?

Data in the **Gold layer** of the lakehouse is typically organized in consumption-ready "project-specific" databases. The Gold layer is for reporting and uses more de-normalized and read-optimized data models with fewer joins. The final layer of data transformations and data quality rules are applied here. Final presentation layer of projects such as Customer Analytics, Product Quality Analytics, Inventory Analytics, Customer Segmentation, Product Recommendations, Marking/Sales Analytics etc. fit in this layer. We see a lot of Kimball style star schema-based data models or Inmon style Data marts fit in this Gold Layer of the lakehouse.

## Frequently Asked Questions

### What's the difference between ETL and ELT when using a medallion architecture?
A medallion architecture typically follows an ELT (extract, load, transform) approach rather than traditional ETL. Only minimal, "just-enough" cleansing and standardization happens while data is loaded into the Silver layer, prioritizing speed and agility in getting data into the lake. The bulk of complex, project-specific transformations and business rules are applied later, when data moves from the Silver layer into the Gold layer.

### Can medallion architecture work with streaming data, or is it only for batch processing?
Medallion architecture supports both batch and streaming data. You can build streaming pipelines using streaming tables and materialized views built on Apache Spark Structured Streaming, which are incrementally refreshed and updated as new data arrives.

### How does the Bronze layer help with data governance and auditing?
The Bronze layer preserves raw source data "as-is," along with metadata columns such as load date/time and process ID, which supports change data capture, historical archiving and full data lineage. Because the original data is retained untouched, teams can reprocess it at any time without having to re-read it from the source system, which strengthens auditability.

---

# Part 2: Building Multi-Hop Data Processing using MinIO, Kafka & Airflow

> Source: [Medium - Prabakar S](https://medium.com/@prabakar193/medallion-architecture-a-step-by-step-guide-to-multi-hop-data-processing-563e26e8bf7a)

## Practical Implementation Overview

* **Foundation Blocks:** Set up **MinIO** for S3-compatible object storage (with `bronze`, `silver`, and `gold` buckets), **Apache Kafka** for event messaging, and **Apache Airflow** for workflow orchestration, all containerized via Docker Compose.
* **Event-Driven Triggering:** Configure **Lambda events** in MinIO (`arn:minio:sqs:bronze_layer:kafka` and `arn:minio:sqs:silver_layer:kafka`) subscribed to `PUT — Object Uploaded` events.
* **End-to-End Data Journey:**
  1. **Bronze Ingestion:** Raw `data.parquet` is uploaded into the `bronze` bucket in MinIO.
  2. **Bronze -> Silver Trigger:** MinIO triggers a notification to the Kafka topic `bronze_layer_data`.
  3. **Silver Transformation (Airflow DAG):** The `2_silver_layer_processing` Airflow DAG consumes the Kafka event, pulls `data.parquet` from the `bronze` bucket, performs data cleaning/deduplication, and writes `cleaned_data.parquet` to the `silver` bucket.
  4. **Silver -> Gold Trigger:** Uploading to the `silver` bucket triggers another event to the Kafka topic `silver_layer_data`.
  5. **Gold Aggregation (Airflow DAG):** The Gold Airflow DAG consumes the event, retrieves `cleaned_data.parquet` from `silver`, aggregates records by date into consumption-ready JSON/analytical structures, and writes them to date-partitioned folders in the `gold` bucket.
