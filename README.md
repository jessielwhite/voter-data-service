```markdown
# voter_data_service

A small ETL and schema utility for ingesting county-specific voter files from Google Cloud Storage into BigQuery, with helper utilities for parsing document-based file layouts. The repository contains scripts for:

- Downloading and parsing a DOCX voter file layout into CSV
- Converting delimited TXT voter files to CSV
- Defining BigQuery schemas and creating tables (voter_registry, voting_history)
- An Airflow DAG (and an ETL DAG) to run the ingestion/cleaning and load to BigQuery
- Utility and orchestration scripts for running locally or in Google Cloud Composer

Important: This project works with highly sensitive personal data (voter records). Read and follow the Security & Privacy section before running anything.

Table of contents
- Project overview
- Repository layout
- Quick start (local)
- Environment variables / .env
- Running the components
  - main.py utilities
  - schema.py (BigQuery table creation)
  - voter_data_etl.py (Airflow DAG)
  - voter_data_dag.py (liveness / orchestration)
- GCP & IAM requirements
- Development notes, TODOs and improvements
- Security & privacy considerations
- Contributing
- License

Project overview
----------------
This repo automates key parts of a pipeline that:
1. Downloads a DOCX file describing the fixed-width or delimited voter file layout from GCS.
2. Parses and converts that DOCX layout to a CSV describing columns/positions.
3. Converts county TXT exports into CSV (auto-detecting delimiter).
4. Cleans and uploads the CSV to Cloud Storage and imports it to BigQuery.
5. Provides schema definitions and table creation helpers for BigQuery, including partitioning/clustering and an example hashing helper for searchable hashed fields.

Repository layout
-----------------
- `main.py` — Utilities:
  - Authenticate to GCS
  - Download a DOCX file from a bucket
  - Convert DOCX layout table to CSV
  - Convert delimited TXT files to CSV (auto-detects delimiter \t, |, or ,)
- `schema.py` — BigQuery schema creation:
  - `create_voter_registry_table(dataset_id)`
  - `create_voting_history_table(dataset_id)`
  - `hash_columns_for_search(fields_to_hash)` (returns expression string for hashing)
- `voter_data_etl.py` — Airflow DAG:
  - DAG to clean CSV for BigQuery, upload cleaned file to GCS, and load to BigQuery
  - Uses `PythonOperator`, `LocalFilesystemToGCSOperator`, and `BigQueryInsertJobOperator`
- `voter_data_dag.py` — Additional DAG showing orchestration/liveness tasks and example of ExecuteAirflowCommand usage
- `Makefile` — convenience: `make install`, `make run` (relies on `.venv` path & requirements.txt)
- `README.md` — (this file)

Quick start (local)
-------------------
Prerequisites:
- Python 3.8+ (3.9 or 3.10 recommended)
- A Google Cloud project with:
  - Cloud Storage bucket for the source files
  - BigQuery dataset for tables
- A service account with the required roles (see GCP & IAM Requirements)
- `git` and access to the repository

Suggested packages (put in `requirements.txt`):
- python-docx
- pandas
- python-dotenv
- google-cloud-storage
- google-cloud-bigquery
- apache-airflow (or the relevant Google Cloud Composer operators / packages when deployed)
(Adjust versions for your environment; the repo's Makefile expects `requirements.txt` to exist.)

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

Environment variables (.env)
----------------------------
Add a `.env` file or set environment variables in your runtime environment. Variables used across scripts:

- PROJECT_ID — GCP project id
- BUCKET_NAME — Cloud Storage bucket name (for downloads/uploads)
- BUCKET_URI — URI used in DAGs (e.g., gs://my-bucket)
- DATASET_ID — BigQuery dataset id where tables will be created
- TABLE_ID — BigQuery table to load into (used in DAG)
- Additional variables might be required depending on deployment: e.g., AIRFLOW_HOME, COMPOSER instance names, etc.

Example `.env`
PROJECT_ID=my-gcp-project
BUCKET_NAME=my-bucket
BUCKET_URI=gs://my-bucket
DATASET_ID=my_dataset
TABLE_ID=voter_registry

Running the components
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

GCP & IAM requirements
----------------------
Service account for running this pipeline should have:
- Storage: roles/storage.objectViewer (or roles/storage.admin for uploads)
- BigQuery: roles/bigquery.dataEditor (or roles/bigquery.admin for table creation)
- Composer/Cloud Composer: roles/composer.worker (as appropriate)
- If you use separate service accounts for Composer, ensure they have appropriate access to buckets & datasets.

Security & privacy
------------------
This repository deals with personally-identifiable information (PII). Important best practices:
- Never commit production credentials or sample PII to the repo.
- Use least-privilege service accounts and rotate keys.
- Encrypt data at rest in GCS (default GCP SSE enabled) and in BigQuery (customer-managed keys if required).
- Minimize local copies of files; clean up temporary files (`/tmp/cleaned_data.csv`) after use.
- Mask, redact, or pseudonymize personally-identifiable fields when storing or sharing.
- Ensure your project and pipeline comply with relevant laws and policies (FERPA, state laws, etc.) and your organization's data governance.

Development notes, TODOs and improvements
---------------------------------------
- Add a `requirements.txt` with pinned versions.
- Add unit tests for parsing functions (`voter_file_layout_docx_to_csv`, `voter_data_txt_to_csv`) and integration tests for BigQuery load jobs (can be mocked).
- Improve error handling and logging (replace print() with structured logging).
- Consider streaming ingestion for large files rather than downloading whole files.
- Hashing helper currently returns SQL expressions — integrate into a transformation step that writes hashed search columns to BigQuery.
- Add CI (GitHub Actions) to lint and run tests.

Contributing
------------
- Open issues for bugs or feature requests.
- For code changes, open a PR with tests and a clear description.
- Please avoid committing secrets or raw production data.

Contact / Author
----------------
Repository owner: @jessielwhite

```
