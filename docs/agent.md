# BECOME MCP — agent contract

Beloved BECOME accepts a **sealed Gallina question** (`Question.v` = Context + `Definition Target`) and type-checks **the buyer's bytes**. English is a label only. Smoke does not invent the numeral (`Open` / `no_answer` without `answer_v`). `certify=true` + `bid_wei=0` stages Generate BECOME DVM Proof. Poll; not Settled yet.

MCP URL: `https://mcp.belovedecosystem.com/mcp`

## Connect (Claude, Grok, Cursor, any MCP client)

Add the MCP URL as a custom remote MCP connector (Claude: **Settings → Connectors**). Leave auth empty. Approve OAuth if offered; leave a key field blank (never paste keys into chat; there is no public demo key). Claude steps: [/CLAUDE-CONNECTOR.md](/CLAUDE-CONNECTOR.md).

## Naive path

Use MCP tools only (`become_orchestrate`). Do not call REST `/v1`. REST `POST /v1/orchestrate` is operators only — not the naive path. There is **no public demo key** and no public self-serve key mint.

Auth errors (REST or gated tools/call without a vaulted token):

| code | means |
|---|---|
| `unauthorized_missing_key` | no Bearer / X-Api-Key and no OAuth |
| `unauthorized_invalid_key` | a key was sent and not accepted |
| `oauth_required` | connect through the client connector, never paste a key into chat |
| `bid_not_allowed` | `bid_wei>0` without `allow_bid=true` on `become_orchestrate` |

Public origin is **fail closed**: unknown origins 403; tools/call without session/OAuth does not compute.

Smoke (`certify: false`) is **coqchk only** (Question.v type-check; `answer_v` for Settled). `certify: true` + `bid_wei: 0` stages the DVM guest.

If a tool answers with an auth error, connect through the client — never paste a key:

```json
{"code": "oauth_required", "how": "Add Beloved BECOME as a custom connector with https://mcp.belovedecosystem.com/mcp and approve OAuth if offered; never paste a key into chat."}
```

`initialize` is a closed MCP `InitializeResult` (`protocolVersion`, `capabilities`, `serverInfo` only — no `experimental`, no top-level `afford`/`auth`). Job status/code/how/afford **and Define-page fields** (`tools/list` inputSchema, submit gates) are a page of `schema/corpus.cspo` (`define field.*` / `define page` / `define tool.*`).

## One tool: `become_orchestrate`

Send a Gallina **target**. English is a label; the server never turns it into a target. Shape for a sum: `exists n : nat, A + B = n`.

**Smoke (seconds, free) — type-check only, no witness:**

```json
{"target": "exists n : nat, 5 + 1 = n", "bid_wei": 0, "certify": false}
```

This node does not invent the numeral on smoke. The live page is `Open` / `no_answer` until you pass `answer_v`.

**`answer_v` (copy-paste):** a full `.v` file, or a bare term which the host wraps.

```coq
Require Import Question.
Definition answer : Question.Target := ex_intro _ 5 eq_refl.
```

Bare `ex_intro _ 5 eq_refl` is accepted and wrapped. On Settled smoke the numeral is in `answer` / `inbox` (e.g. `5`). `certified` is false. `coqc` failures include `coqc_stderr`.

**Certified (`certify=true`, `bid_wei=0`) — DVM path:**

```json
{"target": "exists n : nat, 45 + 8 = n", "bid_wei": 0, "certify": true, "chain": "mainnet"}
```

Question.v type-checks. Host packs a plus witness for the DVM guest (does not rewrite Question.v) and returns **StagedAwaitingProve** or **Proving** (`run_url` when auto-dispatch is on). Zero bid is honored. Not Settled.

```json
{"ok": true, "terminal": false, "agent_status": "StagedAwaitingProve",
 "code": "certify_requires_ec2", "certified": false, "answer": null,
 "poll_after_ms": 7200000,
 "how": "Job staged for Generate BECOME DVM Proof. Poll become_status with job_id. Not Settled."}
```

`ok: true` means accepted, not Settled. Trust `answer` only when Settled and `terminal`. After PayHook fill: Settled, `certified` true, numeral in `answer`/`inbox`.

Non-plus Targets (`True`, `forall …`) return `fill_not_plus`. `a`/`b` are not client fields (`a_b_not_allowed`).

`become_result` returns the same job page plus the Coq answer source when present. `become_ask` is an alias that requires an explicit `bid_wei`; you do not need it.

## Reading any response

| Field | Rule |
|---|---|
| `ok` | Request accepted and not Failed. **Not** the same as Settled. |
| `terminal` | Stop polling when true. Drive your loop from this, not from `summary_for_model`. |
| `agent_status` | Smoke without `answer_v` is `Open`. Kernel-checked `answer_v` is `Settled`. `certify=true` is `StagedAwaitingProve` or `Proving` until the DVM guest settles. |
| `answer` / `inbox` | Trust **only** when `agent_status` is `Settled` and `terminal` is true. |
| `certified` | `true` only after DVM fill. Staging is not certified. |
| `poll_after_ms` | Top-level wait; 0 when terminal. Ignore nested poll clocks. |
| `how` | The one next action, in words. |

A `Failed` result carries `code` and `how` (for example `gallina_required` when no target was sent, `fill_not_plus` when certify Target is not a plus).

## What "certified" means (honesty)

- Smoke runs `coqc` on Question.v. It does **not** invent a plus witness. `Settled` requires client `answer_v` accepted by host `coqchk`.
- `certify=true` + `bid_wei=0` **does** enter the DVM (`schema/corpus.cspo`: `deployment node slm.fill groth16`). Poll `become_status`. Do not treat `StagedAwaitingProve` as Settled.
- Guest fill is plus/ELF (`uint32` visible). Settlement is on Ethereum mainnet PayHook (BecomeJobHash/v0). Settlement honesty = T2CERT0 (not full coqchk-in-guest). Sepolia was the rehearsal network; mainnet is live.
