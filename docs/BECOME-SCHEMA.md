# BECOME schema v1

Projections of the BECOME data model, field by field: the pins
(Deployment), the order, the Nostr transport, the chain, and the MCP view.
**The data model itself is the quad dataset in
[`SERVICE-REQUEST-QUADS.md`](SERVICE-REQUEST-QUADS.md)** (owner decision
2026-09-10: all BECOME schema are quads). Every struct, tag and JSON field
below is derived from a graph and a predicate there; if a field is wanted
here, it is added as a predicate first. Rationale for the ERC-7683 choice is
in [`ORDER-SCHEMA.md`](ORDER-SCHEMA.md); the Nostr registry text is in
[`nostr-kind-5700.yaml`](nostr-kind-5700.yaml). Status: design. Nothing here
is implemented yet; the live host still uses the interim JSON shapes in
`/agent.md`.

One job, one id. The order id is `jobHash`, and it appears in every layer:

```
Nostr (NIP-90)          Ethereum (ERC-7683 + PayHook)        MCP (JSON)
kind 5700  ──x tag───►  jobHash ◄──── publicValues[0]  ◄──── job_hash
kind 7000  status       jobs(jobHash).status                  agent_status
kind 6700  settlement   fill tx                               fill_tx / receipt_url
```

## Layer 0: `Deployment` (the pins)

Everything a job depends on that is not the job. One entry is live per
chain; superseded entries stay so old receipts still resolve. A `Job`
references its deployment by `payhook` and nothing else. Source of truth:
one file in the server repo (`config/deployment.json`, to be created from
the operator log `docs/toy-prod-pins.json`). The docs, the NIP-89
announcement and the MCP server info are derived from it, never typed twice.

Live entry, 2026-09-10:

```json
{
  "id":               "sepolia-t2cert0-docker-2026-09-07",
  "status":           "live",
  "chain_id":         11155111,
  "network":          "sepolia",
  "payhook":          "0x8FA889E7C6d9C74EA5ee2b4BFaf4DB8cE8e964B8",
  "payhook_deploy_tx":"0x804c4d91a386ca60d81109473626ae16c46949d65110066963b8500340933221",
  "verifier":         "0xb69f2584CBcFf99a58C4e7002E8b89Af54a6f4e2",
  "program_vkey":     "0x00116101c20b687297ae11e1fea4bd4bd003ef3320da147a3db1f65bc81f388c",
  "guest":            "T2CERT0",
  "elf_sha256":       "854cd998ab847cd2d85f3c954e0449e7c64046909e5ce98d8b6aa5ff0a31cbb9",
  "elf_build":        { "how": "docker", "sp1_sdk": "6.6.0",
                        "ci": "https://github.com/BelovedEcosystem/become-sp1-skeleton/actions/runs/34074962548",
                        "commit": "f26f129fd258cf5bcaf9b494bd5da32ff2e08017" },
  "claimant":         "0x11bD4139BaAfcd9F19DA44D924b499AfD29032Eb",
  "resolver":         null,
  "order_data_type":  null,
  "jobhash_scheme":   "BecomeJobHash/v0",
  "accepted_assumptions": [
    { "id": "sha256_inj", "statement": "forall a b, SHA256 a = SHA256 b -> a = b",
      "role": "T2 digest => fixture / T2_certificate_soundness", "accepted_by": "Chinedu", "accepted_at": "2026-09-07" }
  ],
  "deployed_at":      "2026-09-07",
  "signed_off_by":    "Chinedu"
}
```

`resolver` and `order_data_type` are null until the resolver contract exists
and the type string is frozen; every other value is live today. Superseded
entries (PayHooks `0x0c85…d5f7`, `0x3adb…DDE7`, `0x66D0…eba1`,
`0x7076…7199` and their keys) move under `"status": "superseded"` with the
same shape.

The resolved-order assumptions (Layer 1b) are this object rendered as
name/value pairs; the NIP-89 announcement (Layer 2b) is this object minus
the build details; the MCP `labels` field is its `guest`, `jobhash_scheme`,
`network` and `question_class`.

