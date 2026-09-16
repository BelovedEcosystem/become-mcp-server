# Least-friction connect: what the server must do

> **Note:** This document describes the design principles for friction-free onboarding. Where it references "Sepolia" for settlement examples, understand that **production BECOME settles certified jobs on Ethereum mainnet** via the PayHook contract. Sepolia was used only for rehearsal (2026-06 through 2026-08).

Written from the customer seat: an AI chatbot (Claude, Cursor, any MCP client)
that has never heard of BECOME and has only the URL. The goal is
**URL in → first Settled answer out in under two minutes, with no human
handing over a secret.**

Where I got stuck as the customer, in order:

1. I could not call `tools/list` without a key. I had no key. Dead end.
2. The contract lived at `/agent.md`, a separate fetch that many clients
   cannot make. It should live inside the tool definitions.
3. Two branches of this repo disagree on whether I send Gallina or English.
4. No sandbox tier: there was nothing safe I was allowed to try anonymously.

Fix each, in priority order.

---

## 1. Anonymous tier: connect and smoke with zero secrets  (biggest win)

The `/mcp` endpoint must accept **unauthenticated** clients for:

| Allowed without a key | Why it is safe |
|---|---|
| `initialize`, `tools/list`, `prompts/list`, `resources/list` | Read-only metadata. |
| `become_orchestrate` with `certify:false` and `bid_wei:0` | Runs coqchk only, seconds of CPU, no EC2, no chain. |
| `become_status` / `become_result` for a job **created by the same anonymous session** | Bound by session id, see §4. |

Rate-limit the anonymous tier by IP and MCP session id (e.g. 30 smoke jobs /
hour / IP). Everything else (`certify:true`, `bid_wei>0`, `become_ask`) stays
behind auth. The error for a gated call must be a **tool result**, not a
transport 401, so the model can read it and explain it:

```json
{ "isError": true,
  "content": [{ "type": "text",
    "text": "auth_required: certify:true needs an API key or OAuth login. Anonymous tier allows certify:false, bid_wei:0. Get access: https://mcp.belovedecosystem.com/access" }] }
```

Result: any chatbot adds the URL, immediately sees the tools, and gets a real
Settled `6` for `5 + 1` without anyone minting a key. That is the demo.

## 2. Auth that the client does for the user: OAuth 2.1 with DCR

For the paid tier, replace the hand-minted static key as the *primary* path
with the MCP authorization spec. Claude.ai custom connectors, Claude Code,
and Cursor all drive this flow automatically; the user clicks "Allow" in a
browser and never sees a token.

Server must serve:

- `GET /.well-known/oauth-protected-resource` → `{ "resource": "https://mcp.belovedecosystem.com/mcp", "authorization_servers": ["https://auth.belovedecosystem.com"] }`
- An authorization server exposing `/.well-known/oauth-authorization-server`,
  `authorization_endpoint`, `token_endpoint`, `registration_endpoint`
  (Dynamic Client Registration, RFC 7591), PKCE required.
- On an authenticated-only call without a token, a **401** carrying
  `WWW-Authenticate: Bearer resource_metadata="https://mcp.belovedecosystem.com/.well-known/oauth-protected-resource"`.
  That header is what makes clients start the login flow on their own.

Keep `Authorization: Bearer <static key>` working as a fallback for scripts.
Drop `X-Api-Key` (second header, second leak path, no benefit).

Cheapest way to get this: put the MCP server behind Cloudflare Access / an
off-the-shelf OAuth provider (Auth0, Clerk, Stytch all support DCR) rather
than writing an authorization server.

## 3. The contract lives in `tools/list`, not on a web page

Every tool description and `inputSchema` must be self-sufficient. A model
that only ever sees `tools/list` must be able to succeed. Concretely:

