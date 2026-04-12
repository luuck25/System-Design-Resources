# Dimensional Modeling Standards

For data practitioners, the design of a dimensional model follows a rigorous four-step process to ensure data is organized for high-performance analysis and reporting.

---

## 1. The Four-Step Design Process
1.  **Select the Business Process:** Identify a discrete, measurable operational activity, such as "processing a sale" or "taking an insurance claim."
2.  **Declare the Grain:** Define exactly what one row in the fact table represents. Best practice is to aim for the **lowest possible atomic detail** (e.g., a single line item on a receipt) to allow for maximum analytical flexibility.
3.  **Identify the Dimensions:** Determine the descriptive "who, what, where, when, why, and how" context surrounding the process.
4.  **Identify the Facts:** Identify the quantitative, numeric measurements resulting from the event that align with the declared grain.

---

## 2. Fact vs. Dimension: How to Decide
The distinction is based on whether the data represents an **event measurement** or the **context** surrounding that event.

| Feature | Fact Table | Dimension Table |
| :--- | :--- | :--- |
| **Content** | Numerical metrics and measures. | Descriptive, textual context and attributes. |
| **Purpose** | Captures **what** happened and **how much**. | Answers **who**, **where**, and **when**. |
| **Structure** | Compact and deep (many rows); primarily keys and numbers. | Wide and shallow; many descriptive columns. |
| **Operation** | Supports aggregation (**SUM**, **AVG**). | Optimized for **filtering** and **grouping**. |

> **💡 Interview Tip:** If a value is used for calculation (like `sales_amount`), it belongs in a **fact table**. If it is used for labeling, filtering, or categorizing (like `product_category`), it belongs in a **dimension table**.

---

## 3. Unique Keys: Natural vs. Surrogate
Choosing the right identifier is a critical strategic decision for warehouse stability.

* **Natural Keys:** Attributes that already exist in the real world (e.g., SSN, SKU, or email). These are often **unstable**, as business rules can change or keys may be recycled by source systems.
* **Surrogate Keys:** Artificial, system-generated identifiers assigned during the ETL process. They are the **gold standard** because they:
    * Provide **immutability**, decoupling the warehouse from source system volatility.
    * Offer **superior performance** for join operations because they are compact.
    * Are **essential** for tracking historical changes via SCD Type 2.

---

## 4. Impact of SCD on Uniqueness
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
