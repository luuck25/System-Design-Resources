# Data Mesh / Data Fabric/Centralized Architecture - how data is owned, governed, and accessed across an organization
# Centralized Data Architecture

## Overview
In a centralized architecture, all data flows from various sources into a single, unified repository (like a Data Warehouse or Data Lake) managed by a central IT or Data team.

### Core Structure
* **Data Sources:** Operational systems (CRM, ERP, Web Logs).
* **Ingestion (ETL):** Centralized pipelines that move and transform data.
* **The Hub:** A single "Source of Truth" (e.g., Snowflake, BigQuery, or Databricks).
* **Consumers:** BI Tools, Analysts, and AI models pull from the Hub.

### Pros & Cons
| Pros | Cons |
| :--- | :--- |
| **Consistency:** One single source of truth. | **Bottlenecks:** Central team is often overloaded. |
| **Security:** Only one "vault" to protect. | **Lack of Context:** Central team may not understand niche data. |
| **Cost:** Lower storage costs (no duplication). | **Rigidity:** Hard to change one part without breaking others. |

> **Best For:** Startups, small-to-medium businesses, or organizations with highly standardized data needs.

# Data Mesh Architecture

## Overview
Data Mesh is a decentralized, sociotechnical approach that treats **Data as a Product**. Instead of one central team, ownership is distributed to the business domains (Sales, Marketing, HR) that actually create the data.

### The Four Pillars
1.  **Domain Ownership:** The people closest to the data own it.
2.  **Data as a Product:** Data must be discoverable, usable, and high-quality.
3.  **Self-Serve Platform:** A central team provides the "tools," but not the data itself.
4.  **Federated Governance:** Global rules (like privacy) are automated across all domains.

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
