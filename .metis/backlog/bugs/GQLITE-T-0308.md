---
id: cross-type-order-comparison-should
level: task
title: "Cross-type order comparison should return null (1 < []), but operand re-transform crashes"
short_code: "GQLITE-T-0308"
created_at: 2026-05-21T12:30:00.000000+00:00
updated_at: 2026-05-21T12:30:00.000000+00:00
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

# Cross-type order comparison returns true/false instead of null

## Reproducer

```sql
SELECT cypher('RETURN 1 < "a" AS r');    -- got: true,  expected: null
SELECT cypher('RETURN 1 < [] AS r');     -- got: true,  expected: null
SELECT cypher('RETURN [1] < 1 AS r');    -- got: false, expected: null
SELECT cypher('RETURN 1 < {} AS r');     -- got: true,  expected: null
SELECT cypher('RETURN "a" < 1 AS r');    -- got: false, expected: null
SELECT cypher('RETURN true < false AS r'); -- got: false, expected: null
```

Per the openCypher spec, ordering comparisons `<`, `>`, `<=`, `>=`
across incompatible type classes return null. graphqlite currently
relies on SQLite's native operators which silently coerce types.

## Affected TCK (~10 scenarios)

- Comparison2 [3] examples 1-4 (cross-type yields null)
- Comparison4 [1] (chained comparisons over null-yielding operands)
- Likely others in Aggregation / Comparison families

## Iter 23 update — capture-string approach ALSO crashes

Tried the capture-expr approach proposed in this doc (call
transform_expression only TWICE — once per operand — and substitute
captured strings into a printf template). Built and ran the TCK:
**still crashed in WITH WHERE execution with SIGABRT** at exactly the
same site.

This rules out "multiple invocations of transform_expression have
side effects" as the cause. Even invoking it just once per operand
in a fresh temp buffer triggers the crash on property access through
the order-cmp path.

Hypotheses to investigate:
1. `transform_property_access` writes to `ctx->pending_prop_joins`
   or `ctx->unified_builder->joins` — those buffers DO mutate ctx
   even when we switch ctx->sql_buffer. So the FIRST capture writes
   joins into the real builder, the SECOND capture writes the SAME
   joins again — leading to duplicate aliases or buffer overflow.
2. `transform_expression` may also mutate `ctx->global_alias_counter`
   — calling twice gives two different aliases for the same property.
3. The realloc-and-free dance for the temp buffer may leak/corrupt
   memory if `transform_expression` realloc's the buffer and we then
   free the original pointer (not the new one).

Next attempt should:
- Snapshot ALL ctx state (sql_buffer, pending_prop_joins,
  unified_builder buffers, alias_counter) before each capture.
- Restore everything after the capture.
- Or: do ONE pass that emits operand-once, and use SQLite's CASE/WITH
  to bind it to a name for re-use.

## Attempted fix and failure mode (iter 22)

Tried wrapping LT/GT/LTE/GTE in `transform_expr_ops.c` with:

```sql
CASE WHEN L IS NULL OR R IS NULL THEN NULL
     WHEN (typeof(L) IN ('integer','real') AND typeof(R) IN ('integer','real'))
       OR (typeof(L) = 'text' AND substr(L,1,1) NOT IN ('[','{')
           AND typeof(R) = 'text' AND substr(R,1,1) NOT IN ('[','{'))
     THEN L op R
     ELSE NULL END
```

Check is semantically correct (verified on literal queries). But emitting
the CASE requires referencing L and R **multiple times** in the SQL
output, which means calling `transform_expression()` repeatedly on the
same AST node.

For property-access expressions like `n.var`, `transform_expression()`
has **side effects**: it allocates aliases, registers variables, etc.
Calling it 10× per ordering comparison crashes the worker with SIGABRT
on queries like `WHERE i.var > 'te'`.

TCK impact of naive attempt: pass=3436 → 3425 (-11), with 4 extension
crashes added (WithWhere5 [1]-[4]).

## Fix sketch

Capture each operand's emitted SQL ONCE using a temporary buffer (mirror
the `transform_expression_to_string` pattern from
`transform_return.c:147`), then substitute the captured strings into the
CASE template using printf. This way `transform_expression` is called
only twice (once per operand), avoiding side-effect repetition.

```c
char *lhs_sql = capture_expr(ctx, binary_op->left);
char *rhs_sql = capture_expr(ctx, binary_op->right);
append_sql(ctx, "(CASE WHEN %s IS NULL OR %s IS NULL THEN NULL "
                "WHEN (typeof(%s) IN ('integer','real') "
                     "AND typeof(%s) IN ('integer','real')) "
                "OR (typeof(%s) = 'text' AND substr(%s,1,1) NOT IN ('[','{') "
                    "AND typeof(%s) = 'text' AND substr(%s,1,1) NOT IN ('[','{')) "
                "THEN %s %s %s ELSE NULL END)",
    lhs_sql, rhs_sql,
    lhs_sql, rhs_sql,
    lhs_sql, lhs_sql, rhs_sql, rhs_sql,
    lhs_sql, op_sql, rhs_sql);
free(lhs_sql); free(rhs_sql);
```

## Affected files

- `src/backend/transform/transform_expr_ops.c`
- Possibly add `capture_expr` helper to `transform_helpers.c`

## Acceptance Criteria

- [ ] `RETURN 1 < "a"` returns null (was true)
- [ ] `RETURN 1 < []` returns null (was true)
- [ ] `RETURN n.foo > 'te'` on text property still works (no crash)
- [ ] No TCK regression from the previous baseline
- [ ] Comparison2 [3] examples flip fail → pass

## Discovered

2026-05-21 during iteration 22 of the open-work queue.
