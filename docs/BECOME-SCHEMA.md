# BECOME schema v1

Normative reference for a BECOME job across its three layers. The rationale
is in [`ORDER-SCHEMA.md`](ORDER-SCHEMA.md); the Nostr registry text is in
[`nostr-kind-5700.yaml`](nostr-kind-5700.yaml). Status: design, 2026-09-10.
Nothing here is implemented yet; the live host still uses the interim
JSON shapes in `/agent.md`.

One job, one id. The order id is `jobHash`, and it appears in every layer:

```
Nostr (NIP-90)          Ethereum (ERC-7683 + PayHook)        MCP (JSON)
kind 5700  ──x tag───►  jobHash ◄──── publicValues[0]  ◄──── job_hash
kind 7000  status       jobs(jobHash).status                  agent_status
kind 6700  settlement   fill tx                               fill_tx / receipt_url
```

## Layer 1: the order (ERC-7683 payload)

The thing the client authorizes and the resolver reads. ABI-encoded
`BecomeOrderData`; identified by the EIP-712 typehash of its type string.

```solidity
struct BecomeOrderData {
    uint16  schemaVersion;      // 1
    bytes32 jobHash;            // order id; first field of publicValues
    bytes32 specRoot;           // manifest hash; 0x0 while BecomeJobHash/v0 is used
    bytes32 acceptanceRuleHash; // hash(target FQN, permitted axioms, coq version, flags)
    address payHook;            // settlement contract; pins verifier + programVKey
    address claimant;           // prover allowed to fill; address(0) = any
    address refundTo;           // receives an expired stake
    uint256 bidWei;             // stake in wei; 0 allowed
    uint8   tier;               // 0 = smoke (coqchk only), 1 = certified (proof + chain)
    bytes32 payloadHash;        // sha256 of the encrypted request bundle (Layer 2 content)
    bytes32 payloadLocator;     // Nostr event id of the 5700 request, or 0x0
}
bytes32 constant BECOME_ORDER_DATA_TYPE_HASH = keccak256(
  "BecomeOrderData(uint16 schemaVersion,bytes32 jobHash,bytes32 specRoot,"
  "bytes32 acceptanceRuleHash,address payHook,address claimant,address refundTo,"
  "uint256 bidWei,uint8 tier,bytes32 payloadHash,bytes32 payloadLocator)"
);
```

Authorization: EIP-712 signature over this struct by the funder, or a direct
`open` transaction by the funder. `openDeadline` and `fillDeadline` are
envelope fields, not struct fields; `fillDeadline` is the PayHook `deadline`.

Job hash rule in force: `BecomeJobHash/v0 = sha256(Question.v bytes)`. Target
rule: `jobHash = H(specRoot, nonce)` once the manifest exists. Either way the
answer file is never hashed.

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

Rules: no `i`, `output` or `bid` tags. `x` equals `order.jobHash`.
`sha256(content ciphertext) == orderData.payloadHash`. The 5700 event id
becomes `payloadLocator` in the on-chain order. An order with
`claimant = address(0)` has no `p` tag and clear content.

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
| variables | `StepCaller(0)`; `PaymentRecipient` (= caller); `Witness("sp1-groth16", abi.encode(programVKey, payloadLocator), [jobHash]) -> proof`; `Witness("sp1-public-values", same) -> publicValues`; `Query(payHook.jobs(jobHash))` |
| payments[0] | native `bidWei` from payHook to `PaymentRecipient`, on step 0, delay 0 |
| assumptions | `sp1-verifier=<addr>`, `program-vkey=<bytes32>`, `tcb=T2CERT0`, `jobhash-scheme=BecomeJobHash/v0`, `network=sepolia`, `question-class=nat-sum`, `native-payment`, `exclusive-claimant=<addr>` when set |

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
`{ "name": "Beloved BECOME", "resolvers": ["<R>"], "payHook": "<addr>",
"programVKey": "<bytes32>", "orderDataType": "<typehash>", "mcp":
"https://mcp.belovedecosystem.com/mcp" }`.

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

### `Job` (output of every job tool)

