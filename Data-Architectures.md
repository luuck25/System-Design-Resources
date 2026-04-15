# Data Architectures — A Comprehensive Guide

> How data is **owned, governed, stored, and queried** across an organization.

---

## Table of Contents

1. [Part I — Organizational Architectures](#part-i--organizational-architectures)
   - [Centralized Data Architecture](#1-centralized-data-architecture)
   - [Data Mesh (Decentralized)](#2-data-mesh-architecture-decentralized)
2. [Part II — Storage Architectures](#part-ii--storage-architectures)
   - [Technical Evolution](#3-technical-evolution-the-plumbing)
   - [Data Lakes in Depth](#4-data-lakes)
   - [Data Warehouses in Depth](#5-data-warehouses)
   - [The Two-System Problem](#6-the-two-system-problem-lake--warehouse)
   - [The Data Lakehouse](#7-the-data-lakehouse-architecture)
3. [Part III — Design Patterns & Querying](#part-iii--design-patterns--querying)
   - [Medallion Architecture (Bronze / Silver / Gold)](#8-the-medallion-data-architecture-mda)
   - [Querying Delta Data as Tables in Spark](#9-querying-delta-data-as-tables-in-spark)
4. [Part IV — Data Governance](#part-iv--data-governance)
   - [What is Data Governance?](#10-data-governance)
   - [Governance Across Architectures](#governance-across-architectures)
5. [Appendix — Why External Tables Weren't Enough](#appendix-why-external-tables-werent-enough)

---

# Part I — Organizational Architectures

*How data is **owned and accessed** across teams.*

---

## 1. Centralized Data Architecture

In a centralized architecture, all data flows from various sources into a single, unified repository (like a Data Warehouse or Data Lake) managed by a central IT or Data team.

### Core Structure

| Component | Description |
| :--- | :--- |
| **Data Sources** | Operational systems (CRM, ERP, Web Logs) |
| **Ingestion (ETL)** | Centralized pipelines that move and transform data |
| **The Hub** | A single "Source of Truth" (e.g., Snowflake, BigQuery, or Databricks) |
| **Consumers** | BI Tools, Analysts, and AI models pull from the Hub |

### Pros & Cons

| Pros | Cons |
| :--- | :--- |
| **Consistency:** One single source of truth. | **Bottlenecks:** Central team is often overloaded. |
| **Easier Security & Governance:** Only one "vault" to protect. | **Lack of Context:** Central team may not understand niche data. |
| **Cost:** Lower storage costs (no duplication). | **Rigidity:** Hard to change one part without breaking others. |

> **Best For:** Startups, small-to-medium businesses, or organizations with highly standardized data needs.

<img width="571" height="307" alt="Centralized Architecture Diagram" src="https://github.com/user-attachments/assets/1470e0f4-b56e-44c9-9999-a9a62a325a99" />

---

## 2. Data Mesh Architecture (Decentralized)

Data Mesh is a decentralized, sociotechnical approach that treats **Data as a Product**. Instead of one central team, ownership is distributed to the business domains (Sales, Marketing, HR) that actually create the data.

### The Four Pillars of Data Mesh

#### Pillar 1 — Domain-Driven Ownership

Instead of a central IT team managing Sales, Finance, and Web data, the **Sales team** manages Sales data, and the **Finance team** manages Finance data. They are the **"Domain Experts"** who actually understand what the numbers mean.

#### Pillar 2 — Data as a Product

The domain teams don't just "dump" data into a lake; they treat it like a product they are **selling** to the rest of the company. To be a viable product, the data must be:

* **Discoverable:** Easy to find in a central data catalog.
* **Trustworthy:** Clean, accurate, and high-quality.
* **Interoperable:** Follows standard formats so it can be easily joined with data from other teams.

#### Pillar 3 — Self-Serve Data Platform

To prevent every team from having to build their own database from scratch, a **Central Platform Team** provides the "tools." They provide the **"grid"** (cloud storage, compute power, and security tools) so the domain teams can simply plug in and start building their data products without worrying about infrastructure.

#### Pillar 4 — Federated Computational Governance

Even though teams are independent, they must still follow global rules. This is usually **automated**. For example, if a company rule states *"all emails must be encrypted,"* the platform automatically enforces that policy across all domains, ensuring compliance without slowing down innovation.

### Pros & Cons

| Pros | Cons |
| :--- | :--- |
| **Scalability:** Teams can grow independently. | **Complexity:** Requires high technical maturity. |
| **Speed:** No waiting on a central "middleman." | **Duplication:** Risk of teams "reinventing the wheel." |
| **Quality:** Experts manage their own domain data. | **Governance:** Hard to enforce standards across teams. |

### Architecture Shift

* **From:** "Big Bang" Centralized Warehouse.
* **To:** An ecosystem of interconnected **Data Products**.

> **Best For:** Large enterprises with complex domains and high-velocity data requirements.

---

# Part II — Storage Architectures

*How data is **stored and queried** — the "plumbing."*

---

## 3. Technical Evolution (The "Plumbing")

This describes how we physically store and process the bits and bytes of organizational data.

### Era 1 — The Data Warehouse (1980s – 2010s)

<img width="517" height="126" alt="Data Warehouse Era" src="https://github.com/user-attachments/assets/e32edca3-06df-4960-8825-11678c73dee3" />

* **The Vibe:** "Cleanliness is Next to Godliness."
* **The Tech:** Relational databases like Oracle, Teradata, and later Snowflake/BigQuery.
* **The Philosophy:** **Schema-on-Write.** You had to perfectly model and structure your data *before* loading it into the system.
* **The Limitation:** It was expensive and strictly limited to **structured data** (rows/columns). It couldn't handle "messy" data like images, logs, or long-form text. It acted as a "Black Box"—accessing the data was difficult and costly.

### Era 2 — The Data Lake (2010s – 2020)

* **The Vibe:** "Save Everything, Figure it Out Later."
* **The Tech:** Hadoop (HDFS) and Cloud Storage (Amazon S3, Azure Blob).
* **The Philosophy:** **Schema-on-Read.** You simply dump raw files (JSON, CSV, MP4, PDF) into cheap storage and only apply structure when you are ready to use the data.
* **The Limitation:** It frequently turned into a **"Data Swamp."** Without strict rules, the data became impossible to find or trust. While great for Data Scientists, it was terrible for Business Analysts who needed clean reports. It also lacked **ACID transactions** (the ability to update specific rows without rewriting entire files).

### Era 3 — The Data Lakehouse (2020 – Present)

* **The Vibe:** "The Best of Both Worlds."
* **The Tech:** Databricks (Delta Lake), Apache Iceberg, Apache Hudi.
* **The Philosophy:** You store data in a **Lake** (cheap, raw, and flexible) but add a **Metadata Layer** on top that provides the features of a **Warehouse** (SQL support, row-level deletes, and quality enforcement).
* **Why it's the winner:** It allows organizations to run BI reports and AI/ML models on the **same exact files**. This eliminates the need to move or duplicate data across different systems.

<img width="514" height="127" alt="Lakehouse Era" src="https://github.com/user-attachments/assets/e93d35b6-e936-4a79-96a8-9f9810d4199c" />

<img width="1005" height="487" alt="Architecture Comparison" src="https://github.com/user-attachments/assets/28515374-9b41-482a-aad0-eb863e1a977f" />

---

## 4. Data Lakes

Data Lakes are designed to store **large volumes of raw data at low cost**.

They are typically built on object storage systems such as:

* Amazon S3
* Azure Data Lake Storage
* Google Cloud Storage
* Hadoop HDFS

### Advantages

* Highly scalable storage
* Low storage cost
* Ability to store structured and unstructured data
* Flexible for data science workloads

### Challenges

* Lack of strong governance
* Poor data quality management
* No schema enforcement
* Difficult for BI tools to query efficiently

> Over time many data lakes turned into what engineers call **"Data Swamps."**

---

## 5. Data Warehouses

Data Warehouses were built to support **fast analytical queries and reporting**.

Popular platforms include:

* Ocient
* BigQuery
* Amazon Redshift
* Snowflake

### Advantages

* High-performance SQL queries
* Strong schema enforcement
* Mature governance and security
* Excellent integration with BI tools

### Challenges

* Expensive storage costs
* Limited support for unstructured data
* Heavy ETL pipelines required
* Data duplication between lake and warehouse

In many organizations data had to be **copied from the lake into the warehouse** before analytics could be performed, increasing infrastructure cost, pipeline complexity, and data latency.

---

## 6. The Two-System Problem (Lake + Warehouse)

Organizations were historically forced to maintain two separate silos: a **Data Lake** for raw storage and a **Data Warehouse** for high-performance analytics.

### 6.1 Implementation & Data Flow

Organizations relied on a complex, multi-stage pipeline to manage their information:

* **Initial Ingestion (ELT):** Companies used an **Extract-Load-Transform (ELT)** process to dump raw, unstructured data into a **Data Lake** (utilizing cheap object storage like Amazon S3 or Google Cloud Storage).
* **Secondary Movement (ETL):** Because the lake was technically too slow and unoptimized for business reporting, a second **Extract-Transform-Load (ETL)** pipeline was required. This pipeline would move cleaned, structured, and aggregated data into a physically separate **Data Warehouse** (built on expensive, high-performance block storage).

### 6.2 Data Quality (DQ) and Governance

Under this model, governance was split and often wildly inconsistent:

* **The Warehouse:** Offered high data quality, robust security, and strict **"schema-on-write"** rules. It was the "gold standard" for reliability but was limited in scope.
* **The Data Lake:** Frequently devolved into a **"data swamp"**—a messy, unmanaged repository. In this environment, quality was low, metadata was missing, and finding specific data points was difficult for anyone but the original engineer.

### 6.3 The Main Problem: Operational Friction

This architecture created several critical pain points for the modern enterprise:

* **Massive Data Duplication:** Organizations were forced to pay for the storage and management of the same data in two different places.
* **High Operational Costs:** Maintaining two different tech stacks, two sets of security protocols, and two different engineering teams led to bloated budgets.
* **Stale Data:** Because of the time-consuming process required to move and sync information between the "two boxes," the data available to business leaders in the warehouse was often hours, days, or even weeks behind the actual raw events captured in the lake.

---

## 7. The Data Lakehouse Architecture

<img width="505" height="140" alt="Lakehouse Architecture" src="https://github.com/user-attachments/assets/54c9d426-4ff4-4565-a7bd-18196520368c" />

**Lakehouse** = Data **Lake** + Data Ware**house**. Here we don't maintain a separate Warehouse — stored data in files acts as a warehouse with the help of a Metadata / Table Management Layer.

The **Data Lakehouse** architecture was introduced to eliminate the traditional separation between the Data Lake and the Data Warehouse. Instead of maintaining two disjointed systems, a Lakehouse enables **warehouse-style analytics directly on top of data lake storage.**

> **In simple terms:** A Lakehouse keeps all data in the data lake but adds technology layers that make it behave like a high-performance data warehouse.

This creates a **single, unified platform** that supports:

* Data Engineering & Batch Processing
* Business Intelligence (BI) & Analytics
* Machine Learning (ML)
* AI Workloads

### 7.1 How a Lakehouse Stores Data

A common misunderstanding is that a Lakehouse stores "Lake data" and "Warehouse data" in separate locations. In reality, **all data lives in the same object storage layer.** The defining difference is the introduction of a **Table Management Layer** that converts flat files into structured, queryable tables.

#### Physical Storage Comparison

| Storage Type | Structure Example | Characteristics |
| :--- | :--- | :--- |
| **Traditional Data Lake** | `/raw/clickstream/file1.json` | Simple files; no transaction control; no schema. |
| **Lakehouse Storage** | `/sales_table/part-001.parquet` | Parquet files + **Metadata** + **Transaction Logs**. |

In a Lakehouse, when you run `SELECT * FROM sales_table;`, the user sees a table, but the system is actually reading optimized Parquet files while using metadata to ensure structure and consistency.

### 7.2 The Core Technology: Table Formats

The "magic" that enables the Lakehouse is the **Modern Table Format Layer**. Popular formats include **Delta Lake**, **Apache Iceberg**, and **Apache Hudi**.

These formats add database-like capabilities to standard files:

* **ACID Transactions:** Multiple pipelines can safely read and write data concurrently without corruption.
* **Schema Enforcement:** Prevents "Data Swamps" by ensuring data matches the defined table structure.
* **Time Travel:** Allows users to query historical versions of data (e.g., `SELECT * FROM sales_table VERSION AS OF 5;`).
* **Updates and Deletes:** Traditional lakes only supported "appends." Lakehouses allow standard SQL updates:

  ```sql
  UPDATE sales_table SET price = 100 WHERE product_id = 10;
  ```

---

# Part III — Design Patterns & Querying

---

## 8. The Medallion Data Architecture (MDA)

While traditional centralized systems like Data Warehouses and Data Lakes have concepts of raw and refined data, the **Medallion Data Architecture (MDA)** is specifically defined as the standard implementation for a **Data Lakehouse**.

It is a multi-layered, quality-focused design approach used to ensure that lakehouse data is progressively cleansed and validated as it moves from ingestion to consumption. The architecture organizes data into three distinct stages to bridge the gap between the raw flexibility of a lake and the structured reliability of a warehouse:

### 8.1 Bronze Layer (Raw)

This layer acts as the **landing zone**, preserving the immutable raw state of data exactly as it was received from source systems.

* **Purpose:** To provide a permanent record of the original data.
* **Action:** Minimal transformation; data is stored "as-is" to allow for reprocessing if logic changes later.

### 8.2 Silver Layer (Conformed)

In this stage, data is cleansed, deduplicated, and unified to provide an **"enterprise view"** of key business entities and transactions.

* **Purpose:** To create a reliable, consistent version of the truth for data science and engineering.
* **Action:** Applying schemas, handling null values, and merging disparate data sources.

### 8.3 Gold Layer (Presentation)

Data is highly refined and optimized for **business reporting and analytics**.

* **Purpose:** To serve as a single source of truth for strategic decision-making.
* **Action:** This layer typically utilizes read-optimized dimensional models, such as **Kimball star schemas**, to provide high-speed performance for BI tools.

---

## 9. Querying Delta Data as Tables in Spark

Yes—and this is exactly how a **Lakehouse** is meant to be used. You don't query files directly; you expose them as tables via a **Catalog Layer** so Spark (and other engines) can use SQL naturally.

To achieve a "Delta files underneath, but query like a table" experience, you need:

> **Delta Files + Metadata Layer + Catalog + Query Engine (Spark)**

### Option 1 — Spark SQL Table (Basic Registration)

You register a Delta path as a table in your local Spark session:

```sql
CREATE TABLE orders
USING DELTA
LOCATION '/mnt/data/orders';
```

**Internals:**

* **Spark:** Reads the `_delta_log` and registers the table in the session metastore.
* **Table Points to:** A specific path, not internal proprietary storage.
* **Limitation:** The table is often only visible inside that specific Spark session or metastore; it is not easily shareable across different engines.

### Option 2 — Hive Metastore (The Traditional Way)

**Architecture:** Spark → Hive Metastore (Table Definitions) → Delta Files (GCS/S3/ADLS)

* **Benefits:** Multiple Spark jobs can share the same table definitions. Provides a centralized schema.
* **Limitation:** Hive was not originally designed for versioned tables or advanced metadata (like Iceberg or Delta), leading to only **partial** lakehouse support.

### Option 3 — Unity Catalog (Modern – Databricks)

Unity Catalog provides a central management layer across Spark Clusters, SQL Warehouses, and fine-grained access control (Security).

**Architecture:** Spark / Databricks SQL → Unity Catalog → Delta Tables → Parquet Files

> Provides a **true "database-like experience"** over a raw lake.

### Option 4 — Delta + External Query Engines (Cross-Engine Power)

You can expose Delta tables to engines outside of Spark, such as **Trino** or **Presto**, using dedicated connectors.

**Architecture:** Trino / Spark → Delta Connector → `_delta_log` → Parquet Files

You can run standard SQL without Spark even being active:

```sql
SELECT * FROM delta.orders;
```

### Catalog Comparison

| Feature | Local Spark Table | Hive Metastore | Unity Catalog / Iceberg |
| :--- | :--- | :--- | :--- |
| **Visibility** | Session only | Organization-wide | Multi-workspace / Multi-engine |
| **Security** | None (File-level) | Basic | Fine-grained (Column/Row) |
| **Best For** | Ad-hoc Analysis | Legacy Pipelines | Modern Enterprise Lakehouse |

---

# Part IV — Data Governance

---

## 10. Data Governance

In simple terms, **Data Governance** is the blueprint or "rulebook" for how an organization collects, structures, manages, and uses its information. Its primary goal is to ensure that data is accurate, complete, and consistent so that business users can trust it for making decisions.

### Core Pillars

Effective governance is built on four essential foundations:

* **Data Quality:** Ensuring there are no errors, duplicates, or missing pieces of information.
* **Security and Privacy:** Protecting sensitive data from breaches and ensuring that only authorized people can see specific files.
* **Compliance:** Following legal rules and industry regulations (such as GDPR, HIPAA, or financial reporting standards).
* **Ownership and Accountability:** Defining who is responsible for specific sets of data so there is no confusion about who should fix errors or manage access.

> **The Risk:** Without effective governance, a company's data can turn into a **"data swamp"**—a vast, messy repository of untrustworthy and undocumented files that are impossible to use for reliable analysis.

### Governance Across Architectures

| Architecture | Governance Style | Characteristics |
| :--- | :--- | :--- |
| **Centralized (Warehouse)** | **Rigid & Top-Down** | A single central team cleans and organizes every piece of data before it is allowed into the system. This ensures high quality but often creates **bottlenecks**. |
| **Data Lakehouse** | **Unified & Automated** | Uses technology like **schema enforcement** to automatically block "bad" data and **ACID transactions** to ensure data remains reliable during concurrent updates. |
| **Data Mesh** | **Federated & Decentralized** | Individual business units (like Finance or Marketing) own their data as a **"product."** They follow global standards set by the company but are locally responsible for quality and usability. |

> Ultimately, data governance is about creating a **trustworthy environment** where the right data is available in the right shape, at the right time, for the right person.

---

# Appendix: Why External Tables Weren't Enough

A common question is: *If tools like BigQuery (BQ) supported external tables long ago, why did we ever need a separate warehouse?* The reality is that those early "schema-on-read" connections lacked the performance, reliability, and governance required for enterprise-grade analytics. Technically querying a file in GCS did not provide the same experience or safety as a managed data warehouse.

### A1. The Performance Gap (Optimized vs. Raw)

Traditional warehouses are designed for speed because they use proprietary, highly optimized storage formats.

* **The Warehouse Approach:** When data is loaded into native storage, it is indexed, partitioned, and stored in a specialized columnar format that the engine understands perfectly.
* **The External Table Limitation:** Querying raw files (CSV/JSON) meant the engine had to read unoptimized data over the network every time. It lacked the **indexing, clustering, and materialized views** that make warehouse queries near-instant.

### A2. Lack of ACID Transactions

In the earlier era, data lakes—where external tables reside—lacked **ACID** properties (Atomicity, Consistency, Isolation, Durability).

* **The Risk:** If you queried an external file while another process was writing to it, the query would likely fail or return partial, corrupted data.
* **The Warehouse Solution:** A warehouse ensures a transaction is either **100% complete or it doesn't happen at all**, providing a "single source of truth" trusted for financial or regulatory reporting.

### A3. Schema Enforcement vs. Schema-on-Read

External tables use a "schema-on-read" philosophy, which offers flexibility but creates massive data quality risks.

* **The "Data Swamp" Problem:** Because raw storage enforced no rules, "bad" data (e.g., a string in a numeric column) would land in the lake. The external table would only "break" at the moment someone ran a report, leading to unreliable dashboards.
* **The Warehouse Requirement:** Organizations needed a warehouse to perform **"schema-on-write,"** where data is cleaned and verified *before* it becomes available to users.

### A4. Technical Management Features

Early external table setups lacked the advanced table management capabilities that are now standard in a Lakehouse:

* **Small File Problem:** Data lakes often suffer performance hits when thousands of tiny files are stored. Warehouses handle this automatically; raw external tables do not.
* **Updates and Deletes:** You could not easily "update" a specific row in a CSV file via an external table—you had to rewrite the entire file. Warehouses made these **DML (Data Manipulation Language)** operations efficient.

---

### What Changed?

A modern **Data Lakehouse** is fundamentally different from the old "BigQuery + External Tables" setup.

The Lakehouse uses **Open Table Formats** (Delta, Iceberg, or Hudi) to bring missing warehouse features—ACID, schema enforcement, and indexing—directly to the files in your cloud storage. In the earlier era, you needed the warehouse because an external table was just a view of a messy file; today, the Lakehouse **makes the file behave like a database.**

---

# Appendix B: Inmon vs Kimball — Warehouse Design Approaches

These are the two foundational philosophies for designing a Data Warehouse. Almost every warehouse you'll encounter in the real world is influenced by one (or a hybrid) of these.

---

## Bill Inmon — Top-Down (Enterprise-First)

**Core idea:** Build one massive, normalized, enterprise-wide data warehouse *first*, then derive smaller data marts for individual departments.

### How It Works

```
Source Systems → ETL → [ Enterprise Data Warehouse (3NF) ] → Data Marts (Star Schema)
                              (Single Source of Truth)         ↙     ↓      ↘
                                                           Sales  Finance  Marketing
```

1. **Extract** data from all source systems across the org
2. **Load** it into a single, centralized **Enterprise Data Warehouse (EDW)** modeled in **Third Normal Form (3NF)** — highly normalized, minimal redundancy
3. **Derive** department-specific **data marts** from the EDW for reporting

### Key Characteristics

| Aspect | Detail |
| :--- | :--- |
| **Model** | 3NF (normalized) in the central warehouse |
| **Data Marts** | Created *after* the warehouse, as subsets |
| **Scope** | Enterprise-wide from Day 1 |
| **Redundancy** | Very low — data stored once |
| **Complexity** | High upfront design effort |

### Pros

- **Single source of truth** — one consistent, integrated view of the entire business
- **Low data redundancy** — normalized model avoids duplication
- **Flexibility** — because raw/granular data is preserved, you can build *any* mart later

### Cons

- **Massive upfront investment** — you need to model the *entire* enterprise before delivering value
- **Slow time to delivery** — months/years before business users see their first dashboard
- **Complex queries** — 3NF requires many JOINs, making ad-hoc reporting slow for analysts

---

## Ralph Kimball — Bottom-Up (Department-First)

**Core idea:** Build small, department-specific **dimensional data marts** first (star schemas), and the "warehouse" is simply the *union* of all these marts over time.

### How It Works

```
Source Systems → ETL → [ Data Mart: Sales ]     ──┐
Source Systems → ETL → [ Data Mart: Finance ]    ──┼──→  "Data Warehouse" (virtual/conformed)
Source Systems → ETL → [ Data Mart: Marketing ]  ──┘
                        (Star Schemas)
```

1. Pick the **highest-priority business process** (e.g., Sales)
2. Build a **star schema** data mart for it with **fact** and **dimension** tables
3. Repeat for the next department
4. Use **conformed dimensions** (shared dimensions like `dim_date`, `dim_customer`) to glue the marts together — this *is* your warehouse

### Key Characteristics

| Aspect | Detail |
| :--- | :--- |
| **Model** | Dimensional (Star Schema / Snowflake Schema) |
| **Data Marts** | Built *first* — they ARE the warehouse |
| **Scope** | One business process at a time |
| **Redundancy** | Higher — some denormalization by design |
| **Complexity** | Lower per-iteration, higher over many iterations |

### Core Concepts You Must Know

- **Fact Table** — stores measurable events/metrics (e.g., `fact_sales` with `revenue`, `quantity`, `discount`). Usually very tall (billions of rows) and narrow.
- **Dimension Table** — stores descriptive context (e.g., `dim_product` with `product_name`, `category`, `brand`). Usually short and wide.
- **Star Schema** — fact table at center, dimension tables radiating out. Simple JOINs, fast queries.
- **Snowflake Schema** — dimensions are further normalized (e.g., `dim_product` → `dim_category` → `dim_department`). Saves space but adds JOIN complexity.
- **Conformed Dimensions** — shared dimensions (like `dim_date` or `dim_customer`) that mean the *exact same thing* across all marts. This is what makes cross-department analysis possible.

### Pros

- **Fast time to value** — deliver the first mart in weeks, not years
- **Simple for analysts** — star schemas are intuitive and BI-tool friendly
- **Query performance** — denormalized design = fewer JOINs = faster reports

### Cons

- **Data redundancy** — denormalization means some data is stored multiple times
- **Governance risk** — without disciplined conformed dimensions, marts can diverge and create conflicting "truths"
- **Harder to retrofit** — if you skip conformed dimensions early on, integrating marts later is painful

---

## Head-to-Head Comparison

| Factor | Inmon (Top-Down) | Kimball (Bottom-Up) |
| :--- | :--- | :--- |
| **Starting point** | Model the entire enterprise | Model one business process |
| **Central warehouse model** | 3NF (normalized) | Dimensional (star schema) |
| **Data marts** | Derived from warehouse | *Are* the warehouse |
| **Time to first delivery** | Months–Years | Weeks–Months |
| **Upfront cost** | Very High | Lower |
| **Query complexity** | Many JOINs (slow ad-hoc) | Few JOINs (fast ad-hoc) |
| **Data redundancy** | Low | Higher (by design) |
| **Best for** | Large enterprises needing a single integrated truth | Orgs wanting quick wins per department |
| **Maintenance** | Easier long-term (one model) | Can get messy if conformed dims aren't enforced |

---

## The Modern Hybrid (How Medallion Connects)

In practice, most modern orgs use a **hybrid**: Kimball-style star schemas for the presentation/Gold layer, but with an Inmon-like normalized staging area (Silver layer in Medallion) to preserve raw granularity. The **Lakehouse Medallion architecture** is essentially this hybrid:

- **Bronze** = raw (like Inmon's staging)
- **Silver** = conformed/normalized (Inmon-ish)
- **Gold** = star schemas (Kimball)

> **Interview tip:** Don't pick sides — explain both, then show you understand the modern hybrid approach through the Medallion architecture. That demonstrates depth.

---

# Appendix C: Data Catalog & Data Lineage

## Data Catalog

**What it answers:** *"What data exists and where can I find it?"*

A Data Catalog is a **searchable inventory** of all your data assets — like a library catalog that tells you what books exist, which shelf they're on, and what they're about.

### What a Catalog Stores

- **Table/dataset names** and their locations
- **Schema** — column names, types, descriptions
- **Ownership** — who owns this table, who to ask questions
- **Tags / classifications** — "PII", "financial", "marketing"
- **Usage stats** — how popular is this table, who queries it most
- **Business glossary** — what does "ARR" or "churn" mean in *our* company

### Popular Catalog Tools

| Tool | Type | Notes |
| :--- | :--- | :--- |
| **Unity Catalog** | Commercial (Databricks) | Native to Lakehouse, fine-grained security |
| **Google Data Catalog** | Commercial (GCP) | Auto-discovers BigQuery, GCS, Pub/Sub assets |
| **Apache Atlas** | Open source | Hadoop-ecosystem, common in on-prem setups |
| **DataHub** (LinkedIn) | Open source | Modern, supports Spark, Airflow, dbt, Kafka, Snowflake |
| **Amundsen** (Lyft) | Open source | Lighter weight, discovery-focused |
| **Atlan** | Commercial | Full catalog with collaboration and governance |
| **Collibra** | Commercial | Enterprise governance + catalog + lineage |
| **Alation** | Commercial | Data intelligence platform with strong search |

---

## Data Lineage

**What it answers:** *"Where did this data come from and what happened to it?"*

Data Lineage is the **end-to-end map of how data flows** through an organization — from its origin (source system) through every transformation, join, and aggregation — all the way to the final dashboard or ML model. Think of it as **package tracking** for data.

### Why Lineage Matters

#### 1. Debugging / Root Cause Analysis

A dashboard shows revenue dropped 40% overnight. With lineage, you trace backwards:

```
Dashboard (Gold) → aggregation job → Silver table → Bronze ingestion → Source API
                                          ↑
                                  Schema changed here — new column name broke the JOIN
```

You find the exact pipeline step that broke. **Minutes instead of hours.**

#### 2. Compliance & Audit (GDPR, HIPAA, SOX)

Regulators ask: *"Show us every system that touched this customer's PII."* Lineage gives you:

- Where was the email collected?
- Which pipelines processed it?
- Where is it stored now?
- Was it encrypted/masked at the right step?

#### 3. Impact Analysis

Before changing a column in a source table, lineage tells you **every downstream table, report, and model** that depends on it. Without this, you push a change and 15 dashboards silently break.

#### 4. Data Trust

Business users can click on a metric and see *exactly* how it was calculated — which source tables, which filters, which aggregation logic.

### Types of Lineage

| Type | What It Tracks | Example |
| :--- | :--- | :--- |
| **Table-level** | Which tables feed into which tables | `raw_orders` → `clean_orders` → `fact_sales` |
| **Column-level** | Which specific columns map to which columns | `raw_orders.total_amt` → `fact_sales.revenue` |
| **Job/Pipeline-level** | Which ETL job or query created the data | Airflow DAG `sales_etl` → `fact_sales` |
| **Row-level** | Which source rows contributed to a specific output row | Rare, expensive, but needed for some audit scenarios |

> **Column-level lineage** is the sweet spot most companies aim for.

### How Companies Capture Lineage — Three Approaches

#### Approach 1 — Manual / Documentation-Based (Small Teams)

Engineers maintain a spreadsheet or wiki. **Always out of date, doesn't scale.**

#### Approach 2 — Embedded in Transformation / Orchestration Tools

Modern tools capture lineage *as a side effect* of running pipelines:

| Tool | How It Captures Lineage |
| :--- | :--- |
| **dbt** | Parses SQL models and auto-generates a DAG with table→table and column→column lineage. `dbt docs generate` gives an interactive graph. |
| **Databricks (Unity Catalog)** | Automatically captures table-level and column-level lineage for any query. Zero extra setup. |
| **Google Cloud (Dataplex)** | Auto-captures lineage for BigQuery queries and Dataflow jobs. |
| **Apache Airflow** | Tracks DAG dependencies (task-level), but not column-level without plugins. |

> This is the **most common approach today** — use tools that already emit lineage as a byproduct.

#### Approach 3 — Dedicated Metadata Platforms

For enterprise-scale, a central platform aggregates lineage from *all* tools:

```
┌──────────────┐     Lineage Events      ┌─────────────────────┐
│  Spark Job   │ ──── (OpenLineage) ────→ │                     │
│  Airflow DAG │ ──── (OpenLineage) ────→ │   DataHub / Atlan   │  ← Search, browse,
│  dbt Model   │ ──── (OpenLineage) ────→ │   (Metadata Store)  │     trace lineage
│  BQ Query    │ ──── (API / Logs)  ────→ │                     │
└──────────────┘                          └─────────────────────┘
                                                    ↓
                                            Interactive Lineage Graph
```

**OpenLineage** is the emerging open standard — a vendor-neutral spec for lineage events emitted by Airflow, Spark, dbt, etc.

### Real-World Example

```
POS System (source)
    → Kafka topic: raw_transactions
        → Spark job → Bronze: bronze.transactions
            → dbt model → Silver: silver.clean_transactions
                → dbt model → Gold: gold.daily_sales_by_store
                    → Looker dashboard: "Store Performance"
```

When the CFO asks *"Why does Store #42 show zero revenue yesterday?"*, the data team opens DataHub, clicks on `gold.daily_sales_by_store`, traces upstream, and finds the dbt model had a filter bug introduced in yesterday's PR.

---

## Catalog vs Lineage — How They Relate

```
┌─────────────────────────────────────────────┐
│             DATA CATALOG                     │
│                                              │
│  "Here are all our tables, who owns them,   │
│   what columns they have, and what they      │
│   mean."                                     │
│                                              │
│   ┌───────────────────────────────────┐      │
│   │         DATA LINEAGE              │      │
│   │                                   │      │
│   │  "And here's how table A flows    │      │
│   │   into table B into table C."     │      │
│   └───────────────────────────────────┘      │
│                                              │
└─────────────────────────────────────────────┘
```

| Aspect | Data Catalog | Data Lineage |
| :--- | :--- | :--- |
| **Core question** | *What data do we have?* | *Where did this data come from?* |
| **Analogy** | Library catalog | Package tracking |
| **Focus** | Discovery, documentation, ownership | Flow, transformation, dependencies |
| **Used by** | Analysts finding data, governance teams classifying it | Engineers debugging pipelines, compliance teams auditing |
| **Standalone?** | Yes — a catalog can exist without lineage | Rarely — lineage is almost always viewed *inside* a catalog |

> **Interview one-liner:** A **Data Catalog** is the *what and where* of your data; **Data Lineage** is the *how and from where*. Lineage is typically a feature embedded within a catalog platform, not a separate system.
