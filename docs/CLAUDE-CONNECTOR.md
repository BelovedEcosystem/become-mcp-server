# Claude — Beloved BECOME custom connector

Ship path for Claude.ai / Claude Desktop **custom remote MCP** connectors.

## Happy path (Wave-2 OAuth)

1. Open **Claude Settings → Connectors**, add **custom / remote MCP**:
   - Name: `Beloved BECOME`
   - MCP URL: `https://mcp.belovedecosystem.com/mcp`
2. **Approve OAuth** (`ask:zero-bid`). No key paste. Never put a key in chat.
3. Ask BECOME with Gallina Target (prefer `become_ask`; interim `become_orchestrate` OK), e.g. `exists n : nat, 16 + 16 = n`.
4. Prefer `wait: false` on `certify: true`. Honor `poll_after_ms` — **do not poll before 2 hours** on certify Proving. Settled as soon as ready; within **24 hours**. Then read `answer` / `inbox`.

Public contract: https://mcp.belovedecosystem.com/agent.md

Anthropic guide: https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-MCP

## Break-glass only (Channel A — not done)

If OAuth is unavailable: open **https://mcp.belovedecosystem.com/connect/claude**, complete CAPTCHA v1, mint a browser-once key, vault it in Settings → Connectors. Remint there if the key was lost before vaulting. Never chat-paste; never use owner GitHub Actions artifact download as the happy path. **No public demo key.**

## Acceptance

- [ ] Connector attached (OAuth approve, or break-glass vault) — no key in the thread
- [ ] `tools/call` no longer returns `unauthorized_missing_key`
- [ ] One ask returns a `job_id`; after ≥2h (or `poll_after_ms`) → `Settled` with `answer`/`inbox` numerals

## Notes

- Custom connector you add — not (yet) an Anthropic directory listing.
- Claude Desktop uses the same custom remote MCP + Connectors flow as claude.ai.
- `certify: true` + `Proving` / `StagedAwaitingProve` is normal; wait `poll_after_ms` (**2h**) — no early poll.
- Prefer Gallina Target shape `exists n : nat, A + B = n`.
