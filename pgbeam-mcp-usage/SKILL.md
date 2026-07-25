---
name: pgbeam-mcp-usage
description: Drive PgBeam's hosted Postgres MCP tools well once an agent is connected. Use this when the agent is already wired to a PgBeam MCP server (query, validate_sql, list_tables, describe_table, explain, schema_catalog, plus search_docs and read_doc) and needs to explore a schema and run SQL efficiently against policy-enforced, read-only-by-default, PII-masked, audited access. For the initial wiring and credential setup, use pgbeam-connect first.
---

# Use PgBeam's hosted Postgres MCP tools well

This skill is for an agent that is already connected to a PgBeam hosted MCP
server (see the `pgbeam-connect` skill for wiring). It explains how to explore a
schema and run SQL efficiently, and how the policy layer shapes what you get
back so you can work with it instead of fighting it.

The server exposes eight tools. Six are database tools: `query`, `validate_sql`,
`list_tables`, `describe_table`, `explain`, and `schema_catalog`. Every database
call runs through the same wire-level policy as a normal connection: read-only by
default, table and column allowlists, PII masking, per-credential budgets, and a
full audit trail. The other two, `search_docs` and `read_doc`, look up how PgBeam
works; they are read-only and not database-scoped.

## Start with schema_catalog, not information_schema

Call `schema_catalog` first. One call returns a compact, LLM-optimized view of
the whole database you are allowed to see: tables with their columns (name,
type, nullable, default, comment), primary keys, foreign keys, indexes,
approximate row counts, and table and column comments. This replaces multiple
round trips against `information_schema` or `pg_catalog`.

Two properties matter for how you read the result:

- **It is already filtered by policy.** Tables and columns this credential may
  not see are omitted from the catalog. If a table you expected is missing, it
  is denied by the allowlist, not absent from the database. Do not try to route
  around this; query a different table or ask the operator to widen the policy.
- **Masked columns are flagged, not hidden.** A column marked `masked` exists
  and you can reference it, but its values come back masked. Use it for joins
  and shape, not for reading real PII.

For very large schemas the catalog paginates with a keyset cursor: if the result
has `truncated: true` and a `next_cursor`, call `schema_catalog` again with that
cursor to get the next page.

Reach for `list_tables` and `describe_table` only when you want a single table's
detail and do not need the whole catalog.

## Running queries

Use `query` for SQL. Assume read-only: `SELECT` and read-side CTEs work; writes
(`INSERT`, `UPDATE`, `DELETE`, DDL) are rejected unless the policy profile
explicitly allows them, which it does not by default. Do not attempt writes to
probe the policy; a blocked write is an audited event.

Use `validate_sql` to check a statement's table and column references against the
schema you are allowed to see before you run it. It returns any unknown or
ambiguous names with ranked suggestions, so you can fix a hallucinated name
without spending a failed query on it.

Use `explain` (which returns `EXPLAIN (FORMAT JSON)`) before running a query you
expect to be expensive, so you can check the plan against the row-count estimates
from `schema_catalog` and avoid burning the query budget on a full scan.

## Read the errors; they are written for you

When the policy blocks a query, the error text explains why in plain language:
which table or column was not allowed, that the credential is read-only, or that
a budget was exceeded. Treat a block as information, not a dead end:

- **Not allowed / relation denied:** the table or column is outside the
  allowlist. Query an allowed relation instead.
- **Read-only:** the statement tried to write. Rephrase as a read, or the
  operator must grant the write in the policy profile.
- **Budget exceeded:** you hit the per-credential row or cost limit. Narrow the
  query (add a `WHERE`, a `LIMIT`, or an aggregate) rather than retrying the same
  broad scan.

Adjust and retry based on the reason. Do not loop on the identical failing query.

## What you can rely on

- The masked values you receive are safe to surface; the real PII never entered
  your context.
- Everything you run is audited with an allow, block, or mask decision, tagged
  as coming from the MCP. Behave as if a human will read the audit log, because
  they can.
- The credential is revocable and kill-switchable out of band. If calls start
  failing wholesale, the credential may have been rotated or disabled; stop and
  report it rather than retrying.

## More

- Connect and credential setup: the `pgbeam-connect` skill
- MCP server card: https://pgbeam.com/.well-known/mcp/server-card.json
- Docs: https://pgbeam.com/docs/mcp
