Product Launch Performance Data Modeling

## Interview Problem Statement

> **Role:** Staff Data Engineer at a global consumer electronics manufacturer  
> **Stakeholder:** Merchandising and Category Management Team  
> **Request:** "We want to understand how our products performed in their early years in the market. Specifically, we want to track revenue growth on sales across the first 3 years from launch. The team should be able to compare products in the same launch cohort to identify which products had strong early adoption, which ones took time to pick up, and which ones never really took off."

---

## Business Definitions: Adoption Patterns

Before diving into the technical solution, it's critical to define what the business means by each adoption pattern:

| Adoption Pattern | Definition | Business Indicators | Example |
|------------------|------------|---------------------|----------|
| **Strong Early Adoption** | Products that achieve significant revenue quickly after launch and maintain momentum | High Year 1 revenue (>50% of 3-year total), consistent month-over-month growth, rapid market penetration | Flagship phone launch with pre-orders, viral product |
| **Took Time to Pick Up (Slow Burn)** | Products with modest initial sales that grow substantially over time through word-of-mouth or market education | Low Year 1 revenue (<30% of 3-year total), accelerating growth in Year 2-3, increasing customer reviews/ratings | Niche accessory that gains cult following |
| **Never Really Took Off** | Products that fail to gain traction despite being in market | Flat or declining revenue trajectory, low cumulative total vs. cohort peers, high return rates | Failed product line, poor market fit |

### Quantitative Thresholds (Example)

```
Strong Early Adoption:
  - Year 1 Revenue >= 50% of Total 3-Year Revenue
  - OR Year 1 Revenue ranks in Top 25% of cohort

Slow Burn:
  - Year 1 Revenue < 30% of Total 3-Year Revenue
  - AND Year 3 Revenue > Year 1 Revenue * 2

Never Took Off:
  - Total 3-Year Revenue ranks in Bottom 25% of cohort
  - OR Year-over-Year growth consistently < 5%
```

---

## Step-by-Step Solution Approach

### Step 1: Requirements Gathering & Conceptual Modeling

Before designing any schema, clarify business rules and definitions with stakeholders.

#### Key Questions to Ask

| Question | Assumption for This Use Case |
|----------|------------------------------|
| What defines "Launch Date"? | Global product release date (not store-specific) |
| What is a "Cohort"? | Products released in the same month/quarter (e.g., "2026-Q2 Launch") |
| What does "early adoption" mean? | Revenue trajectory in first 3 years (Year 1, Year 2, Year 3) |
| What granularity is needed? | Ability to drill down by store, region, or category |

#### Conceptual Entity Mapping

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Product   │     │    Store    │     │  Location   │
│  (Phone,    │     │  (Retail    │     │  (City,     │
│   Laptop)   │     │   Outlet)   │     │   Region)   │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
                    ┌──────▼──────┐
                    │    Sales    │
                    │ Transaction │
                    └─────────────┘
