# OpenMetadata Ship Room — Release Readiness Document

Version: **1.12.0-SNAPSHOT**
Assessed: 2026-03-09

---

## Executive Summary

OpenMetadata is a production-grade, open-source metadata platform with a mature multi-layer architecture (Java API backend, Python ingestion framework, React UI). The platform supports 50+ data source connectors, enterprise authentication, data lineage, data quality, and governance workflows. The Snowflake and dbt connectors are among the most feature-complete in the ecosystem.

---

## Major Capabilities

### Metadata Management
- Centralized catalog for Tables, Databases, Schemas, Dashboards, Pipelines, Topics, ML Models, Containers, Stored Procedures, Glossary Terms, APIs, Data Products, Domains
- JSON document store (MySQL/Postgres) with 40+ entity types
- Full entity versioning and change history via `change_event` table
- Soft delete with restore support
- Custom properties on any entity

### Lineage Tracking
- Column-level lineage (not just table-level)
- Automated lineage from SQL query parsing (Snowflake QUERY_HISTORY, views, stored procedures)
- dbt lineage from `manifest.json` node dependencies
- Manual lineage editing in UI
- OpenLineage protocol support for cross-platform lineage federation
- Configurable lineage depth in graph traversal API

### Data Quality
- Native test framework: TestSuite → TestCase → TestRun
- Built-in test types: row count, column values, null checks, uniqueness, referential integrity
- dbt test results imported and linked to entities
- Trend tracking via `entity_extension_time_series`
- KPI tracking for quality metrics

### Governance
- Tag-based classification with propagation
- Glossary with hierarchical terms
- Policy engine: data access policies, PII classification, retention rules
- Governance workflow automation (approval flows, review cycles)
- Automation rules triggered by entity lifecycle events

### Search & Discovery
- Full-text search powered by Elasticsearch / OpenSearch
- Faceted search by owner, tag, service, tier, domain
- Autocomplete / typeahead
- Relevance ranking

### Observability & Usage
- Table/column usage stats from warehouse query logs
- Query cost tracking
- Data profiling (column statistics, histograms, null %, cardinality)
- Data Insights dashboards for catalog adoption metrics

### AI & LLM Features
- MCP server (`openmetadata-mcp`) for LLM agent access to catalog
- AI-powered description generation (LLM integration)
- AI governance entities (AI models, agent executions, AI applications)

### Authentication & Authorization
- Multi-provider: JWT (local), OIDC (Google, Auth0, Okta), SAML, LDAP, Azure MSAL
- RBAC: Roles → Policies → Rules
- SCIM 2.0 for user provisioning
- Team-based access control
- Bot accounts for service-to-service auth

---

## Integration Points

### Snowflake Connector

**Capabilities:**
| Feature | Status | Details |
|---------|--------|---------|
| Table/Schema/DB metadata | Production | Full schema extraction including column types, constraints |
| Column-level lineage | Production | Parses Snowflake ACCESS_HISTORY and QUERY_HISTORY |
| Usage statistics | Production | Daily/weekly query counts per table from QUERY_HISTORY |
| Tag ingestion | Production | Reads Snowflake object tags, maps to OpenMetadata classification |
| Query parsing | Production | SQL-based lineage extraction via `query_parser.py` |
| Data diff | Available | `data_diff/` submodule for comparing table snapshots |
| Stored procedures | Supported | Listed as entity type in spec |
| Dynamic tables | Supported | Via standard table metadata path |

**Configuration required (`snowflake.json` spec):**
```yaml
type: Snowflake
username: <user>
password: <password>  # or use privateKey
account: <account-identifier>
warehouse: <warehouse>
role: <role>           # optional, defaults to user default
database: <db>         # optional filter
schemaFilterPattern: {}
tableFilterPattern: {}
```

**Permissions needed:** `USAGE` on database/schema/warehouse, `SELECT` on `SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY` for lineage/usage.

---

### dbt Connector

**Capabilities:**
| Feature | Status | Details |
|---------|--------|---------|
| Model → Table mapping | Production | Links dbt models to physical tables in catalog |
| Column-level lineage | Production | From `manifest.json` node `depends_on.nodes` |
| Descriptions | Production | Model + column descriptions from dbt docs |
| Tags | Production | dbt meta tags mapped to OpenMetadata tags |
| Owner mapping | Production | dbt meta owner mapped to OpenMetadata user |
| Test results | Production | `run_results.json` → TestCase + TestSuite entities |
| dbt Cloud support | Supported | Via `dbt_service.py` API integration |
| dbt Core support | Supported | Reads local artifact files (manifest, catalog, run_results) |
| Sources | Supported | dbt sources mapped to upstream tables |
| Exposures | Partial | Not all exposure types surfaced |

**Configuration required:**
```yaml
type: Dbt
dbtConfigSource:
  dbtSecurityConfig: {}       # for dbt Cloud: API token
  dbtArtifactDirPath: /path/to/target   # for dbt Core
# OR for dbt Cloud:
  dbtCloudUrl: https://cloud.getdbt.com
  dbtCloudAccountId: <id>
  dbtCloudAuthToken: <token>
  dbtCloudProjectId: <project-id>
```

---

## Deployment Architecture

### Minimum Required Services

