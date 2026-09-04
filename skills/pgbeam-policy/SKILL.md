---
name: pgbeam-policy
description: Author a PgBeam policy profile as code (access mode, table allow and deny lists, PII masking rules, row filters, query budgets, write mode). Use this when you need to define or tighten what a PgBeam agent credential is allowed to do against Postgres, either with the CLI or as a reviewed Terraform resource. For first-time wiring of an agent to a database, use pgbeam-connect first.
---

# Author a PgBeam policy profile

A policy profile is a named bundle of enforcement rules that PgBeam applies in the Postgres wire protocol for any agent credential attached to it: access mode, table allow and deny lists, per-statement-kind rules, PII masking, per-relation row filters, query and egress budgets, and write mode. Define it once, attach it to a project default or to a specific agent credential, and it is enforced on every query with no change to the upstream database.

Start restrictive and widen deliberately. The safe default is read-only, no tables allowed until you name them, PII masked, and a budget set.

## The pieces of a profile

- **`access_mode`** (`read_only` or `read_write`). Read-only is the default and the right starting point for an agent. Read-write still respects the statement, allowlist, masking, and budget rules below.
- **`table_allowlist` / `table_denylist`**. What the credential may touch. If an allowlist is set, only those relations are visible and queryable; everything else is dropped, including from schema discovery, so the agent never learns it exists. Relations may be schema-qualified (`public.users`) or bare (`users`).
- **`statement_rules`** (`allow` / `deny` over statement kinds: `select`, `insert`, `update`, `delete`, `ddl`, `copy`, `set`, `show`, `explain`, `transaction`). An empty allow means every kind the access mode permits.
- **`masking_rules`**. Per-column masking applied to results in flight. Each rule has a `table`, a `column`, and a `kind`: `redact` (replace with a token), `null` (return NULL), or `hash` (SHA-256 hex, which keeps the same value mapping to the same output so joins still work).
- **`row_filters`**. A boolean SQL expression per relation that scopes which rows the credential can read, applied like an always-on `WHERE`.
- **Budgets and limits**: `budget_queries_per_hour`, `budget_queries_per_day`, `max_rows`, `statement_timeout_ms`, `egress_bytes_per_day`. Cap runaway loops and large scans.
- **`write_mode`** and the approval fields (`approval_mode`, `approval_auto_max_rows`, `approval_timeout_seconds`) gate writes and can route risky statements through human approval when read-write is enabled.

## Author it with the CLI

Fastest path. Create a profile, set masking and access mode, and attach it:

```bash
pgbeam policies create --name "Read-only analytics" \
  --access-mode read_only \
  --allow-table public.orders \
  --allow-table public.customers \
  --mask customers.email=hash \
  --mask customers.ssn=redact \
  --mask customers.phone=null \
  --max-rows 10000

# Attach as the project default, or to one credential:
pgbeam projects update --default-policy-profile <pol_id>
pgbeam agents create --name analytics --policy-profile <pol_id>
```

See `pgbeam policies --help` and https://pgbeam.com/docs/policies for the full flag set (row filters, budgets, timeouts, write mode).

## Author it as code (Terraform)

Preferred for anything reviewed or reproducible. The `pgbeam_policy_profile` resource defines the whole policy; other resources reference it by id.

```hcl
resource "pgbeam_policy_profile" "analytics" {
  project_id  = pgbeam_project.app.id
  name        = "Read-only analytics"
  access_mode = "read_only"

  table_allowlist = ["public.orders", "public.customers"]

  masking_rules = [
    { table = "public.customers", column = "email", kind = "hash" },
    { table = "public.customers", column = "ssn", kind = "redact" },
    { table = "public.customers", column = "phone", kind = "null" },
  ]

  row_filters = [
    { table = "public.orders", expression = "region = 'eu'" },
  ]

  max_rows                = 10000
  budget_queries_per_hour = 1000
  statement_timeout_ms    = 5000
  write_mode              = "normal"
}

resource "pgbeam_agent_credential" "analytics" {
  project_id        = pgbeam_project.app.id
  name              = "analytics-agent"
  policy_profile_id = pgbeam_policy_profile.analytics.id
}
```

Set a project-wide floor with `default_policy_profile_id` on `pgbeam_project`, or scope per credential with `policy_profile_id` on `pgbeam_agent_credential`. The same profile shape is available through the TS SDK (`createPolicyProfile`) and the Go SDK. Import an existing profile with `terraform import pgbeam_policy_profile.analytics <project_id>/<id>`.

## How to reason about a profile

- **Deny by construction, not by hope.** An allowlist plus read-only means the agent cannot reach a table you did not name, so a prompt injection has a small blast radius by default.
- **Mask, do not just hide.** Masked columns stay visible in the schema so the agent can join and reason about shape, but real PII never leaves the wire. Prefer `hash` for identifiers the agent needs to join on, `redact` or `null` for free-text or sensitive fields it should ignore.
- **Budget every credential.** A `max_rows` and a per-hour query cap turn a runaway agent loop into a bounded, audited event instead of a database incident.
- **Changes stream live.** Updated profiles propagate to the data plane without a redeploy, so tightening a policy takes effect on the next query.

## More

- Policies: https://pgbeam.com/docs/policies
- Masking: https://pgbeam.com/docs/masking
- Row-level policies: https://pgbeam.com/docs/row-level-policies
- Terraform provider: https://pgbeam.com/docs/terraform