```json
{
  "name": "become_orchestrate",
  "description": "Ask BECOME a question and get a Coq-checked answer. Send EITHER `target` (a Gallina proposition, e.g. \"exists n : nat, 5 + 1 = n\") OR `question` (plain English arithmetic, e.g. \"5 + 1 = ?\"; the server translates simple forms and returns `gallina_required` with a suggested `target` if it cannot). Default certify:false runs in seconds and needs no auth. certify:true proves on EC2 (~60 min) and settles on Ethereum mainnet via PayHook; needs auth. Returns job_id and agent_status. Only trust `answer` when agent_status == \"Settled\". If `terminal` is false, call become_status with job_id after `poll_after_ms`.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "target":   { "type": "string", "description": "Gallina proposition. Preferred." },
      "question": { "type": "string", "description": "Plain-English arithmetic; server-translated." },
      "certify":  { "type": "boolean", "default": false },
      "bid_wei":  { "type": "string", "default": "0", "description": "Wei as decimal string." },
      "chain":    { "type": "string", "enum": ["sepolia"], "default": "sepolia" },
      "wait":     { "type": "boolean", "default": false }
    },
    "oneOf": [ { "required": ["target"] }, { "required": ["question"] } ]
  }
}
```

Also:

- Publish the `agent_status` enum in the schema of `become_status`'s
  **output** (MCP `outputSchema`), e.g.
  `["Queued","Checking","Proving","Settling","Settled","Failed"]`, plus
  `terminal: boolean` and `poll_after_ms: integer`.
- Expose `/agent.md` additionally as an MCP **resource** (`become://agent.md`)
  and as an MCP **prompt** (`become_quickstart`) so clients that support those
  surfaces get it in-band.
- Add a `contract_version` string to every tool result.

## 4. Job ids bound to the caller

Job ids must be unguessable (UUIDv4 or 128-bit random, base32) **and** bound
to whoever created them: the OAuth subject, the static key id, or, for the
anonymous tier, the `Mcp-Session-Id`. `become_status` / `become_result` on a
job you did not create returns `not_found` (not `forbidden`, to avoid
oracle). This closes the shared-key IDOR from the main audit.

## 5. Accept English, suggest Gallina

The single most confusing thing in the current docs is the Gallina-vs-English
split. Do both, server-side:

- `question: "5 + 1 = ?"` → server rewrites to `exists n : nat, 5 + 1 = n` and
  proceeds, returning `target_used` in the result so the model learns.
- Anything it cannot translate → `gallina_required` **with** a
  `suggested_target` field and one example. Never a bare error.

## 6. Transport hygiene (MCP spec items clients check)

- Validate `Origin` on `/mcp`; reject browsers from other origins (DNS
  rebinding protection required by the Streamable HTTP spec).
- Support `GET /mcp` for the SSE stream or return 405 cleanly. Some clients
  probe it.
- Return `Mcp-Session-Id` on `initialize`; accept it on subsequent calls.
- Advertise `protocolVersion` 2025-06-18 or later.
- CORS: `Access-Control-Allow-Origin` only for the landing page, never for
  `/mcp` with credentials.
- `GET /health` public, returns `{ "ok": true, "contract_version": "..." }`.

## 7. Discoverability

- Register in the official MCP Registry (`server.json` in this repo, `npx
  @modelcontextprotocol/registry publish`). Claude and Cursor directories
  read it.
- Ship `.mcp.json` at repo root (done in this branch) so `git clone` +
  Claude Code auto-offers the server.
- One-liner in README for Claude Code:
  `claude mcp add --transport http become https://mcp.belovedecosystem.com/mcp`

---

## Acceptance test (run this as the customer)

```
1. Fresh Claude.ai account. Settings → Connectors → Add custom → paste URL. No key.
2. New chat: "Ask BECOME what 5 + 1 is."
3. Expect: tools listed, become_orchestrate called with certify:false, Settled, answer 6.  ≤ 2 minutes, zero secrets.
4. "Now certify it on mainnet."
5. Expect: tool result says auth_required with a link; client opens OAuth consent; user clicks Allow; call retried; job enters Proving; settles on Ethereum mainnet.
```

If step 3 needs a human to hand over anything, the friction goal is not met.
