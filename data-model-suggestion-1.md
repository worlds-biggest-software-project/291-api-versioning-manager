# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: API Versioning Manager (Candidate #291)
> Approach: Fully normalized relational schema using PostgreSQL

---

## Summary

A traditional normalized relational data model using PostgreSQL, designed around third-normal-form (3NF) principles. Every entity has its own table with well-defined foreign-key relationships. This approach prioritises data integrity, referential consistency, and the ability to run complex analytical queries across versions, consumers, and deprecation timelines using standard SQL joins.

This is the most straightforward and operationally familiar approach. It maps naturally to the domain's core entities: APIs, versions, endpoints, consumers, deprecation policies, and migration tracking.

---

## Key Entities and Relationships

### Entity-Relationship Overview

```
Organisation ──1:N──> API
API ──1:N──> ApiVersion
ApiVersion ──1:N──> Endpoint
ApiVersion ──1:N──> BreakingChange
ApiVersion ──1:N──> DeprecationPolicy
Endpoint ──1:N──> EndpointParameter
Endpoint ──1:N──> EndpointSchema
Consumer ──M:N──> ApiVersion (via ConsumerSubscription)
Consumer ──1:N──> ConsumerTrafficLog
DeprecationPolicy ──1:N──> DeprecationNotification
BreakingChange ──M:N──> Consumer (via ConsumerImpact)
MigrationGuide ──N:1──> ApiVersion (source)
MigrationGuide ──N:1──> ApiVersion (target)
```

### Core Tables

```sql
-- Organisations that own APIs
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Top-level API registry
CREATE TABLE api (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL,
    description     TEXT,
    api_type        VARCHAR(32) NOT NULL DEFAULT 'REST',
        -- REST, GraphQL, AsyncAPI, gRPC
    versioning_strategy VARCHAR(32) NOT NULL DEFAULT 'SEMVER',
        -- SEMVER, DATE_BASED, URI_PATH, HEADER, QUERY_PARAM
    base_url        VARCHAR(2048),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

-- Individual versions of an API
CREATE TABLE api_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES api(id),
    version_label   VARCHAR(128) NOT NULL,
        -- e.g. "2.1.0", "2026-04-22", "2026-04-22.dahlia"
    version_major   INT,
    version_minor   INT,
    version_patch   INT,
    release_date    DATE,
    status          VARCHAR(32) NOT NULL DEFAULT 'DRAFT',
        -- DRAFT, ACTIVE, DEPRECATED, SUNSET, RETIRED
    openapi_spec_id UUID, -- FK to stored spec
    changelog_summary TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version_label)
);

-- Endpoints within a version
CREATE TABLE endpoint (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    http_method     VARCHAR(10) NOT NULL,
    path            VARCHAR(2048) NOT NULL,
    summary         VARCHAR(512),
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    deprecated_at   TIMESTAMPTZ,
    sunset_date     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_version_id, http_method, path)
);

-- Parameters for each endpoint
CREATE TABLE endpoint_parameter (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES endpoint(id),
    name            VARCHAR(255) NOT NULL,
    location        VARCHAR(16) NOT NULL, -- path, query, header, cookie
    data_type       VARCHAR(64) NOT NULL,
    required        BOOLEAN NOT NULL DEFAULT FALSE,
    deprecated      BOOLEAN NOT NULL DEFAULT FALSE,
    description     TEXT
);

-- Request/response schemas per endpoint
CREATE TABLE endpoint_schema (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id     UUID NOT NULL REFERENCES endpoint(id),
    direction       VARCHAR(16) NOT NULL, -- request, response
    content_type    VARCHAR(128) NOT NULL DEFAULT 'application/json',
    status_code     INT, -- NULL for request schemas
    schema_json     TEXT NOT NULL, -- JSON Schema definition
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Deprecation & Lifecycle Tables

```sql
-- Deprecation policies attached to a version
CREATE TABLE deprecation_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    deprecated_at   TIMESTAMPTZ NOT NULL,
    sunset_date     TIMESTAMPTZ NOT NULL,
    replacement_version_id UUID REFERENCES api_version(id),
    policy_text     TEXT,
    enforcement_level VARCHAR(32) NOT NULL DEFAULT 'ADVISORY',
        -- ADVISORY, WARNING, BLOCKING
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Notifications sent to consumers about deprecation
CREATE TABLE deprecation_notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deprecation_policy_id UUID NOT NULL REFERENCES deprecation_policy(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    channel         VARCHAR(32) NOT NULL, -- EMAIL, WEBHOOK, HEADER, DASHBOARD
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    status          VARCHAR(32) NOT NULL DEFAULT 'PENDING',
        -- PENDING, SENT, DELIVERED, ACKNOWLEDGED, FAILED
    message_body    TEXT
);