## Layer 1: the order (ERC-7683 payload)

The thing the client authorizes and the resolver reads. ABI-encoded
`BecomeOrderData`; identified by the EIP-712 typehash of its type string.

```solidity
struct BecomeOrderData {
    uint16  schemaVersion;      // 1
    bytes32 jobHash;            // order id = sha256(specRoot || nonce); first field of publicValues
    bytes32 specRoot;           // sha256 of the canonical client graphs (question incl. manifest)
    bytes32 acceptanceRuleHash; // sha256 of canonical sow + manifest + sli + slm graphs
    address payHook;            // settlement contract; pins verifier + programVKey
    address claimant;           // prover allowed to fill; address(0) = any
    address funder;             // who locks the stake (client, or a relayer on its behalf)
    address refundTo;           // receives an expired stake
    uint256 bidWei;             // stake in wei; 0 allowed
    uint8   tier;               // 0 = smoke (unattested host check), 1 = certified (proof + chain)
}
bytes32 constant BECOME_ORDER_DATA_TYPE_HASH = keccak256(
  "BecomeOrderData(uint16 schemaVersion,bytes32 jobHash,bytes32 specRoot,"
  "bytes32 acceptanceRuleHash,address payHook,address claimant,address funder,"
  "address refundTo,uint256 bidWei,uint8 tier)"
);
```

Authorization: EIP-712 signature over this struct by the funder, or a direct
`open` transaction by the funder. `openDeadline` and `fillDeadline` are
envelope fields, not struct fields; `fillDeadline` is the PayHook `deadline`.

Job hash rule in force: `BecomeJobHash/v0 = sha256(Question.v bytes)`. Target
rule (quads v2): `specRoot = sha256(canonical client graphs)`, `jobHash =
sha256(specRoot || nonce)`. Either way the answer file is never hashed. The
former `payloadHash` / `payloadLocator` fields were removed: they depended
on the published event and could not be inside the hash.

## Layer 2: the request on Nostr (kind 5700)

```json
{
  "kind": 5700,
  "pubkey": "<client pubkey, may be ephemeral>",
  "created_at": 1789000000,
  "tags": [
    ["p", "<BECOME pubkey hex>"],
    ["R", "<resolver as ERC-7930 interoperable address; carries chain id 11155111>"],
    ["x", "0x3b3208bf11f9c30a2e5e29016d333eb6e72372a1f68464224be80faf1c869af2"],
    ["expiration", "1789086400"],
    ["relays", "wss://buzz.belovedecosystem.com"]
  ],
  "content": "<NIP-44 v2 ciphertext to the p key>"
}
```

Decrypted `content` is a JSON bundle:

```json
{
  "schema": "become-request/v1",
  "order": {
    "orderDataType": "<BECOME_ORDER_DATA_TYPE_HASH>",
    "orderData": "0x<abi-encoded BecomeOrderData>",
    "openDeadline": 1789003600,
    "fillDeadline": 1789086400,
    "originChainId": 11155111,
    "nonce": "0x<32 bytes>"
  },
  "files": { "Question.v": "<source>", "manifest.json": "<optional>" },
  "target": "exists n : nat, 45 + 8 = n",
  "acceptance": { "coq": "8.x", "flags": [], "permitted_axioms": [] }
}
```

Rules: no `i`, `output` or `bid` tags. `x` equals `envelope.jobHash`, and
the content's `specRoot` must recompute from the client graphs it carries.
The 5700 event id is recorded by the provider in `become:request event`;
it is not part of the order. An order with `claimant = address(0)` has no
`p` tag and clear content.

Intake, in order, silent on failure: addressed and encrypted; PayHook shows
`jobs(jobHash)` open, claimant BECOME, `amount >= bidWei`, `deadline` ahead;
decrypt, recompute `jobHash` from `files`, match `x`; then resolve.

## Layer 3: the chain (PayHook)

Live ABI on Sepolia:

