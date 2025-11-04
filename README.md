# sales-project


## 📊 Sales Data Engineering Project (Event-Driven Pipeline)

### 🧩 Overview
This project demonstrates an **event-driven Sales Data Pipeline** built using **Azure Data Engineering services**.  
The goal is to automate the ingestion, validation, and transformation of sales data files as soon as they land in Azure Data Lake, and then serve curated results to **Azure SQL Database** for analytics and reporting.

---

### 🏗️ Architecture

**Architecture Layers:**
| Layer | Service | Description |
|-------|----------|-------------|
| Storage Layer | **Azure Data Lake Gen2** | Stores raw and processed sales data (`landing`, `staging`, `discarded`, `orderitems` folders). |
| Ingestion Layer | **Azure Data Factory (ADF)** | Orchestrates the pipeline and triggers Databricks notebook execution. |
| Trigger | **Storage Event Trigger** | Automatically starts the pipeline when a new file (e.g., `orders.csv`) is dropped in the landing folder. |
| Compute Layer | **Azure Databricks** | Performs data validation, transformation, and aggregation using PySpark. |
| Security | **Azure Key Vault** | Securely stores credentials and connection strings. |
| Serving Layer | **Azure SQL Database** | Stores curated, aggregated results for downstream analytics. |
| External Source | **Amazon S3** | Contains `order_items.csv` file ingested into Data Lake. |

---

### 📂 Data Lake Folder Structure
```
sales/
│
├── landing/        # Raw incoming data (orders.csv dropped by third-party)
├── staging/        # Validated & clean data
├── discarded/      # Invalid or duplicate data
└── orderitems/     # Data copied from Amazon S3 (order_items.csv)
```

---

### ⚙️ Event-Driven Pipeline Flow

#### 1️⃣ File Drop in Landing Folder
- A **third-party service** drops `orders.csv` into the **landing** folder of Azure Data Lake Gen2.

#### 2️⃣ Storage Event Trigger
- A **Storage Event Trigger** in **Azure Data Factory** automatically starts the pipeline whenever a new file lands.  
- The trigger dynamically retrieves the filename using:
  ```json
  @triggerBody().filename
  ```
- This filename is passed as a **parameter** to the ADF pipeline.

#### 3️⃣ Pipeline Activities
1. **Copy Activity:**  
   - Ingests `order_items.csv` from **Amazon S3** into the `orderitems` folder in Azure Data Lake.
2. **Notebook Activity:**  
   - Invokes the **Databricks notebook** and passes the dynamic filename parameter to it.

#### 4️⃣ Databricks Notebook Execution
Inside Databricks, the notebook dynamically reads the file from the landing folder:

```python
filename = dbutils.widgets.get("filename")
FnameWithoutExt = filename.split(".")[0]
file_path = f"/mnt/sales/{filename}"
```

- No filenames are hardcoded — the file path is constructed dynamically from the trigger input.

---

### 🧪 Data Validation Logic (in Databricks)

#### ✅ Validation 1: Unique Order ID
- Ensures all `OrderID` values are unique.
- If duplicates exist → file is moved to the **discarded** folder.
- If all unique → file moves to the **staging** folder.

#### ✅ Validation 2: Valid Order Status
- Retrieves valid statuses from a **reference table** stored in **Azure SQL Database**.
- Connects using JDBC and Key Vault for credentials:
  ```python
  jdbc_url = "jdbc:sqlserver://<server>.database.windows.net:1433;database=<dbname>"
  connection_properties = {
      "user": dbutils.secrets.get("keyvault_scope", "sql-user"),
      "password": dbutils.secrets.get("keyvault_scope", "sql-password"),
      "driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
  }
  ```
- Invalid statuses → discarded  
- Valid statuses → staging

---

### 🧮 Transformations & Aggregations

After validations:
1. Create DataFrames:
   - `orders_df` → from staging folder  
   - `order_items_df` → from orderitems folder  
   - `customers_df` → from Azure SQL Database  
2. Register DataFrames as **temporary views** in Databricks.
3. Perform joins and aggregations to compute:
   - Total **number of orders per customer**  
   - Total **amount spent per customer**

```python
result_df = spark.sql("""
    SELECT c.CustomerID, c.CustomerName,
           COUNT(o.OrderID) AS TotalOrders,
           SUM(oi.Amount) AS TotalAmountSpent
    FROM orders o
    JOIN order_items oi ON o.OrderID = oi.OrderID
    JOIN customers c ON o.CustomerID = c.CustomerID
    GROUP BY c.CustomerID, c.CustomerName
""")
```

4. Write final results to **Azure SQL Database** (serving layer):
```python
result_df.write.jdbc(url=jdbc_url, table="CustomerSalesSummary", mode="overwrite", properties=connection_properties)
```

---

### 🧠 Technologies Used

| Category | Tools / Services |
|-----------|------------------|
| Cloud | Microsoft Azure |
| Storage | Azure Data Lake Gen2 |
| Trigger | Azure Storage Event Trigger |
| Orchestration | Azure Data Factory |
| Compute | Azure Databricks |
| Security | Azure Key Vault |
| Database | Azure SQL Database |
| Source | Amazon S3 |
| Language | Python, PySpark |
| Connectivity | JDBC (SQL DB connections) |

---

### 🔐 Key Vault Integration
All sensitive credentials and connection strings (for SQL DB and S3 access) are securely stored in **Azure Key Vault** and retrieved within both **ADF linked services** and **Databricks notebooks**.

---

### 📊 Output
Final analytical data written to **Azure SQL Database** (table: `CustomerSalesSummary`):

| Column | Description |
|---------|--------------|
| CustomerID | Unique identifier for customer |
| CustomerName | Full name of customer |
| TotalOrders | Total number of orders placed |
| TotalAmountSpent | Total sales value per customer |

---

### 🚀 Automation Highlights
- **Event-driven execution** using **Storage Event Trigger**
- **Dynamic parameter passing** between ADF → Databricks (`@triggerBody().filename`)
- **No hardcoding** of filenames — full dynamic path construction in Databricks
- **Seamless integration** between Data Lake, S3, SQL DB, and Key Vault

---

### 🧰 Skills Demonstrated
- Azure Data Factory pipeline orchestration  
- Event-driven architecture design  
- PySpark-based data validation & transformation  
- Dynamic parameterization using ADF triggers  
- Secure credential management using Key Vault  
- End-to-end ELT workflow implementation  

---

### 🖼️ Architecture Diagram (Suggested)
Upload the following image to this section:
`architecture_diagram.png`

**High-level flow:**
```
Storage Event Trigger → ADF Pipeline
        ├── Copy from Amazon S3 → Data Lake (order_items)
        └── Databricks Notebook → Validation + Aggregation
                 ↓
          Azure SQL Database (Serving Layer)
```

---

---

## 🖼️ Project Snapshots

All key implementation screenshots (Azure resources, Data Lake container structure, ADF pipeline run, and Databricks notebook execution) are available in the file below:

📎 [Sales_Data_Engineering_Project_Snapshots.docx](./Sales_Data_Engineering_Project_Snapshots.docx)


 

---

### 👤 Author
**Konathala Yaswanth Pochi Sai Deep**  
_Azure Data Engineer | Databricks | ADF | Data Lake_  
📍 Hyderabad, India  
