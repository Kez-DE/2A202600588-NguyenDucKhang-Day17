# Submission Run Log — Day 17

Environment used for this verification:

```text
macOS local run
Lite path: Python 3.14.4 virtualenv from `make setup`
dbt path: Python 3.11.15 virtualenv `.venv-dbt`
No API keys required; the lab runs zero-key by default.
```

## 1. Lite smoke test

Command:

```bash
make verify
```

Result:

```text
=== verify.py: Day 17 lab smoke test ===
=== Day 17 pipeline (lite) ===
  bronze rows in      : 13
  duplicates dropped  : 5  (Silver dedup)
  records quarantined : 3  (failed the gate)
  silver rows         : 8
  gold daily rows     : 5

Gold (completed orders by day):
order_date  n_orders  n_users  revenue  avg_order_value
2026-06-01         2        2    69.49            34.74
2026-06-02         1        1    19.99            19.99
2026-06-03         1        1    49.50            49.50
2026-06-04         1        1     5.00             5.00
2026-06-06         1        1    49.50            49.50
  [OK ] extract loaded raw rows
  [OK ] Silver dropped duplicates (the hook)
  [OK ] gate quarantined bad records
  [OK ] Gold produced daily rows
  [OK ] no duplicate order_id remains in Silver
  [OK ] streaming consumer is idempotent
  [OK ] doc->chunk->embedding ingestion
  [OK ] agent traces flattened into Bronze spans
  [OK ] eval golden set curated from traces
  [OK ] decontamination drops eval-overlapping pairs
  [OK ] at least one clean preference pair survives
  [OK ] ASOF point-in-time join avoids future leakage
  [OK ] knowledge graph built from docs
  [OK ] graph query answers 'what is returnable?' = widget only
  [OK ] graph answers a real 2-hop question (widget->accessory->warehouse)
  [OK ] vector foil: no single chunk answers the multi-hop question

RESULT: 16/16 checks — ALL PASS
```

## 2. Pytest suite

Command:

```bash
make test
```

Result:

```text
..................                                                       [100%]
18 passed in 0.61s
```

## 3. Flywheel artifacts

Command:

```bash
make flywheel
```

Result:

```text
=== Day 17 flywheel: agent traces -> datasets ===
  spans landed in Bronze   : 21 (from 8 traces)
  eval golden rows         : 2  -> eval_golden.jsonl
  preference pairs (raw)   : 3
  preference pairs (clean) : 1  -> preference_pairs.jsonl  (2 dropped by decontamination)

  rows where the naive join LEAKED a future value: 2
```

Generated submission artifacts:

```text
datasets/eval_golden.jsonl          2 rows
datasets/preference_pairs.jsonl     1 row
```

## 4. dbt track

Commands:

```bash
python3.11 -m venv .venv-dbt
. .venv-dbt/bin/activate
pip install -r requirements-dbt.txt
cd dbt_project
DBT_PROFILES_DIR=. dbt build
```

Result:

```text
Running with dbt=1.11.11
Registered adapter: duckdb=1.10.1
Found 2 models, 1 seed, 7 data tests, 485 macros, 1 unit test

Finished running 1 seed, 1 table model, 7 data tests, 1 unit test, 1 view model in 0 hours 0 minutes and 0.23 seconds.

Completed successfully
Done. PASS=11 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=11
```
