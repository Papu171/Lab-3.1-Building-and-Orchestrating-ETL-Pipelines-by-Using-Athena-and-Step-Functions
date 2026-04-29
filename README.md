# Lab-3.1-Building-and-Orchestrating-ETL-Pipelines-by-Using-Athena-and-Step-Functions

Emiliano Corona
**Semester:** 6th Semester  

---

## 📋 Description

This project implements a fully automated ETL (Extract, Transform, Load) pipeline using AWS Step Functions to orchestrate queries in Amazon Athena over NYC Taxi data stored in Amazon S3, using an AWS Glue Data Catalog as the metadata store.

The pipeline handles:
- First-run scenarios (creating databases, tables, views from scratch)
- Subsequent runs (inserting new monthly data into existing Parquet tables)

---

## 🏗️ Architecture

```
Amazon S3 (Raw CSV Data)
        ↓
AWS Glue Data Catalog (Schema/Metadata)
        ↓
Amazon Athena (SQL Queries)
        ↑
AWS Step Functions (Orchestration Workflow)
```

### Workflow Logic
```
Create Glue DB
→ Run Table Lookup
→ Get Lookup Query Results
→ Choice: Tables exist?
   ├── NO → Create CSV Tables → Create Parquet Tables → Create View → END
   └── YES → Check All Tables (Map)
                ├── yellowtaxi_data_parquet → Insert February Data → END
                └── Other tables → Ignore → END
```

---

## 🗂️ Repository Structure

```
project/
├── README.md
├── src/
│   └── workflow/
│       └── step_functions_definition.json   # Complete Step Functions state machine
├── sql/
│   ├── create_database.sql
│   ├── create_yellowtaxi_csv.sql
│   ├── create_lookup_csv.sql
│   ├── create_lookup_parquet.sql
│   ├── create_yellowtaxi_parquet.sql
│   ├── create_view.sql
│   └── insert_february.sql
├── screenshots/
│   └── [lab screenshots]
├── docs/
│   └── lab_report.pdf
├── data/
│   ├── sample/
│   └── .gitkeep
├── .gitignore
└── requirements.txt
```

---

## ⚙️ AWS Services Used

| Service | Purpose |
|---|---|
| **AWS Step Functions** | Workflow orchestration and ETL logic |
| **Amazon Athena** | Serverless SQL query execution |
| **AWS Glue Data Catalog** | Metadata store for tables and schemas |
| **Amazon S3** | Raw data storage and query results |
| **AWS IAM** | Role-based access control (`StepLabRole`) |
| **AWS Cloud9** | IDE for data loading commands |

---

## 📊 Dataset

**NYC Yellow Taxi Trip Data — Early 2020**

| File | Description | Size |
|---|---|---|
| `yellow_tripdata_2020-01.csv` | January 2020 taxi trips | ~500 MB |
| `yellow_tripdata_2020-02.csv` | February 2020 taxi trips | ~500 MB |
| `taxi_zone_lookup.csv` | Location ID to Borough/Zone mapping | Small |

**Source:** [NYC Taxi & Limousine Commission](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)

---

## 🗄️ Tables Created

| Table Name | Format | Description |
|---|---|---|
| `yellowtaxi_data_csv` | CSV (External) | Raw January taxi data |
| `nyctaxi_lookup_csv` | CSV (External) | Raw location lookup data |
| `nyctaxi_lookup_parquet` | Parquet + Snappy | Optimized lookup table |
| `yellowtaxi_data_parquet` | Parquet + Snappy + Partitioned | Optimized taxi data partitioned by `pickup_year` and `pickup_month` |
| `yellowtaxi_data_vw` | View | Joined view combining taxi data and location lookup |

---

## 🚀 How to Run

### Prerequisites
- AWS Academy account with access to the lab environment
- IAM Role `StepLabRole` with permissions for Athena, S3, Glue, and Lake Formation
- S3 bucket with `gluelab` in the name

