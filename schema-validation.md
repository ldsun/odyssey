# Schema validation — Pandera & Pydantic

How to validate **tabular batches** vs **structured records** before writing to a datalake or calling an API.

Related (project-specific): [partition columns & Hive](./Odysse_partition_columns_hive.md) — catalog vs file layout

---

## Mental model

| Library | Validates | Typical use |
|---------|-----------|-------------|
| **Pandera** | **pandas DataFrames** (many rows, columnar) | Index tables, metrics batches, homogeneous Parquet |
| **Pydantic** | **Python objects / dicts** (one record or nested JSON) | API payloads, config, nested attributes |

Both can gate data **before upload**. Neither replaces a **Glue / Databricks catalog** — catalog metadata for SQL is a separate layer.

```text
Pipeline builds DataFrame or model instances
        │
        ▼
  validate (pandera or pydantic)
        │
        ▼
  write to storage / API
```

---

## Pandera — DataFrame validation

**Use when:** you have a **batch of rows** in a DataFrame (ETL output, index upload, artifact Parquet).

### Skeleton: table schema class pattern

Many datalake codebases wrap pandera in a schema class (name, version, partitions + `SCHEMA`):

```python
import numpy as np
import pandera as pa

class PageViewIndexSchema:
    NAME = "page_views"
    VERSION = "1"
    PARTITIONS = ["event_date", "country"]

    SCHEMA: pa.DataFrameSchema = pa.DataFrameSchema(
        {
            "event_id": pa.Column(str, nullable=False),
            "user_id": pa.Column(str, nullable=False),
            "page_url": pa.Column(str, nullable=False),
            "event_date": pa.Column(str, nullable=False),   # partition col
            "country": pa.Column(str, nullable=False),
        },
        strict="filter",   # drop extra columns not in schema
        coerce=True,       # try to cast dtypes
    )

    def validate(self, df: pd.DataFrame) -> pd.DataFrame:
        return self.SCHEMA.validate(df, lazy=True)  # lazy → all errors at once
```

### Validate before write

```python
schema = PageViewIndexSchema()
df = schema.validate(raw_df)

# then upload df to S3 / register partition
write_parquet(df, partition_keys={"event_date": "2025-05-27", "country": "US"})
```

### Skeleton: custom DataFrame-level checks

When rules depend on **multiple columns** or **row subsets**:

```python
def check_premium_users_have_tier(df: pd.DataFrame) -> bool:
    mask = df["plan"] == "premium"
    if not mask.any():
        return True
    return df.loc[mask, "tier"].notna().all()

SCHEMA = pa.DataFrameSchema(
    {
        "user_id": pa.Column(str, nullable=False),
        "plan": pa.Column(str, checks=pa.Check.isin(["free", "premium"])),
        "tier": pa.Column(str, nullable=True),
        "amount": pa.Column(float, checks=pa.Check.ge(0.0)),
    },
    checks=[pa.Check(check_premium_users_have_tier)],
)
```

### Pandera building blocks

| Piece | Role |
|-------|------|
| `pa.Column(dtype, nullable=, checks=)` | One column’s type and constraints |
| `pa.Check.isin([...])` | Enum-like allowed values |
| `pa.Check.ge / le` | Numeric bounds |
| `pa.Check(callable)` | Custom row or DataFrame logic |
| `strict="filter"` | Drop columns not listed |
| `coerce=True` | Cast compatible types automatically |

---

## Pydantic — object validation

**Use when:** data is **structured records** (one event, one config object, nested JSON).

### Skeleton: pipeline / job config

```python
from pydantic import BaseModel, Field
from enum import Enum

class Environment(str, Enum):
    DEV = "dev"
    PROD = "prod"

class JobConfig(BaseModel):
    job_id: str
    start_time: int = Field(..., description="Epoch seconds")
    end_time: int = Field(..., gt=0)
    input_paths: list[str]
    environment: Environment = Environment.DEV

config = JobConfig(
    job_id="batch-001",
    start_time=1710000000,
    end_time=1710003600,
    input_paths=["s3://bucket/raw/"],
)
# raises ValidationError if invalid
```

### Skeleton: nested annotation / API record

