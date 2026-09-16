---
name: become-use
description: >-
  Use this BEFORE calling BECOME MCP tools (become_orchestrate, become_status,
  become_result, become_ask). Covers Gallina target, zero-bid default, certify
  vs smoke, Proving poll rules, and Settled-only answer trust.
disable-model-invocation: false
---

# BECOME MCP — Usage Skill

Remote HTTP MCP: `https://mcp.belovedecosystem.com/mcp`  
Contract: `https://mcp.belovedecosystem.com/agent.md`  
Auth: none required. Leave the connector's auth empty (never chat-paste a key).

## 1. Connect

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

Claude custom connector: name **Beloved BECOME**, same URL, auth empty. See `docs/CLAUDE-CONNECTOR.md`.

## 2. Happy path

1. Prefer **`become_orchestrate`** with Gallina `target` (e.g. `exists n : nat, 5 + 1 = n`), `bid_wei=0`.
2. Keep `job_id` **and** `owner_token` from the response; pass both when polling.
3. If not Settled yet: poll **`become_status`** (then **`become_result`**) with `job_id` + `owner_token` while `terminal=false`. Polling early is harmless; certified proofs take ~1–2 h.
4. Trust `answer` / `inbox` **only** when `agent_status=Settled`.

Do not invent job ids. Prefer orchestrate over `become_ask` unless the user names a non-zero bid.

## 3. Ask shape

- Request details = Gallina `target` only. NL-only → `gallina_required`.
- Client knobs: `bid_wei` + optional `due_date`.
- Smoke: `certify: false`. Full cert: `certify: true` (EC2 prove; often ~60–70m for toys). Prefer `wait: false` then poll.

## 4. While Proving

Also read `how`, `run_url`/`actions_url`, `progress`/`proof_present`, `failed`/`in_progress`.  
`certify_requires_ec2`, staged/`Proving`, and quiet `missing_proof` are **normal** — keep polling.  
If `auto_generate` / `run_id` present → **poll only** (no S3 / GHA yourself).  
Do not treat `ok` or `summary_for_model` alone as fail/success.

## 5. Until Settled

Never trust `answer`, `inbox`, `answer_v`, or `visible_result` before `Settled`.

## 6. Honesty

BecomeJobHash/v0 · T2CERT0 (EC2 attestation + certificate verification; not full coqchk-in-guest) · **Production settles on Ethereum mainnet** via PayHook · Sepolia was rehearsal only (2026-06 through 2026-08) · no private keys in ask.
