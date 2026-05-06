# Standards & API Reference

> Project: API Versioning Manager · Generated: 2026-05-03

## Industry Standards & Specifications

### W3C & IETF Standards

**RFC 8594 — The Sunset HTTP Header Field**
- URL: https://datatracker.ietf.org/doc/html/rfc8594
- Published May 2019. Defines the `Sunset` HTTP response header field, which signals that a URI is likely to become unresponsive at a specified point in the future. An API versioning manager should emit this header on deprecated endpoints and parse it from third-party APIs to alert consumers of upcoming shutdowns.

**RFC 9745 — The Deprecation HTTP Response Header Field**
- URL: https://www.rfc-editor.org/rfc/rfc9745.html
- Published March 2025 (Standards Track). Defines the `Deprecation` HTTP response header field, indicating that a resource has been or will be deprecated. Pairs with RFC 8594: `Deprecation` marks the start of the deprecation period while `Sunset` signals the end. An API versioning manager should emit, validate, and monitor both headers across managed services.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Foundational authorization framework underpinning API access control. An API versioning manager must support OAuth 2.0 flows to authenticate securely against managed gateways and developer portals, and to expose its own management API with appropriate scopes.

**RFC 9700 — Best Current Practice for OAuth 2.0 Security**
- URL: https://datatracker.ietf.org/doc/rfc9700/
- Updates and strengthens the threat model and security guidance for OAuth 2.0. Relevant for API versioning manager's own authentication layer and when consuming protected downstream APIs on behalf of teams.

### Data Model & API Specifications

**OpenAPI Specification (OAS) 3.1 / 3.2**
- URL: https://spec.openapis.org/oas/
- The de-facto standard for describing REST API contracts. Provides built-in `deprecated: true` field at the operation, parameter, and schema level, and the `info.version` field for contract-level versioning. An API versioning manager should ingest, diff, and generate OpenAPI documents as its primary data interchange format. OAS 3.2.0 is the current release.

**Semantic Versioning (SemVer) 2.0.0**
- URL: https://semver.org/
- Specification for encoding breaking (`MAJOR`), additive (`MINOR`), and patch (`PATCH`) changes in version numbers. Widely adopted for SDK and library versioning. An API versioning manager should support SemVer detection and enforcement as one versioning strategy alongside date-based and URI-path strategies.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Standard for describing the structure of JSON data. Used to validate API request/response bodies within versioned contracts. An API versioning manager should track schema changes across versions to detect breaking field-level changes, and support SchemaVer (MODEL / REVISION / ADDITION) as an optional schema-level versioning discipline.

**AsyncAPI Specification 3.1**
- URL: https://www.asyncapi.com/docs/reference/specification/v3.1.0
- The equivalent of OpenAPI for event-driven (message-driven) APIs over protocols such as Kafka, AMQP, MQTT, and WebSockets. API versioning concerns apply equally to async contracts; an API versioning manager should support AsyncAPI documents alongside OpenAPI for teams using event-driven architectures.

**GraphQL Specification**
- URL: https://spec.graphql.org/
- GraphQL recommends continuous schema evolution over discrete versioning, using the `@deprecated` directive to mark fields and enum values. An API versioning manager should be able to ingest GraphQL schemas, track `@deprecated` usage, and alert on field removal across schema versions.

### Security & Compliance Standards

**OWASP API Security Top 10 (2023 Edition)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- API5:2023 (Improper Inventory Management) directly addresses failure to maintain a proper inventory of deployed API versions, deprecated endpoints, and exposed debug paths. Compliance with this category requires the kind of version registry, deprecation tracking, and sunset enforcement that an API versioning manager provides.

**Microsoft REST API Guidelines**
- URL: https://github.com/microsoft/api-guidelines
- Industry-influential opinionated guidelines covering API versioning strategy (path-based, query-string, and header-based), breaking change definition, and deprecation process. Includes the Azure REST API Guidelines sub-document and Microsoft Graph REST API Guidelines. A versioning manager should be able to lint API designs against rule sets derived from these guidelines.

---

## Similar Products — Developer Documentation & APIs