```

---

### Step 2: Select Business Process & Declare the Grain

Following the **Kimball Four-Step Dimensional Design Process**:

| Step | Decision |
|------|----------|
| **1. Select Business Process** | Retail Sales |
| **2. Declare the Grain** | One row per line item on a sales receipt (atomic level) |
| **3. Identify Dimensions** | Product, Date, Store |
| **4. Identify Facts** | Sales Revenue, Quantity Sold, Unit Price |

#### Why Atomic Grain?

- Enables drill-down analysis (which stores drove early adoption?)
- Supports flexible aggregation (daily, weekly, by region)
- Allows future analytical needs without re-modeling

---

### Step 3: Identify Dimensions and Facts

#### Dimension Tables (The "Who, What, Where, When")

| Dimension | Key Attributes | Purpose |
|-----------|----------------|---------|
| `dim_product` | `product_sk`, `product_id`, `product_name`, `category`, `launch_date`, `cohort_group` | Track product attributes and launch timing |
| `dim_date` | `date_sk`, `full_date`, `day_of_week`, `calendar_month`, `calendar_year` | Standard calendar for time-based analysis |
| `dim_store` | `store_sk`, `store_id`, `store_name`, `city`, `region`, `country` | Geographic context for sales |

#### Fact Table (The "How Much")

| Fact | Type | Description |
|------|------|-------------|
| `sales_amount` | Additive | Revenue in currency (can be summed across all dimensions) |
| `quantity` | Additive | Units sold (can be summed across all dimensions) |
| `unit_price` | Semi-Additive | Price at time of sale (cannot be summed, but can be averaged) |

---

### Why Store `unit_price` in the Fact Table?

**Problem with storing price in dimension:**
- Product prices change frequently (promotions, markdowns, regional pricing)
- Each price change would create a new SCD Type 2 row in `dim_product`
- Over 3 years, a single product could have 50+ price changes → **dimension bloat**
- Dimension tables should remain relatively stable; frequent changes defeat their purpose

**Benefits of storing price in fact:**
- Captures the **exact price at the moment of transaction** (point-in-time accuracy)
- No dimension growth from price fluctuations
- Enables accurate revenue calculations: `sales_amount = quantity × unit_price`
- Supports price elasticity analysis and promotional effectiveness

| Approach | Dimension Size Impact | Historical Accuracy | Query Complexity |
|----------|----------------------|---------------------|------------------|
| Price in Dimension (SCD2) | High growth (new row per price change) | Requires careful SCD joins | Complex |
| **Price in Fact** (Recommended) | No impact | Exact at transaction time | Simple |

> **Interview Tip:** This is a classic example of a **degenerate fact** — a measurement that could theoretically be a dimension attribute but is stored in the fact table for practical reasons.

---

### Step 4: Address the Unique Challenge — "Days Since Launch"

The business needs to compare products on a **normalized timeline** (Month 1, Month 2, ... Month 36), not calendar dates.

#### Two Architectural Options

| Approach | Description | When to Use |
|----------|-------------|-------------|
| **Transformational View** (Recommended) | Calculate `months_since_launch = transaction_date - launch_date` at query time | Flexible analysis, ad-hoc queries |
| **Accumulating Snapshot** | Pre-aggregate into columns: `year_1_revenue`, `year_2_revenue`, `year_3_revenue` | Fixed milestone reporting, dashboard performance |

**For this use case:** Use a **Transformational View** for flexibility.

---

### Step 5: Strategic Key & SCD Decisions

#### Why Surrogate Keys?

| Key Type | Example | Purpose |
|----------|---------|---------|
| **Surrogate Key** | `product_sk = 101` (integer) | Fast joins, SCD tracking, warehouse-generated |
| **Natural Key** | `product_id = 'SKU-PHONE-Z1'` | Business identifier, may change or have duplicates |

#### SCD Type 2 for Product Dimension

If a product is **re-categorized** during its first 3 years (e.g., moved from "New Releases" to "Mobile"), SCD Type 2 ensures:
- Historical accuracy: Revenue attributed to correct category at time of sale
- Multiple rows per natural key with `effective_date`, `expiration_date`, `is_current` flags

---

### Step 6: Physical Data Types

| Column | Data Type | Rationale |
|--------|-----------|-----------|
| Surrogate Keys | `INT` or `BIGINT` | Optimized join performance |
| Revenue | `DECIMAL(18,2)` | Avoid floating-point rounding errors |
| Dates | `DATE` | Native date arithmetic support |
| Flags | `BOOLEAN` | Efficient storage for `is_current` |

---

## Physical Implementation (DDL)

### Product Dimension (SCD Type 2)

```sql
CREATE TABLE dim_product (
    product_sk      INT PRIMARY KEY,           -- Surrogate Key (PK)
    product_id      VARCHAR(50),               -- Natural Key (SKU)
    product_name    VARCHAR(255),
    category        VARCHAR(100),
    launch_date     DATE,                      -- Critical for cohort analysis
    cohort_group    VARCHAR(50),               -- e.g., '2026-Q2 Launch'
    effective_date  DATE,                      -- SCD2: When this version became active
    expiration_date DATE,                      -- SCD2: When this version expired
    is_current      BOOLEAN                    -- SCD2: Current version flag
);
```

### Date Dimension (Conformed)

```sql
CREATE TABLE dim_date (
    date_sk         INT PRIMARY KEY,           -- e.g., 20260412
    full_date       DATE,
    day_of_week     VARCHAR(15),
    calendar_month  VARCHAR(20),
    calendar_quarter VARCHAR(10),
    calendar_year   INT,
    fiscal_year     INT
);
```

### Store Dimension

```sql
CREATE TABLE dim_store (
    store_sk        INT PRIMARY KEY,
    store_id        VARCHAR(50),
    store_name      VARCHAR(255),
    city            VARCHAR(100),
    region          VARCHAR(100),
    country         VARCHAR(100)
);
```

### Sales Fact Table (Transaction Grain)

```sql
CREATE TABLE fct_sales (
    sales_sk        BIGINT PRIMARY KEY,
    product_sk      INT REFERENCES dim_product(product_sk),
    date_sk         INT REFERENCES dim_date(date_sk),
    store_sk        INT REFERENCES dim_store(store_sk),
    quantity        INT,                        -- Additive Fact: Units sold
    unit_price      DECIMAL(18, 2),            -- Semi-Additive: Price at time of sale
    sales_amount    DECIMAL(18, 2)             -- Additive Fact: quantity × unit_price
);
```

> **Note:** `unit_price` is stored in the fact table (not dimension) because prices change frequently. Storing in dimension would cause excessive SCD Type 2 row creation, leading to dimension bloat. The fact table captures the exact price at transaction time.

---

## Star Schema Diagram

```
                    ┌─────────────────┐
                    │   dim_product   │
                    │─────────────────│
                    │ product_sk (PK) │
                    │ product_id      │
                    │ product_name    │
                    │ category        │
                    │ launch_date     │◄──── Critical for "Months Since Launch"
                    │ cohort_group    │
                    │ effective_date  │
                    │ expiration_date │
                    │ is_current      │
                    └────────┬────────┘
                             │
                             │ FK
                             ▼
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│  dim_date   │     │    fct_sales    │     │  dim_store  │
│─────────────│     │─────────────────│     │─────────────│
│ date_sk(PK) │◄────│ date_sk (FK)    │────►│ store_sk(PK)│
│ full_date   │     │ product_sk (FK) │     │ store_name  │
│ day_of_week │     │ store_sk (FK)   │     │ city        │
│ calendar_   │     │ quantity        │     │ region      │
│   month     │     │ unit_price      │◄─── Price in Fact (avoids dim bloat)
│ quarter     │     │ sales_amount    │     │ country     │
└─────────────┘     └─────────────────┘     └─────────────┘
```

---

## Query 1: Transformational View — Calculate Months Since Launch

### Purpose
Normalize transaction dates relative to each product's launch date, enabling comparison across products with different launch dates over a 3-year window.

### SQL

```sql
CREATE OR REPLACE VIEW vw_product_launch_performance AS
SELECT 
    p.cohort_group,
    p.product_name,
    p.category,
    p.launch_date,
    d.full_date AS transaction_date,
    -- Calculate months since launch (Month 0 = launch month)
    DATEDIFF('month', p.launch_date, d.full_date) AS months_since_launch,
    -- Calculate year since launch (Year 1, 2, 3)
    CASE 
        WHEN DATEDIFF('month', p.launch_date, d.full_date) < 12 THEN 1
        WHEN DATEDIFF('month', p.launch_date, d.full_date) < 24 THEN 2
        ELSE 3
    END AS year_since_launch,
    s.quantity,
    s.unit_price,
    s.sales_amount,
    st.store_name,
    st.region
