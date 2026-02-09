# Yellow Taxi Trip Data Pipeline

## Overview

This project demonstrates a pipeline for downloading, uploading, and analyzing the **Yellow Taxi Trip Data (2024)** using **Google Cloud Storage (GCS)** and **BigQuery (BQ)**.

---

## Prerequisites

* Python 3.x
* Google Cloud SDK
* `google-cloud-storage` Python package
* GCP project with access to BigQuery and GCS
* GCP service account JSON credentials (optional if authenticated via SDK)

Install dependencies:

```bash
pip install google-cloud-storage
```

---

## Script: `upload_yellow_tripdata.py`

This Python script downloads Yellow Taxi trip data for the first 6 months of 2024, uploads it to a GCS bucket, and verifies the upload.

### Key Features

* Download Parquet files from the NYC Yellow Taxi dataset.
* Upload files to GCS with retries and verification.
* Supports multithreaded downloads and uploads for efficiency.
* Creates the GCS bucket if it doesn't exist.
* Configurable chunk size for large files.

### Usage

```bash
python upload_yellow_tripdata.py
```

### Configuration

* `BUCKET_NAME`: Name of your GCS bucket.
* `BASE_URL`: URL prefix for downloading Yellow Taxi trip data.
* `MONTHS`: List of months to download (default: `01` to `06`).
* `DOWNLOAD_DIR`: Local directory to save downloaded files.

---

## BigQuery Steps

### 1. Create External Table

```sql
CREATE OR REPLACE EXTERNAL TABLE `big-query-prj-486915.yellow_taxi_trip.external_yellow_trip_records`
OPTIONS (
  format = 'PARQUET',
  uris = ['gs://zoomcamp_mod3_datawarehouse_dec_hw3/yellow_tripdata_2024-*.parquet']
);
```

* Data remains in GCS.
* Queries are **serverless**, scanning only the necessary data.

---

### 2. Create Materialized/Regular Table

```sql
CREATE OR REPLACE TABLE big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records AS
SELECT * FROM big-query-prj-486915.yellow_taxi_trip.external_yellow_trip_records;
```

* Materialized table stores the data in BigQuery storage.
* Queries scan the stored data (more efficient for repeated queries).

---

### 3. Sample Queries

**Count all records in 2024 dataset:**

```sql
SELECT COUNT(*)
FROM big-query-prj-486915.yellow_taxi_trip.external_yellow_trip_records;
-- 20,332,093
```

**Count distinct `PULocationID`:**

```sql
-- External table
SELECT COUNT(DISTINCT PULocationID)
FROM big-query-prj-486915.yellow_taxi_trip.external_yellow_trip_records;

-- Regular table
SELECT COUNT(DISTINCT PULocationID)
FROM big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records;
```

* External table: 0 bytes scanned (serverless metadata)
* Regular table: 155.12 MB scanned

**Retrieve pickup/dropoff IDs:**

```sql
SELECT PULocationID
FROM big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records;

SELECT PULocationID, DOLocationID
FROM big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records;
```

**Count trips with `fare_amount = 0`:**

```sql
SELECT COUNT(*)
FROM big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records
WHERE fare_amount = 0;
-- 8,333
```

---

### 4. Partitioned and Clustered Table

```sql
CREATE OR REPLACE TABLE big-query-prj-486915.yellow_taxi_trip.yellow_trip_records_partitioned
PARTITION BY DATE(tpep_dropoff_datetime)
CLUSTER BY VendorID AS
SELECT *
FROM big-query-prj-486915.yellow_taxi_trip.external_yellow_trip_records;
```

* Partitioning by `tpep_dropoff_datetime` reduces scanned data for date-specific queries.
* Clustering by `VendorID` improves performance for queries filtering on VendorID.

**Example: Distinct VendorIDs (March 1–15, 2024)**

```sql
-- Regular table
SELECT DISTINCT VendorID
FROM big-query-prj-486915.yellow_taxi_trip.regular_yellow_trip_records
WHERE tpep_dropoff_datetime BETWEEN '2024-03-01' AND '2024-03-15';

-- Partitioned table
SELECT DISTINCT VendorID
FROM big-query-prj-486915.yellow_taxi_trip.yellow_trip_records_partitioned
WHERE tpep_dropoff_datetime BETWEEN '2024-03-01' AND '2024-03-15';
```

* Partitioned table scans fewer bytes → faster and cheaper queries.

---

## Notes

* Always verify GCS uploads to ensure data integrity.
* Use **external tables** for occasional queries to save storage costs.
* Use **partitioned and clustered tables** for large datasets to improve query efficiency.
