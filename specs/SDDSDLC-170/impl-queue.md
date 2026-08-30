# Implementation Queue — SDDSDLC-170

Generated: 2025-08-25

> Each entry is one `speckit.implement` invocation. Entries are processed in order.
> Tick `[x]` only when the corresponding invocation produces a `## Phase Complete` report.

## Queue

- [ ] 1 · sapphire-event-ingestion-api / Phase 1: Setup (Shared Infrastructure)
- [ ] 2 · sapphire-kafka-pipeline / Phase 1: Setup (Shared Infrastructure)
- [ ] 3 · sapphire-charting-api / Phase 1: Setup (Shared Infrastructure)
- [ ] 4 · sapphire-bff-api / Phase 1: Setup (Shared Infrastructure)
- [ ] 5 · Sapphire / Phase 1: Setup (Shared Infrastructure)
- [ ] 6 · sapphire-event-ingestion-api / Phase 2: Foundational (Blocking Prerequisites)
- [ ] 7 · sapphire-kafka-pipeline / Phase 2: Foundational (Blocking Prerequisites)
- [ ] 8 · sapphire-event-ingestion-api / Phase 3: User Story 1 — Ingest Body Temperature from Smart Device (Priority: P1) 🎯 MVP
- [ ] 9 · sapphire-charting-api / Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2)
- [ ] 10 · sapphire-bff-api / Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2)
- [ ] 11 · Sapphire / Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2)
- [ ] 12 · sapphire-charting-api / Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3)
- [ ] 13 · sapphire-event-ingestion-api / Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3)
- [ ] 14 · sapphire-bff-api / Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3)
- [ ] 15 · sapphire-event-ingestion-api / Phase 6: Polish & Cross-Cutting Concerns
- [ ] 16 · sapphire-charting-api / Phase 6: Polish & Cross-Cutting Concerns
- [ ] 17 · sapphire-bff-api / Phase 6: Polish & Cross-Cutting Concerns
- [ ] 18 · Sapphire / Phase 6: Polish & Cross-Cutting Concerns
- [ ] 19 · sapphire-k6-bootstrap / Phase 6: Polish & Cross-Cutting Concerns

## Invocation Template

For each entry above, invoke:

```
/speckit.implement STORY_ID=SDDSDLC-170 REPO=<repo-name> PHASE=<exact phase label>
```

Phase labels must match the phase headers in `specs/SDDSDLC-170/tasks.md` exactly.

## Dependency Notes

- Entries 1–5 (Phase 1 branch setup) are fully parallel — all 5 can run simultaneously.
- Entries 6–7 (Phase 2 Foundational) must complete before Entry 8 (Phase 3 US1 ingestion).
- Entry 8 (Phase 3 US1) must complete before Entries 9–11 (Phase 4 US2 charts) — ingestion must work to seed test data.
- Entries 9–11 (Phase 4 charting API / BFF / UI) can run in parallel with each other.
- Entries 12–14 (Phase 5 US3 export) depend on Entry 9 (charting API data access established).
- Entries 15–19 (Phase 6 polish) depend on all user-story phases complete.

## Invocations by Entry

| # | Repo | Phase |
|---|------|-------|
| 1 | `sapphire-event-ingestion-api` | Phase 1: Setup (Shared Infrastructure) |
| 2 | `sapphire-kafka-pipeline` | Phase 1: Setup (Shared Infrastructure) |
| 3 | `sapphire-charting-api` | Phase 1: Setup (Shared Infrastructure) |
| 4 | `sapphire-bff-api` | Phase 1: Setup (Shared Infrastructure) |
| 5 | `Sapphire` | Phase 1: Setup (Shared Infrastructure) |
| 6 | `sapphire-event-ingestion-api` | Phase 2: Foundational (Blocking Prerequisites) |
| 7 | `sapphire-kafka-pipeline` | Phase 2: Foundational (Blocking Prerequisites) |
| 8 | `sapphire-event-ingestion-api` | Phase 3: User Story 1 — Ingest Body Temperature from Smart Device (Priority: P1) 🎯 MVP |
| 9 | `sapphire-charting-api` | Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2) |
| 10 | `sapphire-bff-api` | Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2) |
| 11 | `Sapphire` | Phase 4: User Story 2 — View Body Temperature Trend Charts (Priority: P2) |
| 12 | `sapphire-charting-api` | Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3) |
| 13 | `sapphire-event-ingestion-api` | Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3) |
| 14 | `sapphire-bff-api` | Phase 5: User Story 3 — Analytics Export and Partner Sharing (Priority: P3) |
| 15 | `sapphire-event-ingestion-api` | Phase 6: Polish & Cross-Cutting Concerns |
| 16 | `sapphire-charting-api` | Phase 6: Polish & Cross-Cutting Concerns |
| 17 | `sapphire-bff-api` | Phase 6: Polish & Cross-Cutting Concerns |
| 18 | `Sapphire` | Phase 6: Polish & Cross-Cutting Concerns |
| 19 | `sapphire-k6-bootstrap` | Phase 6: Polish & Cross-Cutting Concerns |
