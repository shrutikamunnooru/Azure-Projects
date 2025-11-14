# AirBnB CDC Data Pipeline on Azure

This project implements an end-to-end **Change Data Capture (CDC)** data pipeline on Azure for an AirBnB-style application.  
It demonstrates how to combine batch processing (customer updates) and real-time CDC streaming (booking events) using:

- Azure Data Lake Storage (ADLS)  
- Azure Cosmos DB (Change Feed)  
- Azure Data Factory (ADF)  
- Mapping Data Flows  
- Azure Synapse Analytics  

The goal is to keep customer and booking data synchronized and analysis-ready using **incremental ingestion**, **SCD-1 logic**, and **insert/update handling**.

---

## 1. Architecture Overview

The system handles two different data flows:

### **Customer Data – Micro-Batch**
Customer profile updates arrive as CSV files every 30 minutes.  
These represent new customers or updated details for existing customers.

These files are processed using ADF and loaded into a **customer dimension** table in Synapse using **SCD-1 (overwrite/upsert)** logic.

### **Booking Data – Real-Time CDC**
Booking events from the AirBnB application (simulated with Python) are written to **Azure Cosmos DB**.  
ADF reads only the **incremental changes** from Cosmos DB through the Change Feed.

These booking events are transformed, cleaned, and upserted into a **booking fact** table in Synapse.

### **Orchestration Layer**
A master pipeline ensures correct order:

1. Load Customer Dimension  
2. Load Booking Fact  
3. Send notifications  

---

## 2. Data Sources

### **Customer CSV Data**
Delivered every 30 minutes and stored in ADLS:

/airbnb/customer_raw_data/

After processing, files are archived:

/airbnb/customer_archive/

This prevents reprocessing of old files.

---

### **Booking Events (Cosmos DB)**
Stored in a Cosmos DB Database and Container:

- Database: `AirBnB`
- Container: `Bookings`

The pipeline reads only new/updated documents using Change Feed.

---

## 3. Storage Layers

### **ADLS Folder Structure**

/airbnb
/customer_raw_data # incoming customer CSVs
/customer_archive # processed customer files


### **Azure Synapse Conceptual Tables**

- **customer_dim**
  - Holds latest customer attributes  
  - Updated using SCD-1 (overwrite/upsert)

- **bookings_fact**
  - Flattened, enriched booking events  
  - Populated through CDC logic

- **Booking Aggregation Table**
  - Summary metrics updated after each load

---

## 4. Pipeline 1 – Customer Dimension Load (SCD-1)

This pipeline processes customer CSV files and updates the customer dimension in Synapse.

### **Flow:**

1. **List incoming files**  
   ADF retrieves all files in `customer_raw_data`.

2. **Iterate through each file**  
   A ForEach activity loops over each file dynamically.

3. **Load into Synapse with SCD-1 upsert behavior**  
   The copy activity uses:
   - Write behavior: **Upsert**
   - Key column: `customer_id`

   Meaning:
   - If `customer_id` exists → update the record  
   - If it doesn’t → insert a new record  

4. **Archive the processed file**  
   The file is moved to `customer_archive`.

5. **Delete from raw folder**  
   Removes the file to prevent reprocessing.

   <img width="1393" height="425" alt="image" src="https://github.com/user-attachments/assets/05ecdee9-7542-44bd-a4d7-cc3ba3835f6b" />


### **Why this design?**
Customer attributes must always reflect the *latest* state.  
SCD-1 overwrite logic ensures correctness.  
Archive + deletion ensures file-level incremental ingestion.

---

## 5. Pipeline 2 – Booking Fact Load (CDC from Cosmos DB)

This pipeline processes real-time booking events using Cosmos DB Change Feed.

### **Flow:**

1. **Read from Change Feed**  
   Only new/modified booking events since the previous run are processed.

2. **Filter invalid records**  
   Example: check-in date after check-out date.

3. **Transform and flatten**  
   - Extract nested JSON fields  
   - Calculate stay duration  
   - Derive booking year/month  
   - Build full address  

4. **Lookup in Synapse**  
   Checks if the incoming `booking_id` already exists in the fact table.

5. **Insert/update decision**  
   Using Mapping Data Flow:
   - If booking_id does not exist → insert  
   - If it exists → update  

6. **Load into Synapse**  
   Only new and updated bookings are written.

7. **Refresh reporting table**  
   A stored procedure recomputes aggregated metrics.

<img width="1393" height="524" alt="image" src="https://github.com/user-attachments/assets/01c5cc5f-2dab-431d-9b4c-eaa2cfb09193" />


### **Why this design?**
- Change Feed provides clean incremental CDC.  
- Lookup + Alter Row ensures correct upsert behavior.  
- Fact table always holds most recent booking state.

---

## 6. Pipeline 3 – Master Orchestration

This pipeline ensures the correct sequence:

1. **Run Customer Dimension Load**  
   Customer data is foundational; booking events depend on it.

2. **Run Booking Fact Load**  
   Processes CDC booking events after customer updates.

3. **Send Notification**  
   A Logic App sends a success or failure alert.

### **Why customer first?**
Bookings reference `customer_id`.  
So the customer dimension **must** be updated before booking events are processed.

<img width="2786" height="1538" alt="image" src="https://github.com/user-attachments/assets/d4beddc3-cb21-4ce8-81dd-06f4c4697d9e" />


---

## 7. Incremental Processing Behavior

### **Customer Data**
- Incremental at *file level*  
- Only files in `customer_raw_data` when pipeline runs are processed  
- Processed files → archived → removed from raw folder  

### **Booking Data**
- Incremental at *record level*  
- Only new/updated records from the Change Feed are processed  
- Testing confirmed only new events were ingested, not historical Cosmos DB data  

---

## 8. Strengths of This Design

- Mixes batch and CDC processing seamlessly  
- Uses SCD-1 for the customer dimension (correct for profile data)  
- Implements real CDC using Cosmos DB Change Feed  
- Reliable upsert logic ensures accurate fact & dimension tables  
- Clear orchestration ordering  
- Incremental and idempotent  
- Secure authentication with managed identity  
- Transformations handled cleanly in Mapping Data Flows  
- Business logic isolated in a stored procedure  

---

## 9. End-to-End Run Summary

1. Customer CSVs arrive in ADLS  
2. Pipeline 1 loads customers using SCD-1 upsert  
3. Files are archived and removed  
4. Booking events enter Cosmos DB  
5. Pipeline 2 reads only new CDC events  
6. Data is validated, flattened, enriched  
7. Insert/update logic updates bookings_fact  
8. Aggregations refreshed  
9. Master pipeline orchestrates all steps  

---

## 10. What This Project Demonstrates

- Building a real-world Azure Data Engineering solution  
- Micro-batch ingestion patterns  
- CDC ingestion using Cosmos DB Change Feed  
- SCD-1 dimensional modeling  
- Data Flow transformations & upsert logic  
- Idempotent pipeline design  
- Best-practice orchestration and sequencing  

---

**Author:**  
**Shrutika Munnooru**