FROM fct_sales s
JOIN dim_product p ON s.product_sk = p.product_sk
JOIN dim_date d ON s.date_sk = d.date_sk
JOIN dim_store st ON s.store_sk = st.store_sk
-- Filter for first 3 years only (Month 0 through Month 35)
WHERE DATEDIFF('month', p.launch_date, d.full_date) BETWEEN 0 AND 35
  AND p.is_current = TRUE;  -- Use current product attributes
```

### Sample Input Data

**dim_product**
| product_sk | product_name | category | launch_date | cohort_group |
|------------|--------------|----------|-------------|--------------|
| 101 | Galaxy Phone Z | Mobile | 2023-04-01 | 2023-Q2 |
| 102 | Pro Laptop X | Computing | 2023-04-15 | 2023-Q2 |
| 103 | Wireless Buds | Audio | 2023-04-01 | 2023-Q2 |

**dim_date** (sample months)
| date_sk | full_date | calendar_month |
|---------|-----------|----------------|
| 20230401 | 2023-04-01 | April 2023 |
| 20230501 | 2023-05-01 | May 2023 |
| 20240401 | 2024-04-01 | April 2024 |
| 20250401 | 2025-04-01 | April 2025 |

**fct_sales** (aggregated monthly for illustration)
| product_sk | date_sk | quantity | unit_price | sales_amount |
|------------|---------|----------|------------|---------------|
| 101 | 20230401 | 1000 | 999.00 | 999,000.00 |
| 101 | 20230501 | 1200 | 999.00 | 1,198,800.00 |
| 101 | 20240401 | 800 | 899.00 | 719,200.00 |
| 101 | 20250401 | 400 | 799.00 | 319,600.00 |
| 102 | 20230501 | 500 | 1499.00 | 749,500.00 |
| 102 | 20240401 | 600 | 1399.00 | 839,400.00 |
| 102 | 20250401 | 700 | 1299.00 | 909,300.00 |
| 103 | 20230401 | 2000 | 99.00 | 198,000.00 |
| 103 | 20230501 | 1500 | 99.00 | 148,500.00 |
| 103 | 20240401 | 500 | 79.00 | 39,500.00 |

### Sample Output

| cohort_group | product_name | launch_date | transaction_date | months_since_launch | year_since_launch | quantity | unit_price | sales_amount |
|--------------|--------------|-------------|------------------|---------------------|-------------------|----------|------------|---------------|
| 2023-Q2 | Galaxy Phone Z | 2023-04-01 | 2023-04-01 | 0 | 1 | 1000 | 999.00 | 999,000.00 |
| 2023-Q2 | Galaxy Phone Z | 2023-04-01 | 2023-05-01 | 1 | 1 | 1200 | 999.00 | 1,198,800.00 |
| 2023-Q2 | Galaxy Phone Z | 2023-04-01 | 2024-04-01 | 12 | 2 | 800 | 899.00 | 719,200.00 |
| 2023-Q2 | Galaxy Phone Z | 2023-04-01 | 2025-04-01 | 24 | 3 | 400 | 799.00 | 319,600.00 |
| 2023-Q2 | Pro Laptop X | 2023-04-15 | 2023-05-01 | 0 | 1 | 500 | 1499.00 | 749,500.00 |
| 2023-Q2 | Pro Laptop X | 2023-04-15 | 2024-04-01 | 11 | 1 | 600 | 1399.00 | 839,400.00 |
| 2023-Q2 | Pro Laptop X | 2023-04-15 | 2025-04-01 | 23 | 2 | 700 | 1299.00 | 909,300.00 |

**Key Insight:** Notice how "Month 12" for Galaxy Phone Z (April 2024) is a different calendar date than "Month 12" for Pro Laptop X (April 2024 is Month 11 for Laptop). The view normalizes this for cohort comparison.

---

## Query 2: Yearly Revenue Analysis — Adoption Curves

### Purpose
Calculate yearly and cumulative revenue to visualize adoption patterns and identify:
- **Strong early adoption:** High Year 1 revenue (>50% of 3-year total)
- **Slow pickup (Slow Burn):** Low Year 1, accelerating in Year 2-3
- **Never took off:** Flat or declining trajectory, bottom of cohort

### SQL

```sql
SELECT 
    cohort_group,
    product_name,
    year_since_launch,
    -- Yearly revenue (aggregated from atomic transactions)
    SUM(sales_amount) AS yearly_revenue,
    -- Yearly units sold
    SUM(quantity) AS yearly_units,
    -- Average unit price for the year
    ROUND(AVG(unit_price), 2) AS avg_unit_price,
    -- Cumulative (running) revenue for adoption curve visualization
    SUM(SUM(sales_amount)) OVER (
        PARTITION BY product_name 
        ORDER BY year_since_launch
        ROWS UNBOUNDED PRECEDING
    ) AS cumulative_revenue,
    -- Cumulative units
    SUM(SUM(quantity)) OVER (
        PARTITION BY product_name 
        ORDER BY year_since_launch
        ROWS UNBOUNDED PRECEDING
    ) AS cumulative_units
