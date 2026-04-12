# Data Modeling Overview

At its core, **Data Modeling** is the process of creating a visual representation or "blueprint" of how data is stored, organized, and related within a system. 

Think of it like an **architect's floor plan**: before you build a house (the database), you need a drawing that shows where the walls (tables) go, how the rooms (entities) connect, and what features (attributes) each room has.

---

## The Three Levels of Data Modeling
As a project moves from a business idea to a technical reality, it usually goes through three stages:

<img width="1983" height="1060" alt="image" src="https://github.com/user-attachments/assets/441703ce-1cf7-44e8-bf26-a8bde7a212a8" />

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

<img width="1548" height="1096" alt="image" src="https://github.com/user-attachments/assets/b35c4442-5382-4ecd-b7b4-f8238c1b4c59" />


* **Relationships:** How the entities interact (e.g., A Customer **places** an Order).
* **Constraints:** The rules of the data (e.g., "A Sale Amount cannot be negative").
