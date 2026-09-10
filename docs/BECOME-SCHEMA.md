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

## Layer 4: the MCP mirror

What the JSON tools return, and which schema field each is. Fields marked
"add" do not exist in the live responses yet.

| MCP field | Schema field | |
|---|---|---|
| `job_hash` | `jobHash` | same value, both layers |
| `job_id` | none | host-local handle |
| `bid_wei` | `bidWei` | |
| `certify` | `tier` | false = 0, true = 1 |
| `chain` | `originChainId` | |
| `agent_status`, `terminal` | kind 7000 `status` | same state machine |
| `fill_tx`, `receipt_url` | 6700 `settlement` | |
| `owner_token` | none | session credential; never leaves MCP |
| `run_url` | Witness progress | |
| `order_data_type`, `schema_version` | `orderDataType`, `schemaVersion` | add |
| `spec_root`, `acceptance_rule_hash` | | add, zero for now |
| `refund_to`, `open_deadline`, `fill_deadline` | | add before third-party funding |
| `payload_hash`, `payload_locator` | | add when requests leave the host |

In the anonymous tier the host is client, funder and prover, so there is no
5700 event: the host writes the same bundle to the job directory and
`payloadLocator = 0x0`. Every job still has a Layer 1 order and a Layer 1b
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