-- Breaking changes detected between versions
CREATE TABLE breaking_change (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    change_type     VARCHAR(64) NOT NULL,
        -- ENDPOINT_REMOVED, FIELD_REMOVED, TYPE_CHANGED,
        -- REQUIRED_ADDED, RESPONSE_CHANGED, etc.
    severity        VARCHAR(16) NOT NULL DEFAULT 'ERROR',
        -- ERROR, WARNING, INFO
    endpoint_path   VARCHAR(2048),
    field_path      VARCHAR(512),
    description     TEXT NOT NULL,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Consumer & Traffic Tables

```sql
-- API consumers (teams, services, external clients)
CREATE TABLE consumer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    consumer_type   VARCHAR(32) NOT NULL DEFAULT 'SERVICE',
        -- SERVICE, TEAM, EXTERNAL_CLIENT, SDK
    contact_email   VARCHAR(255),
    webhook_url     VARCHAR(2048),
    repository_url  VARCHAR(2048),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- M:N subscription of consumers to API versions
CREATE TABLE consumer_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
        -- ACTIVE, MIGRATING, MIGRATED, INACTIVE
    api_key_hash    VARCHAR(128),
    UNIQUE (consumer_id, api_version_id)
);

-- Aggregated traffic logs per consumer per endpoint
CREATE TABLE consumer_traffic_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    endpoint_id     UUID NOT NULL REFERENCES endpoint(id),
    period_start    TIMESTAMPTZ NOT NULL,
    period_end      TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    last_seen_at    TIMESTAMPTZ NOT NULL
);

-- Impact assessment linking breaking changes to affected consumers
CREATE TABLE consumer_impact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    breaking_change_id UUID NOT NULL REFERENCES breaking_change(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    impact_level    VARCHAR(16) NOT NULL DEFAULT 'HIGH',
        -- HIGH, MEDIUM, LOW
    affected_endpoints TEXT[], -- Array of endpoint paths used by consumer
    estimated_effort VARCHAR(32),
    migration_status VARCHAR(32) NOT NULL DEFAULT 'NOT_STARTED',
        -- NOT_STARTED, IN_PROGRESS, COMPLETED, WAIVED
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Migration & Documentation Tables

```sql
-- Migration guides between versions
CREATE TABLE migration_guide (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    title           VARCHAR(512) NOT NULL,
    content_markdown TEXT NOT NULL,
    generated_by    VARCHAR(32) NOT NULL DEFAULT 'MANUAL',
        -- MANUAL, AI_GENERATED, HYBRID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_version_id, target_version_id)
);

