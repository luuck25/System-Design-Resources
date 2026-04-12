# Dimensional Modeling Standards

For data practitioners, the design of a dimensional model follows a rigorous four-step process to ensure data is organized for high-performance analysis and reporting.

---

## 1. The Four-Step Design Process
Before writing any SQL, practitioners follow this rigorous process to ensure the model meets business needs:

1.  **Select the Business Process:** Identify a discrete, measurable operational activity (e.g., "processing a sale," "taking an insurance claim," or "logging a login event").
2.  **Declare the Grain:** Define exactly what one single row represents. 
    * **Standard:** Always aim for the **Atomic Grain** (the lowest level of detail, like a single line item on a receipt). 
    * **Nuance:** Never "mix" grains (e.g., having both an "Order" row and an "Order Item" row in the same table) or your aggregations will be incorrect.
3.  **Identify the Dimensions:** Determine the "who, what, where, when, why, and how" context (e.g., Customer, Product, Store, Date).
4.  **Identify the Facts:** Identify the quantitative measurements (e.g., `Sale_Amount`, `Quantity`) that align with the declared grain.
---

## 2. Fact vs. Dimension: How to Decide
| Feature | Fact Tables | Dimension Tables |
| :--- | :--- | :--- |
| **Role** | Captures **what** happened and **how much**. | Answers **who**, **where**, **when**, and **why**. |
| **Content** | Quantitative metrics and foreign keys. | Descriptive, textual attributes and context. |
| **Structure** | "Long and Skinny" (billions of rows, few columns). | "Wide and Short" (fewer rows, many columns). |
| **Operation** | Optimized for math (**SUM**, **AVG**, **COUNT**). | Optimized for **filtering**, **grouping**, and **labeling**. |

> **💡 Interview Tip:** If a value is used for calculation (like `unit_price`), it’s a **Fact**. If it’s used for categorizing (like `product_category`), it’s a **Dimension**.

### Key Nuances
* **Additive vs. Non-Additive:** Additive facts (Quantity) can be summed across any dimension. Non-additive facts (Temperature, Unit Price) cannot be summed and usually require averaging.
* **Degenerate Dimensions:** Attributes like `Invoice_Number` that change with every transaction. These live in the Fact table instead of a separate Dimension table.
* **Conformed Dimensions:** Using the exact same `Dim_Product` across different business units (Sales, Inventory) to ensure "Product A" means the same thing company-wide.

---

## 3. Unique Keys: Natural vs. Surrogate
Choosing the right identifier is a critical strategic decision for warehouse stability.

* **Natural Keys:** Attributes that already exist in the real world (e.g., SSN, SKU, or email). These are often **unstable**, as business rules can change or keys may be recycled by source systems.
* **Surrogate Keys:** Artificial, system-generated identifiers assigned during the ETL process. They are the **gold standard** because they:
    * Provide **immutability**, decoupling the warehouse from source system volatility.
    * Offer **superior performance** for join operations because they are compact.
    * Are **essential** for tracking historical changes via SCD Type 2.

---

