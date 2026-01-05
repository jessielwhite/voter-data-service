# Voter Data Service
A small ETL and schema utility for ingesting county-specific voter files from Google Cloud Storage into BigQuery, with helper utilities for parsing document-based file layouts. The repository contains scripts for:

- Downloading and parsing a DOCX voter file layout into CSV
- Converting delimited TXT voter files to CSV
- Defining BigQuery schemas and creating tables (voter_registry, voting_history)
- An Airflow DAG (and an ETL DAG) to run the ingestion/cleaning and load to BigQuery
- Utility and orchestration scripts for running locally or in Google Cloud Composer

### Project overview
----------------
This repo automates key parts of a pipeline that:
1. Downloads a DOCX file describing the fixed-width or delimited voter file layout from GCS.
2. Parses and converts that DOCX layout to a CSV describing columns/positions.
3. Converts county TXT exports into CSV (auto-detecting delimiter).
4. Cleans and uploads the CSV to Cloud Storage and imports it to BigQuery.
5. Provides schema definitions and table creation helpers for BigQuery, including partitioning/clustering and an example hashing helper for searchable hashed fields.

### Quick start (local)
-------------------
Prerequisites:
- Python 3.8+ (3.9 or 3.10 recommended)
- A Google Cloud project with:
  - Cloud Storage bucket for the source files
  - BigQuery dataset for tables
- A service account with the required roles (see GCP & IAM Requirements)

Local setup:
1. Clone the repo:
   git clone https://github.com/jessielwhite/voter_data_service.git
2. Create virtual environment and install:
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   or
   make install
3. Create a `.env` file in the repository root with the necessary environment variables (see below).

### Environment variables (.env)
----------------------------
Example `.env`
```
PROJECT_ID=my-gcp-project
BUCKET_NAME=my-bucket
BUCKET_URI=gs://my-bucket
DATASET_ID=my_dataset
TABLE_ID=voter_registry
```

### Running the components
----------------------

main.py utilities
- authenticate_implicit_with_adc(project_id)
  - Lists buckets to verify ADC (Application Default Credentials) or service account authentication is working.
- download_blob(bucket, source_voter_file_layout, destination_voter_file_layout)
  - Downloads a blob from the given bucket path to local filename.
  - Example usage (after .env and auth are set):
    python main.py
    (The script as shipped invokes some helpers; adjust for your flow.)
- voter_file_layout_docx_to_csv(docx_layout, csv_layout)
  - Reads the first table in a DOCX and writes cleaned rows to a CSV.
- voter_data_txt_to_csv(txt_voter_data, csv_voter_data, delimiter=None)
  - Converts delimited TXT to CSV; if `delimiter` is None, auto-detects from the first line.

schema.py (BigQuery)
- Defines `create_voter_registry_table(dataset_id)` and `create_voting_history_table(dataset_id)` with many fields, partitioning, clustering, and an example of setting primary/foreign key placeholders in a custom Table class.
- To create tables run:
  python schema.py
- Note: The sample code calls the functions at import time. Ensure environment variables are set and you have BigQuery permissions before running.

voter_data_etl.py (Airflow DAG)
- A DAG definition (`voter_data_etl`) that:
  - Runs `clean_csv_for_bigquery` (clean CSV for load)
  - Uploads cleaned CSV to GCS
  - Submits a BigQuery load job
- Deploy to Cloud Composer / Airflow by copying this file into the Composer DAGs folder (or local Airflow DAGs folder) and configuring the connection/environment variables accordingly.

voter_data_dag.py
- Additional DAG demonstrating a liveness prober and orchestration example that calls a bash script and triggers an ExecuteAirflowCommand to start a pipeline.

### GCP & IAM requirements
----------------------
Service account for running this pipeline should have:
- Storage: roles/storage.objectViewer (or roles/storage.admin for uploads)
- BigQuery: roles/bigquery.dataEditor (or roles/bigquery.admin for table creation)
- Composer/Cloud Composer: roles/composer.worker (as appropriate)
- If you use separate service accounts for Composer, ensure they have appropriate access to buckets & datasets.


### Contact / Author
----------------
Repository owner: @jessielwhite
