---
name: data-quality-and-governance
description: Load when designing data quality checks, data contracts, lineage, or governance for data platforms — choosing validation tooling, setting freshness/completeness alerts, debugging bad data incidents and backfills, defining ownership between producers and consumers, or reviewing a pipeline for trustworthiness.
---

# Data Quality and Governance Engineering

## Core mental model

- **Data quality is an interface problem, not a cleaning problem.** Bad data is almost always a producer change (schema, semantics, timing) hitting a consumer assumption nobody wrote down. Cleaning downstream treats symptoms forever; contracts at the boundary fix the class of bug.
- **Enforcement at write time beats detection at read time.** A rejected write is a producer's pager and a clear diff; a detected anomaly three hops downstream is a day of archaeology plus every intermediate table now suspect. Push checks as far upstream as politically possible — the cost of a data bug grows with every transformation it passes through.
- **Checks are assertions about *expectations*, not descriptions of current data.** Auto-profiled checks ("row count between 4.2M and 4.9M because that's what we saw") alert on change, not on wrongness. Write checks that encode business meaning: "order_total ≥ 0", "every order has a customer that exists", "event_time ≤ ingestion_time".
- **Lineage is the debugging map and the blast-radius calculator.** Without column-level lineage, "can we drop/change this column?" and "what's affected by this bad load?" are guesswork. With it, both are queries.
- **Governance is a product with users, not a compliance document.** The minimum viable version — ownership, classification, access rules that engineers can follow without a ticket — beats the comprehensive framework nobody follows. Measure governance by whether velocity survived it.

## Data contracts as the interface

A contract = **schema + semantics + SLA**, versioned, machine-checkable, owned by the *producer*:
- Schema: columns, types, nullability, enums, uniqueness keys.
- Semantics: units, timezone, grain ("one row per order-item per day"), null meaning, enum definitions. Most production incidents that "passed all checks" are semantic drifts — a status enum gaining a value, an amount switching from gross to net.
- SLA: freshness deadline, completeness expectation, notification-before-change window, deprecation policy.

Tooling map (verified as of 2026): **dbt model contracts** (Core ≥1.5: `contract: {enforced: true}` + column data types and constraints) enforce schema *within* the transformation DAG at build time. For agreements crossing team/system boundaries, the **Open Data Contract Standard (ODCS, Linux Foundation Bitol project, v3.x)** is the de facto YAML standard, with `datacontract-cli` to lint contracts and test actual datasets against them in CI. Practical path: dbt contracts on your marts first; ODCS when the counterparty is outside your dbt project (source system teams, ML consumers, external partners).

Reasoning chain for introducing contracts: *Who breaks whom today?* Find the top 3 incident-causing interfaces from the last quarter — contract those first, not everything. *Can the producer enforce it?* If the producer can run schema checks in their CI/deploy (e.g., protobuf/Avro schema registry with compatibility mode for streams, dbt contract for models), enforce at write. If the producer won't cooperate yet, a consumer-side contract test is a detection layer *and* the evidence you bring to the negotiation.

## Validation layering — where each tool fits

Three layers; the mistake is using one tool for all three:

1. **In-warehouse, in-DAG (dbt tests / dbt model contracts):** cheap declarative checks (`not_null`, `unique`, `accepted_values`, `relationships`) running as part of the build, blocking downstream models on failure. This is the workhorse — 80% of the value. Add `dbt-utils`/`elementary` for expression and anomaly tests.
2. **In-flight, in-Python (Pandera-class):** dataframe schema validation (Pandera supports pandas and Polars) at pipeline and service boundaries — API ingestion, feature engineering, ML training inputs. Types + ranges + custom checks as code next to the code, failing the job before bad data lands.
3. **Rich/standalone validation (Great Expectations GX Core / Soda-class):** expectation suites with data docs, profiling, and cross-system checks — strongest when validation must be shared with less-technical stakeholders or run against systems outside your DAG. GX remains the most established Python framework as of 2026; its cost is framework weight — don't adopt it to do what a dbt `not_null` test does.

