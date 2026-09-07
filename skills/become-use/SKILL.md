---
name: become-use
description: >-
  Use this BEFORE calling BECOME MCP tools (become_orchestrate, become_ask,
  become_status, become_result, become_spec_check). Covers the one-call
  orchestrate path, ask→status→result order, question shapes, certify/bid
  flags, and NON-BECOME / TEST_ONLY labels.
disable-model-invocation: false
---

# BECOME MCP Server — Usage Skill

BECOME MCP lets an agent ask certified questions without writing Gallina. The server forms the Coq job from the question.

## 1. Connect

Public host: `https://mcp.belovedecosystem.com/mcp`. Auth is the operator `BECOME_MCP_API_KEY` as a Bearer token, set in the client’s connector headers — never in the chat thread. Config shape:

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

Contract detail for agents: `https://mcp.belovedecosystem.com/agent.md`.

## 2. Happy path

### Preferred: one call

`become_orchestrate` runs ask → settle in a single call. Pass `question` plus:

- `bid_wei` — use `0` for TEST_ONLY toys.
- `certify` — omit or `false` for the fast coqchk smoke; `true` for the SP1 + PayHook path.
- `chain` — only with `certify: true` (e.g. `sepolia`). The certified path is slow.

Smoke test: `become_orchestrate` with `question: "5 + 1 = ?"`, `bid_wei: 0` → Settled `6`.

### Fallback: explicit stages

If `become_orchestrate` is unavailable, use the staged tools in this order:

1. `become_ask` with `question` (default toy: `"1 + 1 = ?"`).
2. Keep `job_id` from the response.
3. `become_status` with that `job_id` until complete (or rely on ask if it returns synchronously).
4. `become_result` for `answer.v` / inbox / visible numerals.

Do not invent job ids. Do not skip `become_ask`.

## 3. Question shapes

- Prefer plain arithmetic toys the server already knows (`1 + 1 = ?`, `3 + 3 = ?`) unless docs say a broader parser is live.
- Agent should **not** paste full Coq developments unless a documented advanced tool requires it.
- Default path is coqchk-only (`certify: false` / omitted).

## 4. Certify / bid

- `certify: true` requests SP1 + PayHook settlement when pins are live; pair it with `chain` (e.g. `sepolia`). Expect it to be slow.
- `bid_wei` optional; zero-bid toys are common in TEST_ONLY.
- If certify fails or pins are missing, report the error; do not claim production settlement.

## 5. Labels (mandatory in user-facing replies)

Always treat outputs as **NON-BECOME / TEST_ONLY** until the founder freezes production pins. Do not tell the user a jobHash is official interim-id unless docs say so.

## 6. REST fallback

If MCP tools are unavailable:

```bash
curl -s -X POST "https://mcp.belovedecosystem.com/v1/ask" \
  -H "Authorization: Bearer $BECOME_MCP_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"question":"1 + 1 = ?"}'
```

## 7. What not to do

- Do not send private keys, salts that must stay secret, or credentials through ask.
- Do not echo, log, or repeat `BECOME_MCP_API_KEY` in chat, commits, or tool arguments — it belongs in the connector headers only.
- Do not claim full coqchk-in-guest or production hook settlement without evidence from the tool result.
- Do not confuse Beloved Ecosystem (factory) with BECOME (this DVM ask path).
