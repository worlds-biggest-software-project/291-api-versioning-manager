# API Versioning Manager — Phased Development Plan

> Project: 291-api-versioning-manager · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

API Versioning Manager is an AI-native, self-hostable platform that tracks API versions, detects breaking changes from OpenAPI/AsyncAPI/GraphQL contracts, manages deprecation timelines (RFC 9745 `Deprecation` + RFC 8594 `Sunset`), identifies which consumers still call deprecated endpoints from traffic data, and generates per-consumer migration guidance. It integrates with existing gateways (Kong, AWS, Azure, Apigee) rather than replacing them.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The data model adopts **Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the primary store, because it gives a relational backbone for the well-defined entities (APIs, versions, consumers, policies) and JSONB flexibility for variable-shape content (specs across REST/GraphQL/AsyncAPI, AI-generated analysis, test reports) without operating a second database. The graph approach from Suggestion 4 is deferred to an optional analytics module (Phase 9, via the `apache_age` PostgreSQL extension) so no polyglot persistence is required for the MVP.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The domain is API/contract-tooling-heavy, not ML-heavy. The richest OpenAPI/AsyncAPI/GraphQL ecosystem lives in JS (`@apidevtools/swagger-parser`, `@graphql-tools`, `@stoplight/spectral`). `oasdiff` (the best breaking-change engine) ships a CLI we shell out to. One language spans API server, CLI, and any future web UI. |
| Runtime / package manager | Node.js 22 + pnpm workspaces | pnpm gives a fast, disk-efficient monorepo for the `server`, `cli`, `core`, and `web` packages with a shared `core` library. |
| API framework | Fastify 5 + `@fastify/swagger` | High throughput, first-class JSON Schema validation per route, and auto-generates an OpenAPI 3.1 spec for the management API — the product should dogfood OAS for its own surface. |
| Validation / types | Zod + `zod-to-json-schema` | Single source of truth for runtime validation and TypeScript types; feeds Fastify route schemas and validates JSONB column shapes (per Suggestion 3's required validation discipline). |
| Database | PostgreSQL 16 | Per data-model Suggestion 3: relational integrity + JSONB + GIN indexes + native range partitioning, all in one engine. `apache_age` extension reserved for optional graph analytics. |
| ORM / query builder | Drizzle ORM | First-class `jsonb()` columns, generated SQL migrations (`drizzle-kit`), typed queries, and no heavy runtime — fits the hybrid relational+JSONB model better than Prisma's awkward JSONB typing. |
| Migrations | drizzle-kit | Versioned, repeatable SQL migrations checked into the repo from day one. |
| Task queue | BullMQ on Redis | Async workloads exist: spec ingestion/parsing, diff computation, AI calls, notification delivery, traffic ingestion. BullMQ gives retries, scheduled jobs (notification schedules, sunset enforcement sweeps), and dead-letter handling. |
| Cache / queue backend | Redis 7 | Backs BullMQ and caches expensive diff/blast-radius results. |
| Breaking-change engine | `oasdiff` (Go CLI, vendored binary) | Open-source, 450+ rules, three severity levels (ERR/WARN/INFO), OAS 3.0/3.1; reused rather than reinvented. GraphQL diff uses `@graphql-inspector/core`; AsyncAPI uses `@asyncapi/parser` + custom rules. |
| Spec parsing | `@apidevtools/swagger-parser` (OpenAPI), `@asyncapi/parser` (AsyncAPI), `graphql` + `@graphql-inspector/core` (GraphQL) | Standard, well-maintained parsers for each contract type named in `standards.md`. |
| Linting (governance) | `@stoplight/spectral-core` | Rule-based API linting; lets the governance rulebook (Phase 8) compile to Spectral rulesets aligned with Microsoft REST API Guidelines and OWASP API5:2023. |
| LLM integration | Anthropic SDK (`@anthropic-ai/sdk`), provider-abstracted | Powers per-consumer migration guides, NL deprecation notices, governance-rulebook parsing, migration-risk prediction. Abstracted behind an `LlmProvider` interface so OpenAI/local models can be swapped. Prompt caching enabled for repeated spec context. |
| Auth | OAuth 2.0 (RFC 6749) via `@fastify/oauth2` + API keys; bearer JWT for the management API | `standards.md` requires OAuth 2.0 for the management API and for authenticating against downstream gateways; RFC 9700 best practices applied (PKCE, short-lived tokens). |
| Frontend | Next.js 15 (React) dashboard — deferred to Phase 10 | A web dashboard is "should-have", not MVP. API + CLI ship first; the dashboard consumes the same management API. |
| CLI | `commander` + `ora` + `chalk` | The CLI is a first-class MVP surface for CI/CD (register specs, run diff gate, generate changelog). Mirrors `oasdiff`/`redocly` UX expectations. |
| Containerisation | Docker + docker-compose | Self-hosted is a primary deployment mode; compose wires server + worker + Postgres + Redis. |
| Testing | Vitest (unit/integration), `testcontainers` (real Postgres/Redis), Supertest-style Fastify `inject` (HTTP), Playwright (dashboard e2e, Phase 10) | Vitest is fast and TS-native; testcontainers gives real DB integration tests without mocking SQL. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard TS toolchain; enforced in CI and per phase Definition of Done. |
| Output formats | OpenAPI 3.1 (management API + ingestion), SARIF 2.1.0 (diff/lint findings for CI), RFC 9745 `Deprecation` + RFC 8594 `Sunset` headers, JSON & Markdown (changelogs, guides) | Aligns with `standards.md`; SARIF makes CI findings render in GitHub/GitLab natively. |
| Observability | `pino` structured logs + OpenTelemetry traces | Operational visibility for ingestion/diff/notification pipelines. |

### Project Structure

```
api-versioning-manager/
├── pnpm-workspace.yaml
├── package.json                      # root scripts (lint, test, build, dev)
├── tsconfig.base.json
├── .env.example
├── docker-compose.yml                # postgres + redis + server + worker
├── Dockerfile                        # multi-stage build for server & worker
├── drizzle.config.ts
├── bin/
│   └── oasdiff                       # vendored oasdiff binary (per-arch in CI)
├── packages/
│   ├── core/                         # framework-agnostic domain logic (no HTTP)
│   │   ├── src/
│   │   │   ├── db/
│   │   │   │   ├── schema.ts          # Drizzle table + JSONB definitions
│   │   │   │   ├── client.ts
│   │   │   │   └── migrations/        # generated SQL migrations
│   │   │   ├── domain/
│   │   │   │   ├── versioning/        # SemVer + date-based strategy parsing
│   │   │   │   ├── specs/             # parse/normalise OpenAPI/AsyncAPI/GraphQL
│   │   │   │   ├── diff/              # oasdiff wrapper, graphql/asyncapi diff
│   │   │   │   ├── changelog/         # changelog generation from diffs
│   │   │   │   ├── deprecation/       # policy lifecycle, header emission
│   │   │   │   ├── consumers/         # consumer + traffic ingestion + impact
│   │   │   │   ├── migration/         # migration guide assembly
│   │   │   │   ├── compatibility/     # contract / backward-compat testing
│   │   │   │   └── governance/        # rulebook → Spectral rules, linting
│   │   │   ├── ai/
│   │   │   │   ├── provider.ts        # LlmProvider interface + Anthropic impl
│   │   │   │   └── prompts/           # prompt templates
│   │   │   ├── jobs/                  # BullMQ job definitions + processors
│   │   │   ├── schemas/               # Zod schemas (shared types + JSONB shapes)
│   │   │   └── index.ts
│   │   └── test/
│   │       └── fixtures/              # sample OAS/GraphQL/AsyncAPI specs
│   ├── server/                       # Fastify management API
│   │   ├── src/
│   │   │   ├── app.ts
│   │   │   ├── routes/               # /apis, /versions, /consumers, /diffs ...
│   │   │   ├── plugins/              # auth, db, queue, errors
│   │   │   └── worker.ts             # BullMQ worker entrypoint
│   │   └── test/
│   ├── cli/                          # avm CLI (CI/CD surface)
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   └── commands/             # register, diff, changelog, gate, notify
│   │   └── test/
│   └── web/                          # Next.js dashboard (Phase 10)
└── docs/
    └── openapi.json                  # generated management API spec
```

The structure is grouped by concern (domain modules in `core`, transport in `server`/`cli`), so every later phase adds files into existing module folders without restructuring.

---

## Phase 1: Foundation & Data Layer

### Purpose
Establish the monorepo, the PostgreSQL hybrid schema, shared Zod schemas, the database client, and CI scaffolding. After this phase the project builds, lints, type-checks, runs migrations against a real Postgres, and has an empty-but-wired Fastify server with a health check. Everything downstream depends on the schema and shared types defined here.

### Tasks

#### 1.1 — Monorepo & tooling scaffold

**What**: Initialise the pnpm workspace with `core`, `server`, `cli` packages, shared TS config, ESLint/Prettier, Vitest, and root scripts.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*`.
- Root `package.json` scripts: `build` (tsc project refs), `lint`, `format`, `test`, `test:int`, `dev` (server+worker via `tsx watch`), `db:generate`, `db:migrate`.
- `tsconfig.base.json`: `"target": "ES2023"`, `"module": "NodeNext"`, `"strict": true`, `"noUncheckedIndexedAccess": true`, composite project references.
- `.env.example`:
  ```
  DATABASE_URL=postgres://avm:avm@localhost:5432/avm
  REDIS_URL=redis://localhost:6379
  PORT=8080
  AUTH_JWT_SECRET=change-me
  ANTHROPIC_API_KEY=
  LLM_PROVIDER=anthropic
  LOG_LEVEL=info
  ```
- ESLint: `typescript-eslint` recommended + `import/order`. Prettier 100-col, single quotes.

**Testing**:
- `Unit: tsc --noEmit` → exits 0 on empty packages.
- `Unit: pnpm lint` → exits 0 on scaffold.
- `Unit: pnpm test` → Vitest runs with zero tests, exits 0.

#### 1.2 — Database schema (hybrid relational + JSONB)

**What**: Define the full Drizzle schema implementing data-model Suggestion 3, with migrations.

**Design**: Drizzle definitions for all core tables. Enums are PG `text` columns constrained by Zod at the app layer (per Suggestion 3's flexibility note). Key tables (abridged Drizzle):
```typescript
export const organisation = pgTable('organisation', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  settings: jsonb('settings').$type<OrgSettings>().notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const api = pgTable('api', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisation.id),
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  apiType: text('api_type').notNull().default('REST'),       // REST|GRAPHQL|ASYNCAPI|GRPC
  versioningStrategy: text('versioning_strategy').notNull().default('SEMVER'),
                                                              // SEMVER|DATE_BASED|URI_PATH|HEADER|QUERY_PARAM
  metadata: jsonb('metadata').$type<ApiMetadata>().notNull().default({}),
  createdAt, updatedAt,
}, (t) => ({ uq: unique().on(t.organisationId, t.slug) }));