```
open(bytes32 jobId, address claimant, uint64 deadline) payable   // value = bidWei
fill(bytes32 jobId, bytes publicValues, bytes proof)
jobs(bytes32) -> (address client, address claimant, uint256 amount, uint64 deadline, uint8 status)
   status: 0 none, 1 open, 2 settled
```

`publicValues = abi.encode(bytes32 jobHash, ...)`. The hook checks
`publicValues[0] == jobId`, `msg.sender == claimant`, `block.timestamp <=
deadline`, `status == 1`, then `verifier.verifyProof(programVKey,
publicValues, proof)`, then pays `amount` to `msg.sender` in the same
transaction. Verifier address and `programVKey` are immutables of the hook.

## Layer 1b: the resolved order (what a prover reads)

`IResolver.resolve(orderData)` per ERC-7683 (2026 form):

| Part | Value |
|---|---|
| steps[0] | `Call` payHook.`fill(bytes32,bytes,bytes)` with [`jobHash`, var `publicValues`, var `proof`]; `NeedsVariable(proof)`, `NeedsVariable(publicValues)`, `TimingBounds(block.timestamp, 0, fillDeadline)`, `SpendsGas(est)`, `RevertPolicy(abort, "settled")`, `RevertPolicy(abort, "deadline")`, `RevertPolicy(abort, "bad proof")` |
| variables | `StepCaller(0)`; `PaymentRecipient` (= caller); `Witness("sp1-groth16", abi.encode(programVKey, developmentHash), [jobHash]) -> proof`; `Witness("sp1-public-values", same) -> publicValues`; `Query(payHook.jobs(jobHash))` |
| payments[0] | native `bidWei` from payHook to `PaymentRecipient`, on step 0, delay 0 |
| assumptions | `sp1-verifier=0xb69f2584CBcFf99a58C4e7002E8b89Af54a6f4e2`, `program-vkey=0x00116101c20b687297ae11e1fea4bd4bd003ef3320da147a3db1f65bc81f388c`, `tcb=T2CERT0`, `jobhash-scheme=BecomeJobHash/v0`, `network=sepolia`, `question-class=nat-sum`, `native-payment`, `exclusive-claimant=<addr>` when set |

## Layer 2b: progress and delivery on Nostr

Feedback, kind 7000, one per transition, `e` = request, `p` = customer:

| `status` tag | BECOME state | extra |
|---|---|---|
| `processing` | Queued, Proving | `["status","processing","Proving"]`, optional `run_url` in content |
| `success` | Settled | `["amount", "<bidWei>", "<fill tx hash>"]` |
| `error` | Failed | `["status","error","<code>"]` with the MCP `code` |
| `expired` | deadline passed unfilled (NIP-69 word) | |
| `canceled` | client cancel honoured (NIP-69 word) | |

`payment-required` is never sent.

Result, kind 6700:

```json
{
  "kind": 6700,
  "tags": [
    ["e", "<5700 event id>"],
    ["p", "<client pubkey>"],
    ["x", "0x3b3208bf11f9c30a2e5e29016d333eb6e72372a1f68464224be80faf1c869af2"],
    ["settlement", "0xf80dd2c8300717ad8d5f14679a58c9b2d44423a812902fc30a2c7bec67c3860f"]
  ],
  "content": "<NIP-44 v2 ciphertext to the client>"
}
```

Decrypted content: `{ "schema": "become-result/v1", "answer.v": "<source>",
"answer.vo": "<base64>", "visible": "53", "certified": true,
"receipt_url": "https://sepolia.etherscan.io/tx/0xf80d…" }`.

Discovery, kind 31990 (NIP-89), published by BECOME: `["k","5700"]`, content
`{ "name": "Beloved BECOME", "resolvers": [], "payHook":
"0x8FA889E7C6d9C74EA5ee2b4BFaf4DB8cE8e964B8", "programVKey":
"0x00116101c20b687297ae11e1fea4bd4bd003ef3320da147a3db1f65bc81f388c",
"orderDataType": null, "chainId": 11155111, "mcp":
"https://mcp.belovedecosystem.com/mcp" }` (derived from the live
`Deployment`; `resolvers` fills when one is deployed).

