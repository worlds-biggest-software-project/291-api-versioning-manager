# Data Model Suggestion 4: Graph-Based Model for API Dependency Tracking

> Project: API Versioning Manager (Candidate #291)
> Approach: Graph database (Neo4j) for dependency and impact analysis, with PostgreSQL for operational data

---

## Summary

A graph-based data model using Neo4j as the primary store for API relationships, version dependencies, consumer impact chains, and migration paths. This approach treats the API versioning domain as fundamentally a graph problem: APIs depend on other APIs, consumers depend on specific versions, breaking changes propagate through dependency chains, and migration paths form directed acyclic graphs between versions.

The key insight is that the hardest problem in API versioning management -- identified in the project research as "the primary unsolved problem in the space" -- is understanding **which consumers are still calling a deprecated version and what the blast radius is**. This is a graph traversal problem. A graph database makes impact analysis, dependency chain discovery, and migration path planning first-class operations rather than expensive multi-join SQL queries.

This suggestion uses a polyglot architecture: Neo4j for the relationship-rich core, PostgreSQL for time-series traffic data and audit logs, and optionally Redis for caching.

---

## Key Entities and Relationships

### Graph Data Model Overview

```
                    ┌─────────────────┐
                    │  Organisation   │
                    └────────┬────────┘
                             │ OWNS
                             ▼
                    ┌─────────────────┐
         ┌─────────│      API        │─────────┐
         │         └────────┬────────┘         │
         │                  │ HAS_VERSION       │
         │                  ▼                   │
         │         ┌─────────────────┐         │
         │    ┌───>│   ApiVersion    │<───┐    │
         │    │    └──┬──────────┬───┘    │    │
         │    │       │          │        │    │
         │  REPLACES  │          │   DEPENDS_ON│
         │    │       │          │        │    │
         │    │  HAS_ENDPOINT  HAS_DEPRECATION │
         │    │       │          │        │    │
         │    │       ▼          ▼        │    │
         │    │ ┌──────────┐ ┌───────────┐│    │
         │    │ │ Endpoint │ │Deprecation││    │
         │    │ └──────────┘ │  Policy   ││    │
         │    │              └───────────┘│    │
         │    │                           │    │
    DEPENDS_ON│    ┌─────────────────┐    │    │
         │    └────│ BreakingChange  │────┘    │
         │         └────────┬────────┘         │
         │                  │ IMPACTS          │
         │                  ▼                  │
         │         ┌─────────────────┐         │
         └────────>│    Consumer     │<────────┘
                   └─────────────────┘
                      │           │
              SUBSCRIBES_TO   MIGRATING_TO
                      │           │
                      ▼           ▼
               ┌──────────┐ ┌──────────┐
               │ApiVersion│ │ApiVersion│
               └──────────┘ └──────────┘
```

### Node Definitions (Neo4j)

```cypher
// Organisation node
CREATE (org:Organisation {
    id: randomUUID(),
    name: "Acme Corp",
    slug: "acme-corp",
    settings: '{"default_sunset_period_days": 180}',
    created_at: datetime()
})

// API node
CREATE (api:Api {
    id: randomUUID(),
    name: "User Management API",
    slug: "user-management",
    api_type: "REST",
    versioning_strategy: "SEMVER",
    base_url: "https://api.acme.com/users",
    created_at: datetime()
})

// ApiVersion node
CREATE (v:ApiVersion {
    id: randomUUID(),
    version_label: "2.1.0",
    version_major: 2,
    version_minor: 1,
    version_patch: 0,
    status: "ACTIVE",
    release_date: date("2026-03-15"),
    endpoint_count: 12,
    spec_checksum: "sha256:abc123...",
    spec_format: "OPENAPI_3_1",
    created_at: datetime()
})

// Endpoint node
CREATE (ep:Endpoint {
    id: randomUUID(),
    http_method: "GET",
    path: "/users/{id}",
    summary: "Retrieve user by ID",
    deprecated: false,
    sunset_date: null,
    parameters: '[{"name": "id", "in": "path", "type": "string"}]',
    response_schema_checksum: "sha256:def456..."
})

// Consumer node
CREATE (c:Consumer {
    id: randomUUID(),
    name: "Order Service",
    consumer_type: "SERVICE",
    contact_email: "order-team@acme.com",
    repository_url: "https://github.com/acme/order-service",
    webhook_url: "https://hooks.acme.com/order-service",
    created_at: datetime()
})

// BreakingChange node
CREATE (bc:BreakingChange {
    id: randomUUID(),
    change_type: "FIELD_REMOVED",
    severity: "ERROR",
    endpoint_path: "/users/{id}",
    field_path: "response.legacy_id",
    description: "Field legacy_id removed from User response",
    migration_hint: "Use field 'id' instead of 'legacy_id'",
    detected_at: datetime()
})

// DeprecationPolicy node
CREATE (dp:DeprecationPolicy {
    id: randomUUID(),
    deprecated_at: datetime("2026-04-01"),
    sunset_date: datetime("2026-10-01"),
    enforcement_level: "WARNING",
    notification_schedule: '["30d", "14d", "7d", "1d"]',
    status: "ACTIVE"
})

// MigrationGuide node
CREATE (mg:MigrationGuide {
    id: randomUUID(),
    title: "Migrating from v2.1 to v3.0",
    generated_by: "AI_GENERATED",
    content_checksum: "sha256:ghi789...",
    created_at: datetime()
})
```

### Relationship Definitions

```cypher
// Ownership and hierarchy
(org)-[:OWNS]->(api)
(api)-[:HAS_VERSION]->(version)
(version)-[:HAS_ENDPOINT]->(endpoint)
(version)-[:HAS_DEPRECATION]->(deprecation_policy)

// Version lineage and succession
(v3)-[:SUCCEEDS {released_at: datetime()}]->(v2)
(v3)-[:REPLACES]->(v2)

// Consumer relationships (the critical graph)
(consumer)-[:SUBSCRIBES_TO {
    subscribed_at: datetime(),
    status: "ACTIVE",
    api_key_hash: "...",
    last_seen_at: datetime(),
    avg_daily_requests: 12500
}]->(version)

(consumer)-[:USES {
    last_seen_at: datetime(),
    request_count_30d: 45000,
    error_rate: 0.02
}]->(endpoint)

(consumer)-[:MIGRATING_TO {
    started_at: datetime(),
    target_completion: date("2026-07-01"),
    progress_pct: 35
}]->(target_version)

// Breaking change impact chain
(breaking_change)-[:BETWEEN {direction: "source"}]->(source_version)
(breaking_change)-[:BETWEEN {direction: "target"}]->(target_version)
(breaking_change)-[:IMPACTS {
    impact_level: "HIGH",
    affected_endpoints: ["/users/{id}"],
    estimated_effort_hours: 4,
    migration_status: "NOT_STARTED"
}]->(consumer)

// API dependency graph (APIs that depend on other APIs)
(api_a)-[:DEPENDS_ON {
    dependency_type: "RUNTIME",
    endpoints_used: ["/auth/validate", "/auth/refresh"]
}]->(api_b)

(consumer)-[:DEPENDS_ON {
    dependency_type: "BUILD_TIME"
}]->(api)

// Migration guide connections
(migration_guide)-[:FROM]->(source_version)
(migration_guide)-[:TO]->(target_version)

// Notification tracking
(consumer)-[:NOTIFIED_OF {
    channel: "EMAIL",
    sent_at: datetime(),
    acknowledged_at: datetime(),
    status: "ACKNOWLEDGED"
}]->(deprecation_policy)
```

### Critical Graph Queries

```cypher
// 1. BLAST RADIUS: Find all consumers affected by sunsetting a version,
//    including transitive dependencies through API chains
MATCH (v:ApiVersion {id: $versionId})<-[:SUBSCRIBES_TO]-(c:Consumer)
OPTIONAL MATCH (c)-[:DEPENDS_ON*1..5]->(downstream:Api)-[:HAS_VERSION]->(dv:ApiVersion)
WHERE dv.status = 'ACTIVE'
RETURN c.name AS consumer,
       c.contact_email AS email,
       collect(DISTINCT downstream.name) AS transitive_dependencies,
       size(collect(DISTINCT downstream)) AS dependency_depth
ORDER BY dependency_depth DESC

// 2. MIGRATION PATH: Find the shortest migration path between two versions
MATCH path = shortestPath(
    (source:ApiVersion {version_label: $from})-[:SUCCEEDS*]->(target:ApiVersion {version_label: $to})
)
WHERE source.api_id = target.api_id
RETURN [n IN nodes(path) | n.version_label] AS migration_path,
       length(path) AS hops

// 3. AT-RISK CONSUMERS: Find consumers unlikely to migrate before sunset
MATCH (dp:DeprecationPolicy {status: "ACTIVE"})<-[:HAS_DEPRECATION]-(v:ApiVersion)
MATCH (v)<-[sub:SUBSCRIBES_TO {status: "ACTIVE"}]-(c:Consumer)
WHERE NOT (c)-[:MIGRATING_TO]->(:ApiVersion)
  AND dp.sunset_date < datetime() + duration({days: 30})
RETURN c.name AS consumer,
       c.contact_email AS contact,
       v.version_label AS deprecated_version,
       dp.sunset_date AS sunset,
       sub.avg_daily_requests AS daily_traffic,
       sub.last_seen_at AS last_active
ORDER BY sub.avg_daily_requests DESC

// 4. BREAKING CHANGE PROPAGATION: Trace how a breaking change
//    propagates through the dependency graph
MATCH (bc:BreakingChange {id: $changeId})-[:IMPACTS]->(c:Consumer)
MATCH (c)-[:DEPENDS_ON*0..3]->(downstream_api:Api)
OPTIONAL MATCH (downstream_api)-[:HAS_VERSION]->(dv:ApiVersion {status: "ACTIVE"})
              <-[:SUBSCRIBES_TO]-(downstream_consumer:Consumer)
RETURN bc.description AS breaking_change,
       c.name AS directly_impacted,
       collect(DISTINCT {
           api: downstream_api.name,
           consumer: downstream_consumer.name
       }) AS cascade_impact

// 5. CONSUMER DEPENDENCY MAP: Full dependency graph for a consumer
MATCH (c:Consumer {id: $consumerId})
MATCH (c)-[sub:SUBSCRIBES_TO]->(v:ApiVersion)<-[:HAS_VERSION]-(api:Api)
OPTIONAL MATCH (api)-[:DEPENDS_ON*1..3]->(upstream:Api)
OPTIONAL MATCH (v)-[:HAS_DEPRECATION]->(dp:DeprecationPolicy)
RETURN api.name AS api_name,
       v.version_label AS version,
       v.status AS status,
       dp.sunset_date AS sunset,
       collect(DISTINCT upstream.name) AS upstream_dependencies

// 6. VERSION LINEAGE: Full history of an API's versions
MATCH (api:Api {slug: $apiSlug})-[:HAS_VERSION]->(v:ApiVersion)
OPTIONAL MATCH (v)-[:SUCCEEDS]->(prev:ApiVersion)
OPTIONAL MATCH (v)<-[:SUCCEEDS]-(next:ApiVersion)
OPTIONAL MATCH (v)-[:HAS_DEPRECATION]->(dp:DeprecationPolicy)
RETURN v.version_label AS version,
       v.status AS status,
       v.release_date AS released,
       prev.version_label AS previous_version,
       next.version_label AS next_version,
       dp.sunset_date AS sunset_date
ORDER BY v.release_date

// 7. IMPACT SCORE: Rank consumers by total exposure to breaking changes
MATCH (c:Consumer)-[sub:SUBSCRIBES_TO]->(v:ApiVersion {status: "DEPRECATED"})
MATCH (bc:BreakingChange)-[imp:IMPACTS]->(c)
WHERE imp.migration_status <> "COMPLETED"
WITH c,
     count(DISTINCT bc) AS open_breaking_changes,
     sum(sub.avg_daily_requests) AS total_daily_traffic,
     collect(DISTINCT v.version_label) AS deprecated_versions
RETURN c.name AS consumer,
       c.contact_email AS contact,
       open_breaking_changes,
       total_daily_traffic,
       deprecated_versions,
       open_breaking_changes * total_daily_traffic AS risk_score
ORDER BY risk_score DESC
```

### PostgreSQL Companion Schema (Operational Data)

The graph database excels at relationship queries but is not ideal for time-series data, large document storage, or transactional audit logs. These stay in PostgreSQL:

```sql
-- Traffic analytics (time-series, high volume)
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

-- Full API specification storage (large documents)
CREATE TABLE api_spec_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    api_version_id  UUID NOT NULL,
    spec_format     VARCHAR(32) NOT NULL,
    spec_content    TEXT NOT NULL,
    spec_parsed     JSONB,
    checksum        VARCHAR(128) NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Version diff results (large structured documents)
CREATE TABLE version_diff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_version_id UUID NOT NULL,
    target_version_id UUID NOT NULL,
    diff_result     JSONB NOT NULL,
    summary_stats   JSONB NOT NULL DEFAULT '{}',
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_version_id, target_version_id)
);

-- Migration guide content (large text documents)
CREATE TABLE migration_guide_content (
    guide_id        UUID PRIMARY KEY,
    content_markdown TEXT NOT NULL,
    content_structured JSONB,
    code_examples   JSONB,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- AI analysis results
CREATE TABLE ai_analysis (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    analysis_type   VARCHAR(64) NOT NULL,
    entity_type     VARCHAR(64) NOT NULL,
    entity_id       UUID NOT NULL,
    result          JSONB NOT NULL,
    model_version   VARCHAR(64),
    confidence      FLOAT,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now()
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
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);
```

### Synchronisation Between Neo4j and PostgreSQL

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Application │────>│   Neo4j      │     │  PostgreSQL  │
│    Layer     │────>│  (graph)     │     │  (documents, │
│              │     │              │     │   traffic,   │
│              │     │  Entities &  │     │   audit)     │
│              │────>│  Relations   │     │              │
└──────┬───────┘     └──────────────┘     └──────────────┘
       │                                        ▲
       │                                        │
       └────────────────────────────────────────┘
         Writes to both via application-level
         dual-write or change-data-capture (CDC)
```

Synchronisation options:
1. **Application-level dual-write**: Write to Neo4j first (source of truth for relationships), then write supplementary data to PostgreSQL. Use a transactional outbox pattern for reliability.
2. **Change Data Capture**: Use Debezium on PostgreSQL or Neo4j Streams to propagate changes between the two stores asynchronously.
3. **Neo4j as read model**: Write everything to PostgreSQL first, then project relationship data into Neo4j using a sync worker. Neo4j becomes a materialised graph view.

---

## Pros and Cons

### Pros

1. **Blast radius analysis is a first-class operation**: "Which consumers are affected if we sunset version X, and what are their transitive dependencies?" is a single Cypher query with variable-depth traversal. In SQL, this requires recursive CTEs across multiple tables.

2. **Dependency chain discovery**: APIs often depend on other APIs. A graph naturally represents `ServiceA -> UserAPI v2.1 -> AuthAPI v3.0 -> TokenService v1.2`. Finding the full dependency chain is `MATCH path = (start)-[:DEPENDS_ON*1..10]->(end)`.

3. **Migration path planning**: Finding the optimal migration path from v1.0 to v4.0 through intermediate versions is a shortest-path graph algorithm, built into Neo4j.

4. **Visual exploration**: Neo4j's built-in browser and Bloom visualisation tool let platform teams visually explore the API dependency graph, consumer relationships, and breaking change impacts. This is powerful for stakeholder communication.

5. **Schema-free evolution**: Adding new relationship types (e.g. `TESTED_AGAINST`, `GENERATED_FROM`, `BLOCKED_BY`) or new node properties requires no migrations. The graph schema is additive.

6. **Natural fit for AI features**: The AI-native features described in the README (consumer identification, impact ranking, migration prediction) map naturally to graph algorithms: PageRank for consumer importance, community detection for related API clusters, and centrality measures for identifying critical dependencies.

7. **Pattern matching**: Cypher's pattern matching makes complex queries readable. Finding "consumers who subscribe to a deprecated version, have not started migrating, and have high traffic" is expressed as a visual pattern rather than a multi-table join.

### Cons

1. **Operational complexity**: Running Neo4j alongside PostgreSQL doubles the database operations burden. Two databases to monitor, back up, scale, and secure. Two query languages for the team to learn.

2. **Data consistency**: Keeping Neo4j and PostgreSQL in sync requires careful engineering. Without strong consistency guarantees between the two stores, the graph may diverge from the operational data.

3. **Transaction model differences**: Neo4j uses ACID transactions within a single database, but cross-database transactions (Neo4j + PostgreSQL) require distributed transaction patterns (saga, outbox) that add complexity.

4. **Write throughput**: Neo4j's write throughput is lower than PostgreSQL's for bulk operations. Ingesting millions of traffic records or bulk-updating consumer subscriptions is faster in PostgreSQL.

5. **Limited time-series support**: Neo4j is not designed for time-series workloads. Traffic analytics, audit logs, and temporal queries over large time ranges belong in PostgreSQL or a dedicated time-series database.

6. **Smaller ecosystem**: Neo4j's ORM and migration tooling is less mature than PostgreSQL's. Cypher expertise is less common than SQL. Hiring and onboarding take longer.

7. **Cost**: Neo4j Enterprise (required for clustering and production features) is commercially licensed. The Community Edition has limitations on clustering and advanced features. AuraDB (managed cloud) pricing can be significant at scale.

8. **Over-engineering risk**: If the API dependency graph is shallow (most consumers directly use one API version with no transitive dependencies), the graph database adds complexity without proportional benefit. The graph approach shines when dependency chains are deep and complex.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Graph Database** | Neo4j 5.x (Enterprise or AuraDB managed) |
| **Relational Database** | PostgreSQL 16+ (for traffic, specs, audit) |
| **Graph ORM** | Neo4j JavaScript Driver or neo4j-graphql library |
| **Visualisation** | Neo4j Bloom for stakeholder dashboards; custom D3.js for embedded visualisations |
| **Sync Layer** | Debezium CDC or application-level outbox pattern |
| **Graph Algorithms** | Neo4j Graph Data Science library (PageRank, community detection, shortest path) |
| **Cache** | Redis for caching frequent graph traversal results |
| **Time-series** | TimescaleDB extension on PostgreSQL for traffic analytics |

### Alternative Graph Technologies

| Option | Consideration |
|--------|--------------|
| **Amazon Neptune** | Managed graph DB, good if already on AWS, but Gremlin/SPARQL syntax is less readable than Cypher |
| **Memgraph** | Cypher-compatible, in-memory, faster for real-time queries but smaller ecosystem |
| **Apache AGE** | PostgreSQL extension adding graph query capabilities, avoids a second database but less mature |
| **FalkorDB** | Open-source graph DB with Cypher support, lighter weight than Neo4j |

**Apache AGE deserves special mention**: It adds Cypher query support directly to PostgreSQL, potentially eliminating the need for a separate Neo4j instance. If the graph query requirements are moderate (not requiring Neo4j's advanced graph algorithms), AGE can provide graph traversal within a single PostgreSQL deployment.

---

## Migration and Scaling Considerations

1. **Start with Apache AGE**: If the team wants graph capabilities without the operational overhead of Neo4j, start with Apache AGE as a PostgreSQL extension. This provides Cypher queries within the existing PostgreSQL deployment. Migrate to Neo4j only if graph query complexity or performance demands it.

2. **Graph-first, sync later**: Build the core API registry and consumer tracking in Neo4j first. Add PostgreSQL for traffic analytics and spec storage as those features are built. Use application-level dual-write initially; introduce CDC when the sync complexity justifies it.

3. **Neo4j clustering**: For production, use Neo4j Causal Clustering (3+ cores) for high availability. Read replicas handle dashboard and analytics queries; the core cluster handles writes.

4. **Graph algorithm pipeline**: Use Neo4j's Graph Data Science library to periodically compute:
   - **PageRank** on consumer nodes to identify the most critical consumers
   - **Betweenness centrality** on API nodes to find APIs that are bottlenecks
   - **Community detection** to identify clusters of tightly coupled APIs
   - **Shortest path** for migration route planning

5. **Data lifecycle**: Archive retired API versions and their relationships to a cold graph or export to CSV/Parquet for long-term storage. Keep the active graph lean for query performance.

6. **PostgreSQL scaling**: Apply the same scaling strategies as Suggestion 1 (partitioning, read replicas, TimescaleDB) for the PostgreSQL companion. The graph database handles the relationship-heavy queries; PostgreSQL handles volume-heavy data.

7. **Multi-tenancy**: Use Neo4j's multi-database feature (Enterprise) to isolate tenants in separate databases, or use `organisation_id` properties on all nodes with query-level filtering. PostgreSQL uses RLS as in other suggestions.