export const apiVersion = pgTable('api_version', {
  id: uuid('id').primaryKey().defaultRandom(),
  apiId: uuid('api_id').notNull().references(() => api.id, { onDelete: 'cascade' }),
  versionLabel: text('version_label').notNull(),
  status: text('status').notNull().default('DRAFT'),         // DRAFT|ACTIVE|DEPRECATED|SUNSET|RETIRED
  releaseDate: date('release_date'),
  versionMajor: integer('version_major'),
  versionMinor: integer('version_minor'),
  versionPatch: integer('version_patch'),
  spec: jsonb('spec').$type<ParsedSpec | null>(),
  endpoints: jsonb('endpoints').$type<EndpointSummary[]>().notNull().default([]),
  changelog: jsonb('changelog').$type<ChangelogEntry[]>().notNull().default([]),
  specChecksum: text('spec_checksum'),
  createdAt, updatedAt,
}, (t) => ({ uq: unique().on(t.apiId, t.versionLabel) }));
```
Remaining tables follow Suggestion 3 verbatim: `deprecation_policy` (with `policy_config` JSONB), `breaking_change` (`changes` JSONB), `version_diff` (`diff_result` + `summary_stats` JSONB, unique on source+target), `consumer` (`metadata` JSONB), `consumer_subscription` (`usage_profile` JSONB), `consumer_traffic` (PARTITION BY RANGE on `time_bucket`), `consumer_impact` (`analysis` JSONB), `deprecation_notification` (`content` JSONB), `migration_guide` (`content` JSONB), `compatibility_test` (`report` JSONB), `audit_log` (`detail` JSONB, partitioned by `occurred_at`).
- Indexes: all from Suggestion 3 (B-tree on FKs/status/release_date; GIN `jsonb_path_ops` on `endpoints`, `changelog`, `metadata`, `analysis`, `changes`, `settings`; partial indexes on deprecated versions and active subscriptions).
- Partitioning helper: a migration creates the current + next 3 monthly partitions for `consumer_traffic` and `audit_log`; a recurring job (Phase 6) provisions future partitions.

**Testing**:
- `Integration (real, testcontainers): run all migrations on fresh PG 16 → all tables + indexes exist (query pg_catalog).`
- `Integration: insert api → api_version with FK → succeeds; insert api_version with nonexistent api_id → FK violation.`
- `Integration: insert consumer_traffic row in current month → lands in correct partition.`
- `Unit: GIN index DDL present for each JSONB column listed.`

#### 1.3 — Shared Zod schemas & JSONB shape registry

**What**: Define Zod schemas for every entity and every JSONB column shape; export inferred TS types used by `$type<>()` in the schema.

**Design**:
- `schemas/jsonb.ts` exports `OrgSettings`, `ApiMetadata`, `EndpointSummary`, `ChangelogEntry`, `ParsedSpec`, `PolicyConfig`, `BreakingChangeDetail`, `ImpactAnalysis`, `NotificationContent`, `MigrationGuideContent`, `CompatibilityReport` as Zod schemas + inferred types.
- Example:
  ```typescript
  export const EndpointSummary = z.object({
    method: z.enum(['GET','POST','PUT','PATCH','DELETE','HEAD','OPTIONS']),
    path: z.string(),
    operationId: z.string().optional(),
    summary: z.string().optional(),
    deprecated: z.boolean().default(false),
    sunsetDate: z.string().datetime().nullable().default(null),
    parameters: z.array(ParameterSummary).default([]),
    requestSchemaRef: z.string().optional(),
    responseSchemas: z.record(z.string(), z.unknown()).default({}),
  });
  ```
- A `validateJsonb(column, value)` helper validates before any write to a JSONB column (enforces Suggestion 3's discipline).

**Testing**:
- `Unit: valid EndpointSummary object → parses with defaults applied.`
- `Unit: missing 'path' → ZodError naming 'path'.`
- `Unit: validateJsonb('endpoints', badArray) → throws with column + path in message.`

#### 1.4 — Fastify app skeleton, config, health, errors

**What**: Bootable Fastify server with config loading, DB plugin, structured error handler, and `/health`.

**Design**:
- `loadConfig()` parses `process.env` with a Zod `EnvSchema`; fails fast on missing required vars.
- DB plugin decorates `fastify.db` (Drizzle client over `pg.Pool`).
- Error handler maps thrown `AppError` subclasses (`NotFoundError`, `ValidationError`, `ConflictError`, `UnauthorizedError`) to RFC 9457 Problem Details JSON (`type`, `title`, `status`, `detail`, `instance`).
- `GET /health` → `{ status: 'ok', db: 'up'|'down' }` (runs `SELECT 1`).

**Testing**:
- `Integration (fastify.inject): GET /health with DB up → 200, db:'up'.`
- `Integration: GET /health with DB down → 503, db:'down'.`
- `Unit: loadConfig with missing DATABASE_URL → throws listing the var.`
- `Unit: NotFoundError → 404 Problem Details body.`

---

## Phase 2: Spec Ingestion & Version Registry

### Purpose
Make the product able to register APIs and ingest versioned contracts. This is the data foundation for everything else: every diff, changelog, deprecation, and impact analysis operates on parsed specs. After this phase a user can register an API and upload OpenAPI/AsyncAPI/GraphQL specs as versions, with endpoints parsed into the queryable `endpoints` JSONB.

### Tasks

#### 2.1 — Versioning strategy parser

**What**: Parse and order version labels under SemVer, date-based, and URI-path strategies.

**Design**:
```typescript
interface ParsedVersion { major?: number; minor?: number; patch?: number;
  date?: string; channel?: string; raw: string; }