## Layer 4: MCP

The MCP surface is a JSON view of the same job. One object, `Job`, is
returned by every job tool; the create tools also accept a `Request`. Field
names are snake_case mirrors of Layers 1 to 3. "live" marks what the host
returns today (`become-mcp-orchestrate/v0`); "v1" marks fields the schema adds.

### Tools

| Tool | Input | Returns | Anonymous | Notes |
|---|---|---|---|---|
| `become_orchestrate` | `Request` | `Job` | yes, creates bind ownership | the default ask |
| `become_status` | `job_id`, `owner_token` | `Job` | yes, with token | reads never rate-limited |
| `become_result` | `job_id`, `owner_token` | `Job` + `answer_v` | yes, with token | same object, adds the Coq source |
| `become_cancel` | `job_id`, `owner_token` | `Job` (terminal) | yes, with token | best effort; never un-settles |
| `become_ask` | `Request` with `bid_wei` required | `Job` | yes | advanced alias; will be retired once `Request` carries the order |
| `become_encode_service_request` | `job_id` | `Order` (Layer 1) + EIP-712 typed data | yes | the only tool that emits the ERC-7683 payload; unsigned, no broadcast |
| `become_spec_check`, `become_engine_query` | | | | operator-only; not part of the data model |

### `Request` (input to create)

```json
{
  "target":        "exists n : nat, 45 + 8 = n",   // Gallina Prop; exactly one of target | question_v
  "question_v":    "<full Question.v>",            // alternative to target
  "request":       "45 plus 8",                    // optional English gloss, echoed, never the ask
  "certify":       true,                           // live: bool  -> v1 tier 0|1
  "chain":         "sepolia",                      // live: "sepolia" | "anvil" -> v1 origin_chain_id
  "bid_wei":       0,                              // live -> v1 bid_wei (same)
  "due_date":      1789086400,                     // live alias fill_deadline -> v1 fill_deadline
  "wait":          false,                          // transport only: block up to timeout_s
  "timeout_s":     120,                            // transport only
  "refund_to":     "0x…",                          // v1; defaults to the funder
  "claimant":      "0x…",                          // v1; defaults to BECOME; address(0) = open
  "open_deadline": 1789003600                      // v1
}
```

`a`, `b` (operand overrides) and `allow_bid` are live-only and drop in v1:
the target is the ask, and the bid policy is host acceptance logic, not a
client flag.


### Deadlines

`due_date` is one value with five names. Only the PayHook enforces it.

| Layer | Name | Role |
|---|---|---|
| `Request` | `due_date` (live alias) -> `fill_deadline` (v1) | what the client asks for; unix seconds or ISO-8601 |
| ERC-7683 envelope | `fillDeadline` | what the client signs |
| kind 5700 | `expiration` tag | when relays may drop the request (NIP-40) |
| PayHook | `open(…, deadline)`, `jobs().deadline` | enforced: `fill` reverts after it; stake reclaimable by `refundTo` |
| `Job` | `fill_deadline` | echoed, never null (server default fills it) |

On expiry: kind 7000 `expired`; `Job.agent_status = "Failed"`, `code =
"expired"`, `terminal = true`. `open_deadline` is separate: the last moment
the stake may be locked, relevant only when a relayer or fiat checkout
opens the order on the client's behalf. In the anonymous tier the host funds
at creation and `open_deadline` is creation plus a few minutes.

Live behaviour (2026-09-10): `due_date` is parsed and echoed, then ignored;
`payhook_open_fill_sepolia` opens with `now + 24h` and fills in the same
call, because the interim host opens on chain only after the proof exists.
v1 locks the stake first, so the client's value (or the server default,
generous for tier 1) is the one written on chain.

### `Job` (output of every job tool): closed schema

Projection of the `become:order`, `become:status`, `become:settlement`,
`become:result` and `become:deployment` graphs; each field is the
snake_case of one predicate.

