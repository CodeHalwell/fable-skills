---
name: data-quality-and-governance
description: Load when designing data quality checks, data contracts, lineage, or governance for data platforms — choosing validation tooling, setting freshness/completeness alerts, debugging bad data incidents and backfills, defining ownership between producers and consumers, or reviewing a pipeline for trustworthiness.
---

# Data Quality and Governance Engineering

Frontier models already argue shift-left enforcement, invariants-over-profiles, block-vs-warn classification, OpenLineage, and producer ownership cold. This sheet keeps the placement algorithm, the calibrations they miss, and the operational pitfalls that only show up in practice.

## Anchors

1. Bad data is a producer change hitting an unwritten consumer assumption — contracts at the boundary fix the class; cleaning downstream treats symptoms forever. Cost grows with every hop.
2. Checks encode *expectations*, not descriptions of current data. Auto-profiled ranges alert on change, not wrongness; profiling proposes, a human justifies.
3. Contract = **schema + semantics + SLA**, producer-owned, machine-checked. Most incidents that "passed all checks" are semantic drift — an enum gaining a value, gross→net, daily→hourly grain. Encode units in column names, grain as a `count(*) = count(distinct key)` check.
4. Invariants **block**; statistical/anomaly checks **warn** — a blocking anomaly check is disabled by the third false 3 a.m. page, then catches nothing. Audit dbt `severity:` config; it drifts.
5. Governance is a product with users: ownership + 4-level classification + role grants + PII retention. Measure it by whether velocity survived; a freeze teaches the org to smuggle data through spreadsheets — govern the paved road instead.

## Where a new check belongs (the placement algorithm)

1. Producer-enforceable invariant → producer CI / schema registry / dbt contract at the source. Everything downstream inherits it.
2. One table, in-DAG → dbt test at the *earliest* model where the assertion is meaningful.
3. Python/service boundary (ingestion, features, training inputs) → Pandera-class validation in the job, blocking.
4. Cross-system reconciliation or stakeholder-facing docs → GX/Soda-class suite on a schedule.
5. Statistical → anomaly monitor, warn-only, owner-routed.

Highest-value refactor of an existing suite is usually *moving* checks upstream, not adding more. Uncooperative producer? A consumer-side contract test is a detection layer *and* the evidence for the negotiation.

## Tooling anchors (verified 2026)

dbt model contracts (`contract: {enforced: true}`, Core ≥1.5) enforce schema within the DAG. Across team/system boundaries: **ODCS (Open Data Contract Standard, Linux Foundation Bitol, v3.x)** is the de facto YAML standard, with `datacontract-cli` to lint and test actual datasets in CI. Path: dbt contracts on marts first; ODCS when the counterparty is outside your dbt project. Lineage: **OpenLineage** (Airflow/Spark/dbt/Flink/GX emit; Marquez or Datahub/Atlan/Collibra consume); dbt column-level lineage derives from `catalog.json`. Two queries lineage must answer or it isn't earning its keep: impact analysis before every contracted change, and root-cause walking upstream to the first bad node. Column-level matters because table-level over-alarms — a table feeding 40 dashboards usually has any given *column* feeding 3.

## Calibrations cold models get wrong

- **The most-neglected dimension is freshness, and staleness is the most common real incident.** The reflex answer is "cross-system consistency is under-invested" — true but second-order; in practice teams ship ten uniqueness tests and no freshness check, and freshness must be measured as data recency (`max(event_time)`) plus non-empty volume, never "the job succeeded." Page at SLA breach, warn at 80% of budget.
- **Alert-precision floor: if under ~50% of anomaly alerts are true incidents, tune before adding any coverage** — the team has already started ignoring them. Seasonality corrections in effort order: same-weekday-last-4-weeks baseline → holiday calendar widening bands → STL/Prophet-class decomposition only if simpler baselines still page falsely.
- **Magnitude-mismatch triage:** an 8% row-count dip cannot explain a 40% revenue drop — a size mismatch between volume anomaly and metric anomaly says *value* problem (bad join, NULLed rates, unit flip), not *volume* problem. Check `git log` before suspecting SQL unchanged for months.

## Incident response (order matters)

