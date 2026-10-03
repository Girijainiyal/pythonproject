# DE Concepts — Quick Revision Layer
*Your workbook (dbt sheet, rows 5,6,8,14 + SQL sheet row 14) already has full depth on these. This is the fast-recall version — read this once, say each answer out loud once, done.*

---

## CDC (Change Data Capture)

**One-liner:** CDC is a way to capture only the ROWS THAT CHANGED at the source, instead of re-reading the whole table every time.

**Three ways it's implemented (know all three, pick based on source):**
1. **Timestamp-based (your usual pattern):** source has `updated_at` — pull `WHERE updated_at > last_watermark`. Simple, but misses hard DELETEs.
2. **Log-based CDC (true CDC):** reads the database's transaction log (e.g., Oracle GoldenGate, Debezium, Snowflake's Streams) — captures INSERT/UPDATE/DELETE as they happen, even deletes. More accurate, more infra.
3. **Trigger-based:** DB triggers write changes to a shadow/audit table. Older pattern, adds write overhead to the source system.

**Interview line:** *"My incremental dbt models use timestamp-based CDC via a watermark — `updated_at > MAX(updated_at) in target`. The gap is deletes don't show up that way, so I'd pair it with periodic full reconciliation or a soft-delete flag from source."*

**Snowflake-native CDC tool worth naming:** **Streams** — a Stream object tracks row-level changes (insert/update/delete) on a table since it was last consumed, then a Task can process just those changed rows. This is Snowflake's built-in log-based CDC mechanism — mention it if asked "how would you do TRUE CDC in Snowflake."

---

## SCD (Slowly Changing Dimension) — Type 1 vs Type 2

**One-liner:** how do you handle a dimension value changing over time (a customer moves cities) — overwrite it, or keep history?

- **Type 1 — overwrite.** Old value is gone. Simple, but you lose history. Use when the old value has zero business value (e.g., fixing a typo).
- **Type 2 — keep every version as a new row**, with `valid_from`, `valid_to`, `is_current` (or dbt's `dbt_valid_from`/`dbt_valid_to`). Use when history matters for reporting (e.g., "what was this patient's insurance plan on the date of service").

**How you build SCD2 in dbt — say this exact mechanism:**
> "dbt **snapshots** automate SCD2. You give it a `unique_key` and a strategy — `timestamp` (trust an `updated_at` column) or `check` (compare specific column values row to row). Every run, dbt compares current source state to what's stored: unchanged rows do nothing, changed rows close the old version (`dbt_valid_to` filled in) and insert a new current version (`dbt_valid_to` = NULL)."

```sql
{% snapshot patients_snapshot %}
{{ config(target_schema='snapshots', unique_key='patient_id',
          strategy='timestamp', updated_at='updated_at') }}
SELECT * FROM {{ source('raw','patients') }}
{% endsnapshot %}
```

---

## Seeds in dbt

**One-liner:** seeds are small, static, rarely-changing CSV files that live IN your dbt repo and get loaded into the warehouse via `dbt seed`.

**When to use a seed vs. a source table:** seeds are for reference/lookup data that's small and controlled by YOU (the analytics engineer), not the source system — think: a country-code-to-region mapping, a list of holiday dates, a manual list of excluded customer IDs. If it changes often or comes from an operational system, it's a `source`, not a `seed`.

```
seeds/country_codes.csv   →   dbt seed   →   becomes a real table, ref()'able like any model
```

**Interview line:** *"I've used seeds for small static mapping tables — reference data that's version-controlled alongside the code, so a change to the mapping goes through the same PR review as any model change, rather than being a silent manual edit in the warehouse."*

---

## Tests in dbt (you have this well-covered already — quick recall)

- **Generic tests** (in `schema.yml`): `unique`, `not_null`, `accepted_values`, `relationships` (FK check) — reusable, config-driven, no SQL to write yourself.
- **Singular tests**: a standalone `.sql` file under `tests/` — any query that should return **zero rows** if data quality is good (e.g., "claims with negative amounts").
- **Packages**: `dbt_utils`, `dbt_expectations` — pre-built advanced tests (e.g., `expression_is_true`, distribution checks).
- **Where they run**: CI on every PR (catch before merge) + after prod runs via orchestration (catch data drift) — failures block downstream promotion.

---

## Facts and Dimensions (rapid recall)

**Fact table** = the "what happened" table — one row per event/transaction, holds the **numbers** (measures) you aggregate.
**Dimension table** = the "who/what/when/where" context around that event — descriptive attributes, joined to the fact.

**Fact table types (say all three if asked "types of facts"):**
1. **Transactional** — one row per individual event (a single charge/transaction).
2. **Periodic snapshot** — one row per entity per time period, capturing a state (e.g., account balance at each month-end).
3. **Accumulating snapshot** — one row per PROCESS, updated in place as it moves through stages (e.g., claim: submitted → adjudicated → paid, with a date column for each milestone).

**Measure types:**
- **Additive** — safely SUM across any dimension (amount, quantity).
- **Semi-additive** — sum across some dimensions but NOT time (account balances — you can sum across accounts, but summing a balance across months is meaningless).
- **Non-additive** — never sum (percentages, ratios) — must be recalculated from underlying additive components.

**Dimension flavors worth naming if pushed:**
- **Conformed** — same dimension (e.g., `dim_date`) shared/reused across multiple fact tables/marts.
- **Role-playing** — one physical dimension used multiple times with different meaning (`dim_date` as both `admit_date` and `discharge_date` in the same fact).
- **Degenerate** — a dimension-like value that lives directly IN the fact table with no separate dimension table (e.g., invoice number).
- **Junk** — several small unrelated flags/indicators bundled into one dimension table to avoid many tiny dimension tables.

---

## The one-sentence version of each, if they want speed-round answers

- **CDC:** "Capturing only what changed at the source, not re-reading everything."
- **SCD2:** "Keeping every historical version of a dimension row with valid_from/valid_to dates."
- **Seed:** "A small static CSV, version-controlled in the repo, loaded as a real table."
- **Test:** "An automated check that fails the build if data quality breaks."
- **Fact:** "One row per event, holds the numbers."
- **Dimension:** "The descriptive context around that event."
