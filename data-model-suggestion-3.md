# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: API Versioning Manager (Candidate #291)
> Approach: PostgreSQL with strategic JSONB columns for semi-structured data

---

## Summary

A hybrid data model that uses PostgreSQL's relational tables for stable, well-defined entities (APIs, versions, consumers, deprecation policies) while leveraging JSONB columns for semi-structured, schema-flexible data (OpenAPI specs, endpoint details, breaking change descriptions, traffic metadata, and AI-generated content). This approach gives the best of both worlds: referential integrity and SQL joins where structure is known, and document-style flexibility where the schema is variable or evolving.

This is particularly well-suited for an API versioning manager because the domain has a clear relational backbone (APIs have versions, versions have consumers, consumers receive notifications) but also substantial semi-structured content that varies by API type (REST endpoints look different from GraphQL schemas, AsyncAPI channel definitions differ from gRPC service descriptors).

---

## Key Entities and Relationships

### Design Principles

1. **Relational for identity, status, and relationships**: Entity IDs, lifecycle status, foreign keys, and timestamps are always proper columns.
2. **JSONB for variable-shape content**: OpenAPI specs, endpoint definitions, schema diffs, AI analysis results, and notification templates are stored as JSONB.
3. **GIN indexes on JSONB**: Key fields within JSONB documents are indexed using GIN indexes for efficient querying.
4. **No separate document store**: Everything lives in PostgreSQL, avoiding the operational complexity of a second database.

### Core Schema

```sql
-- Organisations
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
        -- governance_rules, default_sunset_period, notification_preferences,
        -- versioning_defaults, branding, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- API registry
CREATE TABLE api (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(128) NOT NULL,
    api_type        VARCHAR(32) NOT NULL DEFAULT 'REST',
    versioning_strategy VARCHAR(32) NOT NULL DEFAULT 'SEMVER',
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- base_url, repository_url, owner_team, tags[], labels{},
        -- gateway_config, custom_headers, rate_limits, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

-- API versions with embedded spec and endpoint data
CREATE TABLE api_version (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_id          UUID NOT NULL REFERENCES api(id) ON DELETE CASCADE,
    version_label   VARCHAR(128) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'DRAFT',
    release_date    DATE,

    -- Structured version components (nullable, depends on strategy)
    version_major   INT,
    version_minor   INT,
    version_patch   INT,

    -- The full API specification stored as JSONB
    spec            JSONB,
        -- The complete OpenAPI 3.x, AsyncAPI 3.x, or GraphQL SDL
        -- stored as a parsed JSON document, queryable with JSONB operators

    -- Parsed endpoint summary (denormalized from spec for fast queries)
    endpoints       JSONB NOT NULL DEFAULT '[]',
        -- Array of objects:
        -- [
        --   {
        --     "method": "GET",
        --     "path": "/users/{id}",
        --     "summary": "Get user by ID",
        --     "deprecated": false,
        --     "sunset_date": null,
        --     "parameters": [...],
        --     "request_schema": {...},
        --     "response_schemas": {"200": {...}, "404": {...}}
        --   }
        -- ]

    -- Changelog entries for this version
    changelog       JSONB NOT NULL DEFAULT '[]',
        -- [
        --   {"type": "ADDED", "description": "New /users endpoint", "path": "/users"},
        --   {"type": "CHANGED", "description": "Updated response format", "path": "/orders"}
        -- ]

    spec_checksum   VARCHAR(128),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (api_id, version_label)
);
```

### Deprecation & Breaking Change Tables

