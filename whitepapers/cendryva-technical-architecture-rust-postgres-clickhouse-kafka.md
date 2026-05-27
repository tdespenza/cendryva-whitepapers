# Cendryva Technical Architecture: Rust, PostgreSQL, ClickHouse, Kafka, Vault, and ONNX

**Audience:** Platform engineers, infrastructure architects, security engineers, ML platform leads evaluating Cendryva for self-hosted deployment, technical due diligence reviewers  
**Canonical URL:** `/whitepapers/cendryva-technical-architecture-rust-postgres-clickhouse-kafka/`  
**Related papers:** Cendryva self-hosted ML observability; sub-5ms inference at scale; ClickHouse for high-volume ML and statistical observability; HIPAA-ready ML decision logs  
**Author:** Cendryva  
**Published:** 2026-05-25  
**Version:** 1.0  
**License:** CC BY 4.0
**Contact:** research@cendryva.com

## Abstract

Cendryva is a self-hosted ML observability, statistics, compliance, and real-time inference platform built for regulated enterprises. The architecture is a polyglot stack assembled around clear separation of concerns: a Rust core for performance-sensitive inference and data primitives, a TypeScript application tier for the API and business logic, PostgreSQL for transactional state and identity, ClickHouse for high-volume analytical telemetry, Kafka for event-driven workflows, Redis for low-latency access patterns, HashiCorp Vault for secrets and key management, and ONNX as the portable model artifact format. The platform is packaged for Kubernetes, Docker Compose, and Terraform-driven private cloud deployments.

This paper is the engineer-facing deep dive. It covers the full stack, the reason each component is in the architecture, the data flow from inference request to audit evidence, the consistency model across PostgreSQL and ClickHouse, the security boundaries, the testing strategy, and the file paths that implement each layer. It is the document a platform engineer should read before deploying Cendryva or before doing technical due diligence on the codebase.

## Executive Summary

The architecture is built around five principles:

- **Separation by workload.** Transactional state lives in PostgreSQL. High-volume time-series telemetry lives in ClickHouse. Event-driven workflows flow through Kafka. Low-latency cache lives in Redis. Secrets live in Vault. Conflating these into a single store is a common mistake.
- **Rust where predictability matters; TypeScript where iteration matters.** The inference path, threshold evaluators, and data primitives are Rust. The API, business logic, and integration glue are TypeScript on Node. The boundary is documented and intentional.
- **Audit evidence is a first-class data shape.** Every state-changing operation produces a signed, chain-linked WORM audit row. The audit log is verifiable end-to-end.
- **ONNX as the model artifact boundary.** Training framework choice does not leak into the serving layer. The serving layer reads ONNX.
- **Self-hosted from day one.** The platform is packaged for the organization's own Kubernetes, on-premises, or air-gapped infrastructure. There is no required call to a vendor service for normal operation.

The platform is open-source code that the deploying organization can read, audit, and modify.

## Component Inventory

| Layer | Component | Purpose |
|---|---|---|
| Core inference and data primitives | Rust (`src/`) | ONNX inference, threshold evaluators, data CRUD primitives, ClickHouse and Postgres adapters, compliance catalogs, integration helpers |
| API and business logic | TypeScript on Node (`apps/api/`) | REST and OpenAPI surfaces, business services, RBAC, entitlements, workflow orchestration |
| Web UI | SvelteKit (`apps/web/`) | Operator console, dashboards, audit views, ML registry UI |
| Desktop | Electron (`apps/desktop/`) | Local workstation client |
| E2E tests | Playwright (`apps/e2e/`) | Browser-driven end-to-end coverage |
| Shared types | TypeScript (`packages/types/`) | Cross-app contracts |
| Transactional store | PostgreSQL | Identity, model registry, audit chain, configuration, RBAC, multi-tenant data with RLS |
| Analytical store | ClickHouse | Metrics, inference logs, drift samples, anomaly scores, condition history, rollups |
| Event bus | Kafka | Event-driven workflows, agent runtime, async ingestion, dead-letter handling |
| Cache | Redis | Session, rate limit, hot lookups |
| Secrets | HashiCorp Vault | Audit signing keys, integration credentials, encryption key material |
| Model artifact | ONNX | Portable serving format |
| Packaging | Kubernetes, Docker Compose, Terraform | Self-hosted deployment options |

