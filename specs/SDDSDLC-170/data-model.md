# Data Model: SDDSDLC-170 — Body Temperature Metric

**Date**: 2025-08-25
**Branch**: SDDSDLC-170

---

## 1. Core Entities

### 1.1 TemperatureReading (Ingestion / Event)

Represents a single body temperature measurement submitted by a device or API client.
This is the canonical inbound payload; it is never stored directly — it is validated,
transformed, and published to Kafka.

| Field | Type | Required | Constraints | Notes |
|-------|------|----------|-------------|-------|
| `user_id` | UUID | ✅ | Valid UUID | Resolved from Keycloak JWT sub claim |
| `value` | float | ✅ | See range below | Raw value in the unit supplied |
| `unit` | enum | ✅ | `C` or `F` | Input unit |
| `timestamp` | datetime (ISO 8601 UTC) | ✅ | Not more than 5 min in the future | Device measurement time |
| `device_source` | string | ✅ | 1–128 chars | Device identifier / model |
| `ingestion_source` | string | ✅ | 1–64 chars | API, SDK, manual |
| `measurement_method` | string | ❌ | 0–64 chars | Oral, axillary, tympanic, etc. |

**Validation rules**:
- Celsius: `30.0 ≤ value ≤ 45.0` (configurable via `TEMP_MIN_CELSIUS` / `TEMP_MAX_CELSIUS`)
- Fahrenheit: `86.0 ≤ value ≤ 113.0` (configurable via `TEMP_MIN_FAHRENHEIT` / `TEMP_MAX_FAHRENHEIT`)
- Fahrenheit → Celsius conversion: `(F - 32) × 5 / 9`
- Batch max: 100 records per request (configurable via `TEMP_BATCH_MAX_SIZE`)

---

### 1.2 BodyTemperatureEvent (Kafka Avro Message)

The event published to the health telemetry Kafka topic after validation and Fahrenheit→Celsius
conversion.

| Field | Avro Type | Notes |
|-------|-----------|-------|
| `event_id` | string (UUID) | Generated on ingest |
| `user_id` | string (UUID) | |
| `metric_type` | enum `MetricType` | Value: `BODY_TEMPERATURE` |
| `value_celsius` | double | Canonical storage value |
| `original_value` | double | Raw submitted value |
| `original_unit` | string | `C` or `F` |
| `device_source` | string | |
| `ingestion_source` | string | |
| `measurement_method` | union [null, string] | Default null |
| `recorded_at` | long (timestamp-millis) | Device timestamp |
| `ingested_at` | long (timestamp-millis) | Server-side ingest time |

---

### 1.3 HealthTelemetryBodyTemperature (PostgreSQL / TimescaleDB)

Time-series storage table for persisted temperature records, populated by Kafka Connect sink.

| Column | Type | Nullable | Constraints |
|--------|------|----------|-------------|
| `id` | BIGSERIAL | NOT NULL | PK |
| `user_id` | UUID | NOT NULL | Indexed (with recorded_at) |
| `value_celsius` | DOUBLE PRECISION | NOT NULL | |
| `original_value` | DOUBLE PRECISION | NOT NULL | |
| `original_unit` | VARCHAR(1) | NOT NULL | CHECK IN ('C','F') |
| `device_source` | VARCHAR(128) | NOT NULL | |
| `measurement_method` | VARCHAR(64) | NULL | |
| `ingestion_source` | VARCHAR(64) | NOT NULL | |
| `recorded_at` | TIMESTAMPTZ | NOT NULL | Hypertable partition key |
| `ingested_at` | TIMESTAMPTZ | NOT NULL | DEFAULT NOW() |

**Unique constraint**: `(user_id, device_source, recorded_at)` — ensures idempotent re-ingestion.

**Hypertable**: Partitioned on `recorded_at` with 7-day chunks (default TimescaleDB).

**Continuous aggregates**:
| View | Bucket | Used for range |
|------|--------|----------------|
| `temp_hourly_agg` | 1 hour | `range=day` (24 data points) |
| `temp_daily_agg` | 1 day | `range=week` (7 data points) |
| `temp_weekly_agg` | 7 days | `range=month` (4–5 data points) |

Each aggregate stores: `user_id`, `device_source`, `bucket`, `min_celsius`, `max_celsius`,
`avg_celsius`, `record_count`.

---

### 1.4 TemperatureTrendPoint (Charting API / BFF Response DTO)

Aggregated data point returned by the charting API and BFF for chart rendering.

| Field | Type | Notes |
|-------|------|-------|
| `window_start` | ISO 8601 UTC | Aggregate bucket start |
| `window_end` | ISO 8601 UTC | Aggregate bucket end |
| `min_value` | float | In the requested display unit |
| `max_value` | float | In the requested display unit |
| `avg_value` | float | In the requested display unit |
| `record_count` | int | Raw reading count in bucket |

---

## 2. Entity Relationships

```
TemperatureReading (inbound payload)
    │  validated + converted
    ▼
BodyTemperatureEvent (Avro, Kafka topic: health-telemetry)
    │  Kafka Connect sink
    ▼
HealthTelemetryBodyTemperature (TimescaleDB hypertable)
    │  continuous aggregate
    ▼
temp_hourly_agg / temp_daily_agg / temp_weekly_agg
    │  queried by
    ▼
TemperatureTrendPoint (charting API response)
    │  surfaced via
    ▼
BodyTemperatureChartResult (BFF GraphQL response)
    │  consumed by
    ▼
BodyTemperatureChart (React UI component)
```

---

## 3. Enum Extensions

### MetricType (extended)
```
BLOOD_PRESSURE  (existing)
SPO2            (existing)
ACTIVITY        (existing)
BODY_TEMPERATURE (new)
```

### TemperatureUnit
```
C   (Celsius — canonical storage unit)
F   (Fahrenheit — accepted on input; returned if client requests F)
```

### TemperatureChartRange
```
DAY   → hourly aggregates, last 24h
WEEK  → daily aggregates, last 7 days
MONTH → weekly aggregates, last 30 days
```

---

## 4. Validation Rules Summary

| Rule | Value | Configurable |
|------|-------|--------------|
| Min Celsius | 30.0°C | ✅ `TEMP_MIN_CELSIUS` |
| Max Celsius | 45.0°C | ✅ `TEMP_MAX_CELSIUS` |
| Min Fahrenheit | 86.0°F | ✅ `TEMP_MIN_FAHRENHEIT` |
| Max Fahrenheit | 113.0°F | ✅ `TEMP_MAX_FAHRENHEIT` |
| Max batch size | 100 records | ✅ `TEMP_BATCH_MAX_SIZE` |
| Future timestamp tolerance | 5 minutes | Fixed |
| Idempotency key | (user_id, device_source, recorded_at) | Fixed |
| Rate limit | 1,000 readings/user/minute | ✅ `TEMPERATURE_RATE_LIMIT_PER_USER_PER_MIN` |
