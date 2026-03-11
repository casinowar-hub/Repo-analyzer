# OpenMetadata Component Summary

Version: **1.12.0-SNAPSHOT** | Java 21 | Python 3.x | React/TypeScript

---

## 1. `openmetadata-spec`

**What it does:** Defines the canonical JSON schemas for all OpenMetadata entities, services, and configuration. It is the contract between all layers.

**Key contents:**
- 26+ data entity schemas: `table.json`, `database.json`, `pipeline.json`, `dashboard.json`, `topic.json`, `mlmodel.json`, `glossary.json`, `container.json`, `query.json`, `dataContract.json`, etc.
- 50+ database connection schemas (Snowflake, BigQuery, Postgres, MySQL, Oracle, Redshift, etc.)
- Service schemas: database, dashboard, pipeline, storage, search, messaging, metadata
- Configuration schemas: authentication, authorization, JWT, Kafka, Fernet, limits
- Ingestion workflow schema: `metadataIngestion/workflow.json`
- AI/LLM entity schemas, governance workflow schemas, SCIM schemas

**Dependencies:** None (foundational module)

**Entry point:** `src/main/resources/json/schema/` — all `.json` schema files

---

## 2. `openmetadata-service`

**What it does:** The core Java backend. Exposes the REST API, manages all metadata entities, handles authentication/authorization, search indexing, event publishing, and governance rule execution.

**Key contents:**
- **Entry point:** `OpenMetadataApplication.java` — Dropwizard 5 Application class
- **REST Resources:** 53+ resource classes under `resources/` (one per entity type: tables, databases, pipelines, dashboards, lineage, users, teams, glossary, policies, etc.)
- **Repository layer:** 109 JDBI3 repository classes under `jdbi3/` with `CollectionDAO` as the aggregation point
- **Auth:** JWT filter, OIDC/SAML/LDAP integration, RBAC policy enforcement
- **Search:** Elasticsearch/OpenSearch integration for full-text entity search
- **Events:** Change event capture, Kafka publisher, WebSocket broadcaster, webhook subscriptions
- **Governance:** Policy engine, automation rules, workflow execution
- **Data Insights:** Analytics and dashboard data
- **SCIM:** User provisioning protocol support
- **OpenLineage:** Integration with OpenLineage standard
- **Apps framework:** Pluggable native applications

**Dependencies:** `openmetadata-spec`, `openmetadata-sdk`, MySQL/Postgres, Elasticsearch

**Entry point:** `OpenMetadataApplication.java` → registers all resources and starts Jetty on port 8585

---

## 3. `ingestion` (Python)

**What it does:** The Python-based ETL/ingestion framework. Pulls metadata from external data sources and pushes it to the OpenMetadata API. Implements 50+ source connectors.

**Key contents:**
- **Workflow engine:** `api/topology_runner.py` orchestrates Source → Processor → Sink pipeline
- **Source base:** `api/steps.py` defines abstract `Source`, `Processor`, `Sink` classes
- **Connectors (`source/`):**
  - `database/` — 50+ database connectors (Snowflake, BigQuery, Postgres, MySQL, Oracle, Redshift, Athena, Trino, dbt, etc.)
  - `dashboard/` — Tableau, Looker, Superset, Metabase, PowerBI, Redash
  - `pipeline/` — Airflow, dbt, Dagster, Glue, Flink
  - `messaging/` — Kafka, Pulsar, Kinesis
  - `mlmodel/` — MLflow, Vertex AI, SageMaker
  - `storage/` — S3, GCS, ADLS
  - `search/` — Elasticsearch, OpenSearch
- **Snowflake connector** (`source/database/snowflake/`): `metadata.py`, `lineage.py`, `usage.py`, `query_parser.py`, `queries.py`, `data_diff/`
- **dbt connector** (`source/database/dbt/`): `metadata.py`, `dbt_service.py`, `dbt_config.py`, `dbt_utils.py`
- **ometa/ client:** Python API client with mixins for each entity type
- **Sinks:** `MetadataRestSink` (pushes to API), `ElasticsearchSink`
- **Status tracking:** Per-step success/failure counters written back to API

**Dependencies:** `openmetadata-spec` (for config schemas), SQLAlchemy, requests, pydantic

**Entry point:** `metadata/ingestion/api/topology_runner.py` — `TopologyRunner.run()`

---

## 4. `openmetadata-ui`

