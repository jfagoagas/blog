---
title: "RLS and Full-Text Search - LEAKPROOF matters"
description: "Postgre planner skips index use under RLS if operators aren't leakproof."
date: 2024-12-01T10:00:00+02:00
draft: false
showToc: true
TocOpen: false
tags: ["postgres", "rls"]
UseHugoToc: true
---
When [Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) (RLS) is enabled on a table, PostgreSQL must guarantee that any user-supplied predicate cannot see rows they shouldn't. Concretely, the RLS-derived filter is enforced **before** any user-provided condition. If PostgreSQL were to apply a user's condition first, for example:

```sql
SELECT finding.id
FROM findings
WHERE tenant_id = '81884830-05c9-4178-aef6-d3e2a5e70284' 
  AND text_search @@ to_tsquery('secret');
```

it could inadvertently reveal whether hidden rows match that condition. To prevent this, PostgreSQL only allows user predicates to be "pushed down" into an index scan if they are marked [**LEAKPROOF**](https://www.postgresql.org/docs/current/sql-createfunction.html)—i.e., proven never to leak information about rows.

Unfortunately, none of PostgreSQL's full-text search (FTS) operators (such as `@@`) or the typical TS-query constructors (`plainto_tsquery`, `to_tsquery`, etc.) are marked `LEAKPROOF`. In practice, this means that under RLS:

* A condition like `WHERE text_search @@ to_tsquery('secret')` cannot be executed at the index level.
* PostgreSQL is forced to scan all rows that pass the RLS filter, then apply the `@@`, or any other TS operator, check on each one.

## RLS and LEAKPROOF

> From https://www.postgresql.org/docs/current/ddl-rowsecurity.html#DDL-ROWSECURITY

To specify which rows are visible or modifiable according to a policy, an expression is required that returns a Boolean result. This expression will be evaluated for each row prior to any conditions or functions coming from the user's query. The only exceptions to this rule are **leakproof** functions, which are guaranteed to not leak information; the optimizer may choose to apply such functions ahead of the row-security check.

## Limitations of Multicolumn Indexes for RLS with Full‐Text Search Vectors

When using RLS, you generally need to include `tenant_id` in every index so that PostgreSQL can filter rows by tenant before applying any additional predicates. In theory, you might create a `btree_gin` index on `(tenant_id, text_search)` to accelerate full‐text queries per tenant, but because full-text operators like `@@` are not marked `LEAKPROOF`, the planner is not allowed to push them below the RLS filter. In practice, this means PostgreSQL will enforce RLS first (filtering by `tenant_id` and any other RLS rules) and only then evaluate the `@@` condition on the remaining rows—so the TS component of your `btree_gin` can't be used. Moreover, if your table is joined or referenced by a tenants table, the planner will still choose the simple index on `tenant_id` to satisfy RLS, and only afterward apply the text‐search operator, effectively making the combined `(tenant_id, text_search)` index pointless under RLS.

### Note from a PostgreSQL Core Maintainer

>  https://www.postgresql.org/message-id/flat/18941-90236c38c6101bf0%40postgresql.org

_You would be okay, probably, if the system thought that the @@ operator is "leakproof", because then it can be applied before or concurrently with the RLS condition, which is what's needed to allow an indexscan using the @@ condition.  But our rules for marking things leakproof are pretty strict and TS search operators don't qualify._

_If you're desperate enough, you could mark ts_match_vq() as leakproof and hope that (a) there's not a way to abuse it or at least (b) your users aren't smart enough to find one. I think you might also need to use the 2-parameter form of plainto_tsquery(), to allow that call to be reduced to a constant before these decisions are made._


## Conclusion

In short, PostgreSQL's insistence on strict leakproof guarantees for pushing down predicates is by design: it ensures RLS cannot be bypassed. But it also means that any full-text search operator—none of which is leakproof—can't benefit from a `btree_gin(tenant_id, text_search)` index under RLS. If you need to ensure row-level security constraints and fast TS queries, you may have to look into alternative architectures or accept the necessary performance trade-offs.

