# Azure End-to-End Sales Data Pipeline & Analytics Dashboard

**Batch ELT pipeline that ingests raw CRM sales data, transforms it in Spark, and serves it to a Power BI dashboard — built on the Azure free-tier in a one-week sprint.**

![Azure](https://img.shields.io/badge/Azure-0078D4?style=flat&logo=microsoftazure&logoColor=white)
![Data Factory](https://img.shields.io/badge/Azure%20Data%20Factory-0062AD?style=flat&logo=azuredataexplorer&logoColor=white)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat&logo=databricks&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=flat&logo=apachespark&logoColor=white)
![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=flat&logo=powerbi&logoColor=black)
![Logic Apps](https://img.shields.io/badge/Logic%20Apps-0062AD?style=flat&logo=microsoftazure&logoColor=white)

---

## 📌 Overview

This project simulates a real-world CRM sales analytics use case: a company's sales pipeline data (opportunities, accounts, products, and sales teams) lands as raw CSV files and needs to be cleaned, validated, and made available for business reporting — reliably, on a schedule, with failure alerts and no manual intervention.

I built a **medallion-style ELT pipeline** entirely on Azure that:

- Ingests 5 source tables (**8,800 sales opportunities**, 85 accounts, 7 products, 35 agents across 3 regions) into a Data Lake
- Cleans and transforms the data with **PySpark on Databricks**
- Publishes curated data to a Power BI dashboard for sales performance reporting
- Runs on a **daily schedule via Azure Data Factory**, with **email alerts on every success/failure** via Logic Apps
- Is fully **version-controlled** (ADF Git integration + ARM templates) and secured with **Key Vault + Managed Identity**

> Built solo, end-to-end, on a 7-day Azure free-trial credit budget — including architecture design, infrastructure provisioning, pipeline development, and dashboarding.

---

## 🏗️ Architecture

![Architecture Diagram](assets/architecture-diagram.png)

| Stage | Service | Purpose |
|---|---|---|
| Ingestion | **Azure Data Factory** (Copy Activity, Self-Hosted Integration Runtime) | Pulls raw CSVs into ADLS Gen2 (`raw-data` container) |
| Storage | **Azure Data Lake Storage Gen2** | Raw and transformed data zones (`raw-data`, `transformed-data` containers) |
| Transformation | **Azure Databricks (PySpark, Serverless compute)** | Null handling, type casting, standardization, dedup |
| Governance | **Unity Catalog external locations + Access Connector** | Secure, credential-based access from Databricks to ADLS |
| Secrets | **Azure Key Vault** | Storage keys and connection secrets, no hardcoded credentials |
| Identity | **Microsoft Entra ID + Managed Identity** | Service-to-service auth between ADF, Databricks, and storage |
| Orchestration | **ADF Pipeline** (`OnPremtoCloudPipeline`), daily trigger | Sequenced Copy Data activities per table with success/fail branching |
| Alerting | **Azure Logic Apps + Gmail connector** | Sends a pipeline run report (name, status, run ID, timestamp) on every trigger |
| Monitoring | **Azure Monitor** | Tracks succeeded vs. failed pipeline runs over time |
| Source control | **GitHub (ADF Git integration)** | `main` / `dev` / `adf_publish` branch strategy, ARM template CI artifacts |
| Serving | **Power BI Desktop** | Sales performance dashboard on top of the curated layer |

**Data flow:** `External CSV sources → ADF (Copy Activity) → ADLS raw-data → Databricks (PySpark cleaning) → ADLS transformed-data → Power BI`

---

## 🔧 Pipeline Walkthrough

### 1. Infrastructure
All resources were provisioned under a single resource group (`azure_us_east_rg`) — Data Factory, Storage Account, Databricks workspace, Key Vault, Logic App, and an Access Connector for Databricks — keeping the footprint small enough to fit inside the free-trial credit window.

![Resource Group](assets/01-resource-group.png)

### 2. Ingestion — Azure Data Factory
A single pipeline (`OnPremtoCloudPipeline`) runs 5 parallel **Copy Data** activities — one per source table (`accounts`, `data_dictionary`, `products`, `sales_pipeline`, `sales_teams`) — from a self-hosted integration runtime into the `raw-data` container, followed by conditional **Web** activities that report `Success`/`Fail` downstream.

![ADF Pipeline Canvas](assets/05-adf-pipeline-canvas.png)
![Pipeline Run Succeeded](assets/06-pipeline-run-success.png)

The factory is connected to GitHub for source control, with a `dev` branch for development and an `adf_publish` branch holding the generated ARM templates for deployment.

![ADF Git Integration](assets/07-adf-git-integration.png)

### 3. Storage — ADLS Gen2
Data lands in a **raw-data** container as-is, and lands again in **transformed-data** after processing — a simple two-zone (bronze/silver) layout.

![Raw Data Container](assets/03-raw-data-container.png)

### 4. Transformation — Databricks (PySpark)
A Databricks notebook (serverless compute) reads all 5 raw CSVs directly from ADLS via `abfss://`, applies cleaning logic (e.g. filling nulls in `parent_company` and `account`, auditing null counts per column), and writes the cleaned frames back to the `transformed-data` container as CSV.

![Notebook — Ingest](assets/08-databricks-notebook-ingest.png)
![Notebook — Cleaning](assets/09-databricks-notebook-cleaning.png)
![Notebook — Write](assets/10-databricks-notebook-write.png)

Access from Databricks to storage is governed through **Unity Catalog external locations** backed by a dedicated Databricks Access Connector — not raw storage keys embedded in code.

![Unity Catalog External Locations](assets/11-unity-catalog-external-locations.png)

### 5. Alerting — Logic Apps
An HTTP-triggered Logic App receives the pipeline's run metadata and emails a **Pipeline Run Report** (name, status, run ID, timestamp) after every execution — success or failure — so the pipeline is monitored without needing to log into the Azure portal.

![Logic App Designer](assets/13-logic-app-email-alert-designer.png)
![Email Alert](assets/14-email-alert-notification.png)

### 6. Monitoring
Azure Monitor tracks pipeline run outcomes over time as a metric, giving a quick view of reliability (succeeded vs. failed runs).

![Azure Monitor Metrics](assets/15-azure-monitor-metrics.png)

### 7. Serving — Power BI
The curated `transformed-data` layer feeds a single-page **Sales ETL Dashboard** in Power BI with:
- Revenue by sales agent, segmented by deal stage
- Revenue trend by month
- Won revenue / Lost revenue / Total pipeline value cards + Win Rate %
- Opportunity count by product
- Account revenue by geography (map) and by sector (donut)
- Deal funnel by account

---

## 📊 Key Results (from the underlying dataset)

| Metric | Value |
|---|---|
| Opportunities processed | 8,800 |
| Accounts / Products / Agents | 85 / 7 / 35 |
| Deals Won | 4,238 |
| Deals Lost | 2,473 |
| **Win rate** (Won ÷ Won+Lost) | **63.1%** |
| Total Won revenue | **$10,005,534** |
| Top sales agent (by won revenue) | Darcel Schlecht — $1.15M |
| Top product (by won revenue) | GTX Pro — $3.51M |
| Pipeline schedule | Daily trigger, ~2 min run time |
| Pipeline reliability (observed) | 78 succeeded / 0 failed activity runs (7-day monitor window) |

---

## 🗂️ Repository Structure
```
├── datafactory/
│   ├── pipeline/              # OnPremtoCloudPipeline definition
│   ├── dataset/               # Source & sink dataset definitions
│   ├── linkedService/         # AzureBlobStorageLocation, AzureDataLakeStorage1
│   ├── integrationRuntime/    # Self-hosted IR config
│   ├── trigger/               # Daily schedule trigger
│   └── Azurejadenfactory/     # ARM template + parameters (adf_publish branch)
├── notebooks/
│   └── azure_sales_data_cleaning.py   # Databricks PySpark ingestion & cleaning
├── data/                      # Source CSVs (accounts, products, sales_pipeline, sales_teams, data_dictionary)
├── dashboard/
│   └── Azure_ETL_Dashboard.pbix
└── assets/                    # Architecture diagram + pipeline screenshots
```

---

## ⚙️ Tech Stack

**Cloud & Orchestration:** Azure Data Factory, Azure Logic Apps, Azure Monitor
**Compute:** Azure Databricks (PySpark, serverless)
**Storage:** Azure Data Lake Storage Gen2 (hierarchical namespace)
**Security:** Azure Key Vault, Microsoft Entra ID, Managed Identity, Unity Catalog Access Connector
**BI:** Power BI Desktop
**DevOps:** GitHub (ADF Git integration), ARM templates

---

## 🚀 Reproducing This Project

1. **Provision resources**: resource group, ADLS Gen2 storage account (with 4 containers: `raw-data`, `transformed-data`, and any staging containers), Data Factory (V2), Databricks workspace (Premium tier, for Unity Catalog + Access Connector), Key Vault, Logic App.
2. **Wire up security**: create a Databricks Access Connector, grant it `Storage Blob Data Contributor` on the storage account, register external locations in Unity Catalog pointing at `raw-data` and `transformed-data`.
3. **Build the ADF pipeline**: 5 Copy Data activities (one per CSV) from source → `raw-data`, followed by success/fail Web activities calling the Logic App's HTTP trigger.
4. **Author the Databricks notebook**: read from `abfss://raw-data@<account>.dfs.core.windows.net/`, clean, and write to `abfss://transformed-data@<account>.dfs.core.windows.net/`.
5. **Connect Power BI** to the `transformed-data` container (or a serving layer of your choice) and build the report.
6. **Schedule** a daily trigger on the pipeline and confirm the Logic App email fires on both success and failure paths.

> ⚠️ This was built to fit inside a 7-day Azure free-trial credit window. Serverless Databricks compute and Serverless SQL (where applicable) were chosen deliberately to control cost — swap for provisioned clusters/dedicated pools for production workloads.

---

## 🔭 Future Work
- Add Synapse Serverless SQL as a query layer over `transformed-data` instead of Power BI reading CSV directly
- Introduce CI/CD (GitHub Actions) to auto-deploy ARM templates from `adf_publish` on merge
- Extend the Databricks notebook into a proper Bronze → Silver → Gold Delta Lake structure with schema enforcement
- Add data quality checks (Great Expectations / custom PySpark assertions) as a pipeline gate before publish

---

## 📬 Contact
Built by **Sanjay** — [GitHub](https://github.com/Sanjayvk98) · open to AI/ML Engineer and Data Engineering roles.
