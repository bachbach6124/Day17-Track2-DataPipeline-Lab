# Verification Evidence

Environment used:

```text
Lite pipeline: Python 3.14.6
dbt pipeline: Python 3.13.6
```

## `make verify`

```text
bronze rows in      : 16
duplicates dropped  : 5
records quarantined : 3
silver rows         : 8
gold daily rows     : 5

RESULT: 16/16 checks — ALL PASS
```

The 16 checks cover Medallion ingestion, quarantine, deduplication, Gold,
streaming idempotency, document embedding, trace flattening, eval/DPO
decontamination, point-in-time features and the knowledge graph.

## `make test`

```text
..................                                                       [100%]
18 passed
```

## `dbt build`

```text
Found 2 models, 1 seed, 7 data tests, 1 unit test
Completed successfully
PASS=11 WARN=0 ERROR=0 SKIP=0 NO-OP=0 TOTAL=11
```

## Generated submission artifacts

```text
datasets/eval_golden.jsonl       2 rows
datasets/preference_pairs.jsonl  1 clean pair
submission/REFLECTION.md         183 words
bonus/DESIGN.md                  1,724 words
```