FROM vw_product_launch_performance
GROUP BY 
    cohort_group, 
    product_name, 
    year_since_launch
ORDER BY 
    cohort_group, 
    product_name,
    year_since_launch;
```

### How the Window Function Works

```sql
SUM(SUM(sales_amount)) OVER (
    PARTITION BY product_name 
    ORDER BY year_since_launch
    ROWS UNBOUNDED PRECEDING
) AS cumulative_revenue
```

| Component | Purpose |
|-----------|---------|
| Inner `SUM(sales_amount)` | Aggregates all transactions for that product in that year |
| `PARTITION BY product_name` | Resets the running total for each product |
| `ORDER BY year_since_launch` | Defines the sequence for accumulation (Year 1 → 2 → 3) |
| `ROWS UNBOUNDED PRECEDING` | Includes all rows from start of partition to current row |
| Outer `SUM(...) OVER (...)` | Calculates running total across ordered years |

### Sample Output

| cohort_group | product_name | year_since_launch | yearly_revenue | yearly_units | avg_unit_price | cumulative_revenue | cumulative_units |
|--------------|--------------|-------------------|----------------|--------------|----------------|--------------------| -----------------|
| 2023-Q2 | Galaxy Phone Z | 1 | 12,000,000 | 12,000 | 999.00 | 12,000,000 | 12,000 |
| 2023-Q2 | Galaxy Phone Z | 2 | 8,000,000 | 9,000 | 899.00 | 20,000,000 | 21,000 |
| 2023-Q2 | Galaxy Phone Z | 3 | 4,000,000 | 5,000 | 799.00 | 24,000,000 | 26,000 |
| 2023-Q2 | Pro Laptop X | 1 | 3,000,000 | 2,000 | 1499.00 | 3,000,000 | 2,000 |
| 2023-Q2 | Pro Laptop X | 2 | 5,000,000 | 3,500 | 1399.00 | 8,000,000 | 5,500 |
| 2023-Q2 | Pro Laptop X | 3 | 7,000,000 | 5,500 | 1299.00 | 15,000,000 | 11,000 |
| 2023-Q2 | Wireless Buds | 1 | 2,000,000 | 20,000 | 99.00 | 2,000,000 | 20,000 |
| 2023-Q2 | Wireless Buds | 2 | 500,000 | 6,000 | 79.00 | 2,500,000 | 26,000 |
| 2023-Q2 | Wireless Buds | 3 | 200,000 | 2,500 | 79.00 | 2,700,000 | 28,500 |

### Interpretation & Adoption Pattern Classification

| Product | Year 1 % of Total | Pattern | Analysis |
|---------|-------------------|---------|----------|
| **Galaxy Phone Z** | 50% (12M/24M) | **Strong Early Adoption** | High Year 1 revenue, flagship launch success, natural decline as product ages |
| **Pro Laptop X** | 20% (3M/15M) | **Slow Burn** | Low Year 1, accelerating growth Year 2-3, word-of-mouth/enterprise adoption |
| **Wireless Buds** | 74% (2M/2.7M) | **Never Took Off** | High initial hype, steep decline, product failed to sustain interest |

---

## Query 3: Year-over-Year Growth Rate (Percentage Change)

### Purpose
Calculate the percentage change between consecutive years to identify growth velocity and trajectory.

### SQL

```sql
WITH yearly_metrics AS (
    SELECT 
        cohort_group,
        product_name,
        year_since_launch,
        SUM(sales_amount) AS yearly_revenue,
        SUM(quantity) AS yearly_units,
        ROUND(AVG(unit_price), 2) AS avg_unit_price
    FROM vw_product_launch_performance
    GROUP BY cohort_group, product_name, year_since_launch
)
SELECT 
    cohort_group,
    product_name,
    year_since_launch,
    yearly_revenue,
    yearly_units,
    avg_unit_price,
    LAG(yearly_revenue) OVER (
        PARTITION BY product_name 
        ORDER BY year_since_launch
    ) AS previous_year_revenue,
    -- Calculate year-over-year percentage change
    CASE 
        WHEN LAG(yearly_revenue) OVER (
            PARTITION BY product_name 
            ORDER BY year_since_launch
        ) = 0 THEN NULL
        ELSE ROUND(
            (yearly_revenue - LAG(yearly_revenue) OVER (
                PARTITION BY product_name 
                ORDER BY year_since_launch
            )) * 100.0 / LAG(yearly_revenue) OVER (
                PARTITION BY product_name 
                ORDER BY year_since_launch
            ), 2
        )
    END AS yoy_pct_change