`Job` is a **closed** object: the fields below and no others
(`additionalProperties: false`). A client can validate every response, and
the server cannot leak by accident. Operator-only data travels under one
`diagnostics` key that is present only when an operator key was sent.
Owner decision 2026-09-10.

```json
{
  "schema":            "become-mcp-job/v1",
  "job_id":            "orch-45plus8-d89ad7cd",
  "job_hash":          "0x3b3208bf11f9c30a2e5e29016d333eb6e72372a1f68464224be80faf1c869af2",
  "owner_token":       "…",                        // create only
  "payhook":           "0x8FA889E7C6d9C74EA5ee2b4BFaf4DB8cE8e964B8",   // -> Deployment

  "target":            "exists n : nat, 45 + 8 = n",
  "gloss":             "45 plus 8",                // the English label, if sent
  "tier":              1,
  "bid_wei":           0,
  "chain_id":          11155111,
  "open_deadline":     1789003600,
  "fill_deadline":     1789086400,
  "claimant":          "0x11bD4139BaAfcd9F19DA44D924b499AfD29032Eb",
  "refund_to":         "0x11bD4139BaAfcd9F19DA44D924b499AfD29032Eb",
  "spec_root":         "0x0000000000000000000000000000000000000000000000000000000000000000",
  "acceptance_rule_hash": "0x0000000000000000000000000000000000000000000000000000000000000000",
  "funder":            "0x11bD4139BaAfcd9F19DA44D924b499AfD29032Eb",
  "request_event":     null,                       // 5700 event id when the request came over Nostr; from become:request

  "agent_status":      "Settled",                  // Queued | Proving | Settled | Failed | Cancelled
  "terminal":          true,
  "ok":                true,
  "poll_after_ms":     0,
  "how":               "Done. answer is trustworthy.",
  "code":              null,
  "error":             null,
  "progress":          { "run_url": null, "proof_present": true, "stage": "settled" },
  "late":              true,                       // settled after due_date (soft); from become:status
  "due_date":          1789050000,                 // soft; from become:order dueDate

  "certified":         true,
  "open_tx":           "0x…",
  "fill_tx":           "0xf80dd2c8300717ad8d5f14679a58c9b2d44423a812902fc30a2c7bec67c3860f",
  "receipt_url":       "https://sepolia.etherscan.io/tx/0xf80dd2c8300717ad8d5f14679a58c9b2d44423a812902fc30a2c7bec67c3860f",

  "answer":            "53",
  "inbox":             "53",
  "answer_v":          "…",                        // become_result only; null elsewhere

  "labels":            { "tcb": "T2CERT0", "jobhash_scheme": "BecomeJobHash/v0",
                         "network": "sepolia", "question_class": "nat-sum" },

  "diagnostics":       null                        // object only when an operator key was presented
}
```

Field count: 38 (quads v2: `payload_hash`/`payload_locator` replaced by `funder`/`request_event`). Every field is always present (null when not applicable),
so clients never branch on absence.

Rules that hold for `Job` in every tool:

- `answer`, `inbox`, `answer_v` are null unless `agent_status == "Settled"`
  and `terminal == true`; the server blanks them otherwise.
- `certified` is true only when `tier == 1` and `fill_tx` is set. A tier 0
  Settled means coqchk passed on the host, nothing more.
- `terminal` drives the client loop; `how` is the one next action;
  `poll_after_ms` is the server's suggested wait and is 0 when terminal.
  There is no prose summary field.
- `job_hash` is identical to the Layer 1 `jobHash`; `job_id` is a host
  handle and may be dropped by clients.
- `labels` is derived from the `Deployment` named by `payhook`; it is never
  free text.

Live-to-v1 field disposition (from the `become-mcp-orchestrate/v0` payload):