## Repository Layout

The repository is a pnpm monorepo with a Rust crate at the root.

```text
.
├── Cargo.toml            # Rust crate (cendryva)
├── src/                  # Rust source
│   ├── rms/              # Inference, metrics, alerts, real-time monitoring
│   ├── statistics/       # Threshold engines, formula engine, aggregation
│   ├── clickhouse/       # ClickHouse schema, setup, encryption, retention
│   ├── postgres/         # Postgres helpers, connection pool, migrations
│   ├── compliance/       # HIPAA, PCI-DSS catalogs, PHI registry
│   ├── data/             # CRUD primitives over Postgres
│   ├── vault/            # Vault client and key management
│   ├── audit/            # Audit primitives
│   ├── rbac/             # RBAC primitives
│   ├── connector/        # Connector marketplace primitives
│   └── bin/              # Binaries (hipaa_report, inference_benchmark, etc.)
├── apps/
│   ├── api/              # Express.js TypeScript API
│   ├── web/              # SvelteKit web UI
│   ├── desktop/          # Electron desktop client
│   ├── e2e/              # Playwright end-to-end tests
│   └── mobile/           # React Native (future)
├── packages/
│   ├── types/            # Shared TypeScript types and schemas
│   └── ...               # Other shared utilities
├── migrations/
│   ├── postgres/         # Versioned Postgres migrations (V001 onward)
│   └── clickhouse/       # ClickHouse DDL migrations
└── infrastructure/
    ├── k8s/              # Kubernetes manifests
    └── terraform/        # Terraform IaC
```

The Rust crate name in `Cargo.toml` is `cendryva`. The Rust source intentionally exposes data primitives only; all business logic lives in `apps/api/` (TypeScript). The boundary is enforced in code review and documented in CLAUDE.md.

## Rust Core (`src/`)

The Rust core handles the workloads where predictability, memory control, and concurrency matter most.

### Inference (`src/rms/inference/`)

ONNX-based inference with a session cache, multi-execution-provider support (CPU, CUDA, CoreML, DirectML, ROCm, with TensorRT deferred), and configurable thread pools.

Key files:

- `src/rms/inference/prediction_service.rs` — `run_onnx_inference` performs input and output tensor marshaling, with distinct error variants for model-not-found, shape-mismatch, and runtime errors.
- `src/rms/inference/session_cache.rs` — Per-`(model_id, version)` session cache backed by `dashmap`. `ExecutionProvider` enum selects the runtime backend.
- `src/rms/inference/model_loader.rs` — Local file, S3, MinIO, and LocalStack model loading via `s3://` URLs. Streaming downloads for multi-GB artifacts.
- `src/bin/inference_benchmark.rs` — HdrHistogram-based benchmark with coordinated-omission correction.

Environment variables: `CENDRYVA_INFERENCE_INTRA_THREADS`, `CENDRYVA_INFERENCE_INTER_THREADS`, `CENDRYVA_INFERENCE_OPT_LEVEL`, `CENDRYVA_INFERENCE_EXECUTION_PROVIDER`, `CENDRYVA_INFERENCE_S3_REGION`, `CENDRYVA_INFERENCE_S3_ENDPOINT_URL`, `CENDRYVA_INFERENCE_STREAM_THRESHOLD_MB`.

### Statistics and Thresholds (`src/statistics/`)

The statistics module contains the formula engine, aggregation primitives, and threshold evaluators.

Key files:

- `src/statistics/thresholds/static_threshold.rs` — Fixed-bound threshold evaluation.
- `src/statistics/thresholds/dynamic.rs` — Rolling-window dynamic baseline evaluation.
- `src/statistics/thresholds/seasonal.rs` — Same-point-last-period seasonal comparison.
- `src/statistics/core/formula.rs` — Formula evaluation engine.
- `src/statistics/core/aggregation.rs` — Aggregation primitives.
- `src/statistics/core/rollup.rs` — Rollup logic.

### ClickHouse Adapters (`src/clickhouse/`)

ClickHouse schema setup, encryption, and retention management.

Key files:

- `src/clickhouse/schema.rs` — DDL for the analytical tables.
- `src/clickhouse/setup.rs` — Cluster setup and bootstrap.
- `src/clickhouse/encryption.rs` — Column-level AES-GCM encryption for sensitive columns.
- `src/clickhouse/retention.rs` — TTL-driven retention policy enforcement.