interface VersioningStrategyHandler {
  parse(label: string): ParsedVersion;          // throws on malformed
  compare(a: ParsedVersion, b: ParsedVersion): -1 | 0 | 1;
  isBreakingBump(from: ParsedVersion, to: ParsedVersion): boolean;
}
```
- SemVer handler uses the `semver` package (SemVer 2.0.0 per `standards.md`).
- Date-based handler parses `YYYY-MM-DD` and optional `.channel` (e.g. `2026-04-22.dahlia`, the Stripe pattern); compare by date then channel.
- URI-path handler extracts `v(\d+)` from a path prefix.
- `getHandler(strategy)` factory.

**Testing**:
- `Unit: SemVer compare('2.1.0','2.0.9') → 1; isBreakingBump 1.x→2.0 → true; 1.0→1.1 → false.`
- `Unit: date-based parse('2026-04-22.dahlia') → {date,channel}.`
- `Unit: malformed '2.x' under SemVer → throws.`

#### 2.2 — Spec parsers & normaliser

**What**: Parse OpenAPI 3.0/3.1, AsyncAPI 3.x, and GraphQL SDL into a common `ParsedSpec` + `EndpointSummary[]`.

**Design**:
- `parseSpec(content: string, format: SpecFormat): ParsedSpec` dispatches by format.
- OpenAPI: `@apidevtools/swagger-parser.validate()` (dereferenced), then extract operations → `EndpointSummary[]`, reading `deprecated`, parameters, request/response schemas; capture `info.version`.
- AsyncAPI: `@asyncapi/parser`; channels/operations → endpoint-like summaries (`method` = `SEND`/`RECEIVE`).
- GraphQL: `graphql.buildSchema()`; enumerate types/fields, capturing `@deprecated` directives → summaries keyed by `Type.field`.
- Compute `specChecksum` = SHA-256 of canonicalised content.
- All failures raise `SpecParseError` with the parser's location info.

**Testing**:
- `Fixture: petstore OAS 3.1 → N endpoints, deprecated flag on the deprecated op.`
- `Fixture: AsyncAPI 3.1 streetlights → channels parsed.`
- `Fixture: GraphQL SDL with @deprecated field → summary marks deprecated + reason.`
- `Unit: invalid YAML → SpecParseError with line info.`
- `Unit: identical content twice → identical checksum; whitespace-only change → identical checksum (canonicalised).`

#### 2.3 — API & Version management endpoints

**What**: CRUD for APIs and versions, including spec upload that triggers parsing.

**Design**: Routes (all under `/v1`, JSON, Zod-validated, OpenAPI-documented):
| Method | Path | Body / Result |
|---|---|---|
| POST | `/apis` | `{name,slug,apiType,versioningStrategy,metadata?}` → `Api` (409 on dup slug) |
| GET | `/apis` / `/apis/:id` | list / one |
| POST | `/apis/:id/versions` | `{versionLabel, spec, specFormat, releaseDate?}` → parses spec, fills `endpoints`, derives `versionMajor/minor/patch` via strategy handler; status `DRAFT` |
| GET | `/apis/:id/versions` | list ordered by strategy `compare` |
| GET | `/versions/:id` | version with endpoints |
| POST | `/versions/:id/publish` | `DRAFT→ACTIVE`; sets `releaseDate` if null |
| DELETE | `/versions/:id` | only if `DRAFT` |
- Writes go through `validateJsonb` before persistence. Every state change writes an `audit_log` row.

**Testing**:
- `Integration: POST /apis then POST version with petstore → 201, endpoints populated, version_* derived.`
- `Integration: duplicate (api,versionLabel) → 409.`
- `Integration: publish DRAFT → status ACTIVE + audit_log row 'CREATED'/'UPDATED'.`
- `Integration: DELETE ACTIVE version → 409.`
- `Integration: upload malformed spec → 422 Problem Details from SpecParseError.`

#### 2.4 — CLI: register & push

**What**: `avm` CLI commands to register an API and push a spec from CI.

**Design**:
- `avm api create --name --slug --type --strategy`
- `avm version push --api <slug> --label <v> --spec <path> [--publish]` reads file, POSTs to server (base URL + token from `AVM_URL`/`AVM_TOKEN` env or `--url`/`--token`).
- Human-readable table output by default; `--json` for machine output; non-zero exit on HTTP error.

**Testing**:
- `Integration (mocked server via nock): version push --spec petstore.yaml → POST body matches, exit 0.`
- `Unit: missing --spec → usage error, exit 2.`
- `E2E (testcontainers + real server): api create then version push → version visible via GET.`

---

## Phase 3: Breaking-Change Detection & Changelog

### Purpose
Ship the analytical heart of the product. Given two versions, detect breaking and non-breaking changes and produce a changelog. This is the capability incumbents bury inside gateways; here it is a first-class, standards-aligned (SARIF) output usable as a CI gate.

### Tasks

#### 3.1 — Diff engine (OpenAPI via oasdiff; GraphQL; AsyncAPI)

**What**: Compute a structured diff and breaking-change list between two versions of the same API.

**Design**:
```typescript
type Severity = 'ERROR' | 'WARNING' | 'INFO';   // maps to oasdiff ERR/WARN/INFO
interface DiffChange { id: string; type: string; severity: Severity;
  breaking: boolean; path?: string; method?: string; fieldPath?: string;
  description: string; }
