# Implementation Plan: SDDSDLC-170 — Add Support for Body Temperature Metric

**Branch**: `SDDSDLC-170` | **Date**: 2025-08-25 | **Spec**: `specs/SDDSDLC-170/spec.md`
**Input**: Feature specification from `/specs/SDDSDLC-170/spec.md`

---

## Summary

Add body temperature as a first-class health metric across the Sapphire FitConnect platform.
The feature spans five repositories: ingestion (Python/FastAPI + Avro/Kafka), time-series
storage (Kafka Connect + TimescaleDB), charting (Java/Spring Boot), BFF (Node.js/Apollo
GraphQL), and UI (TypeScript/React). All changes are additive — no existing metric paths,
schemas, or API contracts are modified. Backward compatibility confirmed (R-06 in
research.md).

---

## Technical Context

**Languages/Versions**:
- `sapphire-event-ingestion-api`: Python 3.11 / FastAPI / Pydantic v2 / Confluent Kafka Python
- `sapphire-kafka-pipeline`: Kafka Connect JSON (no code — configuration only)
- `sapphire-charting-api`: Java 17 / Spring Boot 3.x / Spring Data JPA / TimescaleDB
- `sapphire-bff-api`: Node.js 20 / Apollo Server 4 / TypeScript
- `Sapphire`: TypeScript / React 18 / Vite / Apollo Client 3

**Primary Dependencies**:
- Avro schema registry (existing), Redis (existing, for rate limiting), TimescaleDB (existing)
- No new infrastructure dependencies introduced

**Storage**: TimescaleDB hypertable `health_telemetry_body_temperature` (new table, existing DB)

**Testing**:
- `sapphire-event-ingestion-api`: pytest (unit + integration with Testcontainers)
- `sapphire-charting-api`: JUnit 5 / Mockito / @WebMvcTest / @DataJpaTest
- `sapphire-bff-api`: Jest / Apollo Client mock
- `Sapphire`: Vitest / React Testing Library

**Performance Goals**:
- Ingestion endpoint HTTP response: ≤ 500ms p95
- End-to-end (ingest → queryable in storage): ≤ 5s
- Batch of 100 records → all Kafka events: ≤ 10s

**Constraints**:
- All changes additive; no breaking changes to existing API contracts
- Auth exclusively via Keycloak OIDC/PKCE; no bypass routes
- Rate limit: 1,000 readings/user/minute (Redis sliding window)

**Scale/Scope**: 5 repos, ~5,000 active platform users, existing TimescaleDB handles scale

---

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| # | Gate | Principle | Status |
|---|------|-----------|--------|
| 1 | All public functions/methods/classes have docstrings or Javadoc | I. Code Quality | ✅ Planned — all new classes/functions will include docstrings/Javadoc |
| 2 | No magic numbers or strings — named constants or enums used | I. Code Quality | ✅ All thresholds (30.0°C, 45.0°C, 100 batch max) are @ConfigurationProperties / env vars |
| 3 | Cyclomatic complexity ≤ 10 per function/method | I. Code Quality | ✅ Each component (validator, converter, controller, resolver) is single-responsibility |
| 4 | No commented-out code committed | I. Code Quality | ✅ Standard practice enforced by CI lint |
| 5 | Stack-specific rules applied | I. Code Quality | ✅ Spring layering, PEP 8+ruff, strict TS, DataLoader in BFF |
| 6 | Coverage gates: Java 80%/100% domain, Python 80%, TS 70%, BFF resolvers 100% | II. Testing Standards | ✅ Coverage targets defined in SC-009/SC-010; CI gates enforced per repo |
| 7 | Contract tests for GraphQL schema and Kafka Avro schema changes | II. Testing Standards | ✅ Avro schema evolution test planned; BFF GraphQL resolver test planned |
| 8 | Test pyramid respected | II. Testing Standards | ✅ Unit (mocked I/O) + integration (Docker Compose, pre-merge) + quickstart.md E2E |
| 9 | Data-fetching components handle loading/error/empty states | III. UX Consistency | ✅ FR-011 mandates all three states; quickstart.md Step 6 validates |
| 10 | Auth exclusively Keycloak OIDC/PKCE; no bypass routes | III. UX Consistency | ✅ FR-015, SC-007; ingestion + charting + BFF all validate JWT |
| 11 | URL state is source of truth for filters, pagination, selections | III. UX Consistency | ✅ Range and unit selections persisted in URL query params in Sapphire UI |
| 12 | Apollo cache policies explicit; no implicit cache-first for mutable health data | I. Code Quality | ✅ `fetchPolicy: 'cache-and-network'` required for bodyTemperatureChart query |
| 13 | Structured JSON logs with trace_id and span_id | IV. Observability | ✅ FR-016 mandates; all new paths instrumented |
| 14 | OTEL metrics exported; custom business metrics added | IV. Observability | ✅ FR-017: `temperature_readings_ingested_total` counter with unit+status labels |
| 15 | Distributed traces via OTEL SDK; W3C traceparent; DB/HTTP/Kafka as child spans | IV. Observability | ✅ FR-016; ingestion → Kafka publish + charting → DB query instrumented as spans |
| 16 | OTEL env vars set in every container | IV. Observability | ✅ Existing pattern; no new containers — env vars already present |
| 17 | LangGraph rules (N/A — no LangGraph in this feature) | I. Code Quality | N/A |
| 18 | API backward-compatibility assessment complete | V. API Backward Compatibility | ✅ R-06: all changes additive; no MAJOR version bump required |

