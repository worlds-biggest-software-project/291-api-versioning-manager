# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: API Versioning Manager (Candidate #291)
> Approach: Event Sourcing with Command Query Responsibility Segregation (CQRS)

---

## Summary

An event-sourced architecture where every change to the API versioning lifecycle is captured as an immutable domain event in an append-only event store. The system separates write operations (commands that produce events) from read operations (queries against materialised projections). This approach is a natural fit for an API versioning manager because the domain is inherently about tracking change over time: version releases, deprecation announcements, consumer migrations, and sunset enforcement are all temporal events whose full history is business-critical.

Rather than storing "current state" and discarding how we got there, every state transition (e.g. "Version v2.1 was deprecated", "Consumer X acknowledged the deprecation notice", "Consumer Y completed migration") is an event in the log. Read-optimised projections are built from these events to serve dashboards, API responses, and analytics.

---

## Key Entities and Relationships

### Aggregate Roots and Event Streams

The system is organised around four primary aggregate roots, each with its own event stream:

```
ApiLifecycle         -- aggregate for an API and its versions
ConsumerRelationship -- aggregate for a consumer's interaction with APIs
DeprecationProcess   -- aggregate for managing a deprecation campaign
MigrationCampaign    -- aggregate for tracking migration of consumers between versions
```

### Event Store Schema

```sql
-- Central append-only event store
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,           -- aggregate instance ID
    stream_type     VARCHAR(64) NOT NULL,     -- aggregate type name
    event_type      VARCHAR(128) NOT NULL,    -- e.g. 'ApiVersionPublished'
    event_data      JSONB NOT NULL,           -- event payload
    metadata        JSONB NOT NULL DEFAULT '{}',
        -- correlation_id, causation_id, actor, timestamp, etc.
    version         BIGINT NOT NULL,          -- per-stream sequence number
    global_position BIGSERIAL,               -- global ordering
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, version)              -- optimistic concurrency
);

CREATE INDEX idx_event_stream ON event_store(stream_id, version);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_global ON event_store(global_position);
CREATE INDEX idx_event_time ON event_store(created_at);

-- Snapshot store for performance (optional)
CREATE TABLE snapshot_store (
    stream_id       UUID PRIMARY KEY,
    stream_type     VARCHAR(64) NOT NULL,
    snapshot_data   JSONB NOT NULL,
    version         BIGINT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain Events

#### ApiLifecycle Aggregate Events

```typescript
// Events for API registration and version lifecycle
interface ApiRegistered {
  apiId: string;
  organisationId: string;
  name: string;
  apiType: 'REST' | 'GraphQL' | 'AsyncAPI' | 'gRPC';
  versioningStrategy: 'SEMVER' | 'DATE_BASED' | 'URI_PATH' | 'HEADER';
}

interface ApiVersionDrafted {
  apiId: string;
  versionId: string;
  versionLabel: string;
  specFormat: string;
  specChecksum: string;
}

interface ApiVersionPublished {
  apiId: string;
  versionId: string;
  versionLabel: string;
  releaseDate: string;
  endpointCount: number;
}

interface ApiVersionDeprecated {
  apiId: string;
  versionId: string;
  deprecatedAt: string;
  sunsetDate: string;
  replacementVersionId: string | null;
  reason: string;
}

interface ApiVersionSunset {
  apiId: string;
  versionId: string;
  sunsetAt: string;
  remainingConsumerCount: number;
}

interface ApiVersionRetired {
  apiId: string;
  versionId: string;
  retiredAt: string;
}

interface BreakingChangeDetected {
  apiId: string;
  sourceVersionId: string;
  targetVersionId: string;
  changeType: string;
  severity: 'ERROR' | 'WARNING' | 'INFO';
  endpointPath: string;
  fieldPath: string | null;
  description: string;
}

interface OpenApiSpecUploaded {
  apiId: string;
  versionId: string;
  specFormat: string;
  specChecksum: string;
  endpointsParsed: number;
}