-- Stored OpenAPI / AsyncAPI / GraphQL specifications
CREATE TABLE api_spec (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    spec_format     VARCHAR(32) NOT NULL, -- OPENAPI_3_1, ASYNCAPI_3, GRAPHQL_SDL
    spec_content    TEXT NOT NULL,
    checksum        VARCHAR(128) NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Changelogs (one per version transition)
CREATE TABLE changelog_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    entry_type      VARCHAR(32) NOT NULL,
        -- ADDED, CHANGED, DEPRECATED, REMOVED, FIXED, SECURITY
    description     TEXT NOT NULL,
    endpoint_path   VARCHAR(2048),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Compatibility test results
CREATE TABLE compatibility_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    test_type       VARCHAR(32) NOT NULL,
        -- BACKWARD, FORWARD, CONTRACT
    status          VARCHAR(16) NOT NULL, -- PASSED, FAILED, SKIPPED
    report_json     TEXT,
    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit trail for compliance
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(32) NOT NULL,
        -- CREATED, UPDATED, DEPRECATED, SUNSET, DELETED
    actor_id        UUID,
    actor_email     VARCHAR(255),
    detail          TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Key Indexes

```sql
-- Version lookups
CREATE INDEX idx_api_version_api_status ON api_version(api_id, status);
CREATE INDEX idx_api_version_release_date ON api_version(release_date);

-- Endpoint searches
CREATE INDEX idx_endpoint_path ON endpoint(path);
CREATE INDEX idx_endpoint_deprecated ON endpoint(deprecated) WHERE deprecated = TRUE;
CREATE INDEX idx_endpoint_sunset ON endpoint(sunset_date) WHERE sunset_date IS NOT NULL;

-- Consumer traffic analysis
CREATE INDEX idx_traffic_consumer_period ON consumer_traffic_log(consumer_id, period_start);
CREATE INDEX idx_traffic_endpoint ON consumer_traffic_log(endpoint_id, last_seen_at);

-- Breaking change lookups
CREATE INDEX idx_breaking_change_versions ON breaking_change(source_version_id, target_version_id);
CREATE INDEX idx_consumer_impact_status ON consumer_impact(migration_status);

-- Audit trail queries
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_time ON audit_log(occurred_at);
```

---

## Pros and Cons

### Pros

1. **Data integrity**: Foreign keys and constraints enforce referential integrity across all entities. A consumer cannot be subscribed to a nonexistent version; a deprecation policy always points to a valid version.

2. **Mature tooling**: PostgreSQL has decades of tooling for migrations (Flyway, Liquibase, Alembic), ORMs (SQLAlchemy, Prisma, TypeORM), monitoring, backup, and replication. Every team member will be familiar with the paradigm.

3. **Complex analytics**: Standard SQL joins make it straightforward to answer critical questions like "which consumers are still calling deprecated endpoints sorted by traffic volume?" or "what is the blast radius of sunsetting version X?"

4. **ACID transactions**: Version status transitions (ACTIVE -> DEPRECATED -> SUNSET) and associated notifications can be wrapped in transactions, ensuring the system never enters an inconsistent state.

5. **Well-understood scaling path**: Read replicas, connection pooling (PgBouncer), table partitioning (e.g. partition `consumer_traffic_log` by time period), and eventually Citus for horizontal sharding.

6. **Standards compliance**: Maps cleanly to RFC 8594/9745 concepts (sunset dates, deprecation dates as columns), OpenAPI spec storage, and SemVer version parsing.

### Cons

1. **Schema rigidity**: Adding new fields to endpoints or extending version metadata requires ALTER TABLE migrations. For a rapidly evolving product in its early stages, this can slow iteration.

2. **OpenAPI spec storage is awkward**: Storing full OpenAPI documents as TEXT blobs in a relational table loses the ability to query inside the spec without extracting fields into separate tables, leading to schema bloat.

3. **Traffic log volume**: The `consumer_traffic_log` table can grow extremely large. Time-series workloads (high-volume traffic analytics) are not PostgreSQL's strongest suit compared to purpose-built tools like TimescaleDB or ClickHouse.

4. **Rigid change-type enumerations**: Breaking change types, notification channels, and consumer types are expressed as VARCHAR enums. As the domain evolves, these can proliferate, and adding new types requires application-level changes.

5. **No native graph traversal**: Answering "what is the full dependency chain from consumer A through all APIs it consumes and their upstream dependencies?" requires multiple joins or recursive CTEs, which can be complex and slow.

6. **Version comparison logic lives in application**: Comparing SemVer vs date-based versions, determining "next version" ordering, and computing diffs between specs cannot be done purely in SQL and require application code.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ |
| **Migrations** | Flyway or Liquibase for schema versioning |
| **ORM** | Prisma (TypeScript) or SQLAlchemy (Python) |
| **Time-series extension** | TimescaleDB hypertable for `consumer_traffic_log` if traffic volume is high |
| **Full-text search** | PostgreSQL tsvector for searching changelog entries and migration guides |
| **Connection pooling** | PgBouncer for production workloads |
| **Backup** | pg_basebackup with WAL archiving for point-in-time recovery |

---

## Migration and Scaling Considerations

1. **Initial deployment**: Start with a single PostgreSQL instance. The schema supports hundreds of APIs, thousands of versions, and millions of traffic log rows comfortably on a single well-provisioned instance.

2. **Partitioning strategy**: Partition `consumer_traffic_log` by month using PostgreSQL's native declarative partitioning. Partition `audit_log` similarly. This keeps query performance predictable as data grows.

3. **Read replicas**: Add read replicas for dashboard and analytics queries. The primary handles writes (version registration, traffic ingestion, notification delivery); replicas serve the read-heavy UI and reporting layers.

4. **Traffic log offloading**: If traffic analysis becomes the dominant workload, consider moving `consumer_traffic_log` to TimescaleDB (which is PostgreSQL-compatible) or a dedicated OLAP store like ClickHouse, while keeping the core relational schema in standard PostgreSQL.

5. **Multi-tenancy**: The `organisation_id` column on `api` enables row-level security (RLS) for multi-tenant SaaS deployment. Apply RLS policies to restrict each tenant's visibility to their own data.

6. **Schema evolution**: Use a migration tool (Flyway/Liquibase) from day one. Every schema change is versioned and repeatable. Plan for zero-downtime migrations by following expand-contract patterns for column changes.
