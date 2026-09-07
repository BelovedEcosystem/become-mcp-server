<p align="center">
  <b>Beloved Ecosystem</b>
</p>

<h1 align="center">BECOME MCP Server</h1>

<p align="center">
  <b>The remote Model Context Protocol (MCP) front door for BECOME: a hosted bridge that lets AI tools ask certified questions — plain language in, Coq-checked answers out.</b>
</p>

<p align="center">
  <a href="https://github.com/BelovedEcosystem/become-mcp-server"><img src="https://img.shields.io/badge/Beloved_Ecosystem-BECOME-0ea5e9?logo=github&logoColor=white" alt="Beloved Ecosystem BECOME"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-Apache_2.0-0ea5e9" alt="License: Apache 2.0"></a>
  <a href="#how-to-connect"><img src="https://img.shields.io/badge/Status-Interim_TEST_ONLY-b45309" alt="Status: Interim TEST_ONLY"></a>
</p>

<p align="center">
  <a href="https://modelcontextprotocol.io"><img src="https://img.shields.io/badge/Model_Context_Protocol-compatible-000000?logo=modelcontextprotocol&logoColor=white" alt="MCP compatible"></a>
  <a href="#data-and-security"><img src="https://img.shields.io/badge/Auth-Bearer_API_key-0ea5e9" alt="Auth: Bearer API key"></a>
  <a href="#how-to-connect"><img src="https://img.shields.io/badge/Hosting-Remote_HTTP_MCP-0ea5e9" alt="Hosting: Remote HTTP MCP"></a>
</p>

BECOME MCP is a cloud-reachable MCP endpoint for [Beloved Ecosystem](https://github.com/BelovedEcosystem)’s BECOME path: agents ask questions; the server forms the Coq job, runs acceptance (coqchk on the interim path), and returns a certified answer. Optional certify mode aims at SP1 + PayHook settlement when production pins exist.

**At launch (interim):** this repo documents the remote MCP URL and ships an agent skill so clients call tools correctly. Labels stay **NON-BECOME / TEST_ONLY** until founder production pins are frozen.

## Supported AI platforms

BECOME MCP works with any app that supports remote HTTP MCP, including:

* [Claude](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)
* [Cursor](https://cursor.com/docs/mcp#installing-mcp-servers)
* Other MCP-compatible clients

## Before you start

* An MCP-capable AI client
* Network reachability to the MCP URL below
* For certify / settlement paths: understanding that interim pins are **TEST_ONLY**

## How to connect

Add the BECOME MCP server URL to your AI client’s MCP settings:

**MCP URL (public host):**  
`https://mcp.belovedecosystem.com/mcp`

Landing / health: `https://mcp.belovedecosystem.com/`  
Contract detail for agents: `https://mcp.belovedecosystem.com/agent.md`

Requests are authenticated with your operator `BECOME_MCP_API_KEY` as a Bearer token.

### Cursor / Claude (HTTP MCP)

```json
{
  "mcpServers": {
    "become": {
      "type": "http",
      "url": "https://mcp.belovedecosystem.com/mcp",
      "headers": {
        "Authorization": "Bearer YOUR_BECOME_MCP_API_KEY"
      }
    }
  }
}
```

1. Add the URL **and the `Authorization` header** in your client’s MCP settings (Claude: Settings → Connectors → custom MCP).
2. Start a chat and ask BECOME a question (see prompts below).
3. Prefer loading the [`become-use`](skills/become-use/SKILL.md) skill first so tool calls stay well-formed.

**Never paste your API key into a chat thread** — it belongs only in the connector headers.

### Smoke test

Once connected, one prompt is enough:

> Ask BECOME: 5 + 1 = ? Use become_orchestrate with bid_wei=0.

Default smoke is `certify: false` → expect Settled **6**.

For the full certified path, add `certify: true`, `chain: sepolia` — slower, and still **TEST_ONLY** until production pins are frozen.

## Agent skill

This repo ships a [`become-use`](skills/become-use/SKILL.md) agent skill that guides AI agents on how to call BECOME MCP tools — ask → status → result order, certify flags, and interim labels.

```bash
# Install the become-use skill from this repo (skills CLI)
npx skills install BelovedEcosystem/become-mcp-server

# Or install globally
npx skills install BelovedEcosystem/become-mcp-server -g
```

## Supported capabilities

| Category | What you can do |
| -------- | ---------------- |
| **Orchestrate** | `become_orchestrate` — one-call ask → settle; takes `bid_wei`, `certify`, `chain` |
| **Ask** | `become_ask` — plain-language question; server forms Coq + optional bid/certify |
| **Status** | `become_status` — poll by `job_id` |
| **Result** | `become_result` — `answer.v`, inbox, visible numerals |
| **Spec** | `become_spec_check` — compile NatToy Coq spec (advanced) |
| **REST** | `POST /v1/ask`, `GET /v1/status/{id}`, `GET /v1/result/{id}`, `GET /health` |

### Example prompts

* “Ask BECOME: 5 + 1 = ? Use become_orchestrate with bid_wei=0.”
* “Ask BECOME: 1 + 1 = ?”
* “Run become_ask for 3 + 3 and show the result.”
* “What’s the status of job \<id\>?”

Default toy: `1 + 1 = ?` → visible `2` via coqchk. Set `certify: true` for SP1 + PayHook when pins are live.

## Data and security

* **API key** — keep `BECOME_MCP_API_KEY` in your client’s connector headers only. Do not paste it into a chat thread, a commit, or an issue.
* **No secrets** — do not send private keys, salts you must keep, or credentials through the ask path.
* **Labels** — responses and this repo mark **NON-BECOME / TEST_ONLY** until official `jobHash` / coqchk-in-guest / founder pins exist.
* **Scope** — the assistant can only do what the MCP tools expose; settlement and key custody stay out of band.

## BECOME vs Beloved Ecosystem

| | **Beloved Ecosystem** | **BECOME** |
| --- | --- | --- |
| **What** | Science-to-startup factory | Coq DVM: question → certified answer → optional on-chain settlement |
| **This repo** | Brand / org home | Remote MCP docs + agent skill for the ask path |

Related: [BelovedEcosystem/become](https://github.com/BelovedEcosystem/become) (protocol / product codebase).

## FAQ

### Do I need a paid plan?

No special Trello/Atlassian plan applies. You need an MCP client. Certify/settlement may require chain funds when you leave zero-bid toys.

### What is a BECOME MCP server?

The remote connection layer that lets AI assistants ask BECOME questions and receive checked answers under the tool permissions you configure in your client.

### Will using BECOME MCP consume AI tokens?

Yes. Any MCP server draws on your AI client’s message limits or tokens.

### Is this production BECOME?

Not yet. The hostname and auth are now stable, but treat answers as **interim / TEST_ONLY** until the founder freezes production pins (jobHash, verifier, programVKey).

## License

Apache-2.0 — see [LICENSE](LICENSE).