interface DiffResult { source: string; target: string;
  changes: DiffChange[];
  summary: { added: number; removed: number; changed: number; deprecated: number;
             breaking: number }; }
function diffVersions(source: ApiVersion, target: ApiVersion): Promise<DiffResult>;
```
- OpenAPI: shell out to vendored `bin/oasdiff` with `breaking --format json`; parse to `DiffChange[]`. `breaking === severity==='ERROR'`.
- GraphQL: `@graphql-inspector/core` `diff()`; map its `Change.criticality` (BREAKING/DANGEROUS/NON_BREAKING) to severity.
- AsyncAPI: custom rule set (channel removed, message schema field removed/required-added, operation removed) → `DiffChange[]`.
- Results cached in `version_diff` (unique source+target); recompute only on checksum change.

**Testing**:
- `Fixture: petstore v1 vs v2 (removed endpoint, removed required field) → breaking count ≥ 2 with correct types.`
- `Fixture: additive change (new optional field) → breaking:false, INFO.`
- `Fixture: GraphQL field removed → one BREAKING change at Type.field.`
- `Integration: second diff of same pair (unchanged checksums) → served from version_diff cache, oasdiff not invoked (spy).`

#### 3.2 — Changelog generation

**What**: Turn a `DiffResult` into Keep-a-Changelog-style entries persisted on the target version.

**Design**:
- `generateChangelog(diff): ChangelogEntry[]` mapping change types to categories `ADDED|CHANGED|DEPRECATED|REMOVED|FIXED|SECURITY`.
- Rendered to Markdown via `renderChangelogMarkdown(entries)`.
- Persisted into `api_version.changelog`; exposed `GET /versions/:id/changelog?format=json|md`.

**Testing**:
- `Unit: removed endpoint → REMOVED entry; new endpoint → ADDED; deprecated flag → DEPRECATED.`
- `Integration: GET changelog?format=md → valid Markdown with category headings.`

#### 3.3 — Diff endpoints, SARIF output & CI gate

**What**: Expose diffs over HTTP and via CLI with a pass/fail gate and SARIF for CI annotations.

**Design**:
- `GET /apis/:id/diff?from=<label>&to=<label>` → `DiffResult` (computes + caches).
- `GET /apis/:id/diff?...&format=sarif` → SARIF 2.1.0: each `DiffChange` becomes a `result` with `ruleId=type`, `level` (error/warning/note from severity), `message.text=description`, `locations` synthesised from `path`/`fieldPath`.
- CLI `avm diff --api <slug> --from --to [--fail-on error|warning] [--format sarif|table]` exits non-zero when severity ≥ threshold — the CI breaking-change gate.

**Testing**:
- `Integration: GET diff?format=sarif → schema-valid SARIF 2.1.0 (validate against SARIF JSON Schema).`
- `E2E: avm diff --fail-on error on breaking pair → exit 1; on additive pair → exit 0.`
- `Unit: severity→SARIF level mapping (ERROR→error, WARNING→warning, INFO→note).`

---

## Phase 4: Deprecation Lifecycle & Standards Headers

### Purpose
Add the deprecation timeline management that incumbents treat as an afterthought, with machine-readable RFC 9745/8594 headers. After this phase a version can be deprecated with a sunset date, the system enforces the lifecycle state machine, and provides ready-to-serve headers and a public deprecation feed.

### Tasks

#### 4.1 — Deprecation policy & version lifecycle state machine

**What**: Create deprecation policies and drive the version status transitions.

**Design**:
- State machine: `DRAFT → ACTIVE → DEPRECATED → SUNSET → RETIRED`; legal transitions enforced in `transitionVersion(version, to)`; illegal → `ConflictError`.
- `POST /versions/:id/deprecate` body `{ sunsetDate, replacementVersionId?, enforcementLevel: ADVISORY|WARNING|BLOCKING, policyConfig? }` → creates `deprecation_policy`, sets version `DEPRECATED`, sets `deprecatedAt=now`. `policyConfig` carries `notificationSchedule` (default `["30d","14d","7d","1d"]`), `exemptedConsumers`, `customMessageTemplate`.
- `POST /versions/:id/sunset` (manual) and a scheduled sweep (Phase 6) move `DEPRECATED→SUNSET` at/after sunset date.
- Every transition → `audit_log`.

**Testing**:
- `Unit: transition ACTIVE→SUNSET (skipping DEPRECATED) → ConflictError.`
- `Integration: deprecate ACTIVE version → status DEPRECATED, policy row created, defaults applied.`
- `Integration: deprecate with past sunsetDate → 422.`

#### 4.2 — RFC 9745 / RFC 8594 header service

**What**: Produce and parse `Deprecation` and `Sunset` headers.

**Design**:
- `buildDeprecationHeaders(version): Record<string,string>` → for a DEPRECATED/SUNSET version returns:
  - `Deprecation: @<unix-ts>` (RFC 9745, deprecation timestamp) or `Deprecation: true`,
  - `Sunset: <HTTP-date>` (RFC 8594),
  - `Link: <migration-guide-url>; rel="sunset"` and `rel="deprecation"` / `successor-version` when a replacement exists.
- `parseSunsetHeaders(headers)` → `{ deprecation?: Date|boolean; sunset?: Date; links: {...} }` for monitoring third-party APIs.
- `GET /apis/:id/deprecations` → public JSON feed of deprecated versions + sunset dates; `GET /versions/:id/headers` returns the header set so a gateway plugin can copy it onto responses.

**Testing**:
- `Unit: buildDeprecationHeaders for version sunset 2026-10-01 → Sunset is RFC1123 date, Deprecation present.`
- `Unit: parseSunsetHeaders round-trips buildDeprecationHeaders output.`
- `Unit: ACTIVE version → no deprecation headers.`
- `Integration: GET /versions/:id/headers on DEPRECATED → contains Sunset + Link.`

#### 4.3 — Audit trail query API

**What**: Expose the compliance audit trail (OWASP API5:2023 inventory support).

**Design**:
- `GET /audit?entityType=&entityId=&from=&to=&action=` paginated, ordered by `occurredAt desc`.
- Read-only; backed by the partitioned `audit_log`.

**Testing**:
- `Integration: deprecate a version then GET /audit?entityId=<v> → includes the DEPRECATED action with actor.`
- `Integration: pagination cursor returns stable ordering.`

---

## Phase 5: Consumers, Traffic Ingestion & Impact Analysis

### Purpose
Solve the problem the research names as the primary unsolved one: knowing which consumers still call a deprecated version, ranked by blast radius. This phase registers consumers, ingests traffic, and computes deterministic impact analysis linking breaking changes to affected consumers.

### Tasks

#### 5.1 — Consumer registry & subscriptions

**What**: CRUD consumers and their subscriptions to versions.

**Design**:
- `POST /consumers` `{name,consumerType,contactEmail?,metadata?}` (`metadata`: `webhookUrl`, `repositoryUrl`, `slackChannel`, `sdkLanguage`, `notificationPreferences`, `tags`).
- `POST /consumers/:id/subscriptions` `{apiVersionId}` → `consumer_subscription` (unique consumer+version).
- `usageProfile` JSONB on subscription is updated by traffic ingestion (5.2), not directly.

**Testing**:
- `Integration: create consumer + subscribe → unique constraint blocks duplicate subscribe (409).`
- `Unit: consumer metadata Zod validation rejects malformed webhookUrl.`

#### 5.2 — Traffic ingestion pipeline

**What**: Ingest per-consumer/endpoint request counts into the partitioned `consumer_traffic` table and roll up `usageProfile`.

**Design**:
- `POST /traffic:ingest` accepts batched rows `{consumerId, apiVersionId, endpointPath, httpMethod, timeBucket, requestCount, errorCount, p50?, p99?}` (or NDJSON for bulk). Enqueues a BullMQ `traffic.ingest` job.
- Job upserts into `consumer_traffic` (PK conflict → add counts) and recomputes the subscription `usageProfile`: `endpointsUsed`, `lastSeenAt`, `avgDailyRequests` (30-day), `errorRate7d`.
- Optional gateway adapters (`adapters/kong.ts`, `adapters/aws.ts`) normalise gateway access logs into ingestion rows — interface `TrafficAdapter.normalise(raw): TrafficRow[]`.

**Testing**:
- `Integration: ingest 3 batches across 2 days → usageProfile.avgDailyRequests correct, lastSeenAt = latest.`
- `Integration: re-ingest same time_bucket row → counts summed, not duplicated.`
- `Unit: KongTrafficAdapter.normalise(sample log line) → correct TrafficRow.`

#### 5.3 — Deterministic impact analysis & blast-radius ranking

**What**: For a given deprecated version (or a specific breaking change), compute which consumers are affected and rank them.

**Design**:
- `computeImpact(versionId)`:
  1. Load active subscriptions to the version + each consumer's `usageProfile.endpointsUsed`.
  2. Load breaking changes where `targetVersionId` = the replacement (or all changes from this version).
  3. A consumer is impacted by a change if the change's `path`/`fieldPath` intersects their used endpoints.
  4. Write `consumer_impact` rows: `impactLevel` from a rule (HIGH if any ERROR-severity change on a used endpoint; MEDIUM for WARNING; LOW otherwise), `analysis.affectedEndpoints`, `analysis.blastRadius`.
- `blastRadiusScore = openBreakingChanges * avgDailyRequests` (per Suggestion 4's risk formula, computed in SQL).
- `GET /versions/:id/impact` → consumers ranked by `blastRadiusScore desc` with last-seen, daily traffic, sunset countdown (the headline query from Suggestion 3).

**Testing**:
- `Integration: consumer using removed endpoint → impact HIGH; consumer using untouched endpoint → not impacted.`
- `Integration: GET /versions/:id/impact → ordered by blastRadiusScore desc.`
- `Unit: blastRadiusScore = changes × dailyRequests.`
- `Integration: exempted consumer (policyConfig.exemptedConsumers) → excluded from at-risk set.`

---

## Phase 6: Async Jobs, Notifications & Scheduled Enforcement

### Purpose
Turn deprecation policies into action. A worker process runs scheduled sweeps (notification schedules, sunset enforcement, partition provisioning) and delivers notifications across channels. After this phase deprecations actively drive consumer communication and the lifecycle advances automatically.

### Tasks

#### 6.1 — Worker & scheduler

**What**: A BullMQ worker process with repeatable scheduled jobs.

**Design**:
- `worker.ts` registers processors and repeatable jobs:
  - `deprecation.sweep` (hourly): for each active policy, evaluate `notificationSchedule` offsets against `sunsetDate`; enqueue due `notification.send` jobs (idempotent via a `(policyId,consumerId,offset)` dedupe key in notification content).
  - `sunset.enforce` (hourly): move `DEPRECATED→SUNSET` when `now ≥ sunsetDate`; emit `SunsetEnforced` audit row.
  - `partition.provision` (daily): ensure next 3 monthly partitions exist for `consumer_traffic` and `audit_log`.
- Jobs are idempotent and safe to retry (BullMQ default retries with backoff; failures → dead-letter queue + log).

**Testing**:
- `Integration: policy with sunset in 7 days + schedule includes '7d' → sweep enqueues notifications once; second sweep same window → no duplicates.`
- `Integration: policy past sunsetDate → sunset.enforce sets SUNSET + audit row.`
- `Integration: partition.provision on empty future month → creates partition.`

#### 6.2 — Notification delivery

**What**: Deliver deprecation notifications via Email, Webhook, and Dashboard channels with status tracking.

**Design**:
- `NotificationChannel` interface: `send(notification): Promise<DeliveryResult>`. Implementations: `EmailChannel` (SMTP via `nodemailer`), `WebhookChannel` (signed POST to `consumer.metadata.webhookUrl`, HMAC `X-AVM-Signature`), `DashboardChannel` (in-app row only).
- `notification.send` job: render `content` (subject, `bodyMarkdown`, `personalisedMigrationSteps`), pick channels from `consumer.metadata.notificationPreferences`, persist `deprecation_notification` with `status` PENDING→SENT→DELIVERED/FAILED.
- `POST /notifications/:id/ack` (consumer-facing, token-scoped) → `acknowledged_at`.

**Testing**:
- `Integration (mocked SMTP): send → notification row SENT; SMTP error → FAILED, retried.`
- `Integration (webhook to local test server): signature header verifies with shared secret.`
- `Integration: POST ack → acknowledgedAt set; projection counts update.`

#### 6.3 — Deprecation dashboard status endpoint

**What**: Aggregate per-policy progress (total/notified/acknowledged/migrated/at-risk consumers).

**Design**:
- `GET /policies/:id/status` computes the Suggestion 2 `projection_deprecation_status` shape on demand via SQL aggregation over subscriptions, notifications, impacts.

**Testing**:
- `Integration: after notifications + one ack + one migration → counts reflect totalled/acknowledged/migrated correctly.`

---

## Phase 7: AI-Native Migration Guidance & Compatibility Testing

### Purpose
Deliver the AI-native differentiator and the contract-testing safety net. Generate per-consumer migration guides and personalised deprecation notices from diffs + observed usage, and verify backward compatibility between versions.

### Tasks

#### 7.1 — LLM provider abstraction

**What**: Provider-agnostic LLM interface with the Anthropic implementation and prompt caching.

**Design**:
```typescript
interface LlmProvider {
  complete(req: { system: string; messages: LlmMessage[];
    cacheKeyContext?: string; maxTokens?: number;
    responseSchema?: ZodType }): Promise<LlmResult>;
}
```
- `AnthropicProvider` uses `@anthropic-ai/sdk`, sets a cache breakpoint on the (large, reused) spec/diff context block so repeated per-consumer calls hit the prompt cache. `responseSchema` enforces structured JSON output (validated with Zod; one repair retry on parse failure).
- Configurable via `LLM_PROVIDER`; missing key → feature disabled, endpoints return 503 with a clear message (AI is optional, never blocks core flows).

**Testing**:
- `Unit (mocked SDK): complete() sends system + cache-control on context block.`
- `Unit: malformed JSON response → one repair retry then LlmError.`
- `Unit: no API key configured → provider reports unavailable.`

#### 7.2 — Per-consumer migration guide generation

**What**: Generate a migration guide for a (sourceVersion → targetVersion) pair, optionally personalised to one consumer's observed usage.

**Design**:
- Inputs: `DiffResult`, target spec endpoints, and (if consumer given) the consumer's `usageProfile.endpointsUsed` so the guide highlights only changes touching endpoints they actually call.
- System prompt template (`prompts/migration-guide.ts`): instructs the model to produce `MigrationGuideContent` JSON (`title`, `overview`, `breakingChanges[]`, `stepByStep[]`, `codeExamples{python,javascript,curl}`, `faq[]`) and to omit changes the consumer does not use.
- Persisted to `migration_guide` (generatedBy `AI_GENERATED`); `POST /apis/:id/migration-guide` `{from,to,consumerId?}`; `GET` returns JSON or rendered Markdown.
- Personalised NL deprecation notice reuses the same context to fill `notification.content.bodyMarkdown` + `personalisedMigrationSteps`.

**Testing**:
- `Integration (mocked LLM returning fixed JSON): guide persisted, validates against MigrationGuideContent.`
- `Integration: consumerId given → prompt includes only that consumer's used endpoints (assert on captured prompt).`
- `Integration: LLM unavailable → 503, falls back to deterministic diff-based guide.`

