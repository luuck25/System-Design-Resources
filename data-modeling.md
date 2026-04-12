# Data Modeling Overview

At its core, **Data Modeling** is the process of creating a visual representation or "blueprint" of how data is stored, organized, and related within a system. 

Think of it like an **architect's floor plan**: before you build a house (the database), you need a drawing that shows where the walls (tables) go, how the rooms (entities) connect, and what features (attributes) each room has.

---

## The Three Levels of Data Modeling
As a project moves from a business idea to a technical reality, it usually goes through three stages:

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/441703ce-1cf7-44e8-bf26-a8bde7a212a8" />

### 1. Conceptual Model (The "What")
Focuses on **high-level business concepts**.
* **Example:** "We have Customers, Orders, and Products."

<img width="2031" height="1040" alt="image" src="https://github.com/user-attachments/assets/feaea6fb-db89-45a8-8444-6a92acaa01d8" />


### 2. Logical Model (The "How")
Defines the **structure of the data** without worrying about the specific software.
* **Example:** "A Customer has a Name and Email. An Order must be linked to a Customer_ID."

<img width="2016" height="1052" alt="image" src="https://github.com/user-attachments/assets/53e29c38-3fb7-4aac-a4b5-c0045de34872" />

### 3. Physical Model (The "Implementation")
The **actual code** for a specific database (like BigQuery, Azure SQL, or Postgres).
* **Example:** Defining that Email is a `VARCHAR(255)` and setting up Primary Keys.

---

## Key Components
* **Entities:** The "nouns" or objects (e.g., Customer, Sale, Employee).
* **Attributes:** The details about those objects (e.g., Name, Price, Date).

<img width="1117" height="552" alt="image" src="https://github.com/user-attachments/assets/e488540f-44c2-49fb-a6d0-1cafc5535470" />

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/b35c4442-5382-4ecd-b7b4-f8238c1b4c59" />


* **Relationships:** How the entities interact (e.g., A Customer **places** an Order).
* **Constraints:** The rules of the data (e.g., "A Sale Amount cannot be negative").


# Database Normalization

<img width="1080" height="532" alt="image" src="https://github.com/user-attachments/assets/76d68d6c-4123-4607-8852-e9d46b05b44c" />


**Normalization** is the process of organizing data in a database to reduce **redundancy** (repetition) and improve **data integrity** (accuracy). 

The goal is to isolate data so that additions, deletions, and modifications can be made in just one table and then propagated through the rest of the database using defined relationships.

---

## Why Normalize?
* **Eliminate Duplicate Data:** Saves storage space and prevents **update anomalies** (e.g., updating an address in one place but forgetting another).
* **Data Integrity:** Ensures that related data is stored logically.
* **Query Efficiency:** Smaller tables are often faster to index and scan.

---

## The Normal Forms (Level of Detail)
Database normalization follows a series of steps called **Normal Forms (NF)**. Most production databases aim for **3rd Normal Form**.

<img width="800" height="800" alt="image" src="https://github.com/user-attachments/assets/195effbc-591a-450e-922a-30a7770f94f0" />
---

### 1. First Normal Form (1NF)
* **Rule:** **Atomic values** only. No repeating groups or arrays in a single cell.
* **Example:** Instead of a `Colors` column containing `"Red, Blue"`, you create two separate rows or a related table.

### 2. Second Normal Form (2NF)
* **Rule:** Meet all 1NF requirements + All non-key columns must depend on the **entire Primary Key**.
* **Example:** In a table with a composite key (`Order_ID` + `Product_ID`), `Product_Name` shouldn't be included because it only depends on the `Product_ID`, not the `Order_ID`.

### 3. Third Normal Form (3NF)
* **Rule:** Meet all 2NF requirements + No **transitive dependencies** (non-key columns depending on other non-key columns).
* **Example:** If a table has `Zip_Code` and `City`, the `City` depends on the `Zip_Code`, not the Primary Key. To reach 3NF, you move `Zip_Code` and `City` to their own lookup table.