### Compliance (`src/compliance/`)

Compliance control catalogs and PHI registry.

Key files:

- `src/compliance/hipaa.rs` — 64 HIPAA control entries.
- `src/compliance/pci_dss.rs` — PCI-DSS catalog.
- `src/compliance/phi.rs` — PHI classification primitives.
- `src/compliance/phi_registry.rs` — Field-level PHI registry.
- `src/bin/hipaa_report.rs` — Catalog report generator.

### Data Primitives (`src/data/`)

CRUD primitives over PostgreSQL. The boundary rule (documented in user memory): `cendryva-data` is dumb storage; `cendryva-api` owns business logic. Rust `src/data/*` is CRUD-only; all apps reach data only via the TypeScript API, never directly.

Notable files:

- `src/data/audit_logs.rs` — Audit log CRUD with WORM enforcement.
- `src/data/ml_models.rs` — Model registry CRUD.
- `src/data/ml_promotions.rs` — Promotion record CRUD.
- `src/data/feature_samples.rs` — Drift sample CRUD.

### Vault Integration (`src/vault/`)

Vault client and key management.

Key files:

- `src/vault/client.rs` — Vault HTTP client.
- `src/vault/encryption.rs` — Vault Transit encryption helpers.
- `src/vault/key_manager/` — Key lifecycle management.
- `src/vault/secret_rotation.rs` — Rotation primitives.

## TypeScript API (`apps/api/`)

The API tier is Express.js on Node with strict TypeScript. It enforces the three-layer pattern: routes call services; services call the data layer; the data layer never calls routes.

### Routes (`apps/api/src/routes/`)

REST and OpenAPI surfaces. Notable routes:

- `apps/api/src/routes/ml.ts` — ML model registry, promotion, monitoring endpoints.
- `apps/api/src/routes/metrics.ts` — Metric ingestion and query.
- `apps/api/src/routes/compliance.ts` — Compliance, BAA, breach notification, disclosure accounting.
- `apps/api/src/routes/conditions.ts` — Condition state queries and overrides.
- `apps/api/src/routes/notifications.ts` — Notification dispatch.
- `apps/api/src/routes/data-quality.ts` — Data quality endpoints.
- `apps/api/src/routes/security-monitoring.ts` — Security event surface.

### Services (`apps/api/src/services/`)

Organized into domains:

- `apps/api/src/services/ml/` — Model registry, promotion FSM, drift detection, cohort analysis, feature store, model monitoring, retraining orchestration.
- `apps/api/src/services/domains/audit/` — Immutable audit log, signing key provider, integrity verification.
- `apps/api/src/services/domains/security/` — RBAC policy engine, OPA client, HSM key manager, RASP monitor.
- `apps/api/src/services/domains/compliance/` — BAA, breach notification, disclosure accounting.
- `apps/api/src/services/12-conditions/` — Condition state machine, evaluators, condition event bus.
- `apps/api/src/services/platform/vault/` — Vault service and credential service.
- `apps/api/src/services/notifications/` — Multi-channel dispatcher.
- `apps/api/src/services/realtime-metrics/` — WebSocket fan-out.

### Middleware (`apps/api/src/middleware/`)

- `apps/api/src/middleware/minimum-necessary.ts` — HIPAA minimum-necessary access control.
- Global HTTP audit interceptor wired through the audit service.

## Web, Desktop, and E2E

- `apps/web/` — SvelteKit operator console.
- `apps/desktop/` — Electron client.
- `apps/e2e/tests/` — Playwright end-to-end suite.

The web tier never calls Postgres or ClickHouse directly. It calls the API.

## PostgreSQL Schema and Migrations

PostgreSQL is the transactional source of truth. Migrations live in `migrations/postgres/` and are versioned `V###__Description.sql` with matching `.rollback.sql` files.

Selected migrations:

