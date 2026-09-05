# PgBeam agent skills

Installable [agent skills](https://agentskills.io) that teach an AI coding agent to use [PgBeam](https://pgbeam.com) for safe Postgres access. Each skill is a `SKILL.md` with YAML frontmatter (`name`, `description`) and a body the agent reads when a task matches.

| Skill | What it teaches |
| --- | --- |
| [`pgbeam-connect`](./pgbeam-connect/SKILL.md) | Wire an AI agent to Postgres safely: get a scoped credential, choose a guarded connection string or the hosted MCP endpoint, and paste-ready Claude Code, Cursor, and VS Code config. |
| [`pgbeam-policy`](./pgbeam-policy/SKILL.md) | Author a policy profile as code: access mode, table allow and deny lists, PII masking, row filters, budgets, write mode, with CLI and Terraform. Lint, dry-eval, and replay it before attaching it. |
| [`pgbeam-mcp-usage`](./pgbeam-mcp-usage/SKILL.md) | Drive the hosted Postgres MCP tools well once connected: prefer `schema_catalog`, read the policy errors, work within read-only and masking. |
| [`pgbeam-audit`](./pgbeam-audit/SKILL.md) | Answer "what did this agent do": read and filter the audit trail, summarize a session, export to CSV or a SIEM, verify the tamper-evident chain, and turn recorded traffic into a tighter policy. |
| [`pgbeam-safe-migrations`](./pgbeam-safe-migrations/SKILL.md) | Let an agent write without risking production: lint DDL for locking and data loss, instant branches, always-rollback dry-run mode, and human approvals. |
| [`pgbeam-cli`](./pgbeam-cli/SKILL.md) | Drive the `pgbeam` CLI: install, authenticate, link a project, register a database, pull a guarded `DATABASE_URL`, diagnose with `doctor`, and script anything with `--json`. |

## Install

With the open skills tool:

```bash
npx skills add pgbeam-connect
```

Or copy a `SKILL.md` into your agent's skills directory (for example `.claude/skills/pgbeam-connect/SKILL.md` for Claude Code, or `.agents/skills/` for the cross-agent standard).

Agents can also discover these skills over HTTP from the PgBeam site, which serves a machine-readable index with per-skill integrity digests:

- Index: `https://pgbeam.com/.well-known/agent-skills/index.json`
- Each body: `https://pgbeam.com/skills/<name>/SKILL.md`

## Agent Plugin

The whole set also ships as an [Agent Plugin](https://agent-plugins.org/specification). The plugin root is the repository root: `plugin.json` is the manifest, `mcp.json` declares the MCP servers, and this `skills/` directory is where a client discovers the skills. The manifest is also served at `https://pgbeam.com/.well-known/agent-skills/plugin.json`.

`mcp.json` declares the CLI's stdio management server (`pgbeam mcp`), which any client can start once the [CLI](https://pgbeam.com/docs/cli) is installed. The hosted per-project database MCP endpoint is not declared there, because its URL and bearer token are per project and per credential: `pgbeam agents create` prints ready-to-paste config for it, and `pgbeam-connect` explains the choice between the two.

## Contributing

Issues and pull requests are welcome here. An issue is the right place to start for a bug, a wrong doc, or a missing capability; say what you ran, what happened, what you expected, and which version you were on.

Do not open a public issue for a suspected security vulnerability. Email security@pgbeam.com, or report it privately from this repository's Security tab.

## License

Apache 2.0. See [LICENSE](LICENSE).