FROM yearly_metrics
ORDER BY cohort_group, product_name, year_since_launch;
```

### Sample Output

| cohort_group | product_name | year_since_launch | yearly_revenue | yearly_units | avg_unit_price | previous_year_revenue | yoy_pct_change |
|--------------|--------------|-------------------|----------------|--------------|----------------|----------------------|----------------|
| 2023-Q2 | Galaxy Phone Z | 1 | 12,000,000 | 12,000 | 999.00 | NULL | NULL |
| 2023-Q2 | Galaxy Phone Z | 2 | 8,000,000 | 9,000 | 899.00 | 12,000,000 | -33.33 |
| 2023-Q2 | Galaxy Phone Z | 3 | 4,000,000 | 5,000 | 799.00 | 8,000,000 | -50.00 |
| 2023-Q2 | Pro Laptop X | 1 | 3,000,000 | 2,000 | 1499.00 | NULL | NULL |
| 2023-Q2 | Pro Laptop X | 2 | 5,000,000 | 3,500 | 1399.00 | 3,000,000 | +66.67 |
| 2023-Q2 | Pro Laptop X | 3 | 7,000,000 | 5,500 | 1299.00 | 5,000,000 | +40.00 |
| 2023-Q2 | Wireless Buds | 1 | 2,000,000 | 20,000 | 99.00 | NULL | NULL |
| 2023-Q2 | Wireless Buds | 2 | 500,000 | 6,000 | 79.00 | 2,000,000 | -75.00 |
| 2023-Q2 | Wireless Buds | 3 | 200,000 | 2,500 | 79.00 | 500,000 | -60.00 |

---

## Why Cumulative Sum vs. Percentage Change?

| Metric | Use Case | Limitation |
|--------|----------|------------|
| **Cumulative Sum** | Shows total market penetration, adoption magnitude | Doesn't show yearly volatility |
| **Percentage Change (YoY)** | Shows growth velocity, momentum shifts | Can be misleading (100% growth from $1 to $2) |
| **Both Together** | Complete picture of adoption health | Requires more complex visualization |

### Key Interview Point

> **Percentage is a Non-Additive Fact:** You cannot sum percentages across dimensions. Store the additive components (yearly revenue) in the fact table and calculate percentages in the BI layer or at query time.

### Interpreting YoY Growth for Adoption Patterns

| YoY Pattern | Interpretation | Classification |
|-------------|----------------|----------------|
| Year 1 high, Year 2-3 declining | Natural product lifecycle, strong launch | Strong Early Adoption |
| Year 1 low, Year 2-3 increasing | Building momentum, word-of-mouth | Slow Burn |
| Consistent decline >50% YoY | Product failing to retain interest | Never Took Off |
| Stable (<10% change) | Mature, steady performer | Steady Performer |

---

## Query 4: Cohort Comparison Dashboard Query

### Purpose
Compare all products in the same launch cohort side-by-side for the dashboard. This query enables the merchandising team to **compare products in the same cohort** and classify their adoption patterns.

### SQL

```sql
WITH yearly_performance AS (
    SELECT 
        cohort_group,
        product_name,
        category,
        year_since_launch,
        SUM(sales_amount) AS yearly_revenue,
        SUM(quantity) AS yearly_units,
        ROUND(AVG(unit_price), 2) AS avg_unit_price
    FROM vw_product_launch_performance
    GROUP BY cohort_group, product_name, category, year_since_launch
),
cohort_totals AS (
    SELECT 
        cohort_group,
        product_name,
        category,
        -- Total 3-year revenue
        SUM(yearly_revenue) AS total_3year_revenue,
        -- Total 3-year units
        SUM(yearly_units) AS total_3year_units,
        -- Year 1 revenue for "launch strength"
        SUM(CASE WHEN year_since_launch = 1 THEN yearly_revenue ELSE 0 END) AS year_1_revenue,
        -- Year 2 revenue
        SUM(CASE WHEN year_since_launch = 2 THEN yearly_revenue ELSE 0 END) AS year_2_revenue,
        -- Year 3 revenue for "sustained interest"
        SUM(CASE WHEN year_since_launch = 3 THEN yearly_revenue ELSE 0 END) AS year_3_revenue
    FROM yearly_performance
    GROUP BY cohort_group, product_name, category
)
SELECT 
    cohort_group,
    product_name,
    category,
    total_3year_revenue,
    total_3year_units,
    year_1_revenue,
    year_2_revenue,
    year_3_revenue,
    -- Calculate Year 1 percentage of total
    ROUND(year_1_revenue * 100.0 / NULLIF(total_3year_revenue, 0), 1) AS year_1_pct_of_total,
    -- Classify adoption pattern based on business definitions
    CASE 
        -- Strong Early Adoption: Year 1 >= 50% of total OR top 25% of cohort
        WHEN year_1_revenue >= total_3year_revenue * 0.5 
            AND year_2_revenue <= year_1_revenue 
            THEN 'Strong Early Adoption'
        -- Slow Burn: Year 1 < 30% AND Year 3 > Year 1 * 2
        WHEN year_1_revenue < total_3year_revenue * 0.3 
            AND year_3_revenue > year_1_revenue * 1.5 
            THEN 'Slow Burn (Word of Mouth)'
        -- Never Took Off: Consistent decline AND bottom of cohort
        WHEN year_3_revenue < year_1_revenue * 0.25 
            THEN 'Never Took Off'
        -- Default: Steady Performer
        ELSE 'Steady Performer'
    END AS adoption_pattern,
    -- Rank within cohort by total revenue
    RANK() OVER (
        PARTITION BY cohort_group 
        ORDER BY total_3year_revenue DESC
    ) AS cohort_rank,
    -- Percentile within cohort
    PERCENT_RANK() OVER (
        PARTITION BY cohort_group 
        ORDER BY total_3year_revenue
    ) AS cohort_percentile
