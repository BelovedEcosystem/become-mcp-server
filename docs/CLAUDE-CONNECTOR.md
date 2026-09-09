# Claude — Beloved BECOME custom connector

Ship path for Claude.ai / Claude Desktop **custom remote MCP** connectors.

## What you add

| Field | Value |
| --- | --- |
| Name | `Beloved BECOME` |
| MCP URL | `https://mcp.belovedecosystem.com/mcp` |
| Auth | Vaulted API key → `Authorization: Bearer <key>` or `X-Api-Key: <key>` |

Public contract (no auth): https://mcp.belovedecosystem.com/agent.md

## Steps

1. In Claude, open **Settings → Connectors** (wording may vary by product surface).
2. Add a **custom / remote MCP** connector.
3. Paste the MCP URL above.
4. Put the operator-provisioned `BECOME_MCP_API_KEY` into Claude’s **vault / header** field — **never into chat**.
5. Start a chat and ask BECOME (Gallina Target), e.g. certify `exists n : nat, 16 + 16 = n`.
6. Poll until `agent_status=Settled` before reading `answer` / `inbox`.

Anthropic guide: https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-MCP

## Acceptance

- [ ] Connector connected; no key in the thread
- [ ] `POST /mcp` no longer returns `unauthorized_missing_key`
- [ ] One `become_orchestrate` call returns a `job_id`
- [ ] Poll → `Settled` with `answer`/`inbox` numerals
- [ ] No key in logs or reply

## Notes

- This is a **custom** connector you add — not (yet) an Anthropic-curated directory listing.
- `certify: true` + `certify_requires_ec2` / `Proving` is normal on this host; keep polling.
- Prefer Target shape `exists n : nat, A + B = n` for NatToy.