interface ChangelogGenerated {
  apiId: string;
  sourceVersionId: string;
  targetVersionId: string;
  entries: Array<{ type: string; description: string }>;
}
```

#### ConsumerRelationship Aggregate Events

```typescript
interface ConsumerRegistered {
  consumerId: string;
  organisationId: string;
  name: string;
  consumerType: 'SERVICE' | 'TEAM' | 'EXTERNAL_CLIENT' | 'SDK';
  contactEmail: string;
}

interface ConsumerSubscribedToVersion {
  consumerId: string;
  apiVersionId: string;
  subscribedAt: string;
}

interface ConsumerTrafficRecorded {
  consumerId: string;
  endpointId: string;
  periodStart: string;
  periodEnd: string;
  requestCount: number;
  errorCount: number;
}

interface ConsumerMigrationStarted {
  consumerId: string;
  sourceVersionId: string;
  targetVersionId: string;
  startedAt: string;
}

interface ConsumerMigrationCompleted {
  consumerId: string;
  sourceVersionId: string;
  targetVersionId: string;
  completedAt: string;
}

interface ConsumerUnsubscribedFromVersion {
  consumerId: string;
  apiVersionId: string;
  reason: string;
}
```

#### DeprecationProcess Aggregate Events

```typescript
interface DeprecationPolicyCreated {
  policyId: string;
  apiVersionId: string;
  deprecatedAt: string;
  sunsetDate: string;
  enforcementLevel: 'ADVISORY' | 'WARNING' | 'BLOCKING';
}

interface DeprecationNotificationSent {
  policyId: string;
  consumerId: string;
  channel: 'EMAIL' | 'WEBHOOK' | 'HEADER' | 'DASHBOARD';
  sentAt: string;
  messageDigest: string;
}

interface DeprecationNotificationAcknowledged {
  policyId: string;
  consumerId: string;
  acknowledgedAt: string;
}

interface SunsetDateExtended {
  policyId: string;
  previousSunsetDate: string;
  newSunsetDate: string;
  reason: string;
}

interface SunsetEnforced {
  policyId: string;
  apiVersionId: string;
  enforcedAt: string;
  blockedConsumerCount: number;
}
```

#### MigrationCampaign Aggregate Events

```typescript
interface MigrationGuideCreated {
  guideId: string;
  sourceVersionId: string;
  targetVersionId: string;
  generatedBy: 'MANUAL' | 'AI_GENERATED' | 'HYBRID';
}

interface ConsumerImpactAssessed {
  campaignId: string;
  consumerId: string;
  breakingChangeIds: string[];
  impactLevel: 'HIGH' | 'MEDIUM' | 'LOW';
  affectedEndpoints: string[];
}