1. Stop the spread — pause downstream schedules / quarantine the partition *before* diagnosing.
2. Scope with lineage; notify consumers *now* with blast radius ("sales dash wrong Mar 3–5"), not after the fix.
3. Fix at source, backfill forward in dependency order. Incremental/append models need explicit partition deletion first — appending a "corrected" load onto a bad one is a second incident. Non-reprocessable aggregates → document the permanent scar.
4. Invalidate downstream: caches, and **version-bump ML features trained on the bad window** — a model trained on poisoned features doesn't heal when the table does.
5. Postmortem output = a check that would have caught it, placed as far upstream as possible. Stopping rule for the incident: recomputed numbers reconcile against an *external* source of truth, not "looks plausible."

Late-arriving data is the chronic version: watermark columns, trailing-N-day rebuild windows, and metrics annotated "complete through X" — if consumers don't know the completeness horizon, every late batch is a mini-incident.

## Ownership

Producer owns quality + contract (they can fix root causes); central platform team owns standards/tooling (contract format, lineage backbone, classification scheme); consumers own fitness checks for their use. Every critical table has an owner field that *pages someone*. A central team owning quality for data it doesn't produce becomes a permanent, scapegoated bottleneck.

## Operational pitfalls (the ones that survive contact with production)

- Uniqueness tested at staging but not where BI reads — dedup drift double-counts at consumption.
- **Quarantine tables nobody drains**: months of rejects means either false rejects nobody noticed or real data silently missing from marts. Quarantine gets an SLA and an owner like any queue; reject volume is itself monitored.
- Lineage built once decays in months, and stale lineage is worse than none because people trust it — emit automatically from orchestrator/dbt runs, never hand-drawn.
- PII discovered by grep during an audit → classify at ingestion as part of the contract; scanners are a backstop.
- `lazy=True`-style full-failure reports (Pandera) and reject-to-quarantine-with-reason are the difference between one fix cycle and five; rejects are the evidence for the producer conversation.

## Worked micro-example: contract enforced in dbt

```yaml
# models/marts/fct_orders.yml (dbt Core >= 1.5)
models:
  - name: fct_orders
    config:
      contract: {enforced: true}          # build fails on schema drift
    columns:
      - name: order_id
        data_type: string
        constraints: [{type: not_null}]
        data_tests: [unique]
      - name: order_total_usd             # name encodes unit — semantic guard
        data_type: numeric
        data_tests:
          - dbt_utils.accepted_range: {min_value: 0}
      - name: status
        data_type: string
        data_tests:
          - accepted_values: {values: ['placed','paid','shipped','refunded']}
      - name: customer_id
        data_type: string
        data_tests:
          - relationships: {to: ref('dim_customers'), field: customer_id}
```

Cross-team version: the same agreement as ODCS v3 YAML (schema + SLA blocks), `datacontract test` against the producer's actual table in their CI.

## Verification / self-check

1. Per critical table: one *named* check per dimension (freshness, completeness, uniqueness, validity) — "we have dbt tests" is not an answer.
2. Simulate a producer schema change in a branch — does anything block before consumers break?
3. Lineage answers "what breaks if I drop column X?" in minutes at column granularity.
4. Paper incident drill: 2 a.m. bad partition — who's paged, what's quarantined, what's the backfill command, who tells consumers?
5. Alert precision last month > ~50%, or tune before expanding.
6. One core metric reconciled against an external source of truth.

Stopping rule: critical tables (money, ML, executives) get contracts + four-dimension checks + owners; the long tail gets freshness + volume only. Full coverage of everything is where data-quality programs go to die.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 9 baseline (cut/compressed), 3 partial (sharpened), 0 delta.
- Opus cold nails: shift-left argument, auto-profile critique, ODCS/Bitol + datacontract-cli + dbt contracts, block-vs-warn rule, OpenLineage + column-level benefits, incident order-of-operations, producer/platform/consumer ownership, freshness ≠ job success.
- Sharpened: freshness as the most-neglected/most-incident dimension (Opus points at cross-system consistency instead); the ~50% alert-precision tuning floor; magnitude-mismatch (volume vs value) triage; practice-only pitfalls kept (undrained quarantines, lineage decay, governance-as-freeze).