```python
from pydantic import BaseModel, Field
from typing import Literal

class BaseAttributes(BaseModel):
    timestamp_ms: int = Field(ge=0)
    category: str
    label: str

class ProductAttributes(BaseModel):
    product_id: str
    price_usd: float = Field(ge=0)

class Detection2D(BaseAttributes):
    attributes: ProductAttributes | PersonAttributes  # union / discriminated types
    occlusion_pct: Literal[0, 25, 50, 75, 100]

obj = Detection2D.model_validate(record_dict)
json_str = obj.model_dump_json()
restored = Detection2D.model_validate_json(json_str)
```

Pydantic excels at **nested JSON**, **unions**, **`model_validator`**, and **field constraints** (`ge`, `Literal`, regex, etc.).

---

## Hybrid: Pydantic rows → DataFrame

When each **row is a rich object** but you still upload a **DataFrame**, use a mixin pattern:

```python
class PandasDataFrameMixin:
    @classmethod
    def models_validate_pandas_dataframe(cls, df: pd.DataFrame) -> list["Detection2D"]:
        return [cls.model_validate(row._asdict()) for row in df.itertuples()]

    @classmethod
    def models_dump_pandas_dataframe(cls, objs: list["Detection2D"]) -> pd.DataFrame:
        return pd.DataFrame([o.model_dump() for o in objs])

class Detection2D(BaseAttributes, PandasDataFrameMixin):
    ...

# Validate batch: each row → Pydantic → back to flat DataFrame
models = Detection2D.models_validate_pandas_dataframe(raw_df)
out_df = Detection2D.models_dump_pandas_dataframe(models)
```

A unified `validate(df)` on the schema class can branch:

```python
def validate(self, df: pd.DataFrame) -> pd.DataFrame:
    if isinstance(self.SCHEMA, pa.DataFrameSchema):
        return self.SCHEMA.validate(df, lazy=True)
    # Pydantic row model class
    return self.SCHEMA.models_dump_pandas_dataframe(
        objs=self.SCHEMA.models_validate_pandas_dataframe(df=df)
    )
```

Use **pandera only** when columns are flat and homogeneous; use **Pydantic + mixin** when rows have nested `attributes` blobs.

---

## Which to use?

| Situation | Prefer |
|-----------|--------|
| Index / reference table (many flat columns) | **Pandera** `DataFrameSchema` |
| Homogeneous Parquet (metrics, points, logs) | **Pandera** (+ custom `pa.Check`) |
| Nested JSON with unions / per-type fields | **Pydantic** |
| YAML / CLI / service config | **Pydantic** `BaseModel` |
| Single dict from REST API | **Pydantic** `model_validate` |
| Nested records uploaded as DataFrame batch | **Pydantic** + DataFrame mixin |

**Rule of thumb:** DataFrame in, DataFrame out → **Pandera**. Rich nested record → **Pydantic**. Nested record **batched as DataFrame** → Pydantic + mixin.

---

## Where validation runs (typical pipeline)

| Stage | What validates |
|-------|----------------|
| Before write | `schema.validate(df)` in upload path |
| After read (optional) | Re-validate loaded DataFrame for drift detection |
| Ingest API / managed lake | Server-side schema check (independent of client pandera) |
| Glue / Athena | Catalog types at query time — not Python runtime validation |

Client-side pandera/pydantic does **not** automatically update Glue DDL — catalog and Python validation are separate concerns.

---

## Side-by-side skeleton (same logical row)

**Pandera** — row as part of a DataFrame:

```python
df = pd.DataFrame([{
    "event_id": "evt-1",
    "user_id": "u-42",
    "page_url": "/home",
    "event_date": "2025-05-27",
    "country": "US",
}])
PageViewIndexSchema().validate(df)
```

**Pydantic** — same facts as one object:

```python
class PageViewRow(BaseModel):
    event_id: str
    user_id: str
    page_url: str
    event_date: str
    country: str

row = PageViewRow.model_validate({...})
```

Same data, different validation style: **columnar batch** vs **record model**.

---

## Takeaways

1. **Pandera** = column contract for **DataFrames**.
2. **Pydantic** = **objects** (config, nested JSON, API).
3. A schema wrapper can **unify both** behind one `validate(df)` before write.
4. Python validation ≠ **catalog schema** for SQL engines.