- `V001__Identity_And_Tenancy.sql` — Identity, organization, and tenancy model.
- `V002__Audit_Trail_And_Logs.sql` — WORM audit columns, trigger, and chain table.
- `V003__Encryption_And_Security.sql` — Encryption metadata and security tables.
- `V004__Notifications_Schema.sql` — Notification dispatch tables.
- `V005__Legacy_Metrics_Foundation.sql` — Metrics, thresholds, conditions.
- `V006__ML_Schema_And_RLS.sql` — `ml_models`, `model_promotions`, `model_promotion_approvals`, `model_promotion_events`, plus row-level security on ML tables.
- `V007__Data_Quality_And_HIPAA.sql` — Data quality rules, PHI breach tracking.
- `V008__Reports_And_Pipelines.sql` — Reports and pipeline metadata.
- `V009__Workflows_And_Integrations.sql` — Workflow state and integration metadata.
- `V010__Billing_And_Marketplace.sql` — Billing and marketplace tables.

Row-level security is used to enforce organization scoping at the database layer for tenant-isolated data, including the ML schema.

## ClickHouse Schema and Migrations

ClickHouse is the analytical store for high-volume telemetry. Migrations live in `migrations/clickhouse/`.

Selected migrations:

- `1709424000_clickhouse_multi_region_distribution.sql` — Multi-region distributed table topology.
- `1709424001_add_anomaly_scores_table.sql` — Anomaly score time series.
- `1709424002_add_alerts_log_table.sql` — Alert log.
- `1709424007_add_inference_requests_log.sql` — Inference request history.
- `1709424008_add_kafka_dead_letter_queue.sql` — DLQ for Kafka-driven workflows.
- `1709424012_add_condition_evaluation_log.sql` — Condition state history.
- `1709424013_add_notification_delivery_log.sql` — Notification dispatch history.
- `1709424016_add_connector_execution_log.sql` — Connector run history.
- `1709424020_add_workflow_execution_logs.sql` — Workflow execution history.
- `1709424028_add_feature_samples_table.sql` — Feature samples for drift analysis.

Tables use the MergeTree family with partitioning by time and TTL-based retention. AggregatingMergeTree is used for rollups. Column-level AES-GCM encryption is available for sensitive columns through the `src/clickhouse/encryption.rs` helpers.

## Kafka Event Flow

Kafka carries event-driven workflows: agent runtime events, async ingestion, condition event propagation, and dead-letter handling.

Key TypeScript integration points:

- `apps/api/src/services/domains/agents/async-execution.service.ts` — Agent async execution.
- `apps/api/src/services/domains/agents/async-runtime.service.ts` — Agent runtime.
- `apps/api/src/services/12-conditions/condition-event-bus.service.ts` — Condition state change propagation.
- `apps/api/src/services/data-quality/ingestion-pipeline.service.ts` — Async ingestion.
- `apps/api/src/tests/chaos/kafka-failures.test.ts` and `apps/api/src/tests/kafka-chaos.test.ts` — Chaos tests for Kafka failure modes.

The Kafka DLQ pattern lands failed messages in ClickHouse via `migrations/clickhouse/1709424008_add_kafka_dead_letter_queue.sql` so operators can inspect, replay, or escalate.

## Redis Cache

Redis is used for low-latency cache and session patterns:

- `apps/api/src/services/domains/platform/cache.service.ts` — Cache service.
- `apps/api/src/services/domains/agents/session-store.ts` — Agent session store.
- Rate limit storage and hot lookup patterns.

Redis is treated as a cache, not a source of truth. Cache invalidation is event-driven where possible.

## Vault and Secrets

HashiCorp Vault holds audit signing keys, integration credentials, and encryption key material.

Key files:

- `apps/api/src/services/platform/vault/vault.service.ts` — Vault service wrapper.
- `apps/api/src/services/platform/vault/credential.service.ts` — Credential lifecycle.
- `apps/api/src/services/domains/audit/audit-signing-key-provider.ts` — Vault-backed Ed25519 signing key provider.
- `src/vault/key_manager/` — Rust-side key management.
- `src/vault/secret_rotation.rs` — Rotation primitives.

The audit log signing keys never leave Vault. The audit service requests signatures from Vault Transit; the resulting signature is persisted with the audit row.

## End-to-End Data Flow: An Inference Request

Walking the path of a single inference request through the architecture:

1. **Request arrives.** A client calls `POST /ml/inference` (or the equivalent route). The Express route in `apps/api/src/routes/ml.ts` performs validation and RBAC.
2. **Entitlement check.** The route calls the entitlement checker (`apps/api/src/services/ml/data-client-entitlement.checker.ts`) to verify the caller's plan permits the operation.
3. **Model registry lookup.** The route calls the registry service to resolve the model artifact and version.
4. **Inference.** For the Rust-served path, the API delegates to the inference service in `src/rms/inference/prediction_service.rs`. The session cache returns or loads the ONNX session.
5. **Decision log write.** The result and the input summary are written to the audit service (`apps/api/src/services/domains/audit/audit.service.ts`), which routes through `ImmutableAuditLogService.appendLog`. The audit row is signed via the Vault-backed signing key provider, chained to the previous row, and persisted to PostgreSQL.
6. **Drift sample capture.** A sample of the input features is written to ClickHouse via the feature sample loader (`apps/api/src/services/ml/clickhouse-feature-sample.loader.ts`).
7. **Threshold and condition evaluation.** Drift, latency, error rate, and other measurements are evaluated against thresholds (`src/statistics/thresholds/`). The condition state machine (`apps/api/src/services/12-conditions/state-machine.ts`) classifies the resulting state.
8. **Notification dispatch.** If the state transitions to a degraded condition, the notification dispatcher (`apps/api/src/services/notifications/dispatcher.ts`) routes alerts through the configured channels.
9. **Telemetry to ClickHouse.** Inference latency, error rate, and request metadata land in the ClickHouse inference log table.
10. **Response.** The route returns the prediction to the client.

Every step is documented, traceable, and reconstructable from the audit log and the ClickHouse telemetry.

## Consistency Model

The platform uses different consistency guarantees per workload:

| Workload | Store | Consistency |
|---|---|---|
| Identity, RBAC, configuration | PostgreSQL | Strong (single-region serializable) |
| Model registry and promotions | PostgreSQL | Strong, with RLS |
| Audit log writes | PostgreSQL | Strong, with WORM trigger |
| Inference telemetry | ClickHouse | Eventually consistent, append-only |
| Drift samples | ClickHouse | Eventually consistent, append-only |
| Event-driven workflows | Kafka | At-least-once with DLQ |
| Cache | Redis | Best-effort, event-invalidated |

Cross-store consistency is achieved through the audit log: every state-changing operation in PostgreSQL also emits an event that other stores consume. The audit log is the canonical record.

## Security Boundaries

Layered controls:

- **Network.** Self-hosted Kubernetes or VPC isolation. The API does not require outbound calls to a vendor service.
- **Identity.** PostgreSQL identity tables, with RBAC enforced by the policy engine (`apps/api/src/services/domains/security/rbac-policy-engine.ts`) and OPA integration (`apps/api/src/services/domains/security/opa-client.ts`).
- **Row-level security.** PostgreSQL RLS policies enforce organization scoping at the database layer for tenant-isolated data.
- **Column encryption.** ClickHouse sensitive columns use AES-GCM via `src/clickhouse/encryption.rs`. Postgres encryption helpers are in `src/postgres/encryption.rs`.
- **Audit signing.** Audit rows are signed Ed25519 with keys in Vault.
- **WORM trigger.** A PostgreSQL trigger blocks update or delete on audit rows.
- **HSM key management.** `apps/api/src/services/domains/security/hsm-key-manager.ts` handles HSM-backed keys.
- **RASP monitor.** `apps/api/src/services/domains/security/rasp-monitor.ts` performs runtime application self-protection.
- **HIPAA minimum-necessary middleware.** `apps/api/src/middleware/minimum-necessary.ts` enforces field-level access scoping.

## Testing Strategy

The project enforces an exhaustive coverage contract documented in `CLAUDE.md`. Every feature change must update the coverage matrix and the relevant test layers.

Test types:

- Rust unit and integration tests: `cargo test --lib` runs 805+ tests as of the publication date.
- TypeScript unit and integration tests: `pnpm --filter @cendryva/api test` runs 7460+ passing tests.
- E2E tests: Playwright suite in `apps/e2e/tests/`.
- Chaos tests: `apps/api/src/tests/chaos/` and Kafka chaos suites.
- Security tests: `apps/api/src/tests/security/access-control-route-manifest.test.ts`, `apps/api/src/tests/security/access-control-route-matrix.test.ts`.
- Migration idempotency: `apps/api/src/tests/db/clickhouse-migration-idempotency.test.ts`.

A static guard test ensures that protected API routes, app routes, entitlements, quotas, and permissions are reflected in the coverage matrix.