### Step 1 — Load source data (Cloud9)
```bash
# Set your bucket name
mybucket="your-gluelab-bucket-name"

# Load January taxi data
wget -qO- https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDENG-1/step-lab/yellow_tripdata_2020-01.csv | aws s3 cp - "s3://$mybucket/nyctaxidata/data/yellow_tripdata_2020-01.csv"

# Load February taxi data
wget -qO- https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDENG-1/step-lab/yellow_tripdata_2020-02.csv | aws s3 cp - "s3://$mybucket/nyctaxidata/data/yellow_tripdata_2020-02.csv"

# Load location lookup table
wget -qO- https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACDENG-1/step-lab/taxi+_zone_lookup.csv | aws s3 cp - "s3://$mybucket/nyctaxidata/lookup/taxi _zone_lookup.csv"
```

### Step 2 — Deploy the Step Functions workflow
1. Go to **AWS Step Functions** → Create state machine
2. Name it `WorkflowPOC`
3. Use the definition in `src/workflow/step_functions_definition.json`
4. Assign execution role `StepLabRole`

### Step 3 — Run the pipeline
```
Execution name: TaskTwelveTest
```
The workflow will automatically:
1. Create the Glue database if it doesn't exist
2. Detect whether tables exist
3. Create all tables and the view on first run
4. Insert new monthly data on subsequent runs

### Step 4 — Query the results in Athena
```sql
-- Preview the combined view
SELECT * FROM "nyctaxidb"."yellowtaxi_data_vw" LIMIT 10;

-- Query February data specifically
SELECT * FROM "nyctaxidb"."yellowtaxi_data_vw" WHERE pickup_month = '02';

-- Top pickup locations by fare
SELECT zone, borough, SUM(sum_fare) AS total_fare
FROM "nyctaxidb"."yellowtaxi_data_vw"
GROUP BY zone, borough
ORDER BY total_fare DESC
LIMIT 10;
```

---

## 📈 Key Concepts Applied

- **CTAS (Create Table As Select):** Used to transform CSV data into optimized Parquet format
- **Parquet + Snappy compression:** Reduces storage costs and improves query performance
- **Table partitioning:** Data partitioned by `pickup_year` and `pickup_month` to limit scanned data per query
- **Step Functions Choice state:** Implements conditional routing based on whether tables exist
- **Step Functions Map state:** Iterates over all Glue tables to selectively insert new data

---

## 💰 Estimated AWS Costs

| Service | Usage | Estimated Cost |
|---|---|---|
| Amazon Athena | ~10 queries × ~500 MB scanned | ~$0.025 |
| Amazon S3 | ~1 GB stored + requests | ~$0.023 |
| AWS Glue | Metadata catalog | ~$0.00 (free tier) |
| Step Functions | ~10 state transitions | ~$0.00 (free tier) |
| **TOTAL** | | **~$0.05** |

> Note: Costs are minimal due to Parquet compression reducing scanned data and AWS free tier coverage for Glue and Step Functions.

---

## 🔧 Troubleshooting

| Error | Cause | Solution |
|---|---|---|
| `<TU-BUCKET>` in query | Placeholder not replaced | Replace all `<TU-BUCKET>` with actual bucket name |
| `QueryExecutionId` not found | Missing `.$` suffix | Use `"QueryExecutionId.$"` in Parameters |
| Parquet table creation fails | Old S3 data exists | Delete `optimized-data-lookup/` prefix in S3 before re-running |
| Choice state takes wrong path | Tables still exist in Glue | Delete all Glue tables before testing first-run path |

---

## 📚 References

[1] Amazon Web Services, "AWS Step Functions Developer Guide," AWS Docs. [Online]. Available: https://docs.aws.amazon.com/step-functions/ [Accessed: Apr-2026].

[2] Amazon Web Services, "Amazon Athena User Guide," AWS Docs. [Online]. Available: https://docs.aws.amazon.com/athena/ [Accessed: Apr-2026].

[3] Amazon Web Services, "AWS Glue Developer Guide," AWS Docs. [Online]. Available: https://docs.aws.amazon.com/glue/ [Accessed: Apr-2026].

[4] Amazon Web Services, "Using CTAS and INSERT INTO for ETL and Data Analysis," AWS Docs. [Online]. Available: https://docs.aws.amazon.com/athena/latest/ug/ctas-insert-into-etl.html [Accessed: Apr-2026].

[5] NYC Taxi & Limousine Commission, "TLC Trip Record Data," NYC.gov. [Online]. Available: https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page [Accessed: Apr-2026].

---

## 👨‍💻 Author

**Emiliano**  
Universidad Autónoma de Guadalajara (UAG)  
6th Semester — Analysis and Design of Systems with Big Data
