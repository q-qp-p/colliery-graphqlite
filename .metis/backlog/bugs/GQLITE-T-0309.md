---
id: var-length-return-projection
level: task
title: "Var-length RETURN: projection treats CTE alias as edge row instead of emitting list of edges"
short_code: "GQLITE-T-0309"
created_at: 2026-05-21T16:05:19.794253+00:00
updated_at: 2026-05-21T16:05:19.794253+00:00
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

# Var-length RETURN projects single-edge JSON against varlen CTE alias

## Reproducer

```sql
.load build/graphqlite.dylib
SELECT cypher('CREATE ()-[:T]->()');
SELECT cypher('MATCH (a)-[r*1..1]->(b) RETURN r');
-- Error: "no such column: _gql_default_alias_2.id"
```

Expected: `[[{type: "T", ...}]]` (list containing one path of edges).

## Root cause

For a variable-length pattern `[r*1..1]`, the FROM clause aliases the
recursive CTE as `_varlen_path_1 AS _gql_default_alias_2`. The CTE has
columns `start_id, end_id, depth, path_ids, visited` — but the RETURN
projection emits a single-edge JSON template referencing
`_gql_default_alias_2.id`, `_gql_default_alias_2.type`, etc. as if the
alias were a row from the `edges` table.

Concrete bad SQL (formatted):

```sql
WITH RECURSIVE _varlen_path_1(start_id, end_id, depth, path_ids, visited) AS (...)
SELECT (CASE WHEN _gql_default_alias_2.id IS NULL THEN NULL
             ELSE json_object('id', _gql_default_alias_2.id,
                              'type', _gql_default_alias_2.type, ...) END)
FROM nodes AS _gql_default_alias_0
CROSS JOIN _varlen_path_1 AS _gql_default_alias_2
CROSS JOIN nodes AS _gql_default_alias_1
WHERE _gql_default_alias_2.start_id = _gql_default_alias_0.id
  AND _gql_default_alias_2.end_id = _gql_default_alias_1.id
  AND _gql_default_alias_2.depth BETWEEN 1 AND 1
-- Errors: no such column: _gql_default_alias_2.id
```

## Fix sketch

When `r` is bound to a variable-length pattern (the var's kind is
VAR_KIND_EDGE but it represents a sequence of edges, the alias points
at the path CTE), the RETURN projection must emit a JSON array
constructed from `path_ids` rather than a single edge JSON object:

```sql
SELECT (SELECT json_group_array(json_object(
            'id', e.id, 'type', e.type, 'startNodeId', e.source_id,
            'endNodeId', e.target_id, 'properties', ...))
        FROM edges e
        WHERE e.id IN (
            SELECT CAST(value AS INTEGER)
            FROM json_each('[' || REPLACE(_gql_default_alias_2.path_ids, ',', ',') || ']')
        ))
```

(or similar — use a helper UDF `_gql_path_edges_json(path_ids)`).

This needs the projection code (transform_return.c, or wherever edge
JSON is emitted) to:
1. Detect that the variable being projected is a varlen-bound edge
   (transform_var->is_varlen flag or similar marker exists?).
2. Branch to the list-of-edges projection instead of single-edge.

## Affected TCK (22 errors)

Match4 [1] [5] [6] [7], Match7 [13] [20], Match9 [1] [2] [3] [4] [5]
[6] [7], and likely additional follow-ons that surface after this
fix. ~22 error scenarios share this root cause.

## Affected files

- `src/backend/transform/transform_return.c` — the edge-projection
  template emission (json_object with edge fields).
- `src/backend/transform/transform_match.c` — sets up the varlen CTE
  and alias; the var_kind / is_varlen info has to flow to the
  projector.

## Acceptance Criteria

- [ ] `MATCH (a)-[r*1..1]->(b) RETURN r` returns `[[{type: 'T'}]]` for
  a single edge of type T.
- [ ] `MATCH (a)-[r*1..2]->(b) RETURN r` returns the list of edge
  lists, one per path of length 1 or 2.
- [ ] No regression in non-varlen edge projection
  (`MATCH ()-[r]->() RETURN r`).
- [ ] 22 `_gql_default_alias` errors in TCK flip to pass (or to a
  different fail).

## Discovered

2026-05-21 during iteration 38 of the open-work queue, dumping the
SQL via /tmp/badsql.log debug print.