## Packaging and Deployment

The platform is packaged for self-hosted deployment:

- 26+ Kubernetes manifests under `infrastructure/k8s/`.
- Terraform modules under `infrastructure/terraform/`.
- `Dockerfile` for the API container.
- `docker-compose.yml`, `docker-compose.gpu.yml`, `docker-compose.vagrant.yml` for local and lab deployments.

Deployment profiles:

- Private cloud (VPC isolation, managed Kubernetes, enterprise identity).
- Dedicated tenant (dedicated databases, queues, and analytics stores).
- Air-gapped (offline-compatible releases, private artifact registries).
- Hybrid analytics (separate operational and analytics systems with controlled exports).

There is no required call to a hosted Cendryva service for normal operation.

## Industry Focus: Engineering Due Diligence

A platform engineering team evaluating Cendryva for adoption will typically ask:

- What is the model artifact format and how is it served? ONNX, served by Rust with a configurable execution provider.
- Where does the audit log live and how is integrity assured? PostgreSQL with WORM trigger, Ed25519 signing via Vault, chain-linked rows, integrity verification service.
- How is multi-tenant isolation enforced? Organization scoping at the application layer plus PostgreSQL row-level security on tenant-sensitive tables.
- Where does telemetry land and how is it retained? ClickHouse with partitioned MergeTree, TTL-based retention, optional column-level AES-GCM encryption.
- How is the platform packaged for self-hosted deployment? Kubernetes manifests, Terraform, Docker Compose, with no required vendor calls.
- What does the test surface look like? 805+ Rust tests, 7460+ TypeScript tests, Playwright E2E suite, chaos tests, security route matrix.

The answers are visible in code.

## Implementation Checklist

For a platform engineer deploying Cendryva:

- Provision PostgreSQL with the required extensions and run the `V001` through current migrations.
- Provision ClickHouse and run the migrations in `migrations/clickhouse/`.
- Provision Kafka with the topics referenced in the API services.
- Provision Redis for cache and session storage.
- Provision Vault with Transit and KV secrets engines, and configure the audit signing key.
- Build and deploy the API container.
- Apply the Kubernetes manifests or run the Terraform modules.
- Run the Rust test suite and the TypeScript test suite against the deployment.
- Verify the WORM audit log by running the integrity verification service.
- Verify the entitlement and RBAC matrices through the security route tests.
- Generate the HIPAA mapping report via `cargo run --bin hipaa_report`.
- Run the inference benchmark against a representative model.
- Schedule periodic verification of audit integrity, threshold review, and certificate rotation.

## Conclusion

The Cendryva architecture is a deliberate polyglot stack assembled around clear separation of concerns. Rust handles the inference and data primitives where predictability matters. TypeScript handles the API and business logic where iteration matters. PostgreSQL holds the transactional state. ClickHouse holds the analytical telemetry. Kafka carries the event-driven workflows. Redis is the cache. Vault holds the keys. ONNX is the model artifact boundary.

Every state-changing operation produces a signed, chain-linked WORM audit row. Every tenant boundary is enforced at the database layer for sensitive tables. Every model artifact has a versioned identity. Every threshold breach lands in the audit log alongside the decision it pertains to. Every component is packaged for self-hosted deployment with no required vendor calls.

This paper is a map of the stack for the engineer who needs to read, audit, deploy, or extend it. The implementation references are real file paths in the open-source codebase.

## Implementation Status

This section maps the architectural claims above to the Cendryva codebase as of the publication date in the header. Because this paper is the architecture deep dive, the entire paper is effectively an Implementation Status section.

**Rust core** — SHIPPED
- Crate: `Cargo.toml` (`cendryva` v2.0.0-beta).
- Modules: `src/rms/`, `src/statistics/`, `src/clickhouse/`, `src/postgres/`, `src/compliance/`, `src/data/`, `src/vault/`, `src/audit/`, `src/rbac/`, `src/connector/`, `src/integration/`.
- Binaries: `src/bin/inference_benchmark.rs`, `src/bin/hipaa_report.rs`.
- Tests: 805+ `cargo test --lib` tests passing as of the publication date.

