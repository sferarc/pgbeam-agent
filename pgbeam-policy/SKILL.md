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
  --mode read_only \
  --allow public.orders \
  --allow public.customers \
  --mask customers.email=hash \
  --mask customers.ssn=redact \
  --mask customers.phone=null \
  --max-rows 10000

# Attach to one credential, or set the project default for passthrough sessions:
pgbeam agents create --name analytics --policy pol_1a2b3c
pgbeam projects update --default-policy-profile-id pol_1a2b3c
```

`--allow` and `--deny` are repeatable and also take a comma-separated list; `--table-allowlist` and `--table-denylist` take the whole list at once. `--file ./policy.json` supplies the full profile body (statement rules, row filters, everything), and individual flags overlay whatever the file set. `--dry-run` prints the resolved profile without calling the API, so you can diff a change before you make it.

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

## Prove the profile before you attach it

Three checks, all offline against the data plane's own policy engine, none of which touch the upstream database. Run them in this order: they answer progressively harder questions.

```bash
# 1. Is the profile itself badly shaped?
pgbeam policies lint --draft ./policy.json --strict

# 2. What verdict would one statement get?
pgbeam policies dry-eval --policy pol_1a2b3c --sql "SELECT email FROM users"

# 3. What would change for traffic that already ran?
pgbeam policies replay --draft ./policy.json --bound-policy pol_1a2b3c --json
```

`policies lint` reasons about the shape alone: read-write with no table allowlist, committing writes with no affected-row cap or approval, missing budgets, masking with no read ceiling, masking or row-filter rules on tables the policy has already made unreachable, write settings that are inert on a read-only profile, redundant allow and deny overlaps. `--strict` exits non-zero on any warning-or-worse finding, which makes it a CI gate.

`policies dry-eval` prints the verdict (allow, block, mask, or row-filter) for one statement, with the rule and reason and any injected row-filter predicate. Stateful things a single-statement preview cannot model, per-region budgets, approvals, and write routing, come back as informational notes.

`policies replay` runs the project's recorded audit traffic through the candidate and reports which previously-allowed queries would now be blocked, which blocked ones would now pass, and what would newly be masked or filtered. Pass `--bound-policy` to narrow it to the credentials the policy actually governs, which is the question you want before editing a live profile. Tightening a policy with `newly_blocked` at zero is a tightening nobody will notice.

## How to reason about a profile

- **Deny by construction, not by hope.** An allowlist plus read-only means the agent cannot reach a table you did not name, so a prompt injection has a small blast radius by default.
- **Mask, do not just hide.** Masked columns stay visible in the schema so the agent can join and reason about shape, but real PII never leaves the wire. Prefer `hash` for identifiers the agent needs to join on, `redact` or `null` for free-text or sensitive fields it should ignore.
- **Budget every credential.** A `max_rows` and a per-hour query cap turn a runaway agent loop into a bounded, audited event instead of a database incident.
- **Changes stream live.** Updated profiles propagate to the data plane without a redeploy, so tightening a policy takes effect on the next query.

## More

- Reviewing what the policy actually did: the `pgbeam-audit` skill
- Allowing writes safely: the `pgbeam-safe-migrations` skill
- Policies: https://pgbeam.com/docs/policies
- Masking: https://pgbeam.com/docs/masking
- Row-level policies: https://pgbeam.com/docs/row-level-policies
- Terraform provider: https://pgbeam.com/docs/terraform
