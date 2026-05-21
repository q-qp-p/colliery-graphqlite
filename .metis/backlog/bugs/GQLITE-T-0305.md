---
id: unwind-after-with-loses-list
level: task
title: "UNWIND after WITH loses list variables — \"_prev.X\" column missing"
short_code: "GQLITE-T-0305"
created_at: 2026-05-21T01:58:37.141580+00:00
updated_at: 2026-05-21T01:58:37.141580+00:00
parent: 
blocked_by: []
archived: false

tags:
  - "#task"
  - "#phase/backlog"
  - "#bug"


exit_criteria_met: false
initiative_id: NULL
---

# UNWIND after WITH loses list variables — "_prev.X" column missing

## Reproducer

```sql
.load build/graphqlite.dylib
SELECT cypher('CREATE ()-[:T]->()');
SELECT cypher('MATCH p = (n)-[r]->() WITH [n, r, p] AS types UNWIND range(0, size(types) - 1) AS i UNWIND range(0, size(types) - 1) AS j WITH types[i] AS lhs, types[j] AS rhs WHERE i <> j RETURN lhs, rhs');
-- Got: GraphQLiteError: no such column: _prev.types
```

## Affected TCK scenarios (6+)

- `Comparison2 [3]` — types-mixed list with UNWIND/UNWIND
- `ReturnOrderBy4 [1]` — ORDER BY of column introduced in RETURN (`_prev.prows`)
- `Unwind1 [3]` — Unwinding a concatenation of lists (`_prev.first`)
- Others with `_prev.<col>` not-found patterns

## Root cause hypothesis

`transform_unwind.c` wraps the prior query as an inner CTE aliased `_prev`,
then expects each prior variable to be selectable via `_prev.<name>`. The
carry-projection logic at `transform_unwind.c:117-140` enumerates the
prior variables, but appears to skip variables whose value is a list
literal (e.g. `types`) — only certain `transform_var` kinds get included.
When the next WITH/UNWIND tries to reference `types[i]`, the inner select
emits `_prev.types` but the projection didn't carry that column through.

## Fix sketch

In `transform_unwind.c`, inspect the carry-cols builder (`dbuf_appendf(&carry_cols, ", _prev.%s AS %s", ...)`).
Determine which `transform_var` kinds it includes and add any missing
kinds (likely `VAR_KIND_PROJECTED` list-typed scalars). Verify with the
reproducer above.

## Affected files

- `src/backend/transform/transform_unwind.c` (around the `carry_cols`
  builder, line ~117-140)

## Acceptance Criteria

- [ ] Reproducer query returns rows without "_prev.types" error
- [ ] No regression on existing UNWIND-after-WITH scenarios
- [ ] At least the 6 TCK scenarios above flip from error → pass/fail

## Discovered

2026-05-21 during the I-0042 follow-up TCK error audit (post +75 TCK
session). Multiple test failures share this single root cause.
