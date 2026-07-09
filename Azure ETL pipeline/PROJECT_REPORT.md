# Project Report: Azure End-to-End Sales Data Pipeline & Analytics Dashboard

**Type:** Personal portfolio project (solo build)
**Duration:** ~1 week (constrained by Azure free-trial credit window)
**Domain:** CRM / Sales Analytics
**Dataset:** CRM Sales Opportunities (Accounts, Products, Sales Teams, Sales Pipeline, Data Dictionary) — 8,800 opportunity records

---

## 1. Problem Statement

Sales teams generate opportunity data continuously, but raw CRM exports are rarely analytics-ready: inconsistent nulls, no single source of truth across accounts/products/agents, and no automated path from "data lands somewhere" to "dashboard updates." The goal of this project was to build a small but *production-shaped* pipeline — not just a notebook — that demonstrates the full data engineering lifecycle: ingestion, orchestration, transformation, security, monitoring, alerting, version control, and BI delivery.

## 2. Objectives

- Automate ingestion of 5 related CRM tables into cloud storage on a schedule
- Clean and standardize the data using distributed compute (Spark), not just pandas
- Secure all service-to-service access with managed identity and a secrets vault — no keys in code
- Alert a human whenever the pipeline succeeds or fails, without requiring anyone to check the Azure portal
- Version-control the pipeline itself, not just the code that runs in it
- Turn the curated output into a business-facing Power BI dashboard

## 3. Architecture Decisions & Rationale

| Decision | Why |
|---|---|
| **ADF over Airflow/custom scripts** | ADF is the standard Azure-native orchestrator; demonstrates Copy Activity, linked services, integration runtimes, and Git-based CI/CD patterns that map directly to enterprise Azure data teams. |
| **Self-hosted Integration Runtime** | Simulates a common real-world pattern — pulling data from an on-prem or restricted-network source into the cloud — rather than assuming all sources are already cloud-native. |
| **Databricks (PySpark) over Data Factory Data Flows** | Spark is the industry-standard transformation engine at scale, and gives full control over cleaning logic (null audits, conditional fills) beyond what visual data flows offer. |
| **Serverless Databricks compute** | Free-trial credits are limited; serverless avoids paying for idle provisioned clusters while still using genuine distributed PySpark APIs. |
| **Unity Catalog external locations + Access Connector (instead of raw storage account keys)** | Reflects current Azure best practice for Databricks-to-storage auth — credential-less, auditable, and revocable at the connector level rather than embedding a key in notebook code. |
| **Key Vault for secrets** | No connection strings or keys hardcoded in ADF linked services or notebooks. |
| **Logic Apps + Gmail connector for alerting** | Lightweight, serverless way to get human-readable pipeline status without standing up a separate monitoring stack — appropriate for the scale of this project. |
| **Two-zone (raw / transformed) lake layout instead of full medallion (bronze/silver/gold)** | Deliberately scoped down to fit the 1-week/free-trial budget; the report calls this out explicitly as a scope cut rather than an oversight. |
| **GitHub + ADF Git integration, `dev` → `adf_publish` branch flow** | Mirrors how real ADF CI/CD works: `dev` is the collaboration branch where the ADF Studio authors resources; `adf_publish` holds the auto-generated ARM template + parameters that would feed a release pipeline. |

## 4. Data

The dataset models a mid-size B2B sales organization:

- **`sales_pipeline.csv`** (8,800 rows) — one row per opportunity: agent, account, product, deal stage (`Prospecting → Engaging → Won/Lost`), engage date, close date, close value
- **`accounts.csv`** (85 rows) — company, sector, revenue, employee count, HQ location, parent company
- **`products.csv`** (7 rows) — product, series, list price
- **`sales_teams.csv`** (35 rows) — agent, manager, regional office (Central / East / West)
- **`data_dictionary.csv`** — field-level documentation, ingested and versioned like any other table

## 5. Pipeline Detail

### 5.1 Ingestion (Azure Data Factory)
The `OnPremtoCloudPipeline` runs 5 **Copy Data** activities in sequence, one per source table, landing each as a CSV blob in the `raw-data` container of an ADLS Gen2 account. Two **Web** activities branch off the end of the chain — one fired on success, one on failure — each calling the Logic App's HTTP trigger with the pipeline name, status, run ID, and timestamp. The pipeline runs on a **daily schedule trigger**.

### 5.2 Transformation (Databricks / PySpark)
A single notebook (`Azure Sales Data Cleaning`) does the work:
1. Reads all 5 CSVs from `abfss://raw-data@<storage>.dfs.core.windows.net/` into Spark DataFrames with schema inference
2. Runs a null-count audit across every column of `accounts_df` to quantify data quality before cleaning
3. Applies targeted fixes — e.g., `accounts_df.fillna({"parent_company": "Independent"})` for standalone companies with no parent, `sales_pipeline_df.fillna({"account": "Unknown"})` for orphaned opportunity records
4. Writes all 5 cleaned DataFrames back out as CSV to `abfss://transformed-data@<storage>.dfs.core.windows.net/`, overwriting on each run