**ONNX inference with multi-EP support** — SHIPPED
- Code: `src/rms/inference/prediction_service.rs`, `src/rms/inference/session_cache.rs`, `src/rms/inference/model_loader.rs`.
- Notes: CPU default; CUDA, CoreML, DirectML, ROCm available as cargo features. TensorRT EP wiring is deferred (pattern is identical to CUDA).

**TypeScript API tier** — SHIPPED
- Code: `apps/api/src/`. Routes in `apps/api/src/routes/`, services organized into domains under `apps/api/src/services/`.
- Tests: 7460+ TypeScript tests passing as of the publication date.

**Web UI** — SHIPPED
- Code: `apps/web/` (SvelteKit).

**Desktop client** — SHIPPED
- Code: `apps/desktop/` (Electron).

**E2E tests** — SHIPPED
- Code: `apps/e2e/tests/` (Playwright).

**Shared types package** — SHIPPED
- Code: `packages/types/src/index.ts`.

**PostgreSQL migrations** — SHIPPED
- Code: `migrations/postgres/V001` through `V056` (single strictly-sequential range).
- Notes: Each migration ships with a matching `.rollback.sql`.

**ClickHouse migrations** — SHIPPED
- Code: `migrations/clickhouse/1709424000_` through `1709424028_`.

**Kafka event flow** — SHIPPED
- Code: `apps/api/src/services/domains/agents/async-execution.service.ts`, `apps/api/src/services/domains/agents/async-runtime.service.ts`, `apps/api/src/services/12-conditions/condition-event-bus.service.ts`.
- Tests: `apps/api/src/tests/kafka-chaos.test.ts`, `apps/api/src/tests/chaos/kafka-failures.test.ts`.

**Redis cache** — SHIPPED
- Code: `apps/api/src/services/domains/platform/cache.service.ts`, `apps/api/src/services/domains/agents/session-store.ts`.

**Vault integration** — SHIPPED
- Code: `apps/api/src/services/platform/vault/vault.service.ts`, `apps/api/src/services/platform/vault/credential.service.ts`, `apps/api/src/services/domains/audit/audit-signing-key-provider.ts`, `src/vault/client.rs`, `src/vault/key_manager/`, `src/vault/secret_rotation.rs`.

**WORM audit log with chain integrity** — SHIPPED
- Code: `apps/api/src/services/domains/audit/immutable-audit-log.service.ts`, `apps/api/src/services/domains/audit/audit.service.ts`, `apps/api/src/services/domains/audit/audit-log-integrity.service.ts`, `src/data/audit_logs.rs`.
- Schema: `migrations/postgres/V002__Audit_Trail_And_Logs.sql`.
- Tests: 52 audit tests plus 5 Postgres integration WORM tests.

**Row-level security on tenant-sensitive tables** — SHIPPED
- Schema: `migrations/postgres/V001__Identity_And_Tenancy.sql`, `migrations/postgres/V006__ML_Schema_And_RLS.sql`.

**RBAC and OPA integration** — SHIPPED
- Code: `apps/api/src/services/domains/security/rbac-policy-engine.ts`, `apps/api/src/services/domains/security/opa-client.ts`, `apps/api/src/services/domains/security/opa-http-client.ts`, `apps/api/src/services/domains/security/rbac-admin.service.ts`.
- Tests: `rbac-policy-engine.test.ts`, `opa-client.test.ts`, `apps/api/src/tests/security/access-control-route-manifest.test.ts`, `apps/api/src/tests/security/access-control-route-matrix.test.ts`.

**HSM key manager and RASP monitor** — SHIPPED
- Code: `apps/api/src/services/domains/security/hsm-key-manager.ts`, `apps/api/src/services/domains/security/rasp-monitor.ts`.

**HIPAA controls catalog and PHI registry** — SHIPPED
- Code: `src/compliance/hipaa.rs` (64 entries), `src/compliance/phi.rs`, `src/compliance/phi_registry.rs`, `src/bin/hipaa_report.rs`.

**Compliance services** — SHIPPED
- Code: `apps/api/src/services/domains/compliance/baa.service.ts`, `apps/api/src/services/domains/compliance/breach-notification.service.ts`, `apps/api/src/services/domains/compliance/disclosure-accounting.service.ts`. Route: `apps/api/src/routes/compliance.ts`.

**Self-hosted packaging** — SHIPPED
- Kubernetes manifests under `infrastructure/k8s/`, Terraform under `infrastructure/terraform/`, `Dockerfile`, `docker-compose.yml`, `docker-compose.gpu.yml`, `docker-compose.vagrant.yml`.