| Service | Purpose | Image |
|---------|---------|-------|
| MySQL 8 or PostgreSQL 15 | Metadata store | `docker.getcollate.io/openmetadata/db:1.12.0-SNAPSHOT` |
| Elasticsearch 9.x or OpenSearch | Search + indexing | `docker.elastic.co/elasticsearch/elasticsearch:9.3.0` |
| OpenMetadata Server | API + UI | `docker.getcollate.io/openmetadata/server:1.12.0-SNAPSHOT` |
| OpenMetadata Ingestion | Python ingestion worker | `docker.getcollate.io/openmetadata/ingestion:1.12.0-SNAPSHOT` |

### Optional Services

| Service | Purpose |
|---------|---------|
| Apache Airflow | Pipeline orchestration (required for scheduled ingestion) |
| Kafka | Event streaming / audit log fan-out |
| Redis | Session caching / distributed locking |
| S3 / GCS | Log storage for ingestion runs |

### Ports

| Port | Service |
|------|---------|
| 8585 | OpenMetadata REST API + UI |
| 8586 | Admin API (health, metrics) |
| 3306 | MySQL |
| 5432 | PostgreSQL |
| 9200 | Elasticsearch HTTP |
| 9300 | Elasticsearch transport |
| 8080 | Airflow Web UI |

---

## What Is Needed to Deploy

### Prerequisites Checklist

- [ ] **Java 21** installed on server (for running from tarball) OR Docker
- [ ] **MySQL 8+** or **PostgreSQL 15+** provisioned with a dedicated database and user
- [ ] **Elasticsearch 9.x** or **OpenSearch 2.x** cluster accessible
- [ ] Network connectivity from OpenMetadata server to database + search
- [ ] Network connectivity from ingestion workers to data sources (Snowflake, dbt, etc.)
- [ ] DNS / hostname resolution for all services

### Configuration Required (`openmetadata.yaml`)

```yaml
clusterName: openmetadata

database:
  driverClass: com.mysql.cj.jdbc.Driver
  url: jdbc:mysql://<host>:3306/openmetadata_db
  user: openmetadata_user
  password: <password>

elasticsearch:
  host: <es-host>
  port: 9200
  scheme: http

authenticationConfiguration:
  provider: basic  # or google, okta, auth0, azure, ldap, saml
  # ... provider-specific config

authorizerConfiguration:
  className: org.openmetadata.service.security.DefaultAuthorizer
  adminPrincipals:
    - admin

pipelineServiceClientConfiguration:
  className: org.openmetadata.service.clients.pipeline.airflow.AirflowRESTClient
  apiEndpoint: http://<airflow-host>:8080
  # ... auth config
```

### Deployment Steps

1. **Provision infrastructure** (DB, Elasticsearch, network)
2. **Run database migrations:**
   ```bash
   ./bootstrap/openmetadata-ops.sh migrate
   ```
3. **Start OpenMetadata server:**
   ```bash
   ./bin/openmetadata.sh start
   # OR via Docker Compose:
   docker compose -f docker/docker-compose-quickstart/docker-compose.yml up -d
   ```
4. **Configure ingestion connections** via UI at `http://<host>:8585`
5. **Deploy Airflow plugin** (`openmetadata-airflow-apis`) on Airflow workers for scheduled ingestion
6. **Create initial admin user** and configure authentication provider
7. **Run first ingestion** for Snowflake / dbt via UI pipeline configuration or CLI

### Kubernetes Deployment

- Use `openmetadata-k8s-operator` for CRD-based deployment
- Helm charts available in `openmetadata-k8s-operator/`
- Supports horizontal scaling of ingestion workers

---

## Known Risks / Considerations

| Area | Risk | Mitigation |
|------|------|-----------|
| Version | 1.12.0-SNAPSHOT (pre-release) | Pin to a tagged release for production |
| Database | MySQL image is custom (`docker.getcollate.io`) | Verify compatibility with managed MySQL (RDS, Cloud SQL) |
| Elasticsearch version | Pinned to ES 9.3.0 | Test with your existing ES/OpenSearch cluster version |
| Airflow dependency | Scheduled ingestion requires Airflow 2.3.3+ | Can run ingestion manually via CLI without Airflow |
| Snowflake permissions | Lineage/usage requires `ACCOUNT_USAGE` schema access | May need Snowflake ACCOUNTADMIN to grant this |
| dbt artifacts | dbt Core integration requires artifact files to be accessible to ingestion worker | Ensure CI/CD copies artifacts to shared storage |
| Security | Default config uses basic auth + local JWT | Production deployments should use OIDC/SAML + proper secret management |
| Fernet key | Used for encrypting connector credentials at rest | Must be generated and stored securely; loss = loss of all connector passwords |
| Storage | Elasticsearch index size grows with entity count | Plan for adequate disk — 50-100GB+ for large catalogs |

---

## Integration Test Coverage

- `openmetadata-integration-tests/` contains integration test suite
- Tests cover API contract, entity CRUD, lineage, search
- Playwright E2E tests cover UI flows
- Run with: `mvn verify -pl openmetadata-integration-tests`

---

## Ship Readiness Signal

| Dimension | Status |
|-----------|--------|
| Core API stability | Mature — 50+ resources, versioned entities |
| Snowflake connector | Production-ready |
| dbt connector | Production-ready |
| Docker deployment | Ready (quickstart compose available) |
| Kubernetes deployment | Available via operator |
| Authentication | Enterprise-ready (OIDC, SAML, LDAP) |
| Data quality framework | Production-ready |
| Lineage (column-level) | Production-ready |
| AI/MCP features | Beta (1.12.0 new additions) |
| Documentation | Maintained at open-metadata.org/docs |
