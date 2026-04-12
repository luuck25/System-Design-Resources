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