**Deferred**
- TensorRT execution provider runtime wiring (cargo feature exists; runtime arm not yet added).
- Some output dtypes (`f16`, `bf16`, `i4`, complex) currently return `OutputTypeUnsupported`.
- Mobile client (`apps/mobile/` is reserved for React Native, not shipped).
- Production federated learning (see the FL paper for an honest accounting).
- A few opinionated dashboard templates and report generators that the platform supports but does not yet ship out of the box.

**How to verify locally**

```
cargo test --lib
pnpm --filter @cendryva/api test
cargo run --bin hipaa_report
cargo build --release --bin inference-benchmark
./target/release/inference-benchmark --model tests/fixtures/inference/identity.onnx \
  --input-shape 1,3,224,224 --concurrency 8 --total-requests 10000 --warmup 1000 --output hdr
```

## Scope and Limitations

This is a vendor-authored paper from Cendryva. It is the architecture deep-dive for the platform's open-source codebase. It is not a deployment guide, not a sizing document, and not a procurement endorsement.

**In scope.** The full Cendryva architecture, the role of each component, the data flow from inference request to audit evidence, the consistency model across PostgreSQL and ClickHouse, the security boundaries, the testing strategy, and the file paths that implement each layer.

**Out of scope.** Specific hardware sizing. Network topology design for any individual deployment. Disaster recovery runbook. Capacity planning for a specific workload. Detailed migration paths from incumbent tools. Pricing.

**Time-bounded items.** Codebase file paths reflect the publication date and may move under refactoring. Migration numbers, test counts, and feature flags evolve. Re-verify the current layout in the repository at the time of evaluation.

**Empirical claims.** Inference latency numbers depend on hardware and model and must be measured by the deployer using the included benchmark harness. The architectural claims are traceable through the file paths in the Implementation Status section.

**Not legal or security advice.** Security claims describe architectural controls. A specific deployment's security posture depends on configuration, operator practice, and the threat model of the deploying organization.

## References and Further Reading

Language and runtime foundations

- Rust Project. *The Rust Programming Language*. https://www.rust-lang.org/
- Rust Project. *The Rust Performance Book*. https://nnethercote.github.io/perf-book/
- Tokio Project. *Tokio asynchronous runtime documentation*. https://tokio.rs/
- Node.js Project. *Node.js documentation*. https://nodejs.org/

Database and analytical store foundations

- PostgreSQL Global Development Group. *PostgreSQL documentation*. https://www.postgresql.org/docs/
- ClickHouse. *ClickHouse documentation*. https://clickhouse.com/docs

Event streaming and messaging

- Apache Kafka. *Apache Kafka documentation*. https://kafka.apache.org/documentation/

Cache and session

- Redis. *Redis documentation*. https://redis.io/docs/

Secrets and key management

- HashiCorp. *Vault documentation*. https://developer.hashicorp.com/vault

Model serving and portability

- ONNX project. *ONNX Specification*. https://onnx.ai/
- Microsoft. *ONNX Runtime documentation*. https://onnxruntime.ai/docs/

Observability and standards

- OpenTelemetry. *OpenTelemetry specification*. https://opentelemetry.io/docs/

Architecture and data system design

- Kleppmann, M. *Designing Data-Intensive Applications*. O'Reilly, 2017.

Deployment

- Kubernetes Project. *Kubernetes documentation*. https://kubernetes.io/docs/
- HashiCorp. *Terraform documentation*. https://developer.hashicorp.com/terraform

Cendryva foundations

- Cendryva. *Cendryva: A Self-Hosted ML Observability Platform for Regulated Enterprises*. https://cendryva.com/whitepapers/self-hosted-ml-observability/
- Cendryva. *Sub-5ms Inference at Scale: Why Rust Belongs in Production ML Infrastructure*. https://cendryva.com/whitepapers/rust-sub-5ms-ml-inference/
- Cendryva. *ClickHouse for High-Volume ML and Statistical Observability*. https://cendryva.com/whitepapers/clickhouse-high-volume-observability/
- Cendryva. *Designing HIPAA-Ready ML Systems With Immutable Decision Logs*. https://cendryva.com/whitepapers/hipaa-ready-ml-decision-logs/