FROM cohort_totals
ORDER BY cohort_group, cohort_rank;
```

### Sample Output

| cohort_group | product_name | category | total_3year_revenue | year_1_revenue | year_2_revenue | year_3_revenue | year_1_pct_of_total | adoption_pattern | cohort_rank |
|--------------|--------------|----------|---------------------|----------------|----------------|----------------|---------------------|------------------|-------------|
| 2023-Q2 | Galaxy Phone Z | Mobile | 24,000,000 | 12,000,000 | 8,000,000 | 4,000,000 | 50.0% | Strong Early Adoption | 1 |
| 2023-Q2 | Pro Laptop X | Computing | 15,000,000 | 3,000,000 | 5,000,000 | 7,000,000 | 20.0% | Slow Burn (Word of Mouth) | 2 |
| 2023-Q2 | Wireless Buds | Audio | 2,700,000 | 2,000,000 | 500,000 | 200,000 | 74.1% | Never Took Off | 3 |

---

## Handling Edge Cases

### Products with Different Launch Dates per Location

If launch dates vary by store/region, modify the model:

```sql
-- Option 1: Bridge Table for Product-Store Launch Dates
CREATE TABLE bridge_product_store_launch (
    product_sk      INT,
    store_sk        INT,
    local_launch_date DATE,
    PRIMARY KEY (product_sk, store_sk)
);

