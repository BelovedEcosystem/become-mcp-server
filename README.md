<p align="center">
  <b>Beloved Ecosystem</b>
</p>

<h1 align="center">BECOME MCP Server</h1>

<p align="center">
  <b>The remote Model Context Protocol (MCP) front door for BECOME: AI clients ask Gallina questions; the host coqchk-checks and (optionally) EC2-proves + Sepolia-settles.</b>
</p>

<p align="center">
  <a href="https://github.com/BelovedEcosystem/become-mcp-server"><img src="https://img.shields.io/badge/Beloved_Ecosystem-BECOME-0ea5e9?logo=github&logoColor=white" alt="Beloved Ecosystem BECOME"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-0ea5e9" alt="License: Apache 2.0"></a>
  <a href="https://mcp.belovedecosystem.com/agent.md"><img src="https://img.shields.io/badge/Contract-agent.md-0ea5e9" alt="Agent contract"></a>
</p>

<p align="center">
  <a href="https://modelcontextprotocol.io"><img src="https://img.shields.io/badge/Model_Context_Protocol-compatible-000000?logo=modelcontextprotocol&logoColor=white" alt="MCP compatible"></a>
  <a href="#auth"><img src="https://img.shields.io/badge/Auth-API_key_required-0ea5e9" alt="Auth: API key"></a>
  <a href="#how-to-connect"><img src="https://img.shields.io/badge/Hosting-Remote_HTTP_MCP-0ea5e9" alt="Hosting: Remote HTTP MCP"></a>
</p>

## Production host

| | URL |
| --- | --- |
| **MCP** | `https://mcp.belovedecosystem.com/mcp` |
| **Landing / health** | `https://mcp.belovedecosystem.com/` |
| **Agent contract (public)** | [`https://mcp.belovedecosystem.com/agent.md`](https://mcp.belovedecosystem.com/agent.md) |

Ephemeral `*.trycloudflare.com` tunnels are **obsolete** — do not use them.

## Auth

Public MCP requires a key on protected routes:

- `Authorization: Bearer <BECOME_MCP_API_KEY>`
- or `X-Api-Key: <BECOME_MCP_API_KEY>`

**Vault the key in your client connector settings.** Never paste it into chat. There is no public self-serve mint today — ask a Beloved operator for a scoped key (out-of-band). Preferred long-term: product connectors that vault the key for you.

## How to connect

### Claude (custom remote MCP connector)

1. Open Claude → **Settings → Connectors** (or Custom connectors).
2. Add a remote MCP server named **Beloved BECOME**.
3. URL: `https://mcp.belovedecosystem.com/mcp`
4. Auth: store the API key in Claude’s connector vault / header settings (`Authorization: Bearer …` or `X-Api-Key`).
5. Read [`/agent.md`](https://mcp.belovedecosystem.com/agent.md), then ask with `become_orchestrate`.

Guide: [Anthropic — custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp).  
Details: [`docs/CLAUDE-CONNECTOR.md`](docs/CLAUDE-CONNECTOR.md).

### Cursor / HTTP MCP (`mcp.json`)

```json
{
  "mcpServers": {
    "beloved-become": {
      "type": "http",
      "url": "https://mcp.belovedecosystem.com/mcp",
      "headers": {
        "Authorization": "Bearer ${BECOME_MCP_API_KEY}"
      }
    }
  }
}
```

Set `BECOME_MCP_API_KEY` in your environment or Cursor secrets — never commit a real key.

### Grok / product connectors

See packaging notes in the BECOME interim docs (`GROK-CONNECTOR-PACKAGING.md` in `become-dvm-interim`): name **Beloved BECOME**, base URL above, vault the same API key.

## Default client path

1. Prefer **`become_orchestrate`** with Gallina `target` (e.g. `exists n : nat, 5 + 1 = n`), `bid_wei=0`, `chain=sepolia`.
2. `certify: false` = fast smoke; `certify: true` = EC2 prove (~60–70m toys) then auto-settle when live.
3. Prefer `wait: false` on certify; poll **`become_status`** then **`become_result`**.
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
| `become_status` | Poll `job_id` |
| `become_result` | Settled `answer` / `inbox` |
| `become_ask` | Explicit-bid path only |

## Honesty

- Job hash: **BecomeJobHash/v0**
- Settlement TCB: **T2CERT0** (not full coqchk-in-guest)
- **Sepolia ≠ mainnet**
- Never send private keys through the ask path

## FAQ

### Why 401?

Missing or invalid API key. Connect/vault a key — do not invent one. See `/agent.md` Get access.

### Do I need EC2 myself for certify?

No. With auto Generate live on the public host, the MCP box stages and dispatches prove; you poll until Settled.

## License

Apache 2.0 — see [LICENSE](LICENSE).