interface CompatibilityTestExecuted {
  campaignId: string;
  sourceVersionId: string;
  targetVersionId: string;
  testType: 'BACKWARD' | 'FORWARD' | 'CONTRACT';
  status: 'PASSED' | 'FAILED';
  failureCount: number;
}
```

### Read Model Projections

Projections are materialised views rebuilt from events, optimised for specific query patterns:

```sql
-- Projection: Current state of all API versions (denormalized for fast reads)
CREATE TABLE projection_api_version_summary (
    api_id              UUID NOT NULL,
    api_name            VARCHAR(255),
    organisation_id     UUID,
    version_id          UUID PRIMARY KEY,
    version_label       VARCHAR(128),
    status              VARCHAR(32),
    release_date        DATE,
    deprecated_at       TIMESTAMPTZ,
    sunset_date         TIMESTAMPTZ,
    replacement_version VARCHAR(128),
    endpoint_count      INT DEFAULT 0,
    active_consumer_count INT DEFAULT 0,
    total_request_count BIGINT DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Consumer dashboard view
CREATE TABLE projection_consumer_dashboard (
    consumer_id         UUID PRIMARY KEY,
    consumer_name       VARCHAR(255),
    active_subscriptions INT DEFAULT 0,
    deprecated_subscriptions INT DEFAULT 0,
    pending_migrations  INT DEFAULT 0,
    completed_migrations INT DEFAULT 0,
    last_traffic_at     TIMESTAMPTZ,
    migration_risk_score FLOAT,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Deprecation campaign status
CREATE TABLE projection_deprecation_status (
    policy_id           UUID PRIMARY KEY,
    api_version_id      UUID,
    version_label       VARCHAR(128),
    deprecated_at       TIMESTAMPTZ,
    sunset_date         TIMESTAMPTZ,
    total_consumers     INT DEFAULT 0,
    notified_consumers  INT DEFAULT 0,
    acknowledged_consumers INT DEFAULT 0,
    migrated_consumers  INT DEFAULT 0,
    at_risk_consumers   INT DEFAULT 0,
    enforcement_level   VARCHAR(32),
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: Breaking change impact matrix
CREATE TABLE projection_breaking_change_impact (
    breaking_change_id  UUID NOT NULL,
    change_type         VARCHAR(64),
    severity            VARCHAR(16),
    endpoint_path       VARCHAR(2048),
    consumer_id         UUID NOT NULL,
    consumer_name       VARCHAR(255),
    impact_level        VARCHAR(16),
    migration_status    VARCHAR(32),
    last_traffic_at     TIMESTAMPTZ,
    request_count_30d   BIGINT,
    PRIMARY KEY (breaking_change_id, consumer_id)
);

-- Projection: Traffic analytics (time-bucketed)
CREATE TABLE projection_traffic_analytics (
    endpoint_id         UUID NOT NULL,
    consumer_id         UUID NOT NULL,
    time_bucket         TIMESTAMPTZ NOT NULL,
    request_count       BIGINT DEFAULT 0,
    error_count         BIGINT DEFAULT 0,
    PRIMARY KEY (endpoint_id, consumer_id, time_bucket)
);
```

### Command Handlers (Write Side)

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway    │───>│ Command Handler  │───>│   Event Store   │
│ (POST/PUT/DEL)  │    │   (validates &   │    │ (append-only)   │
└─────────────────┘    │   emits events)  │    └────────┬────────┘
                       └─────────────────┘             │
                                                       ▼
                                              ┌─────────────────┐
                                              │ Event Bus        │
                                              │ (Kafka/NATS)     │
                                              └────────┬────────┘
                                                       │
                              ┌─────────────┬──────────┼──────────┐
                              ▼             ▼          ▼          ▼
                       ┌───────────┐ ┌───────────┐ ┌────────┐ ┌────────┐
                       │Projection │ │Projection │ │Notifi- │ │AI      │
                       │Builder:   │ │Builder:   │ │cation  │ │Analysis│
                       │Dashboard  │ │Analytics  │ │Service │ │Service │
                       └───────────┘ └───────────┘ └────────┘ └────────┘
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design**: Every state change is an immutable event. Compliance audit trails, which are listed as a backlog feature, come for free. You can answer "who deprecated this API, when, and why?" by querying the event store directly.

2. **Temporal queries are trivial**: "What was the state of all deprecation campaigns on March 15th?" is answered by replaying events up to that timestamp. This is extremely valuable for an API lifecycle manager where understanding historical state is a core requirement.

3. **Natural fit for the domain**: API versioning is fundamentally about tracking state transitions over time. Events like `ApiVersionDeprecated`, `ConsumerMigrationCompleted`, and `SunsetEnforced` map directly to business concepts without impedance mismatch.

4. **Decoupled read/write scaling**: Write throughput (event ingestion) scales independently from read throughput (projection queries). Traffic analytics can be served from a read-optimised store without affecting the command path.

5. **Event-driven integrations**: Deprecation notifications, AI-powered impact analysis, and migration guide generation can subscribe to the event bus and react asynchronously. Adding a new integration (e.g. Slack notifications, PagerDuty alerts) requires only a new event subscriber.

6. **Projection flexibility**: Different teams can build different read models from the same events. A consumer-facing dashboard, an internal analytics view, and a compliance report can each have their own optimised projection.

7. **Debugging and replay**: If a projection is incorrect, it can be rebuilt from scratch by replaying the event stream. This eliminates an entire class of data corruption bugs.

### Cons

1. **Operational complexity**: Running an event store, an event bus (Kafka/NATS), and multiple projection builders is significantly more infrastructure than a single PostgreSQL database. The team needs event sourcing expertise.

2. **Eventual consistency**: Read projections are updated asynchronously. A consumer who just migrated might not immediately see their status change on the dashboard. This requires careful UX design to set expectations.

3. **Event schema evolution**: Events are immutable, but event schemas must evolve. Adding a field to `ApiVersionPublished` after 10,000 events have been written requires upcasting logic. This is manageable but adds ongoing maintenance.

4. **Query complexity**: Ad-hoc analytical queries across multiple aggregates require joining across projections, which may not have been designed for that specific query. Adding a new dashboard widget may require building a new projection.

5. **Learning curve**: Event sourcing and CQRS are unfamiliar to many developers. The team must understand eventual consistency, idempotent event handlers, and projection rebuilds. Hiring and onboarding are harder.

6. **Traffic log volume**: High-frequency `ConsumerTrafficRecorded` events could overwhelm the event store. These may need to be handled as a separate "hot" stream with aggressive snapshotting or moved to a dedicated time-series store.

7. **Testing complexity**: Testing event-sourced systems requires verifying both command-side behaviour (correct events emitted) and projection-side behaviour (correct read model state). Integration tests are more elaborate.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event Store** | EventStoreDB (purpose-built) or PostgreSQL with append-only table |
| **Event Bus** | Apache Kafka (durable, partitioned) or NATS JetStream (lighter weight) |
| **Projection Database** | PostgreSQL for structured projections; Redis for real-time dashboard caches |
| **Framework** | Axon Framework (Java/Kotlin), Marten (C#/.NET), or custom with Node.js + pg |
| **Serialisation** | JSON with explicit schema versions in event metadata; consider Avro for production |
| **Snapshotting** | Every 100 events per aggregate, stored in `snapshot_store` |
| **Monitoring** | Track projection lag (time between event publication and projection update) |

---

## Migration and Scaling Considerations

1. **Start simple**: Begin with PostgreSQL as both the event store and projection database. The append-only table pattern works well up to millions of events. Introduce Kafka and separate projection stores only when scale demands it.

2. **Event versioning strategy**: Adopt weak schema versioning from the start. Every event includes a `schemaVersion` field in its metadata. Upcasters transform old event shapes to current shapes during replay.

3. **Projection rebuild strategy**: Design all projections to be fully rebuildable from the event stream. Maintain a `projection_checkpoint` table that tracks the last processed `global_position` for each projection. Rebuilds zero out the projection and replay from position 0.

4. **Traffic event handling**: For high-volume consumer traffic data, use a separate "hot" event stream with shorter retention. Aggregate traffic into hourly/daily summaries in the projection and discard raw events after processing. Alternatively, route traffic events to a dedicated time-series database.

5. **Snapshotting cadence**: Snapshot aggregates with long event histories (APIs with hundreds of versions). The `snapshot_store` caches the latest aggregate state to avoid replaying the full stream on every command.

6. **Multi-region**: Event stores are append-only and conflict-free, making them well-suited to multi-region active-passive replication. In an active-active setup, use partition keys (e.g. `organisation_id`) to assign aggregates to regions.

7. **Gradual adoption**: CQRS does not require event sourcing everywhere. Start with event sourcing for the `DeprecationProcess` and `MigrationCampaign` aggregates (where audit trail is most valuable) and use standard CRUD for less critical entities like `Organisation`.