**No constitution violations.** Complexity Tracking section not needed.

---

## Project Structure

### Documentation (this feature)

```text
specs/SDDSDLC-170/
├── plan.md              ← this file
├── spec.md
├── research.md
├── data-model.md
├── quickstart.md
├── observe-workflow.md
├── workflow-state.md
├── checklists/
│   └── requirements.md
└── contracts/
    ├── ingestion-api.yaml
    ├── charting-api.yaml
    └── bff-graphql-schema.graphql
```

### Source Code (multi-repo)

```text
sapphire-event-ingestion-api/          (Python/FastAPI)
├── app/
│   ├── models/
│   │   └── temperature.py             NEW — Pydantic v2 TemperatureReadingRequest / batch
│   ├── validators/
│   │   └── temperature_validator.py   NEW — range check, unit conversion, rate limit
│   ├── routes/
│   │   └── temperature.py             NEW — POST /telemetry/body-temperature (+ /batch)
│   └── schemas/
│       └── avro/
│           └── health_telemetry.avsc  MODIFY — add BODY_TEMPERATURE to MetricType enum
└── tests/
    ├── unit/test_temperature_*.py     NEW
    └── integration/test_temperature_*.py  NEW

sapphire-kafka-pipeline/               (Kafka Connect JSON)
└── connectors/
    └── body-temperature-sink.json     NEW — Kafka Connect sink connector config

sapphire-charting-api/                 (Java 17 / Spring Boot)
└── src/main/java/com/sapphire/charting/
    ├── temperature/
    │   ├── TemperatureChartController.java   NEW
    │   ├── TemperatureChartService.java      NEW
    │   ├── TemperatureChartRepository.java   NEW
    │   └── dto/
    │       ├── TemperatureTrendResponse.java NEW (record)
    │       └── TemperatureTrendPoint.java    NEW (record)
    └── config/
        └── TemperatureProperties.java        NEW (@ConfigurationProperties)

sapphire-bff-api/                      (Node.js / Apollo Server)
└── src/
    ├── schema/
    │   └── temperature.graphql        NEW — extend Query + types
    ├── resolvers/
    │   └── temperature.resolver.ts    NEW
    └── dataloaders/
        └── temperatureChartLoader.ts  NEW

Sapphire/                              (TypeScript/React)
└── src/features/body-temperature/
    ├── BodyTemperatureChart.tsx       NEW
    ├── useBodyTemperatureChart.ts     NEW (custom hook)
    ├── bodyTemperatureChart.types.ts  NEW
    ├── BodyTemperatureChart.test.tsx  NEW
    └── queries/
        └── bodyTemperatureChart.graphql  NEW
```

**Structure Decision**: Multi-repo brownfield. Each repo receives the minimum additive
change following its existing conventions. No new repos or frameworks introduced.

---

## Complexity Tracking

*No constitution violations — this section is intentionally empty.*