#### 7.3 — Migration-risk prediction

**What**: Flag consumers unlikely to migrate before sunset.

**Design**:
- `predictMigrationRisk(policyId)`: features per consumer = days-to-sunset, has-started-migrating, historical notification responsiveness (ack latency), traffic trend (rising/flat/declining on deprecated version). A deterministic scoring function produces `riskScore` 0–1; optionally refined by the LLM with rationale text.
- Writes `consumer_impact.analysis.confidenceScore` / `predictedMigrationDate`; `GET /policies/:id/at-risk` lists high-risk consumers for proactive outreach.

**Testing**:
- `Unit: consumer not migrating + sunset in 5 days + flat traffic → high riskScore.`
- `Unit: consumer migrating + acknowledged → low riskScore.`
- `Integration: GET /policies/:id/at-risk → ordered by riskScore desc.`

#### 7.4 — Backward-compatibility & contract testing

**What**: Run backward-compatibility and contract tests between versions.

**Design**:
- `runCompatibilityTest({sourceVersionId, targetVersionId, testType})`:
  - `CONTRACT`: validate example requests/responses from the source spec against the target schema (using `ajv` over JSON Schema 2020-12); any failure = incompatibility.
  - `BACKWARD`: every source endpoint must still exist in target with compatible (non-narrowed request / non-removed-required-response) shape — derived from the `DiffResult` breaking set.
