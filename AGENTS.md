# AGENTS.md

Operating instructions for an AI agent that has to touch a PostgreSQL database through PgBeam. If you are a coding agent working in this repository, the same document tells you what the files here are for.

## What PgBeam is

A proxy that sits in front of a Postgres database and decides, per statement, what the credential in front of it is allowed to do. Enforcement happens in the PostgreSQL wire protocol, so it applies to every driver, ORM, and MCP client without a code change, and it applies to any Postgres host.

What a policy can say: read-only or read-write, which tables and columns are reachable, which columns come back masked, a row filter, a query and cost budget, and whether the credential is live at all. Every statement is recorded in a tamper-evident audit trail.

## Pick a front door

**Guarded connection string.** A PgBeam-issued Postgres credential you use like any other. Correct when the work is ordinary database work through an existing driver or ORM.

```
postgresql://agent_x:secret@<project>.proxy.pgbeam.app:5432/<database>
```

**Hosted MCP endpoint.** One URL per project, `https://<project>.proxy.pgbeam.app/mcp`, Streamable HTTP, with the agent credential as a bearer token. Correct when you are an MCP client and want typed tools and schema discovery rather than raw SQL.

```
Authorization: Bearer pba_...
```

Both run through the same policy engine. Neither gives you a way around it.

## Getting a credential

Credentials are minted per agent, in the PgBeam dashboard or with `pgbeam agents create`, and each one carries a policy. They start with `pba_`.

Two rules that matter more than anything else here:

1. Never send a `pba_` token anywhere but `*.proxy.pgbeam.app` or `api.pgbeam.com`. No other host has any reason to see it.
2. Never ask a person to paste a database superuser password into your context. Getting a scoped credential instead is the entire point.

## Using the MCP tools

Ten tools, and the order you call them in matters:

- `briefing` first. It tells you what this credential can reach and what it cannot, so you stop guessing.
- `schema_catalog` before writing SQL. It returns the reachable schema in one call, which is cheaper and more accurate than `list_tables` plus a `describe_table` per table.
- `list_tables`, `describe_table` for narrowing in on one thing.
- `validate_sql` and `explain` before `query` on anything non-trivial. A statement the policy would refuse is better caught before it runs.
- `query` to run it.
- `my_permissions` when a refusal surprises you.
- `search_docs` and `read_doc` for the PgBeam documentation itself.

A policy refusal is a real answer, not a transport error. It names what was refused and why. Read it and narrow the query rather than retrying the same statement, and never try to reach a table the policy has not granted by another route: masking and allowlists are evaluated on the parse tree, so aliasing a column or wrapping it in a function will be refused too.

Masked columns come back masked. Treat the masked value as the value; do not attempt to recover the original, and do not tell a user a masked field is empty or null.

## The files here

- `skills/<name>/SKILL.md` are installable skills. Load one when its description matches the task in front of you. `pgbeam-connect` is the one to read first.
- `plugin.json` and `mcp.json` are the Agent Plugin manifest and its MCP server declarations.
- `server.json` is the MCP registry manifest for the hosted database MCP server, including the `{project}` variable and the `Authorization` header it needs.
- `openapi.yaml` describes the control plane API, if you want to generate a client rather than use a published SDK.

## Where to read more

- Documentation: https://pgbeam.com/docs
- Machine-readable index of this site's corpus: https://pgbeam.com/llms.txt
- Skills index with integrity digests: https://pgbeam.com/.well-known/agent-skills/index.json