### Azure API Management
- **Description:** Microsoft's full-lifecycle API platform supporting path-based, header-based, and query-string versioning; revisions for non-breaking changes; and policy-based traffic routing.
- **API Documentation:** https://learn.microsoft.com/en-us/azure/api-management/api-management-versions
- **REST API Reference:** https://learn.microsoft.com/en-us/rest/api/apimanagement/
- **SDKs/Libraries:** Azure SDK for Python, .NET, Java, JavaScript, Go — https://learn.microsoft.com/en-us/azure/developer/
- **Developer Guide:** https://learn.microsoft.com/en-us/azure/api-management/api-management-get-started-publish-versions
- **Standards:** REST/JSON, OpenAPI import/export, Azure Resource Manager
- **Authentication:** Azure Active Directory (OAuth 2.0 / OpenID Connect), subscription keys

### Apigee (Google Cloud)
- **Description:** Enterprise API gateway with version lifecycle management, API Hub for version cataloguing, analytics, and a developer portal.
- **API Documentation:** https://docs.cloud.google.com/apigee/docs/apihub/versions-intro
- **REST API Reference:** https://docs.cloud.google.com/apigee/docs/reference/apis/apigee/rest
- **Discovery Document:** https://apigee.googleapis.com/$discovery/rest?version=v1
- **SDKs/Libraries:** Google Cloud client libraries for Python, Java, Node.js, Go — https://cloud.google.com/apigee/docs/api-platform/get-started/clients
- **Standards:** REST/JSON, OpenAPI 3.x
- **Authentication:** OAuth 2.0 (Google Identity), API keys

### Kong Gateway
- **Description:** Open-source API gateway with plugin-based versioning and traffic routing. Follows SemVer for its own releases ({MAJOR}.{MINOR}.{PATCH}) with 4 minor releases per year and LTS releases every March.
- **API Documentation:** https://developer.konghq.com/gateway/
- **REST Admin API Reference:** https://developer.konghq.com/api/
- **SDKs/Libraries:** Kong SDK (Go), community clients for Python and Node.js
- **Developer Guide:** https://developer.konghq.com/gateway/upgrade/
- **Standards:** REST/JSON, OpenAPI 3.x, gRPC, GraphQL plugins
- **Authentication:** API key, OAuth 2.0 plugin, JWT plugin, OpenID Connect plugin

### AWS API Gateway
- **Description:** Cloud-native managed gateway using stages as version containers, with support for canary deployments and custom domain path-mapping per version.
- **API Documentation:** https://docs.aws.amazon.com/apigateway/latest/developerguide/welcome.html
- **REST API Reference:** https://docs.aws.amazon.com/apigateway/latest/api/
- **SDKs/Libraries:** AWS SDK for Python (boto3), JavaScript, Java, Go, .NET — https://aws.amazon.com/developer/tools/
- **Developer Guide:** https://docs.aws.amazon.com/apigateway/latest/developerguide/stages.html
- **Standards:** REST/JSON, OpenAPI 2.0/3.0 import/export, HTTP/WebSocket
- **Authentication:** AWS IAM, API keys, Lambda authorisers, Cognito user pools

### Redocly
- **Description:** OpenAPI-centric toolchain for multi-version documentation, linting, breaking-change enforcement, and SDK generation. Primary developer-facing surface for API versioning documentation.
- **API Documentation:** https://redocly.com/docs/
- **CLI Reference:** https://redocly.com/docs/cli/
- **GitHub (Redoc OSS):** https://github.com/Redocly/redoc
- **SDKs/Libraries:** Redocly CLI (Node.js); integrations with GitHub Actions, GitLab CI
- **Developer Guide:** https://redocly.com/blog/getting-started-api-versioning
- **Standards:** OpenAPI 3.x, JSON Schema 2020-12
- **Authentication:** Redocly API key (SaaS), self-hosted via config

### Bump.sh
- **Description:** Modern API documentation platform with first-class changelog support, branch-based version management, and automatic diff-based changelog generation on each deployment.
- **API Documentation:** https://docs.bump.sh/
- **REST API Reference:** https://developers.bump.sh/changes
- **GitHub Action:** https://github.com/marketplace/actions/bump-sh-api-documentation-changelog
- **SDKs/Libraries:** Bump CLI (Node.js)
- **Developer Guide:** https://bump.sh/blog/multiple-api-versions-documentation/
- **Standards:** OpenAPI 3.x, AsyncAPI 3.x
- **Authentication:** API token (Bearer), organization API token

