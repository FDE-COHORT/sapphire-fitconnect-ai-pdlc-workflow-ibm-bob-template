<!--
SYNC IMPACT REPORT
==================
Version change: (template / unversioned) → 1.0.0
Modified principles: N/A — first fill from template
Added sections:
  - Core Principles (I–VI, 6 principles filled)
  - Technology & Stack Constraints (Section 2)
  - Development Workflow & Quality Gates (Section 3)
  - Governance
Removed sections: None (all template sections filled)
Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Constitution Check gates already aligned with principles I–IV; gate for V. API Backward Compatibility may be added as a new row.
  ✅ .specify/templates/spec-template.md — No mandatory constitution-driven sections added; existing structure compatible.
  ✅ .specify/templates/tasks-template.md — Phase structure compatible with all six principles; no changes required.
  ⚠ .specify/templates/commands/*.md — No command files found at expected path; nothing to update.
Deferred TODOs:
  - TODO(RATIFICATION_DATE): Set to the date the team formally adopts this constitution (unknown at time of writing).
-->

# Sapphire FitConnect AI Constitution

## Core Principles

### I. Code Quality

All production code MUST meet the following non-negotiable quality standards:

- All public functions, methods, and classes MUST have docstrings or Javadoc documenting
  intent (not implementation detail).
- Magic numbers and strings are FORBIDDEN; named constants or enums MUST be used.
- Cyclomatic complexity MUST NOT exceed 10 per function or method; violations MUST be
  confirmed via static analysis before merge.
- Commented-out code MUST NOT be committed; feature flags or deletion MUST be used instead.
- Stack-specific conventions MUST be applied: Spring layering for Java services, PEP 8 + ruff
  for Python, strict TypeScript for Node/React, and DataLoader patterns in the BFF.
- Apollo cache policies MUST be declared explicitly; cache-first MUST NOT be used for mutable
  health data; cache TTL MUST be ≥ 30 s per session.
- LangGraph graphs MUST use `TypedDict` state with `Annotated` reducers, compile once at
  startup, use single-responsibility async nodes, and employ a persistent checkpointer in
  production.

**Rationale**: Consistent quality rules reduce review friction across the multi-repo brownfield
estate and prevent progressive degradation as new engineers onboard.

### II. Testing Standards

All features MUST be covered by a meaningful test suite before merge:

- Coverage gates MUST be met: Java 80% overall / 100% domain layer; Python 80%;
  TypeScript/React 70%; BFF resolvers 100%.
- Contract tests MUST be written for every GraphQL schema change and every Kafka event
  schema change before the change ships.
- The test pyramid MUST be respected: unit tests mock all I/O; integration tests run against
  Docker Compose and MUST NOT be used as the sole validation gate; E2E tests cover critical
  user journeys only.
- Tests for a user story MUST be written and confirmed failing before implementation begins
  (red-green-refactor).

**Rationale**: High-coverage, layered tests are the primary safety net in a distributed
microservices platform where cross-service regressions are hard to detect manually.

### III. UX Consistency

All user-facing components and auth flows MUST follow these rules:

- Every data-fetching component MUST handle: loading skeleton state, error boundary state,
  and empty-state presentation.
- Authentication MUST be exclusively Keycloak OIDC/PKCE; no bypass routes are permitted in
  any environment, including local development.
- URL state MUST be the authoritative source of truth for filters, pagination, and
  user-driven selections; in-memory state MUST NOT be used for shareable UI state.

**Rationale**: Uniform UX patterns lower cognitive load for users across the wellness platform
and eliminate subtle security vulnerabilities introduced by auth shortcuts.

### IV. Observability

All services MUST be fully observable from day one:

- Every service MUST emit structured JSON logs (Logback+logstash for Java, structlog for
  Python, pino for Node) containing `trace_id` and `span_id` fields on every log line.
- OTEL metrics MUST be exported to the Collector: request count, duration histogram, error
  rate, in-flight counter; feature-specific business metrics MUST be added per story.
- Distributed traces MUST be emitted via the OTEL SDK using W3C `traceparent` propagation;
  all DB, HTTP, and Kafka operations MUST be instrumented as child spans.
- The environment variables `OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_SERVICE_NAME`, and
  `OTEL_DEPLOYMENT_ENVIRONMENT` MUST be set in every container; services MUST NOT export
  traces directly to a backend.

**Rationale**: Distributed health-data flows span multiple services; without end-to-end
traceability, production debugging and SLA verification are impractical.

### V. API Backward Compatibility

All API changes MUST include a backward-compatibility assessment before implementation:

- Every proposed change to a public API (REST endpoint, GraphQL schema, Kafka event schema,
  gRPC contract) MUST be assessed for breaking vs non-breaking impact as part of the plan
  artifact before implementation begins.
- Breaking changes MUST increment the API MAJOR version and MUST be accompanied by a
  migration guide and a deprecation notice issued at least one release cycle in advance.
- Non-breaking additions (new optional fields, new endpoints) MUST be flagged as additive
  and MUST NOT require consumer changes.
- Integration partners and downstream service owners MUST be notified before any breaking
  change ships.
- The plan.md Constitution Check table MUST include a row verifying backward-compatibility
  assessment for any story that touches a public API boundary.

**Rationale**: The Sapphire platform integrates with smart-device APIs and downstream
healthcare providers; unannounced breaking changes directly impact patient-facing
integrations and create compliance risk.

### VI. Governed PDLC Workflow

All feature work MUST follow the governed PDLC workflow without exception:

- Every feature MUST progress through: Specify → Clarify → Plan → Tasks → Implement → Ship.
- Each gate (spec, plan, tasks) MUST be approved via GitHub PR by the designated role;
  chat confirmation is NOT a valid approval gate.
- The submitter MUST NOT self-approve their own gate PR.
- Implementation MUST NOT begin until spec, plan, and tasks gates have passed.
- All traceability links (JIRA story ↔ spec ↔ plan ↔ tasks ↔ PRs) MUST be maintained
  throughout the lifecycle.

**Rationale**: The PDLC gates enforce separation of concerns between authorship and approval,
ensuring architectural decisions are peer-reviewed before costly implementation work begins.

## Technology & Stack Constraints

The Sapphire FitConnect AI platform operates as a brownfield multi-repo estate under the
`FDE-COHORT` GitHub organisation. The following constraints apply to all repositories:

- **Languages**: Java (Spring Boot), Python (FastAPI / LangGraph), TypeScript/React (frontend),
  Node.js (BFF / Apollo Server).
- **Messaging**: Kafka for all asynchronous event pipelines; schema changes MUST have contract
  tests.
- **Auth**: Keycloak OIDC/PKCE exclusively; no alternative auth providers.
- **Data stores**: Service-owned databases; cross-service data access MUST go through
  published API contracts, never direct DB connections.
- **CI/CD**: All repos MUST pass lint, type-check, unit tests, and contract tests in CI before
  merge; integration tests run in a separate pre-merge pipeline.
- **Deployment**: Container-first; all services run in Docker/Podman-compatible images.

## Development Workflow & Quality Gates

- Feature branches MUST be created from the story branch produced by `speckit.implement`.
- PRs MUST reference the JIRA story ID in the title (e.g., `SDDSDLC-170`).
- All PRs MUST pass: lint, type-check, unit tests, contract tests, and the Constitution Check
  table in `plan.md` before review is requested.
- Code review MUST be completed by at least one peer not involved in authoring the change.
- The Constitution supersedes all other engineering practices; where a conflict exists, the
  Constitution wins and an amendment must be raised if the practice is genuinely needed.
- Constitution amendments MUST be proposed via PR, reviewed by the FDE/FDA role, and
  documented with a rationale and version bump.
- Compliance review MUST occur at each plan-gate PR; the reviewer MUST confirm all
  Constitution Check rows are addressed.

## Governance

This Constitution supersedes all informal conventions, tribal knowledge, and undocumented
practices within the Sapphire FitConnect AI project.

- **Amendment procedure**: Open a PR modifying `.specify/memory/constitution.md`, increment
  the version per semantic versioning rules below, obtain approval from the FDE/FDA role, and
  run `/constitution.resolve` to regenerate `.specify/runtime/effective-constitution.md`.
- **Versioning policy**:
  - MAJOR — backward-incompatible governance change: principle removal or redefinition.
  - MINOR — new principle or section added; materially expanded guidance.
  - PATCH — clarifications, wording fixes, non-semantic refinements.
- **Compliance review**: Every plan-gate PR MUST include a completed Constitution Check table.
  Violations that cannot be remediated MUST be documented in the plan's Complexity Tracking
  table with justification.
- **Effective constitution**: Run `/constitution.resolve` after any local amendment to
  regenerate `.specify/runtime/effective-constitution.md`, which merges global org rules with
  this local constitution (local takes precedence on conflicts).

**Version**: 1.0.0 | **Ratified**: TODO(RATIFICATION_DATE): Set to the date the team formally adopts this constitution | **Last Amended**: 2025-08-25
