---
title: "airflow: exporting data from appmetrica using apache airflow"
date: 2025-08-06T00:00:00+08:00
---

Here's an article in English explaining how to export data from AppMetrica using Apache Airflow and the provided code:

---

# Exporting Data from AppMetrica to ClickHouse Using Apache Airflow

**AppMetrica** is a powerful analytics platform from Yandex used to monitor mobile app usage and user behavior. When managing large-scale data pipelines, automation and scalability become critical. In this article, we’ll walk through how to export data from AppMetrica and ingest it into **ClickHouse** using **Apache Airflow**.

We’ll use a production-ready Airflow DAG that handles data extraction, transformation, and loading (ETL) in a modular, scalable way.

---

## Overview of the Workflow

This Airflow pipeline:

1. **Fetches tabular report data from AppMetrica via HTTP API.**
2. **Validates that data is ready using metadata from HTTP headers.**
3. **Loads data into a staging (temporary) ClickHouse table.**
4. **Transfers processed data into the final data warehouse (DWH) table.**
5. **Uses task groups and sensors for dependency control and efficient parallelization.**

---

## Key Components

### 1. **DAG Definition and Configuration**

```python
dag = DAG(
    dag_id="appmetrica",
    start_date=datetime(2022, 2, 1, 1, 0, 0),
    schedule_interval="0 */1 * * *",
    max_active_runs=1,
    default_args={
        "on_failure_callback": task_fail_slack_alert,
        "pool": "appmetrica",
        "retries": 2,
    },
)
```

This DAG runs **hourly**, handling failures gracefully and using a dedicated pool to avoid overloading external systems.

---

### 2. **Table Creation**

Two `ClickHouseOperator` tasks are used for each resource:

* `create_table` – creates a **temporary table** for staging.
* `create_table_dwh` – creates a **partitioned DWH table** for long-term storage.

Table schemas are dynamically generated using field mappings stored in Airflow variables.

---

### 3. **Data Fetching and Validation**

```python
SimpleHttpOperator(...) → HttpSensor(...) → ShortCircuitOperator(...)
```

* The **HTTP operator** triggers report generation.
* A **sensor** waits until the report is ready, checking for the presence of the `Rows-Number` header.
* A **short-circuit task** verifies that the number of rows is non-zero before proceeding.

This ensures we only attempt downloads when real data is available.

---

### 4. **Downloading and Loading Data**

```python
def load_data(url, api_key, table, **kwargs):
    ...
    for chunk in pd.read_csv(tmp.name, ...):
        insert_to_ch(chunk.fillna(""), table)
```

The function uses `requests` to stream the CSV report from AppMetrica, and `pandas` to load it in chunks into ClickHouse using the `ClickHouseHook`.

The function `insert_to_ch` takes care of the actual insertion using dictionary-style records.

---

### 5. **Data Consolidation**

After the temporary table is populated, data is moved to the DWH:

```python
ClickHouseOperator(
    task_id="insert_into_table",
    sql="INSERT INTO dwh.appmetrica_<table> SELECT ... FROM tmp_<table>"
)
```

This pattern supports efficient time-based partitioning and isolation of staging and production data flows.

---

### 6. **Dynamic Task Generation and Dependency Control**

The DAG iterates over a dictionary of resources (`params_dag["resources"]`) and creates task groups dynamically for each table.

```python
for table in params_dag["resources"].keys():
    with TaskGroup(...) as task_group:
        ...
```

Sequential dependency between task groups is enforced using `ExternalTaskSensor` and `DummyOperator`.

This setup ensures that only one group proceeds at a time, useful when AppMetrica rate limits or processing conflicts are a concern.

---

## Benefits of This Approach

* ✅ **Modular**: Each table has its own isolated logic and data flow.
* ✅ **Scalable**: Easily supports many parallel resources with `TaskGroup`.
* ✅ **Robust**: Uses retries, sensors, and validation logic for fault-tolerance.
* ✅ **Maintainable**: Configuration is separated using Airflow Variables and macros.

---

## When to Use

This pipeline is ideal for teams that:

* Need **hourly snapshots** of mobile analytics data.
* Want to **centralize reporting in ClickHouse**.
* Need to maintain **data quality** with temporary staging before merging.

---

## Conclusion

This DAG is a powerful, production-ready solution for automated data extraction from AppMetrica into ClickHouse. By leveraging Apache Airflow’s modularity and monitoring features, it ensures reliable and scalable data processing pipelines tailored to your analytics needs.

---

Let me know if you'd like a shorter version, a diagram, or a section rewritten in a less technical style.
