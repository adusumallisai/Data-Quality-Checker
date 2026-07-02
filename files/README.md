# Data Quality Checker

A Python tool that validates and cleans tabular datasets by detecting **missing
values, duplicate records, and inconsistent data entry**, then generates a
structured summary report. Includes both a Pandas-based checker and a
SQL-based version of the same checks against a SQLite database.

## Why

Raw datasets pulled from forms, exports, or multiple source systems are
rarely clean — the same city gets typed three different ways, dates arrive
in two formats, and rows get duplicated on import. This tool automates the
first pass of a data-cleaning workflow so issues are caught and quantified
before they reach analytics or reporting.

## Features

- **Missing value detection** — count and % missing per column (including
  blank strings, not just NaN)
- **Duplicate detection** — full-row duplicates and duplicate keys (e.g.
  repeated `order_id`)
- **Inconsistent text formatting** — flags values that only differ by
  casing or whitespace (`"Bangalore"` vs `"bangalore "` vs `" BANGALORE"`)
- **Inconsistent date formats** — detects when a single column mixes
  formats like `YYYY-MM-DD` and `DD/MM/YYYY`
- **Range / validity rules** — flags out-of-range numeric values (e.g.
  negative quantities, unrealistic outliers)
- **Summary report** — printable table, exportable to Markdown or CSV
- **Automated cleaning** — produces a de-duplicated, normalized copy of
  the dataset
- **SQL equivalent** (`sql_checks.py`) — the same checks expressed as SQL
  queries against SQLite, for when data lives in a database rather than
  in memory

## Project structure

```
data_quality_checker/
├── data_quality_checker.py    # core DataQualityChecker class (Pandas)
├── sql_checks.py               # equivalent checks written in SQL (SQLite)
├── generate_sample_data.py     # creates a deliberately messy sample dataset
├── run_example.py              # end-to-end demo: load → check → report → clean
├── requirements.txt
├── sample_data/                # generated sample CSV / SQLite DB (gitignored)
└── reports/                    # generated reports and cleaned output (gitignored)
```

## Setup

```bash
pip install -r requirements.txt
```

## Usage

### 1. Generate a sample messy dataset (optional — try it on your own CSV instead)

```bash
python generate_sample_data.py
```

### 2. Run the full check + report + clean pipeline

```bash
python run_example.py
```

This prints a report like:

```
Data Quality Report: customer_orders.csv  (523 rows total)
| Check                        | Column        | Issue Count | % of Rows | Detail                                             |
|-------------------------------|---------------|-------------|-----------|-----------------------------------------------------|
| Missing Values                | customer_name | 24          | 4.59      |                                                       |
| Missing Values                | city          | 17          | 3.25      |                                                       |
| Duplicate Rows                | (all columns) | 15          | 2.87      | Exact duplicate rows                                 |
| Duplicate Key                 | order_id      | 23          | 4.40      | 23 distinct order_id values repeated                 |
| Inconsistent Text Formatting  | city          | 185         | 35.37     | 5 distinct values collapse once trimmed/lower-cased  |
| Inconsistent Date Format      | order_date    | 327         | 62.52     | Formats found: %d-%b-%Y, %d/%m/%Y, %Y-%m-%d          |
| Out-of-Range Value            | quantity      | 45          | 8.60      | Expected range [1, 100]                              |
```

and writes:
- `reports/summary_report.md` — the report above
- `reports/cleaned_orders.csv` — duplicates removed, text normalized, rows
  missing required fields dropped

### 3. Use it on your own data

```python
from data_quality_checker import DataQualityChecker, RangeRule

dqc = DataQualityChecker.from_csv("your_file.csv")
dqc.run_all_checks(
    key_column="id",
    text_columns=["city", "status"],
    date_column="created_at",
    range_rules=[RangeRule(column="amount", min_value=0, max_value=100000)],
)
print(dqc.report())
dqc.export_report("reports/summary.md")

cleaned = dqc.clean(key_column="id", text_columns=["city", "status"])
cleaned.to_csv("cleaned.csv", index=False)
```

### 4. Run the SQL version

```bash
python sql_checks.py
```

Loads the sample CSV into a local SQLite database and runs the same
checks as raw SQL (`GROUP BY ... HAVING COUNT(*) > 1` for duplicates,
`WHERE quantity <= 0` for invalid ranges, etc.) — useful when the dataset
is too large to load fully into memory or already lives in a database.

## Tech stack

- **Python 3.10+**
- **Pandas** — vectorized data validation and cleaning
- **SQLite / SQL** — equivalent checks expressed as queries
- **tabulate** — formatted console reports

## Possible extensions

- Add support for MySQL/PostgreSQL via SQLAlchemy connection strings
- Add a `--config.yaml` so checks can be defined declaratively per dataset
- Schedule as an Airflow/cron job with email/Slack alerts on threshold breach
- Add a Streamlit dashboard for visual exploration of report results
