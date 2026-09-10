<p align="center">
  <b>Beloved Ecosystem</b>
</p>

<h1 align="center">BECOME MCP Server</h1>

<p align="center">
  <b>The remote Model Context Protocol (MCP) front door for BECOME: any AI client asks a Gallina question with no key and no login; the host Coq-checks it in seconds, and can certify it with an EC2 proof settled on Sepolia.</b>
</p>

<p align="center">
  <a href="https://github.com/BelovedEcosystem/become-mcp-server"><img src="https://img.shields.io/badge/Beloved_Ecosystem-BECOME-0ea5e9?logo=github&logoColor=white" alt="Beloved Ecosystem BECOME"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-0ea5e9" alt="License: Apache 2.0"></a>
  <a href="https://mcp.belovedecosystem.com/agent.md"><img src="https://img.shields.io/badge/Contract-agent.md-0ea5e9" alt="Agent contract"></a>
</p>

<p align="center">
  <a href="https://modelcontextprotocol.io"><img src="https://img.shields.io/badge/Model_Context_Protocol-compatible-000000?logo=modelcontextprotocol&logoColor=white" alt="MCP compatible"></a>
  <a href="#auth"><img src="https://img.shields.io/badge/Auth-none_required-0ea5e9" alt="Auth: none required"></a>
  <a href="#how-to-connect"><img src="https://img.shields.io/badge/Hosting-Remote_HTTP_MCP-0ea5e9" alt="Hosting: Remote HTTP MCP"></a>
</p>

## Fastest connect

**Claude Code (one line):**

```bash
claude mcp add --transport http become https://mcp.belovedecosystem.com/mcp
```

**Claude.ai / Desktop:** Settings → Connectors → Add custom connector → paste `https://mcp.belovedecosystem.com/mcp`.

**Any client with a project config:** this repo ships a root [`.mcp.json`](.mcp.json); clone it and Claude Code offers the server automatically. No header, no key.

**Then ask:** call `become_orchestrate` with `target: "exists n : nat, 5 + 1 = n"`. You get `6`, Settled, in seconds. Add `certify: true` for a Sepolia-settled proof (about 1–2 hours; keep the returned `job_id` and `owner_token` to poll). Full contract with real responses: [`/agent.md`](https://mcp.belovedecosystem.com/agent.md). Design notes: [`docs/LEAST-FRICTION.md`](docs/LEAST-FRICTION.md). Target data model (quads, ERC-7683, Nostr 5700, MCP): [`docs/SCHEMA.md`](docs/SCHEMA.md).

## Production host

| | URL |
| --- | --- |
| **MCP** | `https://mcp.belovedecosystem.com/mcp` |
| **Landing / health** | `https://mcp.belovedecosystem.com/` |
| **Agent contract (public)** | [`https://mcp.belovedecosystem.com/agent.md`](https://mcp.belovedecosystem.com/agent.md) |

Ephemeral `*.trycloudflare.com` tunnels are **obsolete** — do not use them.

## Auth

**None required.** Since 2026-09-10 the host accepts every job as requested from anonymous callers, including certified runs at zero bid. Jobs are bound to the session that created them plus an `owner_token` returned on creation, so only the creator can read a job. A per-client rate limit applies to job creation only.

Operator keys still work (`Authorization: Bearer <key>`) for the advanced tools. If you use one, vault it in the connector settings and never paste it into chat.

## How to connect

### Claude (custom remote MCP connector)

1. Open Claude → **Settings → Connectors** (or Custom connectors).
2. Add a remote MCP server named **Beloved BECOME**.
3. URL: `https://mcp.belovedecosystem.com/mcp`
4. Auth: leave empty.
5. Ask with `become_orchestrate`. Contract with real responses: [`/agent.md`](https://mcp.belovedecosystem.com/agent.md).

Guide: [Anthropic — custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp).  
Details: [`docs/CLAUDE-CONNECTOR.md`](docs/CLAUDE-CONNECTOR.md).

### Cursor / HTTP MCP (`mcp.json`)

```json
{
  "mcpServers": {
    "beloved-become": {
      "type": "http",
      "url": "https://mcp.belovedecosystem.com/mcp"
    }
  }
}
```

No key needed. If an operator gave you one for the advanced tools, add a `headers` block with `Authorization: Bearer` and keep the key in your client's secrets.

### Grok / product connectors

Name **Beloved BECOME**, MCP URL above, auth empty. Grok's first-contact review and the fixes it led to are recorded in [`docs/AUDIT-2026-09-10.md`](docs/AUDIT-2026-09-10.md).

## Default client path

1. Prefer **`become_orchestrate`** with Gallina `target` (e.g. `exists n : nat, 5 + 1 = n`), `bid_wei=0`, `chain=sepolia`.
2. `certify: false` = checked in seconds; `certify: true` = EC2 proof (~1–2 h) then auto-settled on Sepolia with a `receipt_url`.
3. Prefer `wait: false` on certify; keep the returned `job_id` **and** `owner_token`; poll **`become_status`** then **`become_result`** with both.
4. Drive off `agent_status`. **Never trust** `answer` / `inbox` / `answer_v` until `Settled`.
5. If `auto_generate` / `run_id` is present → **poll only** (do not upload to S3 or dispatch GHA yourself).

Full contract: [`/agent.md`](https://mcp.belovedecosystem.com/agent.md).

## Agent skill

```bash
npx skills install BelovedEcosystem/become-mcp-server
# or: npx skills install BelovedEcosystem/become-mcp-server -g
```

Skill: [`skills/become-use/SKILL.md`](skills/become-use/SKILL.md).

## Tools (client-facing)

| Tool | Use |
| --- | --- |
| `become_orchestrate` | Default ask (chat / zero-bid) |
| `become_status` | Poll with `job_id` + `owner_token` |
| `become_result` | Settled `answer` / `inbox` / `receipt_url` |
| `become_ask` | Advanced alias requiring explicit `bid_wei`; not needed |

## Honesty

- Job hash: **BecomeJobHash/v0**
- Settlement TCB: **T2CERT0** (not full coqchk-in-guest)
- **Sepolia ≠ mainnet**
- Never send private keys through the ask path

## FAQ

### Why `auth_required` on `become_status`?

You polled a job without its `owner_token`, or from a session that did not create it. Pass the `job_id` and `owner_token` returned when the job was created.

### Why `jobid_already_settled`?

That exact target was already certified on Sepolia. Change the numbers; the prior receipt is in the error.

### Do I need EC2 myself for certify?

No. With auto Generate live on the public host, the MCP box stages and dispatches prove; you poll until Settled.

## License

Apache 2.0 — see [LICENSE](LICENSE).