## 4. Handling Change: Slowly Changing Dimensions (SCD)
SCDs define how the warehouse tracks history when a dimension attribute (like a customer's address) changes.

| Type | Strategy | Best Use Case | Impact on Uniqueness |
| :--- | :--- | :--- | :--- |
| **Type 0** | **Static** | Fixed data (e.g., Date of Birth). | Natural Key stays unique. |
| **Type 1** | **Overwrite** | Fixing typos; no history needed. | Natural Key stays unique. |
| **Type 2** | **Full History** | Tracking address or status changes. | **Natural Key is NOT unique.** You MUST use a **Surrogate Key**. |
| **Type 3** | **Limited** | Storing "Current" and "Previous" columns. | Natural Key stays unique. |
| **Type 4** | **History Table** | "Mini-dimensions" (Age, Credit Score). | Uses two tables to keep the main table lean. |

---

## 5. Impact of SCD on Uniqueness
Slowly Changing Dimensions (SCD) dictate how uniqueness is managed in a table when attributes change over time.

### SCD Type 2 (Full History)
* **Mechanism:** When an attribute changes, a **new row** is inserted for the same natural key.
* **Impact:** The natural key is **no longer unique** in the table because multiple rows exist for the same entity.
* **Requirement:** You **must** use a surrogate key to uniquely identify each specific historical version.

### SCD Type 3 (Limited History)
* **Mechanism:** History is preserved by adding "previous value" columns to the **existing row**.
* **Impact:** The natural key **remains unique** because you update the row in place rather than adding new ones.
* **Requirement:** While a natural key technically works, surrogate keys are still recommended for a consistent architecture.

> **🏅 Interview Takeaway:** Always use **surrogate keys** as the primary identifier in dimension tables to future-proof the model for **SCD Type 2 tracking**, the industry standard for historical analysis.

---
## 6. Advanced Nuances in Real-Time Modeling

### The "Late Arriving" Problem
In real-time streams, a Sale might arrive at 10:00 AM, but the "New Customer" record doesn't arrive until 10:05 AM.
* **The Solution:** Insert the Sale with a **"Placeholder SK"** (e.g., `-1` or `0`). A background process "re-links" the sale to the correct SK once the customer record arrives.

### Schema Management
* **Schema-on-Write (BigQuery/SQL):** You must define the model perfectly before loading.
* **Schema-on-Read / Evolution (Spark/Delta Lake):** Allows you to add new columns to a Dimension table automatically without crashing the pipeline.

---

## 🏅 Summary Checklist for Modeling
1.  **Define the Grain** (One row = One transaction).
2.  **Separate Facts (Numbers) from Dimensions (Context).**
3.  **Use Surrogate Keys** (Preferably Hashes for distributed systems).
4.  **Assign SCD Types** to every column (usually Type 1 for fixes, Type 2 for history).
5.  **Use Conformed Dimensions** to create a "Single Version of the Truth."

# Understanding Uniqueness in SCD Type 2

In a **Slowly Changing Dimension (SCD) Type 2** environment, managing uniqueness is the most critical part of the design. Because SCD2 tracks history by adding new rows rather than overwriting existing ones, traditional "Business Keys" lose their uniqueness.

---

## 1. The Primary Unique Identifier: The Surrogate Key (SK)

The **Surrogate Key** is a system-generated, meaningless identifier assigned to every single row in the dimension table. It serves as the **Primary Key (PK)**.

* **Why it's necessary:** Since a single customer (e.g., ID 101) might have multiple rows—one for each address change—the database needs a way to distinguish "Version 1" from "Version 2."
* **Fact Table Integration:** The Fact table always stores the **Surrogate Key**, not the Business Key. This "point-in-time" link ensures that a sale is tied to the specific attributes (like the city the customer lived in) that existed at the exact moment of the transaction.
* **Immutability:** Unlike business keys, surrogate keys never change, protecting your warehouse from logic shifts in the source systems.

---

## 2. The "Natural" or Business Key (BK)

The **Business Key** is the identifier used in the source system (e.g., `EMP_005` or `PROD_99`).

* **The Role:** In SCD2, the Business Key acts as a descriptive attribute or a grouping mechanism.
* **The Nuance:** It is **not unique** in an SCD2 dimension table. It will repeat every time a historical change is recorded for that entity.
* **Usage:** It is primarily used during the ETL/ELT process to "look up" existing records and determine if a new version needs to be created.

---

## 3. The Composite Key Alternative

While a single Surrogate Key is the industry standard, uniqueness can technically be defined using a **Composite Key**. This is rarely used as a physical Primary Key in modern warehouses due to join performance, but it is useful for logical validation:

* **Business Key + Effective Date:** Ensures a customer cannot have two changes starting at the exact same time.
* **Business Key + Version Number:** Ensures each version for a specific ID is incremented correctly.

---

## 4. Visual Representation of SCD2 Uniqueness

Notice how the `Customer_ID` repeats as the user moves cities, but the `Dim_SK` remains globally unique:

| Dim_SK (PK) | Customer_ID (BK) | City | Effective_Date | End_Date | Is_Current |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **1001** | CUST_99 | New York | 2023-01-01 | 2023-05-14 | N |
| **1052** | CUST_99 | Chicago | 2023-05-15 | 9999-12-31 | Y |
| **1089** | CUST_42 | Miami | 2024-02-10 | 9999-12-31 | Y |

---

## Summary Checklist for SCD2 Uniqueness

* **Primary Unique Key:** Always use a **Surrogate Key** (`Dim_SK`).
* **Business Key:** Keep the `Customer_ID` for grouping, but expect it to repeat across rows.
* **Version Control:** Use `Start_Date`, `End_Date`, and `Is_Current` flags. While these help filter for the "right" version, they are metadata and should not replace the Surrogate Key for joining to Fact tables.
* **Key Generation:** In modern distributed systems (like BigQuery or Spark), consider using **Hash Keys** (SHA-256) as your Surrogate Keys to allow for parallel generation.


# Implementing Surrogate Keys in BigQuery with Spark

When migrating from a relational source like **Azure SQL** to a distributed warehouse like **BigQuery**, the process of maintaining **Surrogate Keys (SK)** changes. You move away from "Auto-Increment" and toward **Hash-based Keys** and **Set-based Joins**.

---

## 1. Creating the Surrogate Key in the Dim Table
In Spark, we generate the SK by hashing the **Business Key** (Natural Key) and the **Effective Date**. This ensures each historical version has a unique, repeatable ID without needing a central counter.

### The Code (PySpark):
```python
from pyspark.sql.functions import sha2, concat_ws, col

# Generate a unique SK for each version of a customer
dim_with_sk = source_df.withColumn(
    "Customer_SK", 
    sha2(concat_ws("||", col("Customer_ID"), col("Effective_Date")), 256)
)
```

## 2. Inserting Sales Records with the Correct SK

To ensure the **Fact Table** has the correct SK, Spark performs a **Point-in-Time Lookup**. It joins the incoming Sales data against the Dimension table to find which version of the customer was "Active" when the sale happened.

### Does Spark look up every time?
**Yes**, but it doesn't do it row-by-row (which is slow). Instead, it loads the Dimension table into memory and performs a **distributed join**.

### The Logic:
To find the right SK, the join condition must match the **Business Key** and ensure the **Sale Date** falls between the dimension's **Effective Date** and **End Date**.

# The join logic for both initial and future loads
```
fact_table_final = raw_sales.join(
    dim_customer,
    (raw_sales.Customer_ID == dim_customer.Customer_ID) &
    (raw_sales.Sale_Date >= dim_customer.Effective_Date) &
    (raw_sales.Sale_Date <= dim_customer.End_Date),
    "left"
).select(
    raw_sales["*"], 
    dim_customer["Customer_SK"] # This attaches the specific historical version
)
```

## 3. Handling Future Data (The Pipeline Workflow)

For every future batch of sales data, the workflow follows these three steps to ensure consistency:

1.  **Update the Dimension (SCD2):** Run your Spark job to process any customer changes. New rows get new `Customer_SK` values based on their new `Effective_Date`.
2.  **SK Lookup Join:** The Spark job reads the new Sales data and joins it against the **entire** updated Dimension table.
3.  **Append to BigQuery:** The resulting dataframe—now containing the correct `Customer_SK`—is appended to the BigQuery Fact table.

---

### 💡 Summary of Standards

* **Initial Load:** Generate SKs for all history using hashing; join Sales to Dim using the date-range logic.
* **Incremental Loads:** Always perform the join in Spark before writing to BigQuery. This ensures that even "late-arriving" sales are linked to the historically accurate customer record.
* **Performance:** For large joins, use **Broadcast Joins** in Spark if the Dimension table is small enough to fit in memory, making the "lookup every time" extremely fast.
