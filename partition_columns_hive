# Partition columns, Hive, and the modern datalake stack

Notes on how **partition columns**, **Hive-style S3 paths**, and **catalog + SQL** fit together — including where **Databricks** sits vs AWS Glue/Athena.

---

## Partition columns (mental model)

A **partition column** is a column in the **same logical table**, but its value also decides **which S3 folder** a row’s data file lives in.

### One table, two views

**Example:** a `page_views` analytics table.

**Logical (SQL / catalog table):**

```text
page_views
├── data columns: event_id, user_id, page_url, session_id, …
└── partition keys: event_date, country, device_type
```

**Physical (S3):**

```text
s3://my-data-lake/page_views/
  event_date=2025-05-27/
    country=US/
      device_type=mobile/
        part-0000.parquet
```

Same table. Partition columns are the bridge: **column value → path segment** (`key=value/`).

### What they are *not*

- **Not** a separate “path table”
- **Not** only in the path — values usually also appear as columns inside Parquet (and in `SELECT`)
- **Not** every column — only those declared as **partition keys** in the table definition

### Partition vs data column

| | Partition column | Data column |
|--|------------------|-------------|
| **Purpose** | Organize files on S3 | Store row content |
| **In S3 path** | Yes — `key=value/` folders | No |
| **In Parquet** | Often yes (duplicated) | Yes |
| **In catalog** | Listed as **partition keys** | Listed as table **columns** |
| **Query effect** | `WHERE country = 'US'` can skip other prefixes | Filters rows inside files |

Example table definition (conceptual):

```sql
CREATE EXTERNAL TABLE page_views (
  event_id   STRING,
  user_id    STRING,
  page_url   STRING,
  session_id STRING
)
PARTITIONED BY (
  event_date  DATE,
  country     STRING,
  device_type STRING
)
LOCATION 's3://my-data-lake/page_views/';
```

Regular columns: `event_id`, `user_id`, `page_url`, …  
Partition columns: `event_date`, `country`, `device_type`.

### Why pick certain columns as partitions?

- **Good:** fields you filter on often, moderate cardinality (`event_date`, `country`)
- **Bad:** very high cardinality (`event_id`, `user_id`) → too many tiny folders, slow queries

---

## Hive-style S3 path segments

**Hive-style** means folder names look like **`column_name=value/`**, even when you are not running Apache Hive.

```text
.../page_views/event_date=2025-05-27/country=US/device_type=mobile/
```

### Why this layout exists

1. **Query pruning** — engines skip whole prefixes when `WHERE` matches partition keys
2. **Catalog alignment** — Glue/Databricks register partition keys that match folders
3. **Shared convention** — Spark, Athena, Databricks, EMR all understand it

Typical write flow:

```text
ETL job defines partition columns
    → writes Parquet under key=value/ folders on S3
    → catalog registers table + partition keys (Glue, Unity Catalog, …)
    → SQL engines query via catalog + prune paths
```

---

## What is Apache Hive?

**Apache Hive** is a **SQL layer on top of large file stores** (originally HDFS, now often S3). You write SQL; Hive plans **distributed batch jobs** that scan files in parallel.

Built at Facebook (~2008) so analysts could query big data without writing MapReduce.

### Core pieces

| Piece | Role |
|-------|------|
| **Hive Metastore** | Catalog: table names, column types, file locations, partition keys |
| **Tables** | Logical views over files (Parquet, ORC, CSV) — not OLTP row storage |
| **Partitions** | `col=value/` folder layout |
| **HiveQL (HQL)** | SQL-like query language |

Hive is **batch analytics**, not a transactional database like Postgres.

### What a query does (conceptually)

```text
SQL query
  → Metastore: where is the table? what are partition keys?
  → Planner: which S3/HDFS prefixes match the WHERE clause?
  → Distributed engine reads matching files in parallel
  → Returns rows / aggregates / joins
```

It is not only “find matched files” — it is **run SQL analytics over matched files, at scale**. Path pruning is step one; distributed read + compute is the rest.

Example:

```sql
SELECT page_url, COUNT(*) AS views
FROM page_views
WHERE event_date = DATE '2025-05-27'
  AND country = 'US'
GROUP BY page_url;
```

→ only reads `event_date=2025-05-27/country=US/...` prefixes, not the whole bucket.

