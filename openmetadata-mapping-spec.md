# OpenMetadata Data Flow Mapping Specification

## Overview

This document defines the data flows between OpenMetadata components — what each module consumes as input and produces as output.

---

## Component Data Flow Map

### 1. External Data Sources → Ingestion Connectors

| Source | Protocol/Method | Data Produced |
|--------|----------------|---------------|
| Snowflake | SQLAlchemy + native Snowflake connector | Table schemas, column types, query logs, tag bindings |
| dbt | dbt manifest.json / catalog.json / run_results.json | Model definitions, column lineage, test results, descriptions |
| BigQuery | GCP SDK | Datasets, tables, partitioning info, query history |
| PostgreSQL / MySQL | JDBC / SQLAlchemy | Tables, views, stored procedures, FK relationships |
| Tableau / Looker | REST API | Dashboards, charts, data sources, lineage |
| Kafka / Pulsar | Schema Registry API | Topics, schemas, sample messages |
| S3 / GCS / ADLS | Cloud SDK | Buckets, containers, file listings |
| Airflow | Airflow REST API | DAGs, tasks, task runs, lineage |

---

### 2. Ingestion Framework Internal Pipeline

```
Source → [topology_runner.py] → Processor → Sink → ometa/ Python Client → OpenMetadata API
```

| Step | Consumes | Produces |
|------|----------|---------|
| **Source** (e.g., `snowflake/metadata.py`) | Connection config (YAML/JSON), DB credentials | Raw entity records (TableMetadata, Column, Schema) |
| **Processor** | Raw entity records | Enriched/transformed entity objects matching OpenMetadata spec |
| **Sink** (MetadataRestSink) | Transformed entity objects | HTTP PUT/POST/PATCH requests to `/api/v1/*` |
| **ometa/ Client** | API request models | Validated API responses; handles retries + pagination |
| **Status tracker** (`status.py`) | Step results (success/failure counts) | Pipeline run status stored back via API |

#### Snowflake-specific flows

| Flow | Input | Output |
|------|-------|--------|
| `metadata.py` | Snowflake connection, db/schema filters | Tables, columns, primary/foreign keys, tags |
| `lineage.py` | Snowflake ACCESS_HISTORY / query logs | Column-level lineage edges |
| `usage.py` | QUERY_HISTORY table | Usage stats per table (daily/weekly counts) |
| `query_parser.py` | Raw SQL strings | Parsed lineage graph (source → target columns) |
| `data_diff/` | Two table snapshots | Diff results for data quality |

#### dbt-specific flows

| Flow | Input | Output |
|------|-------|--------|
| `metadata.py` | manifest.json, catalog.json | dbt models mapped to OpenMetadata tables |
| `dbt_service.py` | dbt Cloud API / local artifacts | Raw dbt project metadata |
| `dbt_utils.py` | Manifest nodes | Tag mappings, owner mappings, test definitions |
| Test results | run_results.json | TestCase + TestSuite entities pushed to API |
| Column lineage | manifest `node.depends_on` | Column-level lineage edges between models |

---

### 3. OpenMetadata Service (API Layer)

| Resource Endpoint | Consumes | Produces |
|------------------|----------|---------|
| `POST /api/v1/tables` | CreateTable JSON (schema-validated) | Stored table entity, change event |
| `GET /api/v1/lineage/table/{id}` | Entity ID, query params (depth, direction) | Lineage graph (nodes + edges) |
| `PUT /api/v1/tables/{id}/followers` | User entity reference | Updated follower list |
| `POST /api/v1/testSuites` | TestSuite definition | TestSuite entity, linked test cases |
| `POST /api/v1/dataQuality/testCases` | TestCase definition | TestCase entity, schedule |
| `GET /api/v1/search/query` | Lucene query string, filters | Paginated search hits from Elasticsearch |
| `POST /api/v1/events/subscriptions` | Alert/subscription config | Event subscription, webhook registration |
| `GET /api/v1/lineage/export` | Entity filter params | Lineage graph in OpenLineage format |

#### Event Flow

```
Entity Create/Update/Delete
  → ChangeEvent written to MySQL (change_event table)
  → EventPublisher picks up event
  → Fan-out to:
      • Elasticsearch (index update)
      • Kafka (if configured)
      • WebSocket (real-time UI push)
      • Webhook subscribers (HTTP POST)
      • Governance rules engine
```

---

### 4. Metadata Store (MySQL / PostgreSQL)

| Table | Consumes | Produces |
|-------|----------|---------|
| `table_entity` | JSON document (entity body) | Table rows queryable by fqnHash / nameHash |
| `entity_relationship` | fromId, toId, relation type | Graph edges for lineage, ownership, tags |
| `change_event` | Entity diffs | Audit log, feed updates, event replay |
| `entity_extension` | Entity ID + extension JSON | Custom properties, data profiles |
| `entity_extension_time_series` | Time-stamped metric data | Historical profiling, quality trends |
| `ingestion_pipeline_entity` | Pipeline config + status | Pipeline run history |

---

### 5. Elasticsearch Index

| Index | Consumes | Produces |
|-------|----------|---------|
| Entity indexes (per type) | Entity JSON from MySQL sync | Full-text search results |
| Search index | Field-level text, tags, owners | Faceted search with aggregations |
| Suggest index | Entity names, FQNs | Autocomplete suggestions |

---

### 6. UI Layer

| UI Action | Consumes (API calls) | Produces (user action result) |
|-----------|---------------------|------------------------------|
| Browse catalog | `GET /api/v1/tables?fields=...` | Rendered entity list |
| View lineage | `GET /api/v1/lineage/table/{id}` | Interactive DAG visualization |
| Edit description | `PATCH /api/v1/tables/{id}` | Updated entity, change event |
| Search | `GET /api/v1/search/query?q=...` | Search results page |
| View quality | `GET /api/v1/dataQuality/testCases` | Quality dashboard |
| Configure pipeline | `POST /api/v1/ingestionPipelines` | Deployed Airflow DAG |

---

### 7. Airflow APIs Plugin

| Trigger | Consumes | Produces |
|---------|----------|---------|
| `POST /rest_api/api?api=deploy_dag` | Workflow JSON from OpenMetadata | Airflow DAG file on disk, DAG registration |
| Scheduled DAG run | DAG parameters + OpenMetadata connection config | Ingestion pipeline execution → metadata pushed to API |
| `DELETE ?api=delete_dag` | dag_id | DAG removed from Airflow scheduler |

---

### 8. Specification Layer → All Consumers

| Artifact | Consumes | Produces |
|----------|----------|---------|
| `openmetadata-spec` JSON schemas | Entity field definitions | Validated API request/response models (Java + Python codegen) |
| `metadataIngestion/workflow.json` | Workflow YAML/JSON | Parsed pipeline configuration object |
| `entity/services/connections/database/snowflake.json` | Snowflake connection fields | Type-safe connection config used by connector |
