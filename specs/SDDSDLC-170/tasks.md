---
description: "Task list for SDDSDLC-170 — Add Support for Body Temperature Metric Ingestion, Storage, and Reporting"
---

# Tasks: SDDSDLC-170 — Body Temperature Metric

**Input**: Design documents from `specs/SDDSDLC-170/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/ ✅

**Organization**: Tasks are grouped by user story and repo. Each story can be implemented
and tested independently. Stories map to the spec priorities: P1 (Ingestion), P2 (UI Charts),
P3 (Analytics Export + Partner Sharing).

**Affected repos**:
- `sapphire-event-ingestion-api` (Python/FastAPI)
- `sapphire-kafka-pipeline` (Kafka Connect JSON)
- `sapphire-charting-api` (Java 17/Spring Boot)
- `sapphire-bff-api` (Node.js/Apollo Server)
- `Sapphire` (TypeScript/React)
- `sapphire-k6-bootstrap` (k6 JS — supporting task T052 only; no dedicated child story)

## Format: `[ID] [P?] [Story?] Description — repo: file path`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story label (US1, US2, US3)
- Paths shown are relative to each repo root

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Branch setup and schema/config changes that other tasks depend on.

- [ ] T001 [P] Create feature branch `SDDSDLC-170` in `sapphire-event-ingestion-api`
- [ ] T002 [P] Create feature branch `SDDSDLC-170` in `sapphire-kafka-pipeline`
- [ ] T003 [P] Create feature branch `SDDSDLC-170` in `sapphire-charting-api`
- [ ] T004 [P] Create feature branch `SDDSDLC-170` in `sapphire-bff-api`
- [ ] T005 [P] Create feature branch `SDDSDLC-170` in `Sapphire`

**Checkpoint**: Feature branches created — team can begin parallel work.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Schema and infrastructure changes that ALL user stories depend on.
MUST be complete before any user story work begins.

### 2A — Avro Schema Extension (sapphire-event-ingestion-api)

- [ ] T006 Extend `app/schemas/avro/health_telemetry.avsc` — add `BODY_TEMPERATURE` to `MetricType` enum with `"default": "UNKNOWN"` for backward compatibility — repo: `sapphire-event-ingestion-api`
- [ ] T007 Add `BodyTemperaturePayload` Avro record to `app/schemas/avro/health_telemetry.avsc` with fields: `value_celsius` (double), `original_value` (double), `original_unit` (string), `device_source` (string), `ingestion_source` (string), `measurement_method` (union null/string, default null) — repo: `sapphire-event-ingestion-api`
- [ ] T008 [P] Write Avro schema evolution compatibility test in `tests/unit/test_avro_schema_evolution.py` — verify old consumers with default enum can deserialise new `BODY_TEMPERATURE` events — repo: `sapphire-event-ingestion-api`

### 2B — TimescaleDB Schema (sapphire-kafka-pipeline)

- [ ] T009 Create SQL migration `migrations/V2__body_temperature_hypertable.sql` — creates `health_telemetry_body_temperature` table, unique constraint `(user_id, device_source, recorded_at)`, hypertable declaration, and composite index `(user_id, recorded_at DESC)` — repo: `sapphire-kafka-pipeline`
- [ ] T010 Create SQL migration `migrations/V3__body_temperature_continuous_aggregates.sql` — defines `temp_hourly_agg` (1h), `temp_daily_agg` (1d), and `temp_weekly_agg` (7d) continuous aggregates with `min_celsius`, `max_celsius`, `avg_celsius`, `record_count` — repo: `sapphire-kafka-pipeline`
- [ ] T011 Create Kafka Connect sink connector config `connectors/body-temperature-sink.json` — routes `MetricType == BODY_TEMPERATURE` events from `health-telemetry` topic to `health_telemetry_body_temperature` table using SMT field routing — repo: `sapphire-kafka-pipeline`

### 2C — @ConfigurationProperties (sapphire-event-ingestion-api)

- [ ] T012 [P] Create `app/config/temperature_config.py` — Pydantic v2 `BaseSettings` class for `TEMP_MIN_CELSIUS`, `TEMP_MAX_CELSIUS`, `TEMP_MIN_FAHRENHEIT`, `TEMP_MAX_FAHRENHEIT`, `TEMP_BATCH_MAX_SIZE` (default 100), `TEMPERATURE_RATE_LIMIT_PER_USER_PER_MIN` (default 1000) — repo: `sapphire-event-ingestion-api`

**Checkpoint**: Foundation ready — ingestion schema, DB schema, and config in place. User story work can begin.

---

## Phase 3: User Story 1 — Ingest Body Temperature from Smart Device (Priority: P1) 🎯 MVP

**Goal**: Accept valid single and batch temperature readings, validate, convert units, publish
to Kafka. Reject invalid/unauthenticated requests with correct HTTP status codes.

**Independent Test**: Submit a valid Celsius reading via
`POST /telemetry/body-temperature` with a Keycloak JWT → verify HTTP 202 + Kafka event.
Submit out-of-range value → verify HTTP 422. No UI or chart needed.

**Repo**: `sapphire-event-ingestion-api`

### Implementation for User Story 1

- [ ] T013 [P] [US1] Create `app/models/temperature.py` — Pydantic v2 `TemperatureReadingRequest` model (value: float, unit: Literal["C","F"], timestamp: datetime, device_source: str, ingestion_source: str, measurement_method: Optional[str]) and `TemperatureReadingBatchRequest` model (readings: list[TemperatureReadingRequest], min 1, max 100 — enforced via `Field(min_length=1, max_length=100)`) — repo: `sapphire-event-ingestion-api`
- [ ] T014 [P] [US1] Create `app/validators/temperature_validator.py` — `validate_temperature_reading()` function: range check using config values, Fahrenheit-to-Celsius conversion `(F-32)*5/9`, future-timestamp check (≤5 min), returns `value_celsius` and validation errors — repo: `sapphire-event-ingestion-api`
- [ ] T015 [P] [US1] Create `app/services/rate_limit_service.py` — Redis sliding window counter keyed on `user_id`; raises `RateLimitExceededError` with `retry_after_seconds` when limit exceeded; reads limit from `temperature_config.py` — repo: `sapphire-event-ingestion-api`
- [ ] T016 [US1] Create `app/routes/temperature.py` — async FastAPI router with `POST /telemetry/body-temperature` (single) and `POST /telemetry/body-temperature/batch` (batch); validates JWT via `Depends(get_current_user)`; calls `rate_limit_service`, `temperature_validator`, Kafka producer; returns HTTP 202 / 207 / 400 / 401 / 422 / 429 / 503 per spec; emits OTEL spans with W3C traceparent; emits `temperature_readings_ingested_total` counter — depends on T013, T014, T015 — repo: `sapphire-event-ingestion-api`
- [ ] T017 [US1] Register temperature router in `app/main.py` — `app.include_router(temperature_router, prefix="/telemetry")` — repo: `sapphire-event-ingestion-api`
- [ ] T018 [P] [US1] Add structlog fields to all new handlers: `temperature_validator.py` and `temperature.py` route — ensure every log line includes `trace_id` and `span_id` from OTEL context — repo: `sapphire-event-ingestion-api`

### Tests for User Story 1

- [ ] T019 [P] [US1] Create `tests/unit/test_temperature_validator.py` — test range check (valid C, valid F, out-of-range C, out-of-range F), unit conversion accuracy, future timestamp rejection, missing required fields — repo: `sapphire-event-ingestion-api`
- [ ] T020 [P] [US1] Create `tests/unit/test_rate_limit_service.py` — test sliding window counter increment, limit enforcement, retry_after calculation, using Redis mock — repo: `sapphire-event-ingestion-api`
- [ ] T021 [US1] Create `tests/integration/test_temperature_ingestion.py` — integration tests using `TestClient`: single valid C reading (202), single valid F reading (202 + verify conversion), out-of-range (422), unauthenticated (401), batch mixed-validity (207), empty batch (400), oversized batch (422), rate limit (429), Kafka unavailable (503), duplicate submission identical (user_id + device_source + timestamp) returns 202 but does not create a second record (idempotency) — depends on T016 — repo: `sapphire-event-ingestion-api`

**Checkpoint**: User Story 1 complete. Ingestion pipeline functional and tested independently.

---

## Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2)

**Goal**: Expose temperature trend data through charting API, BFF GraphQL, and React UI
chart component. Supports day/week/month ranges, Celsius/Fahrenheit display, loading/error/empty states.

**Independent Test**: With 7 days of pre-seeded data, call `GET /users/{userId}/body-temperature/chart?range=week` → verify 7 daily aggregates. Open React chart component → verify all three states (loading skeleton, data, empty).

**Repos**: `sapphire-charting-api`, `sapphire-bff-api`, `Sapphire`

### 4A — Charting API (sapphire-charting-api)

- [ ] T022 [P] [US2] Create `src/main/java/com/sapphire/charting/temperature/dto/TemperatureTrendPoint.java` — Java record: `windowStart` (Instant), `windowEnd` (Instant), `minValue` (double), `maxValue` (double), `avgValue` (double), `recordCount` (long) — repo: `sapphire-charting-api`
- [ ] T023 [P] [US2] Create `src/main/java/com/sapphire/charting/temperature/dto/TemperatureTrendResponse.java` — Java record: `userId` (String), `range` (String), `unit` (String), `dataPoints` (List\<TemperatureTrendPoint\>) — repo: `sapphire-charting-api`
- [ ] T024 [P] [US2] Create `src/main/java/com/sapphire/charting/temperature/config/TemperatureProperties.java` — `@ConfigurationProperties(prefix = "temperature")` with `minCelsius`, `maxCelsius` bound from `application.yml` — repo: `sapphire-charting-api`
- [ ] T025 [US2] Create `src/main/java/com/sapphire/charting/temperature/TemperatureChartRepository.java` — Spring Data JPA repository with custom `@Query` methods querying `temp_hourly_agg`, `temp_daily_agg`, `temp_weekly_agg` views by `userId`, optional `deviceSource`, and time range; for `range=day`, if `temp_hourly_agg` returns 0 rows (user has < 1h of data), fall back to raw `health_telemetry_body_temperature` records for the current day ordered by `recorded_at` — repo: `sapphire-charting-api`
- [ ] T026 [US2] Create `src/main/java/com/sapphire/charting/temperature/TemperatureChartService.java` — `@Service`: selects correct aggregate view by `range` param, applies optional `deviceSource` filter, converts values to requested `unit` (C→F: `value * 9/5 + 32`), returns `TemperatureTrendResponse` — depends on T022, T023, T025 — repo: `sapphire-charting-api`
- [ ] T027 [US2] Create `src/main/java/com/sapphire/charting/temperature/TemperatureChartController.java` — `@RestController` with `GET /users/{userId}/body-temperature/chart` mapping; validates JWT user matches path `userId` (returns 403 if mismatch); returns 404 with `EmptyTrendResponse` if no data; logs with Logback+logstash including `trace_id` and `span_id` — depends on T026 — repo: `sapphire-charting-api`
- [ ] T028 [P] [US2] Create `src/test/java/com/sapphire/charting/temperature/TemperatureChartControllerTest.java` — `@WebMvcTest` slice: valid week range (200), Fahrenheit conversion (200), no data (404), wrong userId JWT (403), unauthenticated (401) — repo: `sapphire-charting-api`
- [ ] T029 [P] [US2] Create `src/test/java/com/sapphire/charting/temperature/TemperatureChartServiceTest.java` — JUnit 5 + Mockito: unit test range→view routing logic, C→F conversion accuracy, empty result handling — repo: `sapphire-charting-api`

### 4B — BFF GraphQL (sapphire-bff-api)

- [ ] T030 [P] [US2] Create `src/schema/temperature.graphql` — schema extension per contract `specs/SDDSDLC-170/contracts/bff-graphql-schema.graphql`: `BodyTemperatureChartInput`, `BodyTemperatureChartResult`, `BodyTemperatureTrendPoint`, `TemperatureChartRange` enum, `TemperatureUnit` enum, `extend type Query { bodyTemperatureChart(...) }` — repo: `sapphire-bff-api`
- [ ] T031 [US2] Create `src/resolvers/temperature.resolver.ts` — typed resolver for `bodyTemperatureChart`; validates JWT claims first; delegates to charting API REST call via service client; returns null (not error) for empty data; logs via pino with `trace_id` and `span_id` — depends on T030 — repo: `sapphire-bff-api`
- [ ] T032 [P] [US2] Create `src/dataloaders/temperatureChartLoader.ts` — DataLoader batching temperature chart requests per `userId` to prevent N+1 calls — repo: `sapphire-bff-api`
- [ ] T033 [US2] Register temperature schema, resolver, and DataLoader in BFF Apollo Server entry point — depends on T030, T031, T032 — repo: `sapphire-bff-api`
- [ ] T034 [P] [US2] Create `src/resolvers/temperature.resolver.test.ts` — Jest unit tests: valid query (200), null return for empty data, JWT validation rejection, resolver delegates to DataLoader not direct REST — repo: `sapphire-bff-api`

### 4C — React UI (Sapphire)

- [ ] T035 [P] [US2] Create `src/features/body-temperature/queries/bodyTemperatureChart.graphql` — Apollo Client query using `bodyTemperatureChart(input: $input)` with all `BodyTemperatureTrendPoint` fields; `fetchPolicy: "cache-and-network"` to prevent stale health data — repo: `Sapphire`
- [ ] T036 [P] [US2] Create `src/features/body-temperature/bodyTemperatureChart.types.ts` — TypeScript interfaces: `TemperatureChartRange`, `TemperatureUnit`, `BodyTemperatureTrendPoint`, `BodyTemperatureChartResult`; no `any` types — repo: `Sapphire`
- [ ] T037 [US2] Create `src/features/body-temperature/useBodyTemperatureChart.ts` — custom hook wrapping Apollo `useQuery`; accepts `userId`, `range`, `unit`, optional `deviceSource`; returns `{ data, loading, error }`; URL query params (`range`, `unit`) are the source of truth for selections — depends on T035, T036 — repo: `Sapphire`
- [ ] T038 [US2] Create `src/features/body-temperature/BodyTemperatureChart.tsx` — React functional component: renders loading skeleton while `loading=true`, error boundary message when `error` present, empty-state message when `data.dataPoints.length === 0`, chart with min/max/avg lines when data present; day/week/month range selector updates URL param; unit toggle (°C/°F) updates URL param and triggers client-side display conversion (no new network request) — depends on T037 — repo: `Sapphire`
- [ ] T039 [US2] Add `BodyTemperatureChart` to the health dashboard metrics list view — integrate into existing metrics dashboard layout alongside blood pressure, SpO2, activity — depends on T038 — repo: `Sapphire`
- [ ] T040 [P] [US2] Create `src/features/body-temperature/BodyTemperatureChart.test.tsx` — React Testing Library: loading skeleton renders, error boundary renders on Apollo error, empty state renders with no data, chart renders with seeded data for each range, unit toggle updates displayed values, snapshot test — repo: `Sapphire`

**Checkpoint**: User Story 2 complete. Full chart flow from DB → charting API → BFF → UI functional.

---

## Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3)

**Goal**: Include temperature data in analytics export endpoints; update OpenAPI/GraphQL schema
documentation; ensure temperature metric appears in health analytics dashboard metrics list.

**Independent Test**: Call analytics export endpoint for user with seeded temperature data → verify
temperature records in response. Check OpenAPI spec includes `body_temperature` metric type.

**Repos**: `sapphire-charting-api`, `sapphire-bff-api`

- [ ] T041 [US3] Add temperature records to `sapphire-charting-api` analytics export endpoint — extend existing analytics export service/controller to include temperature data from `health_telemetry_body_temperature` table; return `value_celsius`, `original_unit`, `device_source`, `measurement_method`, `recorded_at` fields — repo: `sapphire-charting-api`
- [ ] T042 [US3] Update OpenAPI spec `openapi.yaml` (or equivalent) in `sapphire-event-ingestion-api` — document `body_temperature` metric type, all `TemperatureReadingRequest` fields, valid ranges (30–45°C / 86–113°F), supported units (C/F) as per contract `specs/SDDSDLC-170/contracts/ingestion-api.yaml` — repo: `sapphire-event-ingestion-api`
- [ ] T043 [P] [US3] Update OpenAPI spec in `sapphire-charting-api` — document new `GET /users/{userId}/body-temperature/chart` endpoint as per contract `specs/SDDSDLC-170/contracts/charting-api.yaml` — repo: `sapphire-charting-api`
- [ ] T044 [P] [US3] Add `bodyTemperatureChart` query to BFF GraphQL schema documentation — confirm schema introspection returns `bodyTemperatureChart` with all types and deprecation fields correctly documented — repo: `sapphire-bff-api`
- [ ] T045 [P] [US3] Create `src/test/java/.../AnalyticsExportTemperatureTest.java` — `@WebMvcTest` or `@SpringBootTest`: verify temperature records present in analytics export response for user with data; verify auth enforcement (401/403) — repo: `sapphire-charting-api`

**Checkpoint**: User Story 3 complete. All acceptance criteria met — temperature data in exports, schema documented.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Observability validation, test coverage gates, E2E quickstart validation.

- [ ] T046 [P] Verify `temperature_readings_ingested_total` counter appears in OTEL Collector metrics output — add integration test or manual check per `specs/SDDSDLC-170/quickstart.md` Step 8 — repo: `sapphire-event-ingestion-api`
- [ ] T047 [P] Verify structured logs (trace_id, span_id) appear on all new temperature paths in all 3 backend repos — repo: `sapphire-event-ingestion-api`, `sapphire-charting-api`, `sapphire-bff-api`
- [ ] T048 [P] Run test coverage report for `sapphire-event-ingestion-api` — verify ≥80% coverage on new temperature modules per SC-009 — repo: `sapphire-event-ingestion-api`
- [ ] T049 [P] Run test coverage report for `sapphire-charting-api` — verify ≥80% overall, 100% domain layer coverage per SC-010 — repo: `sapphire-charting-api`
- [ ] T050 [P] Run test coverage report for `Sapphire` — verify ≥70% on `src/features/body-temperature/` per constitution — repo: `Sapphire`
- [ ] T051 [P] Run full E2E quickstart validation per `specs/SDDSDLC-170/quickstart.md` Steps 1–8 — confirm all quickstart checklist items pass — all repos
- [ ] T052 Update `sapphire-k6-bootstrap` data seeder to include body temperature metric generation — required for quickstart Step 4 pre-seeding and load testing — repo: `sapphire-k6-bootstrap`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (Setup)**: No dependencies — start immediately. All 5 branches in parallel.
- **Phase 2 (Foundational)**: Depends on Phase 1. T006–T008 and T009–T011 and T012 are all parallel.
- **Phase 3 (US1 — Ingestion)**: Depends on Phase 2 (T006, T007, T012). Can start while Phase 2B (Kafka) continues.
- **Phase 4 (US2 — Charts)**: Depends on Phase 3 completion (data must be ingestable to seed test data).
  - 4A (Charting API), 4B (BFF), 4C (React UI) can run in parallel with each other.
- **Phase 5 (US3 — Export)**: Depends on Phase 4A foundation (charting API data access established).
- **Phase 6 (Polish)**: Depends on all user story phases complete.

### User Story Dependencies

- **US1 (P1)**: Depends on Phase 2A (Avro schema) + Phase 2C (config). No US dependencies.
- **US2 (P2)**: Depends on US1 (ingestion working) + Phase 2B (storage ready for seeded data).
- **US3 (P3)**: Depends on Phase 4A (charting API data access established). Can partially parallel US2.

### Within Each User Story

- Models/validators/config before services
- Services before routes/controllers
- Routes/controllers before integration tests
- Schema files (Avro, GraphQL, OpenAPI) before dependent implementations

### Parallel Opportunities

```bash
# Phase 2 — all three foundational areas in parallel:
Task Group A (Avro): T006, T007, T008
Task Group B (TimescaleDB): T009, T010, T011
Task Group C (Config): T012