```sql
-- Deprecation policies (relational core with JSONB detail)
CREATE TABLE deprecation_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    deprecated_at   TIMESTAMPTZ NOT NULL,
    sunset_date     TIMESTAMPTZ NOT NULL,
    replacement_version_id UUID REFERENCES api_version(id),
    enforcement_level VARCHAR(32) NOT NULL DEFAULT 'ADVISORY',
    status          VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',

    -- Flexible policy configuration
    policy_config   JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "notification_schedule": ["30d", "14d", "7d", "1d"],
        --   "escalation_rules": [...],
        --   "exempted_consumers": [...],
        --   "sunset_extension_rules": {...},
        --   "enforcement_actions": {...},
        --   "custom_message_template": "..."
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Breaking changes between versions
CREATE TABLE breaking_change (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    severity        VARCHAR(16) NOT NULL DEFAULT 'ERROR',

    -- All change details in JSONB (flexible for different API types)
    changes         JSONB NOT NULL,
        -- REST example:
        -- {
        --   "change_type": "ENDPOINT_REMOVED",
        --   "endpoint": {"method": "DELETE", "path": "/users/{id}/legacy"},
        --   "description": "Legacy delete endpoint removed",
        --   "migration_hint": "Use DELETE /users/{id} instead",
        --   "affected_fields": ["response.legacy_id"]
        -- }
        --
        -- GraphQL example:
        -- {
        --   "change_type": "FIELD_REMOVED",
        --   "type_name": "User",
        --   "field_name": "legacyId",
        --   "description": "Field User.legacyId removed",
        --   "migration_hint": "Use User.id instead"
        -- }

    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Diff between two versions (cached result of OpenAPI/schema comparison)
CREATE TABLE version_diff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    diff_result     JSONB NOT NULL,
        -- Full structured diff output from oasdiff or similar tool
        -- Includes added, removed, changed endpoints, schemas, parameters
    summary_stats   JSONB NOT NULL DEFAULT '{}',
        -- {"added": 3, "removed": 1, "changed": 5, "deprecated": 2}
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_version_id, target_version_id)
);
```

### Consumer & Traffic Tables

```sql
-- API consumers
CREATE TABLE consumer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    consumer_type   VARCHAR(32) NOT NULL DEFAULT 'SERVICE',
    contact_email   VARCHAR(255),

    -- Flexible consumer metadata
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "webhook_url": "https://...",
        --   "repository_url": "https://github.com/...",
        --   "slack_channel": "#api-consumers",
        --   "team_lead": "jane@example.com",
        --   "sdk_language": "python",
        --   "notification_preferences": {"email": true, "webhook": true},
        --   "tags": ["internal", "high-priority"]
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Consumer subscriptions to API versions
CREATE TABLE consumer_subscription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    api_version_id  UUID NOT NULL REFERENCES api_version(id),
    status          VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
    subscribed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Usage profile observed from traffic analysis
    usage_profile   JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "endpoints_used": ["/users", "/users/{id}", "/orders"],
        --   "last_seen_at": "2026-05-20T10:30:00Z",
        --   "avg_daily_requests": 12500,
        --   "peak_hourly_requests": 3200,
        --   "error_rate_7d": 0.02,
        --   "features_used": ["pagination", "filtering", "webhooks"]
        -- }

    UNIQUE (consumer_id, api_version_id)
);

-- Consumer traffic (time-partitioned)
CREATE TABLE consumer_traffic (
    consumer_id     UUID NOT NULL,
    api_version_id  UUID NOT NULL,
    endpoint_path   VARCHAR(2048) NOT NULL,
    http_method     VARCHAR(10) NOT NULL,
    time_bucket     TIMESTAMPTZ NOT NULL,
    request_count   BIGINT NOT NULL DEFAULT 0,
    error_count     BIGINT NOT NULL DEFAULT 0,
    p50_latency_ms  FLOAT,
    p99_latency_ms  FLOAT,
    PRIMARY KEY (consumer_id, api_version_id, endpoint_path, http_method, time_bucket)
) PARTITION BY RANGE (time_bucket);

-- Create monthly partitions
-- CREATE TABLE consumer_traffic_2026_05 PARTITION OF consumer_traffic
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Consumer impact assessments
CREATE TABLE consumer_impact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    breaking_change_id UUID NOT NULL REFERENCES breaking_change(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    impact_level    VARCHAR(16) NOT NULL DEFAULT 'HIGH',
    migration_status VARCHAR(32) NOT NULL DEFAULT 'NOT_STARTED',

    -- AI-generated impact analysis
    analysis        JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "affected_endpoints": ["/users/{id}"],
        --   "estimated_effort_hours": 4,
        --   "blast_radius": "medium",
        --   "migration_steps": [
        --     "Update request body to use 'userId' instead of 'user_id'",
        --     "Handle new 422 response code for validation errors"
        --   ],
        --   "code_references": [
        --     {"file": "src/api/users.ts", "line": 42, "snippet": "..."}
        --   ],
        --   "confidence_score": 0.87,
        --   "predicted_migration_date": "2026-06-15"
        -- }

    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (breaking_change_id, consumer_id)
);
```