Blocking vs. warning: checks on *contracted invariants* block (stop the DAG, quarantine the batch); *statistical/anomaly* checks warn (alert, don't block) because they have a false-positive rate. A blocking anomaly check will be disabled by the third false 3 a.m. page — then it catches nothing.

## Quality dimensions, operationalized

For each critical table, cover four dimensions with *concrete* checks and thresholds — a table with ten uniqueness tests and no freshness check is the norm and it's backwards (staleness is the most common real incident):

| Dimension | Concrete check | Alert design |
|---|---|---|
| **Freshness** | `max(loaded_at)` vs. now against the SLA (e.g., "by 06:00, data through midnight"); dbt source freshness | Page at SLA breach; warn at 80% of budget |
| **Completeness** | Row count vs. same-weekday baseline; null-rate per critical column vs. threshold; reconciliation count vs. source system | Warn on deviation > seasonal band; page on reconciliation mismatch |
| **Uniqueness** | Duplicate count on declared keys = 0 (post-dedup); duplicate *rate* upstream trended | Block build on key violation in marts |
| **Validity** | Type/enum/range conformance; referential integrity (`relationships` tests); cross-field rules ("ship_date ≥ order_date") | Block on contracted rules; warn on new enum values |

## Lineage as the debugging map

- **OpenLineage** is the leading open standard (as of 2026) for lineage event collection — integrations with Airflow, Spark, dbt, Flink, and Great Expectations; events carry facets including schema, data-quality results, and **column-level lineage** where the emitter supports it (the dbt integration derives column lineage from `catalog.json` via `dbt docs generate`, available for the major warehouse adapters). dbt Cloud's Explorer also ships column-level lineage natively. Consume via Marquez (reference backend) or commercial catalogs (Datahub, Atlan, Collibra class) that ingest OpenLineage.
- Two queries lineage must answer or it isn't earning its keep: **impact analysis** ("if I change/drop this column, which models, dashboards, and ML features break?") — run *before* every contracted-schema change; and **root-cause tracing** ("this metric is wrong; walk upstream to the first node where the data went bad").
- Column-level matters because table-level lineage over-alarms: a table feeding 40 dashboards usually has any given *column* feeding 3.

## Reasoning chain: which layer does a new check belong in?

Ask in order:
1. *Is it an invariant the producer can enforce?* → producer's CI / schema registry / dbt contract at the source model. Done — everything downstream inherits it.
2. *Does it need only one table, inside the warehouse DAG?* → dbt test on the earliest model where the assertion is meaningful (staging for structure, marts for business rules that require joins).
3. *Does it guard a Python/service boundary (ingestion, feature pipeline, ML training input)?* → Pandera-class validation in the job itself, blocking.
4. *Does it compare across systems (source-vs-warehouse reconciliation) or need docs for stakeholders?* → GX/Soda-class suite on a schedule.
5. *Is it statistical rather than invariant?* → anomaly monitor, warning-only, owner-routed.
A check placed one layer too far downstream still fires — after the damage has propagated. When reviewing an existing suite, the highest-value refactor is usually *moving* checks upstream, not adding more.

## Anomaly detection on data — and the seasonality trap

- Monitor volume, null rates, distinct-count, and distribution (mean/quantiles per numeric column; category shares) per table per load. Tools: elementary/re_data in the dbt world, Monte Carlo/Bigeye class commercially, or hand-rolled z-scores off a metrics table — the model matters less than the baseline design.
- **The seasonality false-positive problem**: naive thresholds fire every Monday (weekend dip), every holiday, every marketing campaign. Corrections, in order of effort: compare same-weekday-last-4-weeks instead of yesterday; maintain a holiday calendar that widens bands; use STL/Prophet-style decomposition only if the simpler baselines still page falsely. Track alert precision — if under ~50% of anomaly alerts are true incidents, tune before adding coverage, because the team has already started ignoring them.
- Distribution checks catch what row counts miss: the load that arrives on time, right-sized, but with `country` 90% NULL because an upstream join broke.

## Incident response for data

Order of operations when bad data lands:
1. **Stop the spread**: pause downstream schedules / mark the partition quarantined before diagnosing. The costliest minutes are the ones where consumers keep reading poison.
2. **Scope with lineage**: which partitions, which downstream tables/dashboards/features consumed them. Notify consumers *now*, with the blast radius — "sales dash wrong for Mar 3–5" — not after the fix.
3. **Fix at the source**, then **backfill forward through the DAG in dependency order**. Backfill judgment: idempotent, partition-overwrite pipelines make this a re-run; incremental/append pipelines need explicit partition deletion first — appending a "corrected" load onto a bad one creates duplicates and a second incident. If aggregates are non-reprocessable (mutable sources, expired raw data), document the permanent scar.
4. **Downstream invalidation**: recompute derived tables, bust caches, version-bump ML features trained on the bad window (a model trained on poisoned features doesn't heal when the table does — retrain or mark).
5. Postmortem output = a *check* that would have caught it, placed as far upstream as possible.

**Late-arriving data** is the chronic version: design for it explicitly — watermark columns (`event_time` vs. `loaded_at`), reprocessing windows (rebuild the trailing N days each run), and metrics annotated as "complete through X". If consumers don't know the completeness horizon, every late batch is a mini-incident.

## Ownership and pragmatic governance

- Domain ownership with central standards (the defensible core of the data-mesh idea, minus the buzzword): the team that *produces* the data owns its quality and contract — they have the context to fix root causes; a central platform team owns the *standards* (contract format, check tooling, lineage backbone, classification scheme) so domains don't diverge into incompatible fiefdoms. A central data team owning quality for data it doesn't produce becomes a permanent, scapegoated bottleneck.
- Every table has an owner *field that pages someone*. Unowned critical tables are incidents-in-waiting.
- Minimum viable governance: (1) classification at four-ish levels (public / internal / confidential / restricted-PII) applied at the *dataset* level with column tags for PII; (2) access via group/role grants tied to classification, self-service for internal, approval only for restricted; (3) audit logging on restricted access; (4) retention rules on PII. Anything beyond this must justify itself against velocity — a governance process that turns a one-day analysis into a two-week ticket queue teaches the org to smuggle data through spreadsheets, which is *worse* governance.

## How an expert thinks through it: "revenue dashboard looks wrong"

Monday, revenue for Friday shows 40% down. Internal monologue: *First: real or data? Check the business — no known outage, checkout conversion normal in the product-analytics tool. So probably data. Freshness first (highest prior): did Friday's load complete?* Load status green, row count… 8% below same-Friday baseline — low-ish but within the band. *So not a missing load. Walk lineage upstream from the dashboard metric: dashboard ← mart_revenue ← fct_orders ← stg_payments ← payments source. Query each layer for Friday's total — mart and fct agree, stg is fine, but fct_orders' join to currency rates: Friday's rate table shows EUR rows missing.* Check the rates loader: the FX vendor changed a field name Thursday; the loader's lenient parser wrote NULL rates, and the join's COALESCE defaulted to… 0. *Rejected en route: blaming the 8% row dip (too small for a 40% revenue drop — magnitude mismatch is a strong clue it's a value problem, not a volume problem); re-running the dashboard extract (symptom-level); suspecting the mart SQL (unchanged in git for months — check `git log` before suspecting stable code).* Fix: patch the loader, quarantine + backfill rates for Thu–Mon, rebuild fct_orders and mart for those partitions in order, notify finance with the affected window. The postmortem check: NOT NULL + row-count-per-currency contract on the rates table — at the *loader*, not the mart, because that's the first place the wrongness existed. Stopping rule: recomputed Friday matches the payment processor's settlement report (external reconciliation), not just "looks plausible."

## Failure modes & pitfalls

- **Checks that describe instead of assert** (auto-profiled ranges). They page on growth and sleep through semantic breaks. Correction: hand-write invariant checks for critical tables; use profiling only to *suggest*.
- **All checks blocking, or all checks warning.** Blocking anomaly checks get disabled after false pages; warning-only contract checks let poison flow. Correction: invariants block, statistics warn — deliberately classify every check.
- **Testing marts but not sources.** By the mart, the bad data has already contaminated intermediates and the fix requires a full-chain backfill. Correction: dbt source freshness + source-level contract tests; push checks to ingestion.
- **`unique` test on the staging key but no uniqueness declared where BI reads** — dedup logic drifted, dashboard double-counts. Correction: uniqueness tests on the *consumption* layer keys too.
- **Backfilling an incremental model without deleting the bad partitions first** — duplicates. Correction: know each model's materialization; use partition-overwrite (`insert_overwrite`) semantics for anything you expect to backfill; make backfill a documented runbook per pipeline, not tribal knowledge.
- **Schema-only contracts.** Type-stable semantic drift (units, enum meaning, grain change from daily to hourly) sails through. Correction: encode grain and units in the contract and add a grain check (`count(*) = count(distinct key)`).
- **Freshness measured as "job succeeded"** while the job loaded zero rows. Correction: freshness = data recency (`max(event_time)`), plus non-empty and volume checks — a green DAG is not fresh data.
- **Anomaly alerts routed to a channel nobody triages.** Detection without ownership is noise. Correction: alerts route to the table's owner; weekly review of alert precision.
- **Lineage built once, never maintained** — decays in months as pipelines change; stale lineage is worse than none because people trust it. Correction: lineage emitted automatically from the orchestrator/dbt runs (OpenLineage events per run), never hand-drawn.
- **PII discovered by grep during an audit.** Correction: classification at ingestion as part of the contract; automated PII scanners as a backstop, not the primary control.
- **Governance rolled out as a freeze** ("no new datasets until cataloged"). The org routes around it within a month. Correction: govern the paved road — make the compliant path the easiest path (templates that emit contracts, classification, and ownership by default).

## Worked micro-example: contract enforced in dbt

```yaml
# models/marts/fct_orders.yml (dbt Core >= 1.5)
models:
  - name: fct_orders
    config:
      contract: {enforced: true}     # build fails on schema drift
    columns:
      - name: order_id
        data_type: string
        constraints: [{type: not_null}]
        data_tests: [unique]
      - name: order_total_usd        # name encodes unit — semantic guard
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

For the cross-team version of the same agreement, express it as an ODCS v3 YAML (schema + SLA blocks) and run `datacontract test` against the producer's actual table in their CI.

## Worked micro-example: boundary validation in Python (Pandera)

```python
# Ingestion boundary for an events feed — fail the job BEFORE bad data lands.
import pandera.pandas as pa
from pandera.typing import Series
import pandas as pd

class EventSchema(pa.DataFrameModel):
    event_id: Series[str] = pa.Field(unique=True, nullable=False)
    event_type: Series[str] = pa.Field(isin=["view", "click", "purchase", "refund"])
    amount_usd: Series[float] = pa.Field(ge=0, nullable=True)      # null OK, negative not
    event_time: Series[pd.Timestamp] = pa.Field(nullable=False)
    loaded_at: Series[pd.Timestamp] = pa.Field(nullable=False)

    @pa.dataframe_check
    def time_sanity(cls, df: pd.DataFrame) -> Series[bool]:
        return df["event_time"] <= df["loaded_at"]                 # no future events

    @pa.dataframe_check
    def purchase_has_amount(cls, df: pd.DataFrame) -> Series[bool]:
        return ~((df["event_type"] == "purchase") & df["amount_usd"].isna())

validated = EventSchema.validate(raw_df, lazy=True)   # lazy=True → report ALL failures,
                                                      # not just the first — essential for triage
```

`lazy=True` matters operationally: the failure report enumerates every violated check with row
counts, which is the difference between one fix cycle and five. Quarantine the failing rows
(write them to a reject table with the failure reason) rather than dropping them — rejects are
the evidence for the producer conversation.

## Verification / self-check

Before calling a dataset or platform "trustworthy":
1. For each critical table: one concrete check per dimension (freshness, completeness, uniqueness, validity) — name them; "we have dbt tests" is not an answer.
2. Simulate a producer schema change in a branch — does anything *block* before consumers break?
3. Ask lineage: "what breaks if I drop column X?" — answer in minutes, at column granularity, or lineage isn't real.
4. Run the incident drill on paper: bad partition lands at 2 a.m. — who's paged, what's quarantined, what's the backfill command, who tells consumers?
5. Check alert precision over the last month: > ~50% true-positive, or tune before expanding.
6. Reconcile one core metric against an external source of truth (payment processor, source system count).

Stopping rule: critical tables (the ones feeding money, ML, or executives) get contracts + four-dimension checks + owners; long-tail tables get freshness + volume monitoring only. Full coverage of everything is where data-quality programs go to die — coverage breadth past the critical set buys less than alert-precision tuning on what you have.