```json
{
  // identity
  "schema":            "become-mcp-job/v1",        // live: "become-mcp-orchestrate/v0"
  "job_id":            "orch-45plus8-d89ad7cd",    // host-local handle; not in the order
  "job_hash":          "0x3b3208bf…869af2",        // = Layer 1 jobHash = 5700 x tag = publicValues[0]
  "owner_token":       "…",                        // only on create; session credential, never on chain
  "order_data_type":   "0x…",                      // v1: BECOME_ORDER_DATA_TYPE_HASH
  "schema_version":    1,                          // v1

  // the order, as sent (Layer 1 mirror)
  "target":            "exists n : nat, 45 + 8 = n",
  "tier":              1,                          // v1; live: "certify": true
  "bid_wei":           0,
  "chain":             "sepolia",                  // v1 adds "origin_chain_id": 11155111
  "fill_deadline":     1789086400,                 // live: "due_date"
  "open_deadline":     1789003600,                 // v1
  "claimant":          "0x…",                      // v1
  "refund_to":         "0x…",                      // v1
  "spec_root":         "0x0…",                     // v1, zero under BecomeJobHash/v0
  "acceptance_rule_hash": "0x0…",                  // v1
  "payload_hash":      "0x0…",                     // v1, zero when the request never left the host
  "payload_locator":   "0x0…",                     // v1, 5700 event id when it did

  // state (kind 7000 mirror)
  "agent_status":      "Settled",                  // Queued | Proving | Settled | Failed | Cancelled
  "terminal":          true,
  "ok":                true,
  "poll_after_ms":     0,
  "how":               "…",                        // the one next action, in words
  "code":              null,                       // on Failed: gallina_required | jobid_already_settled | wait_timeout | …
  "error":             null,
  "run_url":           "https://github.com/…/actions/runs/…",   // while Proving; Witness progress

  // settlement (Layer 3 / kind 6700 mirror)
  "certified":         true,                       // true only on tier 1 after fill
  "open_tx":           "0x…",
  "fill_tx":           "0xf80dd2c8…3860f",
  "receipt_url":       "https://sepolia.etherscan.io/tx/0xf80dd2c8…3860f",
  "payhook":           "0x…",

  // deliverable (kind 6700 content mirror) — present only when Settled
  "answer":            "53",
  "inbox":             "53",
  "answer_v":          "…",                        // become_result only

  // honesty (resolved-order assumptions mirror)
  "labels":            { "tcb": "T2CERT0", "jobhash": "BecomeJobHash/v0", "network": "sepolia", "question_class": "nat-sum" }
}
```

Rules that hold for `Job` in every tool:

- `answer`, `inbox`, `answer_v` are null unless `agent_status == "Settled"`
  and `terminal == true`; the server blanks them otherwise.
- `certified` is true only when `tier == 1` and `fill_tx` is set. A tier 0
  Settled means coqchk passed on the host, nothing more.
- `terminal` drives the client loop; `poll_after_ms` is the server's
  suggested wait and is 0 when terminal.
- `job_hash` is identical to the Layer 1 `jobHash`; `job_id` is not part of
  any layer and may be dropped by clients.
- Host paths (`job_dir`, `tarball`, `inbox_path`), operator recipes
  (`ec2_stage`), `plan`, `internal_status`, `engine`, `notes`, and
  `summary_for_model` are live-only diagnostics and are not in v1. Clients
  should not read them.

### `Order` (output of `become_encode_service_request`)

```json
{
  "order_data_type": "0x…",
  "order_data":      "0x<abi-encoded BecomeOrderData>",
  "order":           { "schemaVersion": 1, "jobHash": "0x…", "specRoot": "0x…", "acceptanceRuleHash": "0x…",
                       "payHook": "0x…", "claimant": "0x…", "refundTo": "0x…", "bidWei": "0",
                       "tier": 1, "payloadHash": "0x…", "payloadLocator": "0x…" },
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

In the anonymous tier the host is client, funder and prover, so there is no
5700 event: the host writes the same bundle to the job directory and
`payload_locator` is zero. Every job still has a Layer 1 order and a Layer 1b
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