### Notification & Migration Tables

```sql
-- Deprecation notifications
CREATE TABLE deprecation_notification (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    deprecation_policy_id UUID NOT NULL REFERENCES deprecation_policy(id),
    consumer_id     UUID NOT NULL REFERENCES consumer(id),
    channel         VARCHAR(32) NOT NULL,
    status          VARCHAR(32) NOT NULL DEFAULT 'PENDING',
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,

    -- Notification content and delivery details
    content         JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "subject": "API v2.1 deprecation notice",
        --   "body_markdown": "...",
        --   "personalised_migration_steps": [...],
        --   "delivery_metadata": {"message_id": "...", "bounce_status": null}
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Migration guides (content stored as JSONB for rich structure)
CREATE TABLE migration_guide (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    generated_by    VARCHAR(32) NOT NULL DEFAULT 'MANUAL',

    -- Structured guide content
    content         JSONB NOT NULL,
        -- {
        --   "title": "Migrating from v2.1 to v3.0",
        --   "overview": "...",
        --   "breaking_changes": [...],
        --   "step_by_step": [
        --     {"step": 1, "title": "Update authentication", "details": "..."},
        --     {"step": 2, "title": "Replace deprecated fields", "details": "..."}
        --   ],
        --   "code_examples": {
        --     "python": "...",
        --     "javascript": "...",
        --     "curl": "..."
        --   },
        --   "faq": [...]
        -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_version_id, target_version_id)
);

-- Compatibility test results
CREATE TABLE compatibility_test (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL REFERENCES api_version(id),
    target_version_id UUID NOT NULL REFERENCES api_version(id),
    test_type       VARCHAR(32) NOT NULL,
    status          VARCHAR(16) NOT NULL,

    -- Detailed test report
    report          JSONB NOT NULL DEFAULT '{}',
        -- {
        --   "total_tests": 142,
        --   "passed": 138,
        --   "failed": 4,
        --   "failures": [
        --     {
        --       "endpoint": "GET /users/{id}",
        --       "assertion": "response.body.legacy_id should exist",
        --       "expected": "string",
        --       "actual": "undefined"
        --     }
        --   ],
        --   "duration_ms": 3420
        -- }

    executed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID NOT NULL,
    action          VARCHAR(32) NOT NULL,
    actor_id        UUID,
    actor_email     VARCHAR(255),
    detail          JSONB NOT NULL DEFAULT '{}',
        -- Flexible structure captures any context relevant to the action
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);
```

### Indexes (Including JSONB GIN Indexes)

