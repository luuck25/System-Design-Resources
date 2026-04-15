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
| **Easier Security & Governance:** Only one "vault" to protect. | **Lack of Context:** Central team may not understand niche data. |
| **Cost:** Lower storage costs (no duplication). | **Rigidity:** Hard to change one part without breaking others. |

> **Best For:** Startups, small-to-medium businesses, or organizations with highly standardized data needs.

# Data Mesh Architecture

## Overview
Data Mesh is a decentralized, sociotechnical approach that treats **Data as a Product**. Instead of one central team, ownership is distributed to the business domains (Sales, Marketing, HR) that actually create the data.


## The Four Pillars of Data Mesh

### 1. Domain-Driven Ownership
Instead of a central IT team managing Sales, Finance, and Web data, the **Sales team** manages Sales data, and the **Finance team** manages Finance data. They are the **"Domain Experts"** who actually understand what the numbers mean.

### 2. Data as a Product
The domain teams don't just "dump" data into a lake; they treat it like a product they are **selling** to the rest of the company. To be a viable product, the data must be:
* **Discoverable:** Easy to find in a central data catalog.
* **Trustworthy:** Clean, accurate, and high-quality.
* **Interoperable:** Follows standard formats so it can be easily joined with data from other teams.

### 3. Self-Serve Data Platform
To prevent every team from having to build their own database from scratch, a **Central Platform Team** provides the "tools." They provide the **"grid"** (cloud storage, compute power, and security tools) so the domain teams can simply plug in and start building their data products without worrying about infrastructure.

### 4. Federated Computational Governance
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
