# PgBeam for agents

Everything an AI agent needs to reach Postgres through [PgBeam](https://pgbeam.com), in one repository: the agent skills, the plugin and MCP manifests, and the API contract.

PgBeam sits in front of your database and enforces what an agent is allowed to do, in the PostgreSQL wire protocol. Read-only by default, table and column allowlists, PII masking, query budgets, a kill switch, and a tamper-evident audit trail of every statement. It works with any Postgres host, and it needs no code changes: swap the connection string, or point your client at the hosted MCP endpoint.

```
postgresql://agent_x:secret@abc.proxy.pgbeam.app:5432/mydb
```

## What is in here

| Path | What it is |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | Operating instructions for an agent: which front door to pick, how to authenticate, which tool to call first, and the limits to expect. |
| [`skills/`](skills) | Installable [agent skills](https://agentskills.io), one `SKILL.md` per directory. |
| [`plugin.json`](plugin.json) | [Agent Plugin](https://agent-plugins.org/specification) manifest. The skills beside it are the plugin's skills. |
| [`mcp.json`](mcp.json) | The MCP servers the plugin declares. Today that is the `pgbeam mcp` stdio server the CLI runs. |
| [`server.json`](server.json) | The MCP registry manifest for the hosted, per-project database MCP server. |
| [`openapi.yaml`](openapi.yaml) | The OpenAPI 3 description of the PgBeam control plane API, if you would rather generate a client than use one of the published SDKs. |

## Install a skill

```bash
npx skills add sferarc/pgbeam-agent
```

That lists the skills in this repository and installs the ones you pick. To take a single one, copy its `SKILL.md` into your agent's skills directory: `.claude/skills/pgbeam-connect/SKILL.md` for Claude Code, or `.agents/skills/` for the cross-agent layout.

Agents can also fetch the same bodies over HTTP, with an integrity digest per skill:

- Index: `https://pgbeam.com/.well-known/agent-skills/index.json`
- Body: `https://pgbeam.com/skills/<name>/SKILL.md`

## Connect an agent

1. Create a project at [pgbeam.com](https://pgbeam.com) and add your database.
2. Issue a scoped agent credential and attach a policy to it.
3. Give the agent either the guarded connection string or the project's MCP URL, `https://<project>.proxy.pgbeam.app/mcp`, with the credential as a bearer token.

The [`pgbeam-connect`](skills/pgbeam-connect/SKILL.md) skill walks an agent through this on its own, including paste-ready config for Claude Code, Cursor, and VS Code. The full guides are at [pgbeam.com/docs](https://pgbeam.com/docs).

## Elsewhere

The client libraries and providers have their own repositories: [pgbeam-js](https://github.com/sferarc/pgbeam-js) (TypeScript SDK), [pgbeam-go](https://github.com/sferarc/pgbeam-go) (Go SDK), [pgbeam-cli](https://github.com/sferarc/pgbeam-cli), [pgbeam-terraform](https://github.com/sferarc/pgbeam-terraform), [pgbeam-pulumi](https://github.com/sferarc/pgbeam-pulumi), [pgbeam-crossplane](https://github.com/sferarc/pgbeam-crossplane), and [pgbeam-docs](https://github.com/sferarc/pgbeam-docs).

## Contributing

Issues and pull requests are welcome here. For a bug, a wrong instruction in a skill, or a missing capability, an issue is the right place to start: say what you ran, what happened, what you expected, and which version you were on.

Do not open a public issue for a suspected security vulnerability. Email security@pgbeam.com, or report it privately from this repository's Security tab.

## License

Apache 2.0. See [LICENSE](LICENSE).