```sql
-- Standard relational indexes
CREATE INDEX idx_api_org ON api(organisation_id);
CREATE INDEX idx_version_api_status ON api_version(api_id, status);
CREATE INDEX idx_version_release ON api_version(release_date);
CREATE INDEX idx_subscription_consumer ON consumer_subscription(consumer_id);
CREATE INDEX idx_subscription_version ON consumer_subscription(api_version_id);
CREATE INDEX idx_impact_status ON consumer_impact(migration_status);

-- GIN indexes for JSONB querying
CREATE INDEX idx_version_endpoints ON api_version USING GIN (endpoints jsonb_path_ops);
CREATE INDEX idx_version_changelog ON api_version USING GIN (changelog jsonb_path_ops);
CREATE INDEX idx_api_metadata ON api USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_consumer_metadata ON consumer USING GIN (metadata jsonb_path_ops);
CREATE INDEX idx_subscription_usage ON consumer_subscription USING GIN (usage_profile jsonb_path_ops);
CREATE INDEX idx_impact_analysis ON consumer_impact USING GIN (analysis jsonb_path_ops);
CREATE INDEX idx_breaking_changes ON breaking_change USING GIN (changes jsonb_path_ops);
CREATE INDEX idx_org_settings ON organisation USING GIN (settings jsonb_path_ops);

-- Partial indexes for common queries
CREATE INDEX idx_deprecated_versions ON api_version(api_id, sunset_date)
    WHERE status = 'DEPRECATED';
CREATE INDEX idx_active_subscriptions ON consumer_subscription(api_version_id)
    WHERE status = 'ACTIVE';

-- Audit log
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
```

### Example Queries

```sql
-- Find all consumers still using deprecated endpoints, ranked by traffic
SELECT
    c.name AS consumer_name,
    c.contact_email,
    av.version_label,
    cs.usage_profile->>'last_seen_at' AS last_seen,
    (cs.usage_profile->>'avg_daily_requests')::int AS avg_daily_requests,
    dp.sunset_date,
    dp.sunset_date - now() AS time_remaining
FROM consumer_subscription cs
JOIN consumer c ON c.id = cs.consumer_id
JOIN api_version av ON av.id = cs.api_version_id
JOIN deprecation_policy dp ON dp.api_version_id = av.id
WHERE av.status = 'DEPRECATED'
  AND cs.status = 'ACTIVE'
ORDER BY (cs.usage_profile->>'avg_daily_requests')::int DESC;

-- Find all endpoints deprecated across all versions of an API
SELECT
    av.version_label,
    ep->>'method' AS method,
    ep->>'path' AS path,
    ep->>'summary' AS summary,
    ep->>'sunset_date' AS sunset_date
FROM api_version av,
     jsonb_array_elements(av.endpoints) AS ep
WHERE av.api_id = 'some-api-id'
  AND (ep->>'deprecated')::boolean = true
ORDER BY av.version_label, ep->>'path';

-- Get AI-generated migration steps for a specific consumer
SELECT
    ci.impact_level,
    ci.analysis->'migration_steps' AS steps,
    ci.analysis->>'estimated_effort_hours' AS effort,
    ci.analysis->>'confidence_score' AS confidence
FROM consumer_impact ci
WHERE ci.consumer_id = 'some-consumer-id'
  AND ci.migration_status = 'NOT_STARTED';
```

---

## Pros and Cons

### Pros

1. **Best of both worlds**: Relational integrity for core relationships (APIs own versions, versions have consumers) with document flexibility for variable-shape data (endpoint definitions, AI analysis results, test reports).

2. **Single database**: Everything lives in PostgreSQL. No need to operate a separate document store (MongoDB) or search engine (Elasticsearch). This dramatically simplifies deployment, backup, and operations.

3. **Spec storage done right**: OpenAPI/AsyncAPI/GraphQL specs are stored as JSONB, making them queryable with PostgreSQL's JSON operators and `jsonb_path_query`. You can find all endpoints with `deprecated: true` across all specs without extracting them into separate tables.

4. **Evolves gracefully**: Adding new fields to JSONB columns requires no schema migration. When the AI analysis produces a new output format, or when a new API type (gRPC) needs different endpoint metadata, the JSONB structure accommodates it immediately.

5. **Strong query capabilities**: PostgreSQL's JSONB operators (`->`, `->>`, `@>`, `jsonb_path_query`) combined with GIN indexes provide near-document-database query performance. Combined with relational JOINs, this handles both structured and semi-structured queries efficiently.

6. **Natural fit for AI outputs**: AI-generated content (migration guides, impact analyses, personalised notifications) is inherently variable in structure. Storing it as JSONB avoids the need to normalise AI output into rigid table schemas.

