# AWS Serverless ETL Pipeline (S3 + Lambda + Glue + Athena)

This project implements an event‑driven, serverless ETL pipeline on AWS. A JSON orders file is uploaded to an S3 bucket, automatically triggering a Lambda function that flattens the nested JSON, writes the transformed data as Parquet back to S3, and makes it queryable in Amazon Athena via an AWS Glue Data Catalog table.

## Architecture

- **Amazon S3**
  - Source bucket/folder for incoming nested JSON (e.g. `orders_json_incoming/`).
  - Target bucket/folder for transformed Parquet data (e.g. `orders_parquet_datalake/`).

- **AWS Lambda**
  - Triggered by S3 `PUT` events on the incoming JSON prefix.
  - Reads the file from S3 using `boto3`.
  - Flattens nested JSON to a tabular structure using Python and Pandas.
  - Writes the result as a Parquet file to the data lake folder in S3.
  - Optionally triggers an AWS Glue crawler to refresh the catalog.

- **AWS Glue**
  - Glue Crawler scans the Parquet location in S3.
  - Creates/updates a table in the Glue Data Catalog with schema information (columns, data types, location, format).

- **Amazon Athena**
  - Uses the Glue Data Catalog as its metastore.
  - Runs SQL queries directly on the Parquet data stored in S3.
  - Supports ad‑hoc analytics such as total sales per customer, order‑level aggregations, etc.

## Tech Stack

- AWS: S3, Lambda, Glue, Athena, CloudWatch Logs, IAM
- Language: Python (Lambda + local development)
- Libraries: `boto3`, `pandas`, `pyarrow` (or `fastparquet`), `json`, `io`, `datetime`
- Data format: Input JSON, output Parquet

## How the Pipeline Works

1. **Upload JSON to S3**
   - User drops a nested JSON orders file (e.g. `orders_etl.json`) into the configured S3 bucket and prefix.
   - S3 emits an event that triggers the Lambda function.

2. **Lambda Transformation**
   - Lambda receives the S3 event, extracts `bucket` and `key` from the `event` payload.
   - Uses `boto3` to `get_object` and reads the JSON content.
   - Loads JSON into Python, flattens it into a Pandas DataFrame (one row per order‑product combination).
   - Writes the DataFrame to an in‑memory buffer as a Parquet file.
   - Generates a unique filename using a timestamp (for example `orders_etl_YYYYMMDD_HHMMSS.parquet`).
   - Uploads the Parquet buffer to the target S3 folder (the data lake path).

3. **Glue Crawler & Data Catalog**
   - A Glue Crawler is configured to crawl the Parquet folder.
   - On run, it infers:
     - Column names and data types.
     - File format (Parquet).
     - S3 location.
   - It then creates or updates a table in a Glue database (for example: database `etl_pipeline`, table `orders_parquet_datalake`).

4. **Query with Athena**
   - In Athena, select the Glue database and table.
   - Run SQL queries such as:
     - `SELECT * FROM orders_parquet_datalake LIMIT 10;`
     - `SELECT customer_id, SUM(total_amount) AS total_sales FROM orders_parquet_datalake GROUP BY customer_id;`
   - Results are written to an S3 query results location configured in Athena.

## Setup Instructions

1. **AWS Account**
   - Create or use an existing AWS account.
   - Ensure you are in your preferred region.
   - Enable free‑tier eligible services where applicable.

2. **Create S3 Bucket and Folders**
   - Create an S3 bucket (name must be globally unique).
   - Inside the bucket, create:
     - `orders_json_incoming/` for raw JSON.
     - `orders_parquet_datalake/` for processed Parquet data.

3. **Prepare Lambda IAM Role**
   - Create an IAM role for Lambda with:
     - Basic Lambda execution policy (CloudWatch Logs).
     - S3 read/write access for the chosen bucket.
   - Attach the role to the Lambda function.

4. **Create Lambda Function**
   - Runtime: Python 3.x.
   - Add the Lambda layer that provides Pandas and PyArrow (AWS SDK for Pandas layer or a custom layer).
   - Paste the transformation code:
     - Parse S3 event.
     - Read JSON from S3.
     - Flatten with Pandas.
     - Write Parquet to the data lake folder.
   - Increase Lambda timeout (for example to 30 seconds) to avoid timeouts while importing libraries and processing data.
   - Deploy the function.

5. **Configure S3 Trigger**
   - Add an S3 trigger to the Lambda:
     - Event type: ObjectCreated (PUT).
     - Bucket: your project bucket.
     - Prefix: `orders_json_incoming/`.
     - Suffix: `.json`.
   - Save the trigger.

6. **Create Glue Database and Crawler**
   - In AWS Glue, create a database (for example `etl_pipeline`).
   - Create a Crawler:
     - Data source: S3, pointing to the `orders_parquet_datalake/` folder.
     - IAM role: new or existing role with S3 and Glue permissions.
     - Target: the Glue database you created.
     - Schedule: On‑demand (or scheduled if desired).
   - Run the crawler to create the table.

7. **Query Data in Athena**
   - Open Athena and select the Glue data catalog and the `etl_pipeline` database.
   - Confirm table (for example `orders_parquet_datalake`) is visible.
   - Run sample queries to validate the pipeline.

## Project Structure (example)

.
├── lambda_function.py # Lambda handler + JSON flatten logic
├── flatten_utils.py # Optional: helper functions to flatten nested JSON
├── sample_data/
│ └── orders_etl.json # Example input JSON file
└── README.md # This file


## JSON Flattening Logic (Concept)

- Input JSON structure:
  - Top‑level list of orders.
  - Each order contains:
    - Order‑level fields: `order_id`, `order_date`, `total_amount`, `customer` details.
    - Nested list of products.
- Flattening strategy:
  - Loop over each order.
  - For each product within the order, create a separate row combining:
    - Order‑level fields.
    - Customer fields.
    - Product‑level fields.
  - Append all rows to a list and convert to a Pandas DataFrame.

This results in a table where each row represents one product within one order, ready for analytics in Athena.

## Notes

- Parquet provides better compression and query performance than CSV, especially for columnar analytics workloads.
- IAM permissions and least‑privilege should be carefully managed in a real project.
- For production, you can extend this pattern with Step Functions for orchestration, error handling, and retries.