# Phase 3 — US1 internal parallelism:
Task Group (models + validator + rate limit): T013, T014, T015

# Phase 4 — full repo-level parallelism:
Developer A: Charting API (T022–T029)
Developer B: BFF GraphQL (T030–T034)
Developer C: React UI (T035–T040)

# Phase 4 internal parallelism (within each repo):
T022, T023, T024 (charting DTOs + config) — parallel
T030, T035, T036 (schema + types) — parallel across repos
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Branch setup
2. Complete Phase 2A + 2C: Avro schema + config (US1 foundation)
3. Complete Phase 3: US1 Ingestion
4. **STOP and VALIDATE**: Use quickstart.md Step 1–2 to verify ingestion + Kafka event
5. Validate independently before proceeding to charting

### Incremental Delivery

1. Phase 1 → Phase 2 → Phase 3 → **MVP: Temperature ingestion working**
2. Phase 4 → **Charts working end-to-end** (charting API + BFF + UI simultaneously)
3. Phase 5 → **Export + documentation complete**
4. Phase 6 → **Observability, coverage gates, full E2E validation**

### Parallel Team Strategy

With 3+ developers:
- Developer A: `sapphire-event-ingestion-api` (Phase 2A, 2C, Phase 3 fully)
- Developer B: `sapphire-kafka-pipeline` (Phase 2B) + `sapphire-charting-api` (Phase 4A)
- Developer C: `sapphire-bff-api` (Phase 4B) + `Sapphire` (Phase 4C)

---

## Notes

- `[P]` tasks = different files, no dependencies on in-progress tasks in same phase
- `[US1/2/3]` label maps each task to a specific user story for traceability
- Each user story is independently completable and testable
- Phase 2B (TimescaleDB) and Phase 2A (Avro) can proceed in parallel
- Verify quickstart.md checklist items after each story phase
- Total tasks: **52** across 5 repos and 3 user stories
