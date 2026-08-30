# Feature Specification: Add Support for Body Temperature Metric Ingestion, Storage, and Reporting

**Feature Branch**: `SDDSDLC-170`
**Created**: 2025-08-25
**Status**: Draft
**Jira Story**: [SDDSDLC-170](https://jsw.ibm.com/browse/SDDSDLC-170)

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Ingest Body Temperature from Smart Device (Priority: P1)

A user with a compatible smart device or wearable that measures body temperature submits a
temperature reading to the Sapphire platform. The system accepts the reading, validates that
the value falls within a physiologically acceptable range, associates it with the user's
account, and publishes it to the event pipeline for downstream storage and analytics. The
user can submit both Celsius and Fahrenheit readings, individually or in batches.

**Why this priority**: Without ingestion, no downstream capability (storage, charts, UI) can
function. This story is the foundational entry point for the entire feature. Delivering it
first enables end-to-end testing of the pipeline even before the UI is built.

**Independent Test**: Submit a valid temperature reading (e.g. `{"value": 37.1, "unit": "C",
"device_source": "smartwatch-x1", "timestamp": "2025-08-25T09:00:00Z"}`) via the ingestion
REST endpoint. Verify the event appears in the Kafka topic with the correct Avro schema
fields. Submit an invalid value (e.g. `{"value": 200, "unit": "C"}`) and verify a 422
rejection with an error message referencing the range violation.

**Acceptance Scenarios**:

1. **Given** a registered user has a connected smart device, **When** the device submits a
   single temperature reading in Celsius within the valid physiological range (configurable,
   default 30.0°C–45.0°C), **Then** the system accepts the reading with HTTP 202 and
   publishes an event to the health telemetry Kafka topic containing value, unit, timestamp,
   user ID, and device source.

2. **Given** a registered user submits a batch of temperature readings (up to 100 records),
   **When** all readings are within the valid range, **Then** the system accepts all records
   with HTTP 202 and publishes each as an individual event to the Kafka topic.

3. **Given** a registered user submits a temperature reading in Fahrenheit, **When** the
   value is within the valid Fahrenheit range (configurable, default 86.0°F–113.0°F),
   **Then** the system accepts the reading, converts it to Celsius for internal storage, and
   publishes the event with both the original unit and the converted Celsius value recorded.

4. **Given** a smart device submits a temperature reading with a value outside the configured
   physiological range, **When** the ingestion endpoint receives the request, **Then** the
   system rejects it with HTTP 422 and returns an error message identifying the field,
   the invalid value, and the acceptable range.

5. **Given** the ingestion endpoint receives a batch containing a mix of valid and invalid
   temperature readings, **When** the batch is processed, **Then** valid records are published
   to the Kafka topic and the response returns HTTP 207 Multi-Status with a per-record result
   list — each entry indicating success or the specific validation error for that record.

6. **Given** the ingestion endpoint is called without a valid Keycloak JWT, **When** the
   request arrives, **Then** the system returns HTTP 401 and does not publish any event.

7. **Given** the ingestion endpoint receives a request with a missing required field
   (value or timestamp), **When** validation runs, **Then** the system returns HTTP 422
   with a field-level error message.

---

### User Story 2 - View Body Temperature Trend Charts (Priority: P2)

A user opens their Sapphire health dashboard and navigates to the body temperature section.
They can view a chart displaying their temperature history with selectable time ranges (day,
week, month). The chart shows minimum, maximum, and average values per aggregation window.
The user can switch between Celsius and Fahrenheit display. The chart behaves consistently
with existing health metric charts (loading state, error state, empty state).

**Why this priority**: The UI is the primary surface users interact with. Once ingestion
(P1) is working and data is in storage, the chart view delivers the user-visible value of
the feature. It requires the charting API and BFF to be functional first.

**Independent Test**: With at least 7 days of pre-seeded temperature data for a test user
(using the k6-bootstrap tool), open the temperature chart component in the UI. Verify it
renders correctly for day/week/month ranges, displays min/max/average values, and switches
units on demand. Verify loading skeleton appears during data fetch and an empty-state message
appears when no data exists.

**Acceptance Scenarios**:

1. **Given** a logged-in user has temperature records spanning the last 30 days, **When**
   they navigate to the temperature metric view and select the "week" range, **Then** the
   chart displays aggregated daily data points (min, max, average) for each of the last 7
   days with clearly labelled axes and units.

2. **Given** a logged-in user views the temperature chart, **When** the data is loading,
   **Then** a loading skeleton is displayed in place of the chart until data arrives.

3. **Given** a logged-in user views the temperature chart and no temperature records exist
   for the selected time range, **When** the data request completes, **Then** an empty-state
   message is displayed (e.g. "No temperature data for this period") with guidance to
   connect a device.

4. **Given** a logged-in user views the temperature chart in Celsius, **When** they switch
   the unit toggle to Fahrenheit, **Then** all chart values and axis labels update to
   Fahrenheit without a full page reload, and the selection persists for the session.

5. **Given** the charting API is unavailable, **When** the UI attempts to fetch temperature
   data, **Then** an error boundary message is displayed (e.g. "Could not load temperature
   data. Please try again.") and no broken UI state is shown.

6. **Given** a logged-in user selects the "day" time range, **When** the chart renders,
   **Then** hourly aggregated data points (or raw readings if fewer than 24 exist) are
   displayed for the current day.

7. **Given** a logged-in user selects the "month" time range, **When** the chart renders,
   **Then** weekly aggregated data points are displayed for the last 30 days.

---

### User Story 3 - Temperature Data in Analytics Export and Partner Sharing (Priority: P3)

A user can access their own temperature data through the analytics export endpoints using
their own Keycloak credentials. Healthcare provider access is out of scope for this story —
providers may receive exported data only if the user manually shares it. Temperature metrics
appear alongside other health metrics in the user's analytics summary. Schema documentation
for the temperature metric is available to integration partners.

**Why this priority**: Export and partner sharing extend the reach of the feature but are
not required for the core health-tracking use case. This story delivers the remaining
acceptance criterion (schema shared with integration partners) and completes the full scope
of the Jira story.

**Independent Test**: Call the analytics export endpoint for a user with seeded temperature
data and verify the response payload includes temperature records. Confirm the API schema
documentation (OpenAPI spec) has been updated to include the temperature metric type with
all fields defined.

**Acceptance Scenarios**:

1. **Given** a user with temperature records calls the analytics export endpoint, **When**
   the response is returned, **Then** it includes temperature measurements with value, unit,
   timestamp, device source, and measurement method fields.

2. **Given** the temperature metric has been deployed, **When** an integration partner
   accesses the published API schema documentation, **Then** the temperature metric type is
   documented with all field definitions, valid value ranges, and supported units.

3. **Given** a user views their overall health analytics dashboard, **When** the temperature
   metric data exists, **Then** temperature appears in the metrics list alongside blood
   pressure, SpO2, and activity data with the same visual treatment.

4. **Given** the analytics export endpoint is called without a valid Keycloak JWT, **When**
   the request arrives, **Then** the system returns HTTP 401 and returns no data.

5. **Given** a user attempts to access another user's temperature export data using their
   own JWT, **When** the request is processed, **Then** the system returns HTTP 403 and
   returns no data (user-scoped access enforced).

---

### Edge Cases

- What happens when a device submits a temperature reading with a timestamp in the future
  (clock drift)? The system should accept readings up to 5 minutes in the future and reject
  readings more than 5 minutes ahead with HTTP 422.
- What happens when the same temperature reading (identical user, device, timestamp, value)
  is submitted twice? The system should be idempotent: the second submission should be
  accepted (HTTP 202) but not create a duplicate record.
- What happens when a user has temperature readings in both Celsius and Fahrenheit in storage?
  The chart must display all values in a single consistent unit as selected by the user.
- What happens when the Kafka topic is unavailable during ingestion? The API must return
  HTTP 503 and not silently swallow the error.
- What happens when a user has zero temperature readings and requests the month range? The
  empty-state UI must render without JavaScript errors.
- What happens when a batch contains 0 records? The API must return HTTP 400.
- What happens when a batch exceeds the maximum record count (100)? The API must return
  HTTP 422 with a clear error message.
- What happens when a user exceeds the ingestion rate limit? The system must return HTTP 429
  with a `Retry-After` header indicating when the client may retry.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST accept body temperature readings via the health telemetry
  ingestion endpoint, supporting both single-record and batch (up to 100 records) input.
  For batches containing a mix of valid and invalid records, the system MUST use partial-accept
  semantics: valid records are published to Kafka and the response MUST be HTTP 207 Multi-Status
  with a per-record result array identifying successes and field-level errors for invalid records.
- **FR-002**: The system MUST support both Celsius and Fahrenheit as input units for
  temperature readings.
- **FR-003a**: The ingestion endpoint MUST enforce a per-user rate limit of 1,000 temperature
  readings per minute. Requests exceeding this limit MUST be rejected with HTTP 429 and a
  `Retry-After` response header specifying the number of seconds until the limit resets.
- **FR-003**: The system MUST validate temperature values against a configurable
  physiological range (default: 30.0°C–45.0°C / 86.0°F–113.0°F) and MUST reject out-of-range
  values with HTTP 422 and a field-level error message identifying the value and the
  acceptable range.
- **FR-004**: Each temperature measurement MUST be associated with a user ID, a timestamp,
  and a device source identifier.
- **FR-005**: The system MUST publish accepted temperature events to the health telemetry
  Kafka topic using the existing Avro schema structure extended with a `body_temperature`
  metric type.
- **FR-006**: The system MUST store temperature records in the time-series metrics datastore
  with indexing that supports efficient time-range queries (day, week, month).
- **FR-007**: The system MUST support daily, weekly, and monthly rollup aggregations for
  temperature data (min, max, average per window).
- **FR-008**: The charting API MUST expose temperature trend data endpoints returning
  aggregated min, max, and average values for a requested time range, filterable by date
  range and device source.
- **FR-009**: The BFF GraphQL layer MUST expose a query for temperature trend data consumable
  by the frontend.
- **FR-010**: The frontend MUST display a body temperature chart component within the health
  dashboard, with selectable time ranges (day, week, month) and a unit toggle
  (Celsius / Fahrenheit).
- **FR-011**: The temperature chart component MUST display loading skeleton, error boundary,
  and empty-state views in line with existing health metric components.
- **FR-012**: The temperature metric MUST appear in the user's health metrics list alongside
  existing metrics.
- **FR-013**: Temperature data MUST be included in analytics export endpoints.
- **FR-014**: The OpenAPI / GraphQL schema documentation MUST be updated to reflect the
  temperature metric type, all fields, valid ranges, and supported units before the feature
  ships.
- **FR-015**: All ingestion endpoints MUST require a valid Keycloak JWT; unauthenticated
  requests MUST be rejected with HTTP 401.
- **FR-016**: All new temperature-related service paths (ingestion, charting, export) MUST
  be instrumented with OTEL traces (W3C `traceparent` propagation) and emit structured JSON
  logs containing `trace_id` and `span_id` on every log line.
- **FR-017**: The ingestion service MUST emit a `temperature_readings_ingested_total` counter
  metric with labels `unit` (`C` or `F`) and `status` (`accepted` or `rejected`), exported
  to the OTEL Collector.

### Key Entities

- **TemperatureReading**: A single measurement event containing `user_id`, `value` (decimal),
  `unit` (enum: `C` | `F`), `timestamp` (ISO 8601 UTC), `device_source` (string),
  `ingestion_source` (string), `measurement_method` (optional string). The canonical storage
  unit is Celsius; Fahrenheit readings are converted on ingest.
- **TemperatureTrendPoint**: An aggregated data point for charting containing `window_start`,
  `window_end`, `min_value`, `max_value`, `avg_value`, `unit`, `record_count`.
- **MetricType**: An enumeration extended to include `body_temperature` alongside existing
  types (`blood_pressure`, `spo2`, `activity`).

## Clarifications

### Session 2025-08-25

- Q: For a batch containing mixed valid/invalid temperature readings, which behaviour should the system implement? → A: Partial accept — valid records published to Kafka, invalid records returned with per-record errors, HTTP 207 Multi-Status.
- Q: Should the ingestion endpoint enforce rate limiting? → A: Yes — per-user limit of 1,000 readings/minute; excess requests return HTTP 429 with a `Retry-After` header.
- Q: What is the synchronous HTTP response latency target for the ingestion endpoint? → A: 500ms p95 under normal load.
- Q: Can healthcare providers access the analytics export endpoint with their own credentials? → A: No — export endpoint is user-scoped (own Keycloak JWT only); healthcare provider delegation is out of scope for this story.
- Q: What observability is required for the new temperature metric paths? → A: Standard OTEL traces + structured logs on all new paths, plus a `temperature_readings_ingested_total` counter metric (labels: unit, status: accepted|rejected).

## Assumptions

- The existing health telemetry ingestion API (`sapphire-event-ingestion-api`) already has an
  Avro schema registry integration; adding `body_temperature` as a new metric type follows
  the same pattern as existing metrics without requiring a new topic.
- The existing time-series PostgreSQL schema (via `sapphire-kafka-pipeline`) uses a metric
  type discriminator column or table; temperature will use the same pattern.
- The configurable physiological range (30.0–45.0°C) will be stored as environment variables
  or application configuration, not hardcoded.
- Unit conversion (Fahrenheit to Celsius) is performed at ingest time; only Celsius values
  are stored internally. The display unit preference is a UI-side concern resolved at render
  time from the stored Celsius value.
- The charting API already has a time-range aggregation pattern for existing metrics;
  temperature aggregation will follow the same pattern with no new aggregation framework.
- The BFF GraphQL schema already has a health metrics query structure; temperature is added
  as a new field/type within that structure.
- The frontend Sapphire React app already has a reusable chart component or pattern used for
  existing health metrics; the temperature chart component will reuse that pattern.
- "Integration partners" in the acceptance criteria refers to the API schema documentation
  (OpenAPI spec) being updated and published — not a real-time notification to external
  parties, which is out of scope for this story.
- Batch idempotency is enforced by a unique constraint on (user_id, device_source, timestamp)
  in the time-series store.
- The maximum batch size of 100 records is a reasonable default; this value should be
  configurable via environment variable.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A valid single temperature reading submitted to the ingestion endpoint is
  accepted (HTTP 202) and appears as a queryable record in the time-series store within 5
  seconds of submission under normal load conditions. The ingestion endpoint MUST return its
  HTTP response (202 or error) within 500ms at the 95th percentile under normal load.
- **SC-002**: A temperature reading with a value outside the configured physiological range
  is rejected with HTTP 422 and a field-level error message in 100% of test cases.
- **SC-003**: The temperature chart in the UI renders correctly (no JavaScript errors, no
  broken layout) for users with data and for users with zero data across all three time
  ranges (day, week, month).
- **SC-004**: The temperature chart displays the correct aggregated min, max, and average
  values matching the data in the time-series store for a given time range, verified in 100%
  of test cases with pre-seeded data.
- **SC-005**: Unit switching (Celsius ↔ Fahrenheit) on the chart updates all displayed
  values and labels correctly in under 200ms (client-side conversion, no new network
  request required).
- **SC-006**: The OpenAPI specification for the ingestion API and the GraphQL schema for the
  BFF both include the `body_temperature` metric type with all fields documented before the
  feature is merged.
- **SC-007**: Unauthenticated requests to all temperature-related endpoints return HTTP 401
  in 100% of test cases.
- **SC-008**: A batch of 100 valid temperature records is accepted and all 100 events appear
  in the Kafka topic within 10 seconds of submission under normal load conditions.
- **SC-009**: The ingestion endpoint test coverage for the temperature metric path meets the
  project standard of 80% for Python services.
- **SC-010**: The charting API test coverage for temperature endpoints meets the project
  standard of 80% overall with 100% on domain-layer logic.
