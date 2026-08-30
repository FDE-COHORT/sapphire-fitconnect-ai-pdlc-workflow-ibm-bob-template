# Quickstart: SDDSDLC-170 — Body Temperature Metric
**Branch**: SDDSDLC-170 | **Date**: 2025-08-25

This guide describes how to validate the body temperature feature end-to-end once all
implementation tasks are complete.

---

## Prerequisites

- All 5 affected repos cloned as siblings of this sidekick repo under `<PRODUCT_HOME>/`
- Docker / Podman Compose stack running (PostgreSQL, Kafka, Zookeeper, Redis, Keycloak,
  TimescaleDB extension enabled)
- A valid test user Keycloak JWT (obtain via k6-bootstrap or Keycloak admin console)
- `TEMP_MIN_CELSIUS=30.0`, `TEMP_MAX_CELSIUS=45.0` set in `sapphire-event-ingestion-api`
  environment

---

## Step 1 — Verify Ingestion (sapphire-event-ingestion-api)

```bash
# Single valid Celsius reading
curl -X POST http://localhost:8081/telemetry/body-temperature \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "value": 37.1,
    "unit": "C",
    "timestamp": "2025-08-25T09:00:00Z",
    "device_source": "smartwatch-x1",
    "ingestion_source": "api"
  }'
# Expected: HTTP 202 + {"status":"accepted","event_id":"<uuid>"}
```

```bash
# Invalid out-of-range value — should be rejected
curl -X POST http://localhost:8081/telemetry/body-temperature \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"value": 60.0, "unit": "C", "timestamp": "2025-08-25T09:00:00Z", "device_source": "x1", "ingestion_source": "api"}'
# Expected: HTTP 422 + field error for "value"
```

```bash
# Unauthenticated request
curl -X POST http://localhost:8081/telemetry/body-temperature \
  -H "Content-Type: application/json" \
  -d '{"value": 37.1, "unit": "C", "timestamp": "2025-08-25T09:00:00Z", "device_source": "x1", "ingestion_source": "api"}'
# Expected: HTTP 401
```

---

## Step 2 — Verify Kafka Event

```bash
# Using kafka-console-consumer (inside Kafka container)
docker exec -it kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic health-telemetry \
  --from-beginning \
  --max-messages 1
# Expected: Avro-encoded event with metric_type=BODY_TEMPERATURE
```

---

## Step 3 — Verify Storage (TimescaleDB)

```sql
-- Connect to PostgreSQL and verify the record landed
SELECT user_id, value_celsius, original_unit, device_source, recorded_at
FROM health_telemetry_body_temperature
WHERE recorded_at > NOW() - INTERVAL '5 minutes'
ORDER BY recorded_at DESC
LIMIT 5;
-- Expected: row with value_celsius=37.1, original_unit='C'
```

---

## Step 4 — Verify Charting API (sapphire-charting-api)

```bash
# Seed 7 days of data first (use k6-bootstrap with temperature profile)
# Then query the chart endpoint
curl "http://localhost:8082/users/$USER_ID/body-temperature/chart?range=week&unit=C" \
  -H "Authorization: Bearer $JWT"
# Expected: 200 + dataPoints array with 7 daily aggregates (min/max/avg)
```

```bash
# Unit conversion — Fahrenheit
curl "http://localhost:8082/users/$USER_ID/body-temperature/chart?range=day&unit=F" \
  -H "Authorization: Bearer $JWT"
# Expected: values converted to Fahrenheit (37.1°C → 98.78°F)
```

---

## Step 5 — Verify BFF GraphQL (sapphire-bff-api)

```graphql
# Apollo Sandbox or curl — POST to http://localhost:4000/graphql
query {
  bodyTemperatureChart(input: {
    userId: "<USER_ID>"
    range: WEEK
    unit: C
  }) {
    userId
    range
    unit
    dataPoints {
      windowStart
      windowEnd
      minValue
      maxValue
      avgValue
      recordCount
    }
  }
}
```
Expected: `dataPoints` array with 7 entries matching the charting API output.

---

## Step 6 — Verify UI (Sapphire React App)

1. Open `http://localhost:5173` and log in with the test user
2. Navigate to **Health Dashboard** → **Body Temperature**
3. Verify:
   - Chart renders with week range as default
   - Loading skeleton appears while data fetches
   - Day / Week / Month range selector works
   - Unit toggle (°C / °F) updates all values client-side (no new network request)
   - Empty state message appears for a user with no temperature data

---

## Step 7 — Rate Limit Validation

```bash
# Send 1001 readings in quick succession — 1001st should return 429
for i in $(seq 1 1001); do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST http://localhost:8081/telemetry/body-temperature \
    -H "Authorization: Bearer $JWT" \
    -H "Content-Type: application/json" \
    -d "{\"value\":37.0,\"unit\":\"C\",\"timestamp\":\"2025-08-25T09:00:0${i}Z\",\"device_source\":\"x1\",\"ingestion_source\":\"api\"}"
done | sort | uniq -c
# Expected: 1000× "202", 1× "429"
```

---

## Step 8 — Observability Validation

```bash
# Verify OTEL metric counter in Prometheus/Grafana
curl http://localhost:9090/api/v1/query \
  --data-urlencode 'query=temperature_readings_ingested_total{status="accepted"}'
# Expected: counter value ≥ 1000

# Verify structured logs contain trace_id and span_id
docker logs sapphire-event-ingestion-api 2>&1 | grep body_temperature | head -5
# Expected: JSON log lines with "trace_id" and "span_id" fields
```

---

## Validation Checklist

- [ ] Single reading accepted (HTTP 202) and Kafka event produced
- [ ] Invalid value rejected (HTTP 422) with field-level error
- [ ] Unauthenticated request rejected (HTTP 401)
- [ ] Record persisted in `health_telemetry_body_temperature` within 5s
- [ ] Charting API returns correct aggregates for day/week/month
- [ ] Unit conversion (C↔F) works correctly in chart API response
- [ ] BFF GraphQL query returns data matching charting API
- [ ] UI chart renders with loading/error/empty states
- [ ] Rate limit enforced (HTTP 429 + Retry-After after 1,000/min)
- [ ] `temperature_readings_ingested_total` counter visible in OTEL metrics
- [ ] Structured logs contain trace_id and span_id on all new paths
