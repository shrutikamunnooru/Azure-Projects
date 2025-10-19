Retail Project

Tools :
1) ADF - for data pipleine
2) ADLS - for storage
3) Databrocks - for multiple dataset creation
4) Power BI - for data visualization

Business requirements:
1. We have to build end to end data pipeline for the retail clients.
2. We have data coming from multiple data sources and we need to bring it into data lake.
3. We have transaction, store and products data available in Azure SQL DB.
4. We have customer data coming from API in JSON format.

 <img width="2768" height="738" alt="image" src="https://github.com/user-attachments/assets/a9d438d4-e7b7-4d6f-94d9-bbedeeddd23d" />
  
We take data from the databases and API into ADLS using ADF, we create pipelines.

We read the data in darabrcks and do data cleaning and tranformation using medallion architecture.
1. Bronze Layer - Raw data taken directly from ADLS which comes from multiple data source, not clean
2. Silver - Clean data, filtering, removing duplicates
3. Gold Layer - Based on business ask, we build gold layr, like total no. of sales w\ect and build gold layer from it.

4. Finally, we perform data visualization.

  Steps 
Open Azure Account

A.Create SQL DB

Create a resource group, database name, server, location, authentication (admin name , PW)
basic compute

Query editor - login with pw

We have to create 3 datasets using DDLS
1. Product - product_id, product_name, category and price
2. Store - store_id, store_name, location
3. Transaction - transaction_id, customer_id, product_id, store_id, quantity, transaction_date

4. for API - in github - click raw - from the url, fetch the data

B. Create ADLS
Select resource group, name, region, hierarchial namespace(for Azure Data Lake) if not it will create blob storage.

Create container - retail

inside container - create multople directory
a. bronze
b. Silver
c. gold

C. ADF Setup

select resource , group, name , region --> launch studio

new pipeline --> copy data - either paramterize or create multiple copy data 

1. Copy data - name - transaction
   Source - Azure SQL DB - create linked service - new - subscriotion name, databse name, user name and pw and create
   then select table - transaction and click ok

   Sink - ADLS - store in parquet data, linked service, select subscription and select storage acc and create
   then select file path - transaction and click ok

2. Copy data for Product
3. Copy data for store
4. Copy data for Customer
Source - http --.> json  --> linkedinservice, pass the base url in linkedinservice , till .com
then give the relative url - rest url

Sink - ADLS - parquet - retial container - bronze - customer

After connecting all copy data, click debug

It will read all the data and copy

All the data will be copied in the respective folders inside the bronze folder in ADLS

Ex. customer.parquet

C. Read data with spark session

Open Terminal

Code

When launching from your Mac terminal, make sure you start Spark with Azure dependencies:

pyspark --packages org.apache.hadoop:hadoop-azure:3.3.4,com.microsoft.azure:azure-storage:8.6.6

# retail_etl_mac.py

from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum, countDistinct, avg

# -----------------------------
# 1️⃣ Create Spark session
# -----------------------------
spark = SparkSession.builder \
    .appName("Retail ETL - ADLS Gen2") \
    .getOrCreate()

# -----------------------------
# 2️⃣ Set Azure storage account key
# -----------------------------
spark.conf.set(
    "fs.azure.account.key.retailproject10.dfs.core.windows.net",
    "YCHThLXmpkAws1DpQjUiQ1AAX2uYLjpR3K1+3QuAqDJSa8OzSXRoufCaOeBvkiWTr3GvzU1SQODn+ASthopeQQ=="
)

# -----------------------------
# 3️⃣ Read Bronze layer
# -----------------------------
df_transactions = spark.read.parquet(
    "abfss://retail@retailproject10.dfs.core.windows.net/bronze/transaction/"
)

df_products = spark.read.parquet(
    "abfss://retail@retailproject10.dfs.core.windows.net/bronze/product/"
)

df_stores = spark.read.parquet(
    "abfss://retail@retailproject10.dfs.core.windows.net/bronze/store/"
)

df_customers = spark.read.parquet(
    "abfss://retail@retailproject10.dfs.core.windows.net/bronze/customer/manish040596/azure-data-engineer---multi-source/refs/heads/main/"
)

# -----------------------------
# 4️⃣ Clean / Silver layer
# -----------------------------
df_transactions = df_transactions.select(
    col("transaction_id").cast("int"),
    col("customer_id").cast("int"),
    col("product_id").cast("int"),
    col("store_id").cast("int"),
    col("quantity").cast("int"),
    col("transaction_date").cast("date")
)

df_products = df_products.select(
    col("product_id").cast("int"),
    col("product_name"),
    col("category"),
    col("price").cast("double")
)

df_stores = df_stores.select(
    col("store_id").cast("int"),
    col("store_name"),
    col("location")
)

df_customers = df_customers.select(
    "customer_id", "first_name", "last_name", "email", "city", "registration_date"
).dropDuplicates(["customer_id"])

# -----------------------------
# 5️⃣ Join all datasets (Silver)
# -----------------------------
df_silver = df_transactions \
    .join(df_customers, "customer_id") \
    .join(df_products, "product_id") \
    .join(df_stores, "store_id") \
    .withColumn("total_amount", col("quantity") * col("price"))

# Preview Silver data
df_silver.show(5)

# -----------------------------
# 6️⃣ Save Silver layer to ADLS
# -----------------------------
silver_path = "abfss://retail@retailproject10.dfs.core.windows.net/silver_parquet/"
df_silver.write.mode("overwrite").parquet(silver_path)

# -----------------------------
# 7️⃣ Gold layer (Aggregations)
# -----------------------------
gold_df = df_silver.groupBy(
    "transaction_date",
    "product_id", "product_name", "category",
    "store_id", "store_name", "location"
).agg(
    sum("quantity").alias("total_quantity_sold"),
    sum("total_amount").alias("total_sales_amount"),
    countDistinct("transaction_id").alias("number_of_transactions"),
    avg("total_amount").alias("average_transaction_value")
)

# Preview Gold data
gold_df.show(5)

# -----------------------------
# 8️⃣ Save Gold layer to ADLS
# -----------------------------
gold_path = "abfss://retail@retailproject10.dfs.core.windows.net/gold_parquet/"
gold_df.write.mode("overwrite").parquet(gold_path)

# -----------------------------
# ✅ Done
# -----------------------------
print("✅ Silver and Gold layers successfully saved to ADLS (abfss) in Parquet format!")




Power BI

Blank report --> Get data -> text.csv --> load

text box - header - retail project
format background 

build vis
Total number of sales - create card
total quantity sold 
number of transactions
Line chart - based on trnsaction_date/ month , show total sales amount
Pie chart - total number of sales by store name
Bar chart - based on total sales , how are the sales happening (sales amount)
Donut chart - Based on category , sum of quantity sold
Column chart - based on transaction_value , how many product are sold
Based on store name , what are the total sales amount

<img width="1972" height="1106" alt="image" src="https://github.com/user-attachments/assets/2fa3b22e-909f-42a7-8807-b1d008163756" />

