# Core Checklist — 100/100

| # | Status | Rubric criterion | Evidence | Pts |
|---:|:---:|---|---|---:|
| 1 | ✅ | Bronze loads all raw rows; Silver dedups by `order_id` | Bronze 16 rows; 5 duplicates dropped | 8 |
| 2 | ✅ | Gate quarantines exactly 3 bad rows without halting | `quarantine.csv` has 3 data rows; run completes | 10 |
| 3 | ✅ | Gold aggregates completed orders; no duplicate survives | 5 daily Gold rows; Silver uniqueness check passes | 7 |
| 4 | ✅ | Topologically ordered DAG | `extract → validate → transform → report` | 5 |
| 5 | ✅ | Partition-by-key and idempotent streaming consumer | Replayed event ID is ignored by test/verify | 8 |
| 6 | ✅ | dbt staging→Gold, data tests and one unit test | Python 3.13; `PASS=11`, `ERROR=0` | 7 |
| 7 | ✅ | Recursive `gen_ai.*` trace flattening | 8 traces → 21 Bronze span rows | 12 |
| 8 | ✅ | Per-trace cost, latency and outcome summary | 8 summary rows printed by `make flywheel` | 6 |
| 9 | ✅ | Eval set from held-out `split='eval'` traces | `eval_golden.jsonl` has 2 rows | 6 |
| 10 | ✅ | DPO `(prompt, chosen, rejected)` pair mining | 3 valid raw pairs generated | 8 |
| 11 | ✅ | Eval decontamination | 3 raw pairs → 1 clean pair; 2 dropped | 8 |
| 12 | ✅ | ASOF point-in-time features avoid future leakage | Naive join leaks on 2 rows; ASOF does not | 10 |
| — | ✅ | Reproducible end-to-end verification | `16/16 checks — ALL PASS` | 5 |
|  |  | **Core total** |  | **100** |

Additional test evidence: `pytest` reports **18 passed**.

## Commands used

```text
make setup
make run
make flywheel
make kg
make verify
make test
DBT_PROFILES_DIR=. ../.venv-dbt/bin/dbt build
```

## Final results

```text
verify.py: 16/16 checks — ALL PASS
pytest: 18 passed
dbt: PASS=11 WARN=0 ERROR=0 SKIP=0 TOTAL=11
```

## Bonus

| Status | Criterion | Evidence | Pts |
|:---:|---|---|---:|
| ✅ | Real problem and clear constraints, ≥600 words | Legal-document pipeline in Vietnam, 1,724 words | 4 |
| ✅ | 4–6 open questions with explicit trade-offs | 6 explicit decisions with alternatives and reasons | 8 |
| ✅ | Rejected alternative | Page-level streaming and extract-everything graph rejected | 3 |
| ✅ | Architecture sketch | End-to-end ASCII pipeline included | 2 |
| ✅ | Vietnam/cost/failure awareness | Vietnamese OCR, sensitive data, idempotency, backfill and cost | 3 |
|  | **Bonus total** |  | **20** |