-- Option 2: Modify the view to use store-specific launch dates
CREATE OR REPLACE VIEW vw_product_launch_performance_by_store AS
SELECT 
    p.product_name,
    st.store_name,
    st.region,
    COALESCE(b.local_launch_date, p.launch_date) AS effective_launch_date,
    d.full_date AS transaction_date,
    DATEDIFF('day', 
        COALESCE(b.local_launch_date, p.launch_date), 
        d.full_date
    ) AS days_since_launch,
    s.sales_amount
FROM fct_sales s
JOIN dim_product p ON s.product_sk = p.product_sk
JOIN dim_date d ON s.date_sk = d.date_sk
JOIN dim_store st ON s.store_sk = st.store_sk
LEFT JOIN bridge_product_store_launch b 
    ON s.product_sk = b.product_sk 
    AND s.store_sk = b.store_sk
WHERE DATEDIFF('month', 
    COALESCE(b.local_launch_date, p.launch_date), 
    d.full_date
) BETWEEN 0 AND 35;  -- First 3 years (36 months)
```

### Late-Arriving Dimensions

When a sale arrives before the product dimension record:

1. **Use a placeholder row** in `dim_product` with `product_sk = -1` (Unknown)
2. **Reprocess** when the actual product data arrives
3. **SCD Type 2** ensures historical accuracy after correction

### Many-to-Many Categories (Bridge Table)

If products belong to multiple categories:

```sql
CREATE TABLE bridge_product_category (
    product_sk      INT,
    category_sk     INT,
    is_primary      BOOLEAN,
    PRIMARY KEY (product_sk, category_sk)
);
```

---

## Summary: Model Components

| Component | Type | Purpose |
|-----------|------|---------|
| `fct_sales` | Transaction Fact | Atomic sales at line-item grain with `quantity`, `unit_price`, `sales_amount` |
| `dim_product` | SCD Type 2 | Product attributes with launch date (price NOT stored here) |
| `dim_date` | Conformed | Standard calendar dimension |
| `dim_store` | Standard | Store and location context |
| `vw_product_launch_performance` | Transformational View | Calculates `months_since_launch` and `year_since_launch` |
| Analysis Queries | BI Layer | Cumulative sums, YoY growth, cohort rankings, adoption patterns |

---

## Interview Tips

1. **Always start with the grain:** "One row per line item on a sales receipt"
2. **Explain surrogate keys:** Enable SCD tracking and faster joins
3. **Distinguish additive vs. non-additive:** Revenue/quantity can be summed; percentages cannot
4. **Justify price in fact table:** Avoids dimension bloat from frequent price changes
5. **Justify the transformational approach:** Flexibility over pre-aggregation for 3-year analysis
6. **Define adoption patterns clearly:** Strong Early Adoption, Slow Burn, Never Took Off with quantitative thresholds
7. **Address edge cases proactively:** Late-arriving data, regional launches, category changes

---

## Additional Dimension Types Reference

| Dimension Type | Description | Example in This Model |
|----------------|-------------|----------------------|
| **Conformed** | Shared across fact tables | `dim_date` |
| **Role-Playing** | Same dimension used multiple ways | `dim_date` as order_date vs ship_date |
| **Degenerate** | No separate table, stored in fact | `receipt_number` in `fct_sales` |
| **Junk** | Low-cardinality flags combined | `is_online`, `is_promotion` flags |
| **Outrigger** | Dimension of a dimension | `dim_region` linked from `dim_store` |

---

## Conclusion

This data model follows the **Kimball Dimensional Modeling** methodology to solve the product launch performance tracking problem. The key design decisions are:

1. **Atomic grain** for maximum analytical flexibility
2. **Surrogate keys** for SCD Type 2 tracking and join performance
3. **Price in fact table** to avoid dimension bloat from frequent price changes
4. **Transformational view** to normalize "months/years since launch" across products
5. **Window functions** for cumulative adoption metrics over 3 years
6. **Star schema** for query simplicity and BI tool compatibility

## One Query Direct for Short Interview

```sql
SELECT 
    p.cohort_group,
    p.product_name,
    YEAR(d.full_date) - YEAR(p.launch_date) + 1 AS year_number,
    SUM(s.sales_amount) AS yearly_revenue,
    SUM(s.quantity) AS yearly_units,
    -- Cumulative revenue
    SUM(SUM(s.sales_amount)) OVER (
        PARTITION BY p.product_name 
        ORDER BY YEAR(d.full_date) - YEAR(p.launch_date) + 1
    ) AS cumulative_revenue,
    -- Previous year revenue
    LAG(SUM(s.sales_amount)) OVER (
        PARTITION BY p.product_name 
        ORDER BY YEAR(d.full_date) - YEAR(p.launch_date) + 1
    ) AS prev_year_revenue,
    -- YoY % change
    ROUND(
        (SUM(s.sales_amount) - LAG(SUM(s.sales_amount)) OVER (
            PARTITION BY p.product_name 
            ORDER BY YEAR(d.full_date) - YEAR(p.launch_date) + 1
        )) * 100.0 
        / NULLIF(LAG(SUM(s.sales_amount)) OVER (
            PARTITION BY p.product_name 
            ORDER BY YEAR(d.full_date) - YEAR(p.launch_date) + 1
        ), 0), 2
    ) AS yoy_pct_change
FROM fct_sales s
JOIN dim_product p ON s.product_sk = p.product_sk
JOIN dim_date d ON s.date_sk = d.date_sk
WHERE d.full_date >= p.launch_date
  AND d.full_date < DATEADD('year', 3, p.launch_date)
GROUP BY p.cohort_group, p.product_name, 
         YEAR(d.full_date) - YEAR(p.launch_date) + 1
ORDER BY p.cohort_group, p.product_name, 
         YEAR(d.full_date) - YEAR(p.launch_date) + 1;
```

The model enables the merchandising team to:
- **Compare products within the same launch cohort** using cohort_rank and percentile
- **Classify adoption patterns:** Strong Early Adoption, Slow Burn, Never Took Off
- Drill down to store/region level for root cause analysis
- Track historical accuracy even when product attributes change
- Analyze price elasticity using `unit_price` captured at transaction time