- Persist `compatibility_test.report` (`totalTests`, `passed`, `failed`, `failures[]`). `POST /apis/:id/compatibility-test`, `GET` results.

**Testing**:
- `Fixture: backward-compatible additive change → status PASSED.`
- `Fixture: removed required response field → FAILED with the field in failures[].`
- `Integration: result persisted and retrievable.`

---

## Phase 8: Governance Rulebook & Linting

### Purpose
Implement AI-assisted policy enforcement: ingest a plain-English governance rulebook, compile it to executable lint rules, and gate new specs against versioning/naming/deprecation standards (Microsoft REST API Guidelines, OWASP API5:2023).

### Tasks

#### 8.1 — Spectral-based linting baseline

**What**: Lint a spec against a configurable ruleset.

**Design**:
- Wrap `@stoplight/spectral-core` with a default ruleset covering: required `info.version`, operationId presence, naming conventions, mandatory `Deprecation`/`Sunset` documentation on deprecated ops, OWASP API5 inventory hygiene (no undocumented/debug paths).
- `POST /apis/:id/lint` (or version spec) → findings; CLI `avm lint --spec` → table/SARIF, `--fail-on`.

**Testing**:
- `Fixture: spec missing info.version → finding raised.`
- `E2E: avm lint --fail-on error on non-compliant spec → exit 1.`

