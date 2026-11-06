Data Engineering Assessment Project Report

Production-Grade Idempotent Pipeline & Business Intelligence Architecture
1. Overview of the Complete Solution
This documentation maps out the implementation of an enterprise-grade data engineering platform. The project framework showcases best practices in decoupled data architecture, handling unstructured assets, and transforming raw data into high-performance business intelligence layers. The technical system is structured into four major pipeline zones:
•	Extraction Layer: Utilizes Python scripts inside a Jupyter Notebook environment to programmatically ingest loose, semi-structured data files (sales_data.json and customer_data.json).
•	Transformation & Quality Zone: Executes automated cleaning passes to flatten complex multi-tier nested columns, standardizes unparseable date structures, handles logical math inconsistencies, and dynamically expands transaction patterns to 750 records for robust pipeline stress-testing.
•	Warehouse Storage Layer: Streams clean relational records incrementally into an optimized Star Schema layout inside PostgreSQL, isolating data entities using a explicit staging namespace.
•	BI Semantic Layer: Leverages Power BI Desktop to build direct analytical channels into the warehouse, mapping clean physical relationships and explicit DAX metrics to an executive canvas reporting screen.
2. ETL Approach & Database Schema Design
Relational Data Warehouse Star Schema Definition
To eliminate severe text data redundancy and guarantee structural consistency across data layers, the PostgreSQL warehouse is organized around a multi-tier Star Schema. Physical lookup indexes are explicitly mapped to reduce downstream join complexities from O(N) down to O(log N).
-- 1. Customers Dimension Table
CREATE TABLE IF NOT EXISTS staging.customers (
    customer_id VARCHAR(50) PRIMARY KEY,
    customer_name VARCHAR(255),
    email VARCHAR(255),
    region VARCHAR(100),
    join_date TIMESTAMP,
    loyalty_points INT,
    etl_processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 2. Products Dimension Table
CREATE TABLE IF NOT EXISTS staging.products (
    product_id VARCHAR(50) PRIMARY KEY,
    product_name VARCHAR(255),
    category VARCHAR(100),
    unit_price NUMERIC(10, 2),
    etl_processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Transactions Fact Table
CREATE TABLE IF NOT EXISTS staging.transactions (
    transaction_id VARCHAR(50) PRIMARY KEY,
    customer_id VARCHAR(50) REFERENCES staging.customers(customer_id),
    product_id VARCHAR(50) REFERENCES staging.products(product_id),
    quantity INT,
    discount NUMERIC(5, 2),
    transaction_date TIMESTAMP,
    region VARCHAR(100),
    etl_processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Optimization Indexes
CREATE INDEX IF NOT EXISTS idx_transactions_customer ON staging.transactions(customer_id);
CREATE INDEX IF NOT EXISTS idx_transactions_product ON staging.transactions(product_id);
CREATE INDEX IF NOT EXISTS idx_transactions_date ON staging.transactions(transaction_date);
Idempotent Incremental Upsert Load Pattern
To achieve operational idempotency, the data ingestion loop rejects standard drop-and-replace actions. By implementing an automated pipeline mapping loop using text SQL syntax bindings with the engine manager, the loader leverages native PostgreSQL ON CONFLICT (primary_key) DO UPDATE SET declarations. New operational events slide into vacant data nodes, while modified fields smoothly overwrite stale system values without creating duplicates.
3. Power BI Architecture, DAX Logic & Insights
 
Semantic Measure Expressions
// 1. Total Net Sales Revenue Generation
Total Sales = SUMX(transactions, transactions[quantity] * RELATED(products[unit_price]) * (1 - transactions[discount]))

// 2. Average Ticket Value (Division-by-Zero Protected Context)
Average Sale per Transaction = DIVIDE([Total Sales], DISTINCTCOUNT(transactions[transaction_id]), 0)

// 3. Customer Loyalty Point Sump
Total Loyalty Points = SUM(customers[loyalty_points])

// 4. Time-Intelligence Year-To-Date Rolling Metric
Sales YTD = TOTALYTD([Total Sales], transactions[transaction_date])

// 5. High-Value Ingestion Filter Counter (> $1,000 threshold)
High-Value Transactions = COUNTROWS(FILTER(transactions, (transactions[quantity] * RELATED(products[unit_price]) * (1 - transactions[discount])) > 1000))
Core Business Analytics Takeaways
The synthesized data outputs mapped directly onto the Power BI reporting visual grid provide immediate high-value management observations:
•	• Gross Portfolio Health: The system successfully managed and calculated $916,447.41 in absolute net corporate sales across 99 verified unique active buyer accounts, maintaining a premium $1,255.41 average ticket execution value per order event.
•	• Critical Geographic Territory Gaps: Total revenue is deeply concentrated within the North region, which captures an overwhelming $831,458.07 of all money flowing through the warehouse. The secondary markets—East ($39.6K), West ($31.1K), and South ($14.2K)—exhibit significant underperformance and indicate immediate expansion requirements.
•	• Product Catalog Anchors: The Electronics product catalog acts as the primary revenue engine, pulling in $732,166.92 in net total value. This metric is driven almost exclusively by Laptops, which single-handedly secured $692,543.07 across corporate historical activity.
4. Data Quality Challenges Faced & Solutions
To protect system operations, the transformation code contains defensive blocks designed to defuse severe formatting anomalies hidden within the employer's datasets before pushing columns to the database layer:
Data Anomaly Traps	System Impact If Unmanaged	Defensive Solution Implemented
Corrupted String Datetimes
(e.g., 'invalid_date' variables)	Triggers immediate database query crashes due to structural row value mismatches against rigid TIMESTAMP column parameters.	Passed inputs through pd.to_datetime(errors='coerce') to catch formatting errors, forcing a uniform calendar timestamp fallback ('2024-01-01') on invalid entries.
String Length Violations
(Names/Emails exceeding 300 characters)	PostgreSQL instantly halts processing and throws a StringDataRightTruncation error when incoming data overflows text type limits.	Applied strict Python text slicing windows ([:255]) to the variable parameters loop, trimming overflowing strings safely to guarantee clean constraint compatibility.
Negative Operational Counts
(Negative quantities / loyalty numbers)	Distorts corporate report metrics, outputs invalid negative revenue values, and breaks analytical KPIs.	Wrapped the numeric loading variables inside the absolute value mathematical function abs(), forcing structural normalization to clean positive integers.