| Disposition | Live fields |
|---|---|
| kept, same name | `job_id`, `job_hash`, `owner_token`, `target`, `bid_wei`, `agent_status`, `terminal`, `ok`, `poll_after_ms`, `how`, `code`, `error`, `certified`, `open_tx`, `fill_tx`, `receipt_url`, `payhook`, `answer`, `inbox`, `answer_v`, `labels` |
| kept, renamed | `certify` -> `tier`; `chain` -> `chain_id`; `due_date` -> `fill_deadline`; `question`/`request` -> `gloss`; `run_url`, `actions_url`, `proof_present` -> `progress`; `honesty` merged into `labels` |
| operator only, under `diagnostics` | `plan`, `notes`, `internal_status`, `engine`, `ollama_unloaded`, `certified_path`, `bid_honored_zero`, `allow_bid`, `wait`, `timeout_s`, `a`, `b`, `visible_expected`, `inbox_visible`, `role`, `notary`, `interim_job_id` |
| never to any client | `job_dir`, `tarball`, `inbox_path`, `ec2_stage` (staging recipe, S3 keys, command lines) |
| dropped | `summary_for_model` (bots drive off `terminal` and `how`); `status` (duplicate of `agent_status`); `visible` (duplicate of `answer`) |

### `Order` (output of `become_encode_service_request`)

```json
{
  "order_data_type": "0x…",
  "order_data":      "0x<abi-encoded BecomeOrderData>",
  "order":           { "schemaVersion": 1, "jobHash": "0x…", "specRoot": "0x…", "acceptanceRuleHash": "0x…",
                       "payHook": "0x…", "claimant": "0x…", "funder": "0x…", "refundTo": "0x…",
                       "bidWei": "0", "tier": 1 },
  "envelope":        { "originSettler": "0x…", "user": "0x…", "nonce": "0x…", "originChainId": 11155111,
                       "openDeadline": 1789003600, "fillDeadline": 1789086400 },
  "eip712":          { "domain": {…}, "types": {…}, "primaryType": "BecomeOrderData", "message": {…} },
  "signed":          false
}
```

Live today this tool emits `BecomeServiceRequest/v0`, a JSON blob with
`programVKey` and `sp1Verifier` inside it. v1 replaces that with the ABI
struct above; the verifier and key move to the PayHook.

### One model, four views

| Concept | Layer 1 order | Kind 5700 / 6700 / 7000 | PayHook | MCP `Job` |
|---|---|---|---|---|
| identity | `jobHash` | `x` tag | `jobId` | `job_hash` |
| who may fill | `claimant` | `p` tag | `jobs().claimant` | `claimant` |
| stake | `bidWei` | (in payload) | `jobs().amount` | `bid_wei` |
| deadline | envelope `fillDeadline` | `expiration` | `jobs().deadline` | `fill_deadline` |
| mode | `tier` | (in payload) | n/a | `tier` |
| state | resolved-order step status | 7000 `status` | `jobs().status` | `agent_status`, `terminal` |
| settlement | payment on step 0 | 6700 `settlement` | `fill` tx | `fill_tx`, `receipt_url` |
| deliverable | n/a | 6700 content | n/a | `answer`, `answer_v` |
| honesty | assumptions | 31990 content | immutables | `labels` |
| pins | `payHook` (names the deployment) | `R` tag | the contract itself | `payhook` -> `Deployment` |

In the anonymous tier the host is client, funder and prover, so there is no
5700 event: the host writes the same bundle to the job directory and
`request_event` is null. Every job still has a Layer 1 order and a Layer 1b
resolution.

## Invariants

1. `jobHash` is computed from the question only; the answer is never hashed.
2. `jobHash` is the first public value and is checked on chain.
3. The stake is locked before the request exists.
4. Verifier and `programVKey` are hook immutables; the order names only the hook.
5. Settlement is `fill` and only `fill`.
6. Every abort path is a named `RevertPolicy` in the resolved order.
7. Honesty labels are named assumptions, not prose.
8. Encrypted content is the input; clear tags carry routing and money only.
9. An order with no claimant is clear; an order with a claimant is encrypted to it.
10. Smoke and certified are the same schema with `tier` 0 or 1.
