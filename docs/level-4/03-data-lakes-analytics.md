# Data Lakes & Analytics (S3, Athena, Glue, Redshift)

Operational databases (RDS, DynamoDB) are optimized for transactional
reads/writes on current data. Analytics over large historical volumes
— "total revenue by region for the last three years" — needs a
different shape: a **data lake** on S3, queried directly, with a
managed warehouse (Redshift) for the heaviest, most structured
workloads.

## The data lake pattern

Raw data lands in S3 in its native format (CSV, JSON, Parquet); a
**catalog** (Glue Data Catalog) describes its schema; query engines
(Athena) read directly from S3 using that catalog — no data movement,
no cluster to manage for ad-hoc queries.

```bash
aws s3 mb s3://training-datalake-raw
aws s3 cp orders-2026-08.parquet s3://training-datalake-raw/orders/year=2026/month=08/
```

Partitioning by folder structure (`year=2026/month=08/`) lets query
engines skip scanning irrelevant partitions — critical for cost, since
Athena bills per byte scanned.

## Glue: crawl and catalog

```bash
aws glue create-database --database-input '{"Name":"training_lake"}'

aws glue create-crawler \
  --name orders-crawler \
  --role arn:aws:iam::123456789012:role/GlueCrawlerRole \
  --database-name training_lake \
  --targets '{"S3Targets":[{"Path":"s3://training-datalake-raw/orders/"}]}'

aws glue start-crawler --name orders-crawler
```

The crawler infers schema and partition structure from the S3 layout
and registers a table in the Glue Data Catalog — Athena, Redshift
Spectrum, and EMR can all query the same catalog entry without
re-defining the schema per engine.

## Query with Athena

```bash
aws athena start-query-execution \
  --query-string "SELECT region, SUM(amount) AS revenue FROM training_lake.orders WHERE year='2026' AND month='08' GROUP BY region" \
  --query-execution-context Database=training_lake \
  --result-configuration OutputLocation=s3://training-athena-results/

aws athena get-query-execution --query-execution-id abc12345-6789-def0-1234-56789abcdef0 \
  --query 'QueryExecution.{Status:Status.State,BytesScanned:Statistics.DataScannedInBytes}'
# { "Status": "SUCCEEDED", "BytesScanned": 41943040 }

aws athena get-query-results --query-execution-id abc12345-6789-def0-1234-56789abcdef0
```

Athena has no persistent infrastructure to provision — you pay per
query based on bytes scanned, which is why partitioning and columnar
formats (Parquet, compressed) matter so much: the same query against
uncompressed CSV can scan 10x the data and cost 10x as much.

## Glue ETL jobs

For transforming raw data (e.g., CSV → partitioned Parquet) rather than
just querying it in place:

```python
# glue_etl_job.py (PySpark, run by Glue)
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from awsglue.context import GlueContext
from pyspark.context import SparkContext

glueContext = GlueContext(SparkContext.getOrCreate())
raw = glueContext.create_dynamic_frame.from_catalog(database="training_lake", table_name="orders_csv")
glueContext.write_dynamic_frame.from_options(
    frame=raw,
    connection_type="s3",
    connection_options={"path": "s3://training-datalake-raw/orders-parquet/", "partitionKeys": ["year", "month"]},
    format="parquet"
)
```

```bash
aws glue create-job \
  --name orders-csv-to-parquet \
  --role arn:aws:iam::123456789012:role/GlueETLRole \
  --command '{"Name":"glueetl","ScriptLocation":"s3://training-scripts/glue_etl_job.py","PythonVersion":"3"}'

aws glue start-job-run --job-name orders-csv-to-parquet
```

## Redshift: when the lake isn't enough

Athena is great for ad-hoc, infrequent, or exploratory queries.
**Redshift** is a provisioned (or serverless) columnar data warehouse
for heavy, frequent, complex queries (joins across billions of rows,
BI dashboards hit constantly) where consistent low latency matters
more than pay-per-query flexibility.

```bash
aws redshift-serverless create-workgroup \
  --workgroup-name training-wg \
  --namespace-name training-ns \
  --base-capacity 8

# Query the lake directly from Redshift without copying data (Redshift Spectrum)
```
```sql
CREATE EXTERNAL SCHEMA training_lake_ext
FROM DATA CATALOG DATABASE 'training_lake'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftSpectrumRole';

SELECT region, SUM(amount) FROM training_lake_ext.orders GROUP BY region;
```

Redshift Spectrum queries the same S3 data and Glue catalog Athena
uses — you don't have to choose one exclusively; use Athena for
exploration and Spectrum/Redshift for recurring, performance-sensitive
workloads on the same underlying lake.

## Gotchas

- **Athena has no schema enforcement at write time** — a malformed row
  in the raw data (wrong type in a column) causes that row to be
  skipped or the query to error, not caught until query time.
- **Partition projection vs. crawler-discovered partitions** — relying
  on the Glue crawler to re-discover new partitions means paying for
  crawler runs and a lag before new data is queryable; partition
  projection (configured on the table) computes partitions
  algorithmically and needs no crawler, but only works for
  predictable, regular partition schemes.
- **Small files kill performance and cost** — thousands of tiny
  Parquet files force Athena/Spectrum to open many objects for little
  data each; Glue ETL jobs should compact output into fewer, larger
  files.
- **IAM roles for Glue/Athena need both S3 and Glue Catalog
  permissions** — a role with S3 read access but no
  `glue:GetTable`/`glue:GetPartitions` fails at the catalog lookup
  step, not the data read.
- **Redshift Spectrum and Athena bill separately** even when querying
  the same S3 data — Spectrum bills through Redshift's per-byte-scanned
  charge, distinct from Athena's.

## Cheat sheet

| Task | Command |
|---|---|
| Catalog a database | `aws glue create-database` |
| Discover schema/partitions | `aws glue create-crawler` + `start-crawler` |
| Ad-hoc SQL over S3 | `aws athena start-query-execution` |
| Transform data at scale | `aws glue create-job` + `start-job-run` |
| Provisioned warehouse | `aws redshift-serverless create-workgroup` |
| Query lake from Redshift | `CREATE EXTERNAL SCHEMA ... FROM DATA CATALOG` |

## Exercise

Upload a small CSV to S3 partitioned by `year=2026/month=08/`, crawl it
into a Glue database, then run an Athena query with `GROUP BY` and
check `BytesScanned` in the query execution result. Convert the same
data to Parquet with a Glue ETL job and compare bytes scanned for an
equivalent query.
