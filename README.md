# Data Engineering Zoomcamp - Module 3 - Data Warehouse

## General notes (from videos)

OLTP = Online Transaction Processing
OLAP = Online Analytical Processing

Partitioning is only possible on 1 column.
Limited to 4000 partitions.

Clustering generally improves the filtering and aggregating queries.
Really only useful for large datasets > 1GB.
Limited to 4 clustering columns.

Cluster on columns with over 4k different values.
Cardinality == number of unique values in a column.

BigQuery caches query results.

Denormalizing == expanding by adding more columns that can be sorted.

Join by largest_table + smallest_table + 2nd_largest_table + 3rd_largest_table + 4th_largest_table ...

Usually, you can get good performance just by following best practices, without knowing what is going on under the hood.

One-Hot encoding == adding more columns for categorical values, 1 where category is true, else 0.
Converts a category into a sparse vector.
So for [dog, cat, fish] categories, dog = [1, 0, 0].

Some columns contain numerical values, but are a actually categories. Fix this by casting them to strings, then BigQuery can/will do preprocessing in these columns.

## Question 1:

What is count of records for the 2024 Yellow Taxi Data?

- 65,623
- 840,402
- 20,332,093
- 85,431,289

### Steps

1. Use the Module 1 Terraform code to set up the bucket and dataset.
2. Run the python script `load_yellow_taxi_data.py` to download the parquet files and upload them to the GCP Bucket created in step 1.
3. Go to BigQuery in the GCS console, and execute the following:

```sql
CREATE OR REPLACE EXTERNAL TABLE `my_project_name.my_bucket_name.external_table`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://my_bucket_uri/*.parquet']
);

SELECT count(*) FROM `my_project_name.my_bucket_name.external_table`;
```

### Answer

The result should be `20332093` (Option 3)

## Question 2:

### Question

Write a query to count the distinct number of PULocationIDs for the entire dataset on both the tables.</br>
What is the **estimated amount** of data that will be read when this query is executed on the External Table and the Table?

- 18.82 MB for the External Table and 47.60 MB for the Materialized Table
- 0 MB for the External Table and 155.12 MB for the Materialized Table
- 2.14 GB for the External Table and 0MB for the Materialized Table
- 0 MB for the External Table and 0MB for the Materialized Table

### Steps

1. Create the materialized table

```sql
CREATE OR REPLACE TABLE `my_project_name.my_bucket_name.materialized_table`
AS SELECT * FROM `my_project_name.my_bucket_name.external_table`;
```

2. Use the below sql to estimate the amount of data that will be read for the two tables

```sql
SELECT COUNT(DISTINCT(PULocationID)) FROM `my_project.my_dataset.external_table`;
SELECT COUNT(DISTINCT(PULocationID)) FROM `my_project.my_dataset.materialized_table`;
```

### Answer

0 MB for the External Table and 155.12 MB for the Materialized Table (Option 2)

## Question 3:

Write a query to retrieve the PULocationID from the table (not the external table) in BigQuery. Now write a query to retrieve the PULocationID and DOLocationID on the same table. Why are the estimated number of Bytes different?

- BigQuery is a columnar database, and it only scans the specific columns requested in the query. Querying two columns (PULocationID, DOLocationID) requires
  reading more data than querying one column (PULocationID), leading to a higher estimated number of bytes processed.
- BigQuery duplicates data across multiple storage partitions, so selecting two columns instead of one requires scanning the table twice,
  doubling the estimated bytes processed.
- BigQuery automatically caches the first queried column, so adding a second column increases processing time but does not affect the estimated bytes scanned.
- When selecting multiple columns, BigQuery performs an implicit join operation between them, increasing the estimated bytes processed

### Steps

1. Use the below SQL queries to estimate the number of bytes processed

```sql
SELECT PULocationID FROM `my_project.my_dataset.materialized_table`;
SELECT PULocationID, DOLocationID FROM `my_project.my_dataset.materialized_table`;
```

### Answer

BigQuery is a columnar database.
The first query results in an estimate of 155.12 MB.
The second query results in an estimate of 310.24 MB. -> double the estimate of 1 column

The answer is Option 1.

## Question 4:

### Question

How many records have a fare_amount of 0?

- 128,210
- 546,578
- 20,188,016
- 8,333

### Steps

Run the below SQL:

```sql
SELECT count(*) FROM  `my_project.my_dataset.materialized_table`
WHERE fare_amount = 0;
```

### Answer

Running the above SQL gives the result `8333` (Option 4)

## Question 5:

### Question

What is the best strategy to make an optimized table in Big Query if your query will always filter based on tpep_dropoff_datetime and order the results by VendorID (Create a new table with this strategy)

- Partition by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on by tpep_dropoff_datetime and Cluster on VendorID
- Cluster on tpep_dropoff_datetime Partition by VendorID
- Partition by tpep_dropoff_datetime and Partition by VendorID

### Steps

Run the below SQL:

```sql
CREATE OR REPLACE TABLE `my_project.my_dataset.partitioned_tpep_dropoff_clustered_vendorid_table`
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS (
  SELECT * FROM `my_project.my_dataset.materialized_table`
);
```

### Answer

You would partition the table by `tpep_dropoff_datetime` (since filtered by this field) and then cluster by `VendorID` (since sorted by this field) (Option 1).

## Question 6:

### Question

Write a query to retrieve the distinct VendorIDs between tpep_dropoff_datetime
03/01/2024 and 03/15/2024 (inclusive)</br>

Use the materialized table you created earlier in your from clause and note the estimated bytes. Now change the table in the from clause to the partitioned table you created for question 4 and note the estimated bytes processed. What are these values? </br>

Choose the answer which most closely matches.</br>

- 12.47 MB for non-partitioned table and 326.42 MB for the partitioned table
- 310.24 MB for non-partitioned table and 26.84 MB for the partitioned table
- 5.87 MB for non-partitioned table and 0 MB for the partitioned table
- 310.31 MB for non-partitioned table and 285.64 MB for the partitioned table

### Steps

Run the below SQL:

```sql
SELECT COUNT(DISTINCT(VendorID))  FROM  `my_project.my_dataset.materialized_table`
WHERE DATE(tpep_dropoff_datetime) BETWEEN '2024-03-01' AND '2024-03-15';

SELECT COUNT(DISTINCT(VendorID))  FROM `my_project.my_dataset.partitioned_tpep_dropoff_clustered_vendorid_table`
WHERE DATE(tpep_dropoff_datetime) BETWEEN '2024-03-01' AND '2024-03-15';
```

### Answer

310.24 MB for non-partitioned table and 26.84 MB for the partitioned table (Option 2)

## Question 7:

### Question

Where is the data stored in the External Table you created?

- Big Query
- Container Registry
- GCP Bucket
- Big Table

### Steps

N/A

### Answer

The data is stored in a GCP Bucket (Option 3)

## Question 8:

### Question

It is best practice in Big Query to always cluster your data:

- True
- False

### Steps

N/A

### Answer

Clustering is generally only useful for large datasets > 1GB, on columns which have high cardinality. Therefore it is not always 'best practice' (Option 2)

## (Bonus: Not worth points) Question 8:

### Question

No Points: Write a `SELECT count(*)` query FROM the materialized table you created. How many bytes does it estimate will be read? Why?

### Steps

```sql
SELECT count(*) FROM  `my_project.my_dataset.materialized_table`
```

### Answer

It estimates 0 B will be processed, because it will take this information directly from the table's metadata. As a result, no records from the table need to be read.