#### 8.2 — NL rulebook → ruleset compiler

**What**: Convert an organisation's plain-English rulebook into Spectral rules + default sunset timelines.

**Design**:
- `compileRulebook(text)` via LLM (structured output): produces `{ spectralRules: SpectralRule[], defaultSunsetPeriodDays, versioningDefaults }`, stored in `organisation.settings.governanceRules`.
- Unmappable rules are returned as `manualReviewRules[]` rather than silently dropped.
- New version uploads auto-lint against the org ruleset; `enforcementLevel` decides advisory vs blocking.

**Testing**:
- `Integration (mocked LLM): rulebook "all v-major bumps need 180-day sunset" → defaultSunsetPeriodDays=180.`
- `Integration: compiled rule flags a violating spec on upload.`
- `Unit: ambiguous rule → appears in manualReviewRules, not silently dropped.`

---

## Phase 9: Dependency Graph Analytics (Optional Module)

### Purpose
Add transitive blast-radius and dependency-chain analysis from data-model Suggestion 4, using the `apache_age` PostgreSQL extension so no second database is operated. Optional and feature-flagged; the core product is fully functional without it.

### Tasks

#### 9.1 — Graph projection via Apache AGE

**What**: Project APIs, versions, consumers, dependencies into an AGE graph kept in sync with relational tables.