### 5.3 Governance & Security
- A dedicated **Access Connector for Azure Databricks** (`oms-access-connector`) is the identity Databricks uses to reach storage — not a personal account or embedded key
- **Unity Catalog** registers two external locations (`oms-ext-location`, `oms-ext-location-raw`) mapping to the transformed and raw containers respectively, each bound to a named credential (`oms-credential`)
- **Key Vault** (`azureeastjadenkey`) holds the storage secrets referenced by ADF linked services
- **Microsoft Entra ID** (Default Directory) backs the managed identities used across the factory and Databricks workspace

### 5.4 Alerting & Monitoring
The Logic App (`azureuseastlogic`) is a stateful workflow: `When an HTTP request is received → Send email (V2)`. Every pipeline run — success or fail — produces an email titled **"Pipeline Run Report"** with the pipeline name, status, run ID, and UTC timestamp, giving a lightweight audit trail without needing a dashboard open. Over the observed monitoring window, **Azure Monitor** recorded 78 succeeded activity runs and 0 failed runs.

### 5.5 Source Control
The factory is Git-connected to a private GitHub repo (`azure-useast-jaden-repo`) with a `main` default branch, a `dev` collaboration branch (8 commits ahead of `main` at last check), and an `adf_publish` branch holding the ARM template + parameters generated by ADF Studio's publish flow — the standard pattern for promoting ADF changes toward a release pipeline.

### 5.6 Serving (Power BI)
The `Sales ETL Dashboard` (single page) reads the curated layer and reports:
- **KPI cards:** total pipeline value, Won revenue, Lost revenue, Win Rate %
- **Bar chart:** revenue by sales agent, stacked by deal stage
- **Line chart:** revenue trend by close month
- **Filled map:** account revenue by HQ location
- **Donut chart:** account count by sector
- **Clustered column:** opportunity count by product
- **Funnel:** accounts by revenue

## 6. Results

| Metric | Value |
|---|---|
| Total opportunities processed | 8,800 |
| Deals Won / Lost / Engaging / Prospecting | 4,238 / 2,473 / 1,589 / 500 |
| **Win rate** | **63.1%** |
| Total Won revenue | **$10,005,534** |
| Top agent by won revenue | Darcel Schlecht ($1,153,214) |
| Top product by won revenue | GTX Pro ($3,510,578) |
| Regional offices covered | Central, East, West |
| Sectors represented | 10 (technology, medical, retail, software, entertainment, marketing, telecom, finance, employment, services) |
| Pipeline run time | ~2 minutes end-to-end |
| Observed reliability | 78/78 succeeded activity runs, 0 failures |

## 7. Challenges & How They Were Addressed

- **Free-trial time/credit pressure** → scoped the lake to two zones instead of a full medallion architecture, used serverless Databricks compute, and skipped Synapse dedicated pools and full CI/CD, documenting these as explicit cuts rather than gaps.
- **Secure Databricks-to-storage access without managing keys in notebooks** → solved with a dedicated Access Connector + Unity Catalog external locations bound to a named credential, instead of embedding an account key.
- **Needing visibility into pipeline health without constantly checking the portal** → solved with a Logic Apps email alert wired to both the success and failure paths of the pipeline, plus Azure Monitor metrics for a longer-term reliability view.
- **Keeping pipeline changes reviewable and reversible** → solved with ADF's native Git integration and a `dev`/`adf_publish` branch split mirroring how ADF's built-in CI/CD workflow is designed to be used.

## 8. Skills Demonstrated

`Azure Data Factory` · `Self-Hosted Integration Runtime` · `PySpark` · `Azure Databricks` · `Unity Catalog` · `ADLS Gen2` · `Azure Key Vault` · `Microsoft Entra ID / Managed Identity` · `Azure Logic Apps` · `Azure Monitor` · `Power BI` · `Git-based CI/CD for data pipelines` · `ARM templates` · `Data quality auditing` · `Sales/CRM data modeling`

## 9. Future Work

- Move the serving layer to **Synapse Serverless SQL** so Power BI queries a proper SQL endpoint instead of reading CSVs directly
- Add **GitHub Actions** to auto-deploy the `adf_publish` ARM templates on merge, closing the CI/CD loop
- Rebuild the Databricks notebook around **Delta Lake** with a true Bronze → Silver → Gold structure and schema enforcement/evolution
- Add automated **data quality gates** (row count checks, null thresholds, referential integrity between `sales_pipeline` and `accounts`/`products`) that can fail the pipeline before publish
- Add **drift/anomaly detection** on incoming daily volumes as a lightweight MLOps extension

---

## Appendix: Resume-Ready Bullet Points

- Designed and built an end-to-end Azure ELT pipeline (ADF → Databricks/PySpark → ADLS Gen2 → Power BI) processing 8,800+ CRM sales records across 5 source tables on a daily schedule with 100% observed pipeline success rate.
- Implemented secure service-to-service data access using Unity Catalog external locations, a dedicated Databricks Access Connector, and Azure Key Vault, eliminating hardcoded credentials from the pipeline.
- Built automated failure/success alerting via Azure Logic Apps, sending real-time pipeline run reports (status, run ID, timestamp) without manual monitoring.
- Version-controlled the entire Data Factory pipeline through Git integration with a dev/publish branch strategy and ARM template-based deployment artifacts.
- Delivered a Power BI dashboard surfacing sales KPIs (63% win rate, $10M+ won revenue) across agents, products, regions, and sectors from the curated data layer.
