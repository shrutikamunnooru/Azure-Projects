# Fintech Data Migration Pipeline

**Azure Synapse | PySpark | Delta Lake | ADLS | Logic Apps**

---

## 📌 Project Overview
This project implements an **end-to-end fintech data migration and analytics pipeline** using Microsoft Azure and Apache Spark. The goal is to migrate historical transactional data from an **Azure SQL Database** into a scalable **Lakehouse architecture** on **Azure Data Lake Storage (ADLS)** following the **Medallion (Bronze–Silver–Gold) pattern**.

The pipeline is fully automated using **Azure Synapse Pipelines**, data is transformed using **PySpark**, stored as **Delta tables**, and monitored using **Azure Logic Apps** for success and failure alerts.

---

## 🧠 Business Problem
Fintech operational databases are optimized for **OLTP workloads** and are not suitable for analytics at scale. Running reporting queries directly on these systems can:
- Impact production performance
- Increase infrastructure cost
- Limit analytical scalability

This project solves the problem by:
- Migrating historical data out of the operational database
- Structuring it into analytics-friendly layers
- Enabling scalable reporting without affecting source systems

---

## 🏗️ Architecture Overview
**High-level flow:**

Azure SQL Database  
→ Azure Synapse Pipeline  
→ ADLS Bronze (Raw Data)  
→ PySpark Transformations  
→ ADLS Silver (Clean Delta Tables)  
→ PySpark Modeling  
→ ADLS Gold (Fact & Dimension Tables)  
→ Azure Logic Apps (Monitoring & Alerts)

> 📸 *<img width="1368" height="1066" alt="image" src="https://github.com/user-attachments/assets/0830bab3-1c15-455a-a121-cf8adf1d038e" />*

---

## 🗄️ Source System
- **Database**: Azure SQL Database  
- **Schema**: `fintech`  
- **Tables**:
  - Customers
  - Accounts
  - Loans
  - Payments
  - Transactions  
- **Data Type**: Historical batch data

---

## 🔄 End-to-End Data Flow

### 1️⃣ SQL Database → Bronze Layer (Ingestion)
- Azure Synapse Pipeline uses a **Lookup activity** to dynamically retrieve table names
- A **ForEach activity** iterates through the tables
- **Copy activity** loads data into ADLS Bronze as parquet files
- No transformations are applied at this stage

> 📸 *<img width="1270" height="467" alt="image" src="https://github.com/user-attachments/assets/02ab2b2b-52e5-4d44-8761-a89644390d4a" />*

---

### 2️⃣ Bronze Layer
- Stores raw, immutable data
- One-to-one mapping with source tables
- Serves as the system of record in the data lake

bronze/fintech/
├── Customers/
├── Accounts/
├── Loans/
├── Payments/
└── Transactions/



---

### 3️⃣ Bronze → Silver (Data Cleaning & Enrichment)
A PySpark notebook processes each dataset independently and writes **Delta tables** to the Silver layer.

**Key transformations include:**
- Account age calculation
- Customer full name creation and email masking (PII protection)
- Loan interest and duration calculation with explicit decimal casting
- Payment recency calculation
- Transaction categorization (Income / Expense / Other)


---

### 4️⃣ Silver Layer
- Contains clean, standardized, and trusted datasets
- Stored as **Delta tables** to support ACID transactions and schema evolution
- Serves as the foundation for analytics modeling

---

### 5️⃣ Silver → Gold (Analytics Modeling)
A second PySpark notebook transforms Silver data into analytics-ready datasets.

**Gold tables created:**
- **Dimension Tables**
  - `dim_customers`
  - `dim_accounts`
  - `dim_loans`
- **Fact Tables**
  - `fact_payments`
  - `fact_transactions`

Joins are used to correctly associate events with customers through account and loan relationships.



---

### 6️⃣ Gold Layer
- Business-ready, analytics-optimized data
- Modeled using star-schema principles
- Designed for consumption by BI tools such as Power BI

---

## 🔔 Monitoring & Alerts (Azure Logic Apps)
- Azure Synapse Pipeline triggers a **Logic App via HTTP** on success or failure
- Pipeline execution details are passed dynamically
- Logic App sends automated email notifications for:
  - Successful pipeline completion
  - Pipeline failures requiring attention

<img width="963" height="493" alt="image" src="https://github.com/user-attachments/assets/ddbd3194-a5ea-4fd2-8832-35596173afd2" />


---

## 🛠️ Tech Stack
- **Languages**: Python, SQL  
- **Cloud Platform**: Microsoft Azure  
- **Services**:
  - Azure SQL Database
  - Azure Synapse Analytics
  - Azure Data Lake Storage (ADLS Gen2)
  - Azure Logic Apps  
- **Processing Engine**: PySpark  
- **Storage Format**: Delta Lake  

---

## 🎯 Key Learnings
- Designed a metadata-driven ingestion pipeline using Azure Synapse
- Implemented Medallion architecture for scalable data processing
- Built PySpark transformations with business-oriented logic
- Modeled data into fact and dimension tables for analytics
- Integrated monitoring and alerting using Azure Logic Apps

---

## 🔮 Future Enhancements
- Implement CDC / incremental data ingestion
- Add data quality validation checks
- Optimize Delta tables with partitioning and Z-ordering
- Integrate Power BI dashboards for reporting

---

## 🧩 Summary
This project demonstrates how to build a **production-style fintech data pipeline** that combines scalable ingestion, Spark-based transformations, analytics modeling, and operational monitoring using Azure cloud services.