### Postman API Platform
- **Description:** API design, testing, and versioning workspace with changelog, mock servers, and Git-based version control. Published versions create static snapshots for consumer reference.
- **API Documentation:** https://learning.postman.com/docs/designing-and-developing-your-api/versioning-an-api/api-versions/
- **Postman API Reference:** https://www.postman.com/postman/postman-public-workspace/
- **SDKs/Libraries:** Postman SDK (JavaScript/Node.js), Newman CLI
- **Developer Guide:** https://blog.postman.com/automate-versioning-with-postman-api-github-actions/
- **Standards:** OpenAPI 3.x import/export, REST/JSON
- **Authentication:** Postman API key, OAuth 2.0 (workspace integrations)

### Stripe API (Reference Implementation)
- **Description:** Industry gold-standard for date-based API versioning. Combines monthly additive releases with twice-yearly named breaking-change releases (e.g. `2026-04-22.dahlia`). The `Stripe-Version` request header pins clients to a specific version.
- **API Documentation:** https://docs.stripe.com/api/versioning
- **Changelog:** https://docs.stripe.com/changelog
- **SDK Versioning Policy:** https://docs.stripe.com/sdks/versioning
- **SDKs/Libraries:** Official libraries for Python, Ruby, Node.js, Java, PHP, Go, .NET — https://docs.stripe.com/sdks
- **Standards:** REST/JSON, OpenAPI (internal); `Stripe-Version` custom header
- **Authentication:** API key (Bearer), Restricted keys, OAuth for Connect

### oasdiff
- **Description:** Open-source CLI and Go library for OpenAPI diff and breaking-change detection. Supports 450+ rules, three severity levels (ERR/WARN/INFO), and multiple output formats including GitHub Actions annotations.
- **GitHub Repository:** https://github.com/oasdiff/oasdiff
- **GitHub Action:** https://github.com/oasdiff/oasdiff-action
- **Hosted PR Review:** https://www.oasdiff.com/
- **SDKs/Libraries:** Go package (`github.com/oasdiff/oasdiff`); GitHub Action
- **Standards:** OpenAPI 3.0 and 3.1
- **Authentication:** N/A (CLI tool); GitHub token for Action integration

### GraphQL Hive
- **Description:** Open-source GraphQL schema registry with schema history, version-to-version diff, conditional breaking-change detection against real usage, and CDN artifact delivery.
- **API Documentation:** https://the-guild.dev/graphql/hive/docs/schema-registry
- **GitHub:** https://github.com/graphql-hive
- **CLI:** https://www.npmjs.com/package/@graphql-hive/cli
- **SDKs/Libraries:** Hive CLI (Node.js), client SDKs for Apollo, Yoga, Mesh
- **Standards:** GraphQL spec, OpenTelemetry (usage analytics)
- **Authentication:** Registry access token (Bearer)

---

## Notes

**Emerging convergence of Sunset + Deprecation headers:** With RFC 9745 published in March 2025, both the `Sunset` (RFC 8594) and `Deprecation` (RFC 9745) headers are now ratified IETF standards. An API versioning manager should treat parsing and emitting both headers as a core platform capability rather than an optional feature.

**OpenAPI 3.2 adoption:** OpenAPI 3.2 is the current specification version. Tooling compatibility for 3.2 is still maturing; any parser or linter built into an API versioning manager should target OAS 3.1.x as the stable baseline while monitoring 3.2 support.

**Date-based versioning gaining over SemVer for REST:** Stripe's model (and similar approaches from Twilio and Salesforce) demonstrates that date-based versioning combined with named releases provides clearer consumer communication than pure SemVer for REST HTTP APIs. SemVer remains dominant for SDK and library releases. An API versioning manager should support both strategies without mandating one.

**GraphQL anti-versioning pattern:** The GraphQL community intentionally avoids version numbers in favour of schema evolution and the `@deprecated` directive. Support for GraphQL Hive-style schema registry workflows (rather than gateway-style version routing) is the correct integration point for GraphQL-based APIs.