---

## Hive today: runtime vs practices

**What survived (widely adopted):**

- Tables as **metadata over files** (external tables)
- **Hive-style partitions** on object storage
- A **central catalog** (schema + partition keys + locations)
- **SQL** as the analyst-facing interface

**What is often replaced:**

- Hive server as the primary query engine
- MapReduce as the execution backend
- HiveQL as the only SQL dialect in the stack

**Two ways “Hive” shows up now:**

1. **Implicitly** — Glue, Athena, Spark, Databricks use the same partition + catalog model (very common).
2. **Explicitly** — Some teams still run Hive or Spark-with-Hive-metastore on EMR/on-prem.

**Short summary:** Hive pioneered **SQL + catalog + partitioned files on cheap storage**. Modern tools kept the model and swapped in faster engines.

---

## Stack comparison: Hive era vs AWS vs Databricks

| Concern | Classic Hive | AWS lakehouse pattern | Databricks |
|---------|--------------|----------------------|------------|
| **Storage** | HDFS / S3 | **S3** (e.g. `s3://my-data-lake/...`) | Often **same S3** (external) or DB-managed volumes |
| **Catalog** | Hive Metastore | **AWS Glue Data Catalog** | **Unity Catalog** (`catalog.schema.table`) |
| **SQL engine** | Hive / Tez / Spark | **Amazon Athena** (Trino/Presto-class) | **Databricks SQL** / Spark SQL |
| **Partition layout** | `key=value/` | Same Hive-style on S3 | Same convention for external/managed tables |
| **Table model** | External table → files | Glue external table → S3 prefix | External or managed Delta/Parquet |
| **Who writes paths** | ETL / Spark jobs | Pipelines, Lambdas, Spark, custom writers | Spark jobs, ingestion pipelines; can mirror S3 layout |
| **Typical use** | Batch warehouse | Ad hoc SQL, pipelines, shared datalake | ML/analytics notebooks, shared lakehouse, federated reads |

Glue and Unity Catalog both speak **Hive metastore–compatible** ideas (databases, tables, partitions), but they are different products with different governance and APIs.

---

## End-to-end example (generic pipeline)

### Write path

```text
Batch job reads raw events
    → groups by event_date, country, device_type
    → writes Parquet:
        s3://my-data-lake/page_views/event_date=2025-05-27/country=US/device_type=mobile/part-0000.parquet
    → registers partition in catalog (Glue crawler, MSCK REPAIR, or explicit API call)
```

### Read path (two front-ends, same files)

**Via Athena:**

```sql
SELECT * FROM analytics_db.page_views
WHERE event_date = DATE '2025-05-27' AND country = 'DE';
```

**Via Databricks (Unity Catalog):**

```sql
SELECT * FROM prod.analytics.page_views
WHERE event_date = DATE '2025-05-27' AND country = 'DE';
```

Same partition columns and S3 layout; different catalog namespace and SQL service.

### Reference vs payload (optional pattern)

Some lakehouse designs split **index rows** (metadata in Parquet) from **large blobs** (images, JSON dumps):

| Kind | Partition columns | Row points to |
|------|-------------------|---------------|
| Index table | `event_date`, `country` | `s3_uri` column → blob location |
| Artifact store | same or finer keys | the blob itself |

Partition pruning still applies to both; the index table is what SQL usually scans first.

---

## Glossary (quick)

| Term | Meaning |
|------|---------|
| **Partition column** | Column whose value appears in the S3 path *and* (usually) in the file |
| **Hive-style path** | `partition_col=value/` folder segments |
| **Metastore / catalog** | Registry: table location, columns, partition keys |
| **External table** | Catalog entry pointing at files on S3; data not “inside” the catalog |
| **Partition pruning** | Skip reading S3 prefixes that cannot match the query filter |

---

## Takeaways

1. Partition columns are **the same table** — not a separate path table — but they **define S3 layout** and enable **query pruning**.
2. That layout follows **Hive partitioning practice**, adopted by Glue, Athena, Spark, and Databricks.
3. **Hive the project** is often not what you run day to day; **Hive the pattern** (catalog + partitioned files + SQL) is what you use.
4. Choosing partition columns is a design tradeoff: **filter performance** vs **folder/file count** on object storage.