**What it does:** The single-page React application. Provides the user-facing catalog browser, lineage visualizer, data quality dashboard, governance tools, team management, and pipeline configuration UI.

**Key contents:**
- Framework: **React + TypeScript**, built with **Vite**
- UI library: **Ant Design** + Material UI
- Authentication: Auth0, Azure MSAL, Okta integrations
- Rich text editing: TipTap
- Dynamic forms: React JSON Schema Form (`@rjsf`)
- E2E testing: Playwright
- Unit testing: Jest

**Dependencies:** `openmetadata-ui-core-components`, OpenMetadata REST API

**Entry point:** `src/main/resources/ui/` — Vite app, `yarn start` for dev, `yarn build` for prod

---

## 5. `openmetadata-ui-core-components`

**What it does:** Shared React component library extracted from the main UI — reusable across the main app and potentially third-party integrations.

**Dependencies:** React, TypeScript

---

## 6. `openmetadata-airflow-apis`

**What it does:** An Apache Airflow REST plugin that allows OpenMetadata to programmatically deploy, manage, and delete Airflow DAGs for ingestion pipelines.

**Key contents:**
- REST endpoints: `deploy_dag`, `delete_dag`, `trigger_dag`, `dag_state`
- JWT-based auth between OpenMetadata server and Airflow
- Generates Python DAG files from OpenMetadata workflow JSON configs
- Supports Airflow 2.3.3+

**Dependencies:** Apache Airflow, OpenMetadata ingestion Python package

**Entry point:** `openmetadata_managed_apis/` — Airflow plugin registration

---

## 7. `openmetadata-sdk` (Java)

**What it does:** Java client SDK for programmatic access to the OpenMetadata API. Provides a fluent builder API for entity operations.

**Key contents:**
- `OM.java` — main SDK entry point
- Fluent API for CRUD on all entity types
- Entity reference utilities
- WebSocket client support
- HTTP client with auth handling

**Dependencies:** `openmetadata-spec`, Jackson, Jersey/Apache HTTP

---

## 8. `openmetadata-k8s-operator`

**What it does:** Kubernetes operator for deploying and managing OpenMetadata on Kubernetes using Custom Resource Definitions (CRDs).

**Dependencies:** Kubernetes Java client, `openmetadata-service`

---

## 9. `openmetadata-mcp`

**What it does:** Model Context Protocol (MCP) server — exposes OpenMetadata catalog data as MCP tools for use with LLM-based agents and AI assistants.

**Dependencies:** `openmetadata-service`, MCP protocol libraries

---

## 10. `openmetadata-clients` (Java)

**What it does:** Generated Java client stubs for calling the OpenMetadata REST API from external Java applications.

---

## 11. `openmetadata-shaded-deps`

**What it does:** Provides shaded (relocated) Elasticsearch and OpenSearch JAR dependencies to avoid classpath conflicts when used alongside other libraries.

---

## 12. `common`

**What it does:** Shared Java utilities used across `openmetadata-service` and `openmetadata-sdk`.

---

## 13. `openmetadata-dist`

**What it does:** Distribution packaging module — assembles the final deployable tarball/zip with all JARs, config files, scripts, and static UI assets.

---

## Dependency Graph (Simplified)

```
openmetadata-spec
    ↓
openmetadata-sdk ← common
    ↓
openmetadata-service
    ↓
openmetadata-dist (final package)

ingestion (Python) → openmetadata-spec (JSON schemas)
                    → openmetadata REST API

openmetadata-ui → openmetadata REST API
openmetadata-airflow-apis → ingestion + openmetadata REST API
openmetadata-k8s-operator → openmetadata-service
openmetadata-mcp → openmetadata REST API
```

---

## Key Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend runtime | Java 21, Dropwizard 5.0, Jetty 12 |
| Data access | JDBI3, MySQL 8 / PostgreSQL 15 |
| Search | Elasticsearch 9.x / OpenSearch |
| Ingestion | Python 3.x, SQLAlchemy, Pydantic |
| Frontend | React 18, TypeScript, Vite, Ant Design |
| Orchestration | Apache Airflow 2.3+, Kubernetes |
| Auth | JWT, OIDC, SAML, LDAP, Auth0, Azure MSAL |
| Messaging | Kafka (optional, for events) |
| Build | Maven (Java), Yarn/Vite (UI), pip/setuptools (Python) |
