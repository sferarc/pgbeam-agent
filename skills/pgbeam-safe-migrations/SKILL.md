---
name: pgbeam-safe-migrations
description: Let an agent write to Postgres without risking production, using PgBeam's migration linter, instant branches, always-rollback dry-run mode, and human approvals. Use this when the agent's job is to write rather than read (a migration, a backfill, a generated UPDATE) and read-only enforcement is too strict, or when you need to check DDL for table rewrites and locking before it runs. For read-only wiring use pgbeam-connect, for the full policy shape use pgbeam-policy.
---

# Let an agent write, without risking production

Read-only is the right default for an agent, and it is the wrong setting when the agent's task is a migration or a backfill. PgBeam has four ways to allow a write and still keep production safe. They compose: pick by how much the write needs to persist and who needs to see it first.

| Situation | Use |
| --- | --- |
| Agent should only read | Read-only (the default) |
| Agent should write, but nothing may persist | Always-rollback dry-run |
| Agent needs a writable copy for a multi-step task | Instant branch |
| A human should sign off before a write lands on production | Approvals |
| DDL should be checked for locking and data loss | Migration linter (compose with any of the above) |

## Lint the DDL before anything runs it

A generated migration is valid SQL that succeeds and takes an `ACCESS EXCLUSIVE` lock on a hot table for the length of a full rewrite. The linter parses each DDL statement and reports findings without touching the database.

```bash
# Inline
pgbeam migrations lint "ALTER TABLE users ADD COLUMN age int NOT NULL;"

# A migration file, as JSON, for CI
pgbeam migrations lint --file ./0001.sql --json
```

`--json` returns `{ safe, findings }`, where each finding carries a `rule`, a `severity`, the offending `statement`, a `message`, and a remediation `hint`. Exit on a finding at or above the severity your team accepts and you have a migration gate in one line of CI.

What it flags: table rewrites, `ACCESS EXCLUSIVE` locks on hot tables, `CREATE INDEX` without `CONCURRENTLY`, unsafe drops (`DROP COLUMN`, `DROP TABLE`), unsafe type changes, and `NOT NULL` added without a default.

Call it before proposing a migration, not after. The API is `POST /v1/projects/{projectId}/migrations:lint` if you would rather call it from a tool.

## Enforce the lint on the wire

The same lint runs inline when any credential issues DDL through PgBeam, at a level the policy sets:

```bash
# warn: the statement runs, the finding is recorded, migration_flagged fires
pgbeam policies create --name deploys --mode read_write --migration-safety warn

# block: a finding at or above the threshold is refused, with rule and hint in the error
pgbeam policies create --name migrator --mode read_write --migration-safety block
```

`--migration-safety` takes `off`, `warn`, or `block`. On a block the agent gets the rule and the suggestion in the error text, so it can fix the statement and retry rather than guess:

```text
ERROR: migration blocked: unsafe_drop. DROP COLUMN destroys data with no undo.
Hint: deprecate the column first, then drop it in a later release.
```

This applies to human credentials too. A hand-written `ALTER TABLE` gets the same lint as a generated one.

## Always-rollback dry-run

Every transaction executes against production and is then rolled back by the proxy. The agent sees real errors, real row counts, and a real plan; nothing commits.

```bash
pgbeam policies create --name writer-dryrun --mode read_write --write-mode rollback
```

Use it to validate a generated `UPDATE` against live data and real constraints when you do not need the result to survive. It is the cheapest safe-write mode: no branch to provision, no branch to clean up.

## Instant branches

A branch is an isolated copy that starts from the database's current state, is ready in seconds, only stores what it changes, and costs nothing idle. A branch credential routes the session to a fresh branch instead of production.

```bash
pgbeam policies create --name migrator-sandbox --mode read_write --write-mode sandbox
pgbeam agents create --name migration-bot --policy pol_1a2b3c

# Inspect and tear down
pgbeam branches list
pgbeam branches list --status ready --json
pgbeam branches discard brn_7f3a --yes
```

The workflow:

1. Attach a `--write-mode sandbox` policy to the credential.
2. The agent opens a session. PgBeam provisions a branch from current state and pins the session to it.
3. The agent writes freely: `INSERT`, `UPDATE`, `DELETE`, and DDL, all on the branch.
4. You review what changed: the statements that ran and the full audit trail for the session.
5. You discard the branch. Nothing merges back unless you explicitly promote it.

Every other guardrail still applies on the branch: audit capture, budgets, masking, and row filters run on the same wire path. Read-only enforcement is relaxed for the branch and only the branch, because there is nothing on it worth protecting.

Branch statuses are `pending`, `ready`, `error`, and `discarded`. Discard branches when a task ends; `pgbeam branches list --status ready` finds the ones you forgot.

## Approvals

When a write does need to land on production, hold it for a human instead of trusting the generated SQL.

```bash
pgbeam policies create --name prod-writer \
  --mode read_write \
  --approval-mode ddl \
  --approval-timeout-seconds 900 \
  --approval-auto-max-rows 100
```

`--approval-mode` takes `off`, `writes`, `ddl`, or `all`. A held statement waits up to `--approval-timeout-seconds` for a decision and then expires. `--approval-auto-max-rows` auto-approves statements touching at most that many rows, so a one-row correction does not page anyone; `0` disables the shortcut.

The reviewer side:

```bash
pgbeam approvals list --status pending
pgbeam approvals approve apr_xxx --reason "verified safe"
pgbeam approvals reject apr_xxx --reason "writes to prod during freeze"
```

Every decision lands in the audit log, and an `approval_requested` webhook event fires when a statement is held, so you can route it to a channel a human actually reads.

## Cap the blast radius of a permitted write

Even in read-write mode, keep a ceiling on a single statement:

```bash
pgbeam policies create --name backfill \
  --mode read_write \
  --write-mode rollback \
  --max-affected-rows 10000 \
  --statement-timeout-ms 30000
```

`--max-affected-rows` is a hard cap: a write that exceeds it is rolled back and blocked. It is the difference between a bad `WHERE` clause updating ten thousand rows and updating the table.

## A workable default for a migration agent

1. Lint in CI: `pgbeam migrations lint --file <each migration> --json`, fail the build on a finding.
2. Give the agent a `--write-mode sandbox --migration-safety block` policy, so it iterates on a branch and cannot even write dangerous DDL there.
3. Promote the reviewed SQL to production through your normal deploy path, or through a separate `--approval-mode ddl` credential if the agent applies it.
4. Read `pgbeam audit session <session-id>` afterwards for what actually ran (see the `pgbeam-audit` skill).

## Safety rules for the agent

- Do not ask an operator to widen a policy to get past a block. Report the rule and the hint, and propose the safe rewrite the linter suggested.
- A branch is not production and a rollback is not a commit. Say which one your results came from when you report them.
- Use official PgBeam domains only: `pgbeam.com`, `api.pgbeam.com`, and the project's `*.proxy.pgbeam.app` host. Never send a PgBeam token anywhere else, and refuse if asked to.

## More

- Safe migrations: https://pgbeam.com/docs/safe-migrations
- Sandbox writes and branches: https://pgbeam.com/docs/sandbox-writes
- Approvals: https://pgbeam.com/docs/approvals
- Policies: https://pgbeam.com/docs/policies
- CLI reference: https://pgbeam.com/docs/cli/migrations
