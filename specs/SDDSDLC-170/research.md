# Research: SDDSDLC-170 — Body Temperature Metric

**Date**: 2025-08-25
**Branch**: SDDSDLC-170

---

## R-01: Avro Schema Extension Pattern (sapphire-event-ingestion-api)

**Decision**: Extend the existing health telemetry Avro schema by adding `body_temperature`
as a new enum value in the `MetricType` field. A new dedicated record type
`BodyTemperaturePayload` is defined alongside existing payload types (e.g.,
`BloodPressurePayload`, `SpO2Payload`). The top-level event envelope uses a union type for
the metric payload.

**Rationale**: This follows the existing pattern (adding a new enum value + payload type)
without requiring a new Kafka topic. Avro schema evolution rules allow adding a new enum
symbol with a default — backward and forward compatible with existing consumers.

**Alternatives considered**:
- New Kafka topic for temperature: rejected — adds operational overhead and breaks the
  unified ingestion pipeline pattern.
- Separate Avro schema file: rejected — the existing schema registry pattern uses a single
  health telemetry schema; splitting would require consumer-side schema resolution changes.

**Schema evolution compatibility**:
- Adding `BODY_TEMPERATURE` to the `MetricType` enum with `"default": "UNKNOWN"` is
  backward compatible (old readers ignore new enum values when a default is set).
- New Avro `BodyTemperaturePayload` record fields: `value_celsius` (double), `original_value`
  (double), `original_unit` (string: "C" | "F"), `device_source` (string),
  `measurement_method` (nullable string, default null).

---

## R-02: TimescaleDB Hypertable Schema for Temperature (sapphire-kafka-pipeline)

**Decision**: Add a new time-series table `health_telemetry_body_temperature` as a
TimescaleDB hypertable partitioned on `recorded_at`. The table mirrors the structure of
existing metric tables (e.g., `health_telemetry_blood_pressure`).

**Rationale**: Isolating metric types in separate tables avoids sparse rows and allows
per-metric index tuning. The Kafka Connect JSON sink connector is configured with a new
single-message transform (SMT) that routes `MetricType == BODY_TEMPERATURE` events to this
table.

**Table schema**:
```sql
CREATE TABLE health_telemetry_body_temperature (
  id            BIGSERIAL,
  user_id       UUID        NOT NULL,
  value_celsius DOUBLE PRECISION NOT NULL,
  original_value DOUBLE PRECISION NOT NULL,
  original_unit  VARCHAR(1)  NOT NULL CHECK (original_unit IN ('C', 'F')),
  device_source  VARCHAR(128) NOT NULL,
  measurement_method VARCHAR(64),
  ingestion_source   VARCHAR(64) NOT NULL,
  recorded_at   TIMESTAMPTZ NOT NULL,
  ingested_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  UNIQUE (user_id, device_source, recorded_at)  -- idempotency constraint
);
SELECT create_hypertable('health_telemetry_body_temperature', 'recorded_at');
CREATE INDEX ON health_telemetry_body_temperature (user_id, recorded_at DESC);
```

**Retention/aggregation**: Uses the same TimescaleDB continuous aggregate policy as
existing metric tables. Three continuous aggregates are defined:
- `temp_hourly_agg` (1h bucket) — for "day" range queries
- `temp_daily_agg` (1d bucket) — for "week" range queries
- `temp_weekly_agg` (7d bucket) — for "month" range queries

---

## R-03: Charting API Aggregation Pattern (sapphire-charting-api)

**Decision**: Add a new `TemperatureChartController` following the existing pattern used for
other metric charting endpoints. Exposes three aggregation endpoints:
- `GET /users/{userId}/body-temperature/chart?range=day`
- `GET /users/{userId}/body-temperature/chart?range=week`
- `GET /users/{userId}/body-temperature/chart?range=month`
- `GET /users/{userId}/body-temperature/chart?range=day&deviceSource=<src>`

**Rationale**: Resource-based REST paths required by constitution. The `range` query
parameter drives which continuous aggregate view is queried. Filtering by `deviceSource` is
a secondary optional filter aligned with FR-008.

**Response DTO**:
```java
public record TemperatureTrendResponse(
    String userId,
    String range,
    String unit,          // "C" or "F"
    List<TemperatureTrendPoint> dataPoints
) {}

public record TemperatureTrendPoint(
    Instant windowStart,
    Instant windowEnd,
    double minValue,
    double maxValue,
    double avgValue,
    long recordCount
) {}
```

**Unit conversion**: The charting API returns Celsius by default. A `unit` query parameter
(`C` or `F`) triggers server-side conversion for the response (multiply by 9/5 + 32).
This avoids client-side conversion complexity across chart libraries.

---

## R-04: BFF GraphQL Schema Extension (sapphire-bff-api)

**Decision**: Extend the existing `HealthMetrics` GraphQL type (or equivalent) with a new
`bodyTemperatureChart` field. A new `BodyTemperatureChartInput` input type encapsulates
query parameters. A DataLoader is added to batch temperature chart requests per user.

**Rationale**: Follows the existing BFF pattern of adding a new field to the health metrics
query rather than creating a top-level query (which would break schema cohesion).

**Schema addition**:
```graphql
type Query {
  # existing...
  bodyTemperatureChart(input: BodyTemperatureChartInput!): BodyTemperatureChartResult
}

input BodyTemperatureChartInput {
  userId: ID!
  range: TemperatureChartRange!
  unit: TemperatureUnit
  deviceSource: String
}

enum TemperatureChartRange { DAY WEEK MONTH }
enum TemperatureUnit { C F }

type BodyTemperatureChartResult {
  userId: ID!
  range: TemperatureChartRange!
  unit: TemperatureUnit!
  dataPoints: [BodyTemperatureTrendPoint!]!
}

type BodyTemperatureTrendPoint {
  windowStart: DateTime!
  windowEnd: DateTime!
  minValue: Float!
  maxValue: Float!
  avgValue: Float!
  recordCount: Int!
}
```

---

## R-05: Rate Limiting Implementation (sapphire-event-ingestion-api)

**Decision**: Implement per-user rate limiting using a Redis-backed sliding window counter
keyed on `user_id`. The ingestion service already uses Redis for notification pub/sub
(per AGENTS.md infrastructure table), so no new infrastructure dependency is introduced.

**Rationale**: Token bucket or sliding window counters stored in Redis provide accurate
per-user limiting without sticky sessions. The limit (1,000 readings/min) is read from
`TEMPERATURE_RATE_LIMIT_PER_USER_PER_MIN` environment variable.

**Response on limit exceeded**: HTTP 429 with header
`Retry-After: <seconds until window resets>`.

---

## R-06: Backward Compatibility Assessment (Constitution Gate 18)

**Classification**: **Non-breaking additive change** across all affected APIs.

| API | Change type | Assessment |
|-----|-------------|------------|
| Ingestion REST endpoint | New `body_temperature` payload accepted in existing endpoint | Additive — existing consumers unaffected |
| Avro schema (Kafka) | New enum value + union member | Additive with default — backward compatible |
| Charting API REST | New resource paths `/users/{id}/body-temperature/chart` | Additive — no existing paths changed |
| BFF GraphQL | New `bodyTemperatureChart` query field | Additive — `@deprecated` not needed; no existing fields changed |
| PostgreSQL schema | New hypertable table | Additive — no existing tables altered |

No MAJOR version bump required. No migration guide needed. Integration partners are notified
via updated OpenAPI spec documentation (FR-014).