**Design**:
- Enable `apache_age`; a `graph.sync` job (triggered on subscription/dependency/version changes) upserts nodes/edges (`OWNS`, `HAS_VERSION`, `SUBSCRIBES_TO`, `DEPENDS_ON`, `IMPACTS`, `SUCCEEDS`).
- `POST /apis/:id/dependencies` records `DEPENDS_ON` edges (`{dependsOnApiId, dependencyType, endpointsUsed}`).

**Testing**:
- `Integration (AGE testcontainer): create subscription → SUBSCRIBES_TO edge appears after sync.`
- `Integration: feature flag off → graph endpoints return 404, no sync job runs.`

#### 9.2 — Transitive blast-radius & migration-path queries

**What**: Expose graph queries for transitive impact and shortest migration path.

**Design**:
- `GET /versions/:id/blast-radius?depth=` runs the variable-depth Cypher (Suggestion 4 query 1) over AGE, returning directly and transitively affected consumers.
- `GET /apis/:id/migration-path?from=&to=` returns the shortest `SUCCEEDS` path.

**Testing**:
- `Integration: A depends_on B; B version sunset → A surfaces as transitively affected at depth 2.`
- `Integration: migration-path v1→v3 across v2 → ['v1','v2','v3'].`

---

## Phase 10: Web Dashboard

### Purpose
Provide the platform-team UI over the existing management API: version registry, deprecation timelines, impact rankings, and migration-campaign status. Delivered last because API + CLI are the MVP surfaces; the dashboard adds no new backend capability.

### Tasks

#### 10.1 — Dashboard app & auth

**What**: Next.js 15 app authenticating via the management API's OAuth/JWT.

**Design**:
- App Router; server components fetch from the management API with the user's token. Pages: API list, API detail (versions timeline + lifecycle badges), Diff/Changelog viewer, Deprecation campaign status, Impact/at-risk consumer table (sortable by blast-radius), Migration guide viewer.
- Reuses Zod-inferred types from `core` for typed API responses.

**Testing**:
- `E2E (Playwright): login → API list renders from seeded data.`
- `E2E: deprecate a version in UI → status badge updates, audit entry appears.`

#### 10.2 — Visualisations

**What**: Deprecation timeline (Gantt-style) and consumer blast-radius chart.

**Design**:
- Timeline component renders each version's ACTIVE→DEPRECATED→SUNSET window from policy dates.
- Blast-radius bar chart from `GET /versions/:id/impact`.

**Testing**:
- `E2E: timeline shows sunset marker at the policy's sunset date.`
- `Component test: impact chart sorts bars by blastRadiusScore.`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Layer        ─── required by everything
    │
Phase 2: Spec Ingestion & Registry      ─── requires 1
    │
Phase 3: Breaking-Change & Changelog    ─── requires 2
    │
    ├── Phase 4: Deprecation & Headers   ─── requires 3
    │       │
    │       └── Phase 6: Jobs & Notifications  ─── requires 4 (and 5 for impact-aware notices)
    │
    └── Phase 5: Consumers, Traffic & Impact ─── requires 3 (can parallel with Phase 4)
            │
            └── Phase 7: AI Guidance & Compat Testing ─── requires 3 + 5
                    │
                    └── Phase 8: Governance & Linting  ─── requires 2 (LLM parts share Phase 7's provider)

Phase 9: Dependency Graph (optional)    ─── requires 5
Phase 10: Web Dashboard                 ─── requires 3,4,5,6 (any subset for incremental UI)
```

Parallelism: After Phase 3, **Phase 4 and Phase 5 can be developed concurrently** (different teams). **Phase 8** can start once Phase 2 exists (its LLM compiler reuses the Phase 7 provider, so sequence 8 after 7 if one developer). **Phase 9 and Phase 10** are independent of each other and can both proceed once their data dependencies (5; and 3–6) are met.

Suggested MVP cut line: **Phases 1–6** deliver every "Must-have (MVP)" feature from `features.md` (version registry, deprecation timeline, breaking-change detection, changelog, migration guide templates, version-specific docs via stored specs, backward-compat groundwork, version switching config via lifecycle). **Phase 7** adds the AI-native differentiators and full compatibility testing (v1.1). **Phases 8–10** are v1.1+/backlog.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and mocked-integration tests pass (`pnpm test`).
3. Real-dependency integration tests pass against testcontainers (`pnpm test:int`).
4. ESLint and Prettier pass with no errors (`pnpm lint`).
5. Type checking passes (`pnpm -r exec tsc --noEmit`).
6. `docker compose up` builds and starts server + worker + Postgres + Redis with the phase's features working end-to-end.
7. New database changes are captured as a checked-in `drizzle-kit` migration and apply cleanly on a fresh database.
8. New management-API endpoints appear in the generated OpenAPI 3.1 spec (`docs/openapi.json` regenerated).
9. New JSONB column shapes have a Zod schema in `core/schemas` and are guarded by `validateJsonb` on write.
10. New config/env vars are added to `.env.example` and documented.
11. Any standards-bearing output (SARIF, RFC 9745/8594 headers, OpenAPI) validates against its schema/spec.
12. The phase's headline capability is demonstrated by at least one E2E test exercising the real user workflow (HTTP or CLI).