7. **Familiar to most teams**: Every developer knows SQL. JSONB adds a learning curve but it is well-documented and widely used. No need for specialised event sourcing or graph database expertise.

### Cons

1. **JSONB discipline required**: Without careful conventions, JSONB columns can become unstructured dumping grounds. The team must enforce JSON Schema validation at the application layer and document the expected shape of each JSONB column.

2. **JSONB query performance ceiling**: While GIN-indexed JSONB queries are fast for containment checks (`@>`), complex aggregations across deeply nested JSONB structures are slower than equivalent queries on normalised columns. Heavy analytics may need materialised views.

3. **No referential integrity inside JSONB**: If an endpoint reference inside a JSONB `usage_profile` points to a path that no longer exists, PostgreSQL cannot enforce that constraint. Consistency inside JSONB is the application's responsibility.

4. **Backup size**: JSONB columns storing full OpenAPI specs can be large (specs can exceed 1MB). This increases backup sizes and WAL volume. Consider storing specs in object storage with a reference, if specs are very large.

5. **ORM mapping complexity**: ORMs handle JSONB columns with varying degrees of elegance. Prisma and SQLAlchemy both support JSONB, but typed access to nested fields requires additional TypeScript/Python type definitions that must be kept in sync with the actual JSON shape.

6. **Migration of JSONB structure**: While adding new fields to JSONB is easy, changing the shape of existing JSONB data (renaming a key, restructuring nested arrays) requires data migration scripts that parse and transform JSON, which can be slower and harder to test than ALTER TABLE.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ (native JSONB, GIN indexes, partitioning) |
| **ORM** | Prisma with `Json` type fields, or Drizzle ORM with `jsonb()` columns |
| **Validation** | Zod or AJV for runtime JSON Schema validation of JSONB content |
| **Migrations** | Prisma Migrate or Drizzle Kit; custom scripts for JSONB data transforms |
| **Spec parsing** | `@readme/openapi-parser` or `swagger-parser` for OpenAPI; `@graphql-tools/load` for GraphQL |
| **Diff engine** | `oasdiff` CLI or library for OpenAPI diff; store results as JSONB in `version_diff` |
| **Full-text search** | PostgreSQL `tsvector` on changelog and migration guide content |
| **Time-series** | TimescaleDB extension for `consumer_traffic` hypertable if volume is high |

---

## Migration and Scaling Considerations

1. **Start with the hybrid from day one**: Define clear conventions for which data goes in columns vs JSONB. Rule of thumb: if you need to JOIN on it, filter on it frequently, or enforce uniqueness, make it a column. If it varies by API type, is AI-generated, or is a large nested document, use JSONB.

2. **JSON Schema validation layer**: Implement application-level validation using JSON Schema (or Zod schemas) for every JSONB column. Store the schemas in a `json_schema_registry` table so they can evolve and be versioned alongside the data.

3. **Materialised views for analytics**: For complex cross-entity queries that join relational and JSONB data, create materialised views that are refreshed on a schedule (e.g. every 5 minutes). This avoids repeated expensive JSONB extraction in dashboard queries.

4. **Partitioning**: Partition `consumer_traffic` by time range and `audit_log` by time range. Consider partitioning `api_version` by `status` if the table grows very large (archived/retired versions in cold partitions).

5. **Large spec handling**: If OpenAPI specs regularly exceed 500KB, consider storing the raw spec in object storage (S3/MinIO) and keeping only a parsed summary in the JSONB `spec` column. The `endpoints` JSONB array serves as the queryable cache.

6. **Multi-tenant isolation**: Use PostgreSQL Row-Level Security (RLS) policies on `organisation_id`. JSONB columns are transparent to RLS -- the same policies that protect relational rows protect their JSONB content.

7. **JSONB to columns migration path**: If a field in JSONB becomes critical enough to warrant its own column (e.g. `usage_profile.last_seen_at`), the migration is straightforward: add the column, backfill from JSONB, and update application code. This "promote to column" pattern is a well-trodden path.
