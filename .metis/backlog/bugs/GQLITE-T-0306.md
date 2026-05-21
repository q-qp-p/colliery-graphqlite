---
id: optional-match-after-with-emits
level: task
title: "OPTIONAL MATCH after WITH emits invalid SQL alias (uses column ref as table alias)"
short_code: "GQLITE-T-0306"
created_at: 2026-05-21T02:58:20.251410+00:00
updated_at: 2026-05-21T02:58:20.251410+00:00
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

# OPTIONAL MATCH after WITH emits invalid SQL alias

## Reproducer

```sql
.load build/graphqlite.dylib
SELECT cypher('OPTIONAL MATCH (a:X) WITH a OPTIONAL MATCH (b)-[r:R]->(a) RETURN a, b, r');
-- Got: SyntaxError near ".": syntax error (PARSE_ERROR)
```

## Bad SQL excerpt

```sql
... LEFT JOIN nodes AS _gql_default_alias_1 ON 1=1
    LEFT JOIN edges AS _gql_default_alias_2 ON ...
    LEFT JOIN nodes AS _with_0.a ON _with_0.a.id = _gql_default_alias_2.target_id
```

The trailing `LEFT JOIN nodes AS _with_0.a` is invalid — `_with_0.a` is a
column reference (alias form `<table>.<col>`), not a table alias. SQLite
parses the `.` as a separator and errors with `near '.':` syntax error.

## Root cause hypothesis

`transform_match.c` re-binds variable `a` when it appears in the second
OPTIONAL MATCH pattern. The variable's `table_alias` is the cross-clause
column reference (`_with_0.a`) inherited from the WITH projection. When
the transformer emits a JOIN for the re-used variable, it uses that
reference as if it were a fresh table alias, producing `AS _with_0.a`.

Correct behavior: when a variable is already bound (carried through
WITH), don't add it to the FROM/JOIN — use the existing column reference
and constrain via equality (`_gql_default_alias_2.target_id = _with_0.a`).

## Affected TCK scenarios (5)

- `Match7 [21]` — Handling optional matches between nulls
- `Match7 [27]` — Handling optional matches between optionally matched entities
- `Match4 [8]` — Matching relationships into a list and matching variable length using the list
- `Match9 [6]/[7]` — Matching relationships into a list and matching variable length using the list

All produce the same `near '.':` syntax error from the same alias
construction bug.

## Affected files

- `src/backend/transform/transform_match.c` — the JOIN emission path
  for already-bound variables

## Acceptance Criteria

- [x] Reproducer query parses and runs without "near '.':" error
- [x] No regression on existing OPTIONAL MATCH / WITH chain scenarios
- [x] 2 of 5 TCK scenarios flipped error → fail (Match7 [21]/[27])
- [ ] Remaining 3 (Match4 [8], Match9 [6]/[7]) still error — they use variable-length relationships, a different code path (`generate_varlen_cte`)

## Status Updates

**2026-05-21** — Partial fix in commit 9cac9c6.

The non-varlen `optional` path in `generate_relationship_match` had a
`target_already_added` check that did `strstr(from, target_alias)`.
When `target_alias` was a cross-clause column reference (`_with_0.a`)
from a prior WITH projection, the substring search missed it because
FROM/JOIN held only the CTE name `_with_0`.

Added a check: if `target_node->variable` is a `VAR_KIND_PROJECTED`
variable or has `alias_is_id` set (both indicate WITH-binding), treat
target as already-attached. This skips the bogus
`LEFT JOIN nodes AS _with_0.a ON _with_0.a.id = ...` emission.

The 3 remaining var-length scenarios (Match4 [8], Match9 [6]/[7]) take
a different code path (`generate_varlen_cte` and its CTE-style join
construction) that has the same alias-mishandling issue. Filing as
follow-on work in this same ticket — the source-side and var-length
analog need similar `alias_is_id`-aware skipping.

## Discovered

2026-05-21 during iteration 9 of the open-work queue, after TCK
stabilization (deterministic random()).
