# BECOME order schema, grounded in ERC-7683

Status: design note, 2026-09-10. Nothing here is implemented. It fixes the
vocabulary and the field set so that later code has one shape to follow.
Decision recorded in §8: BECOME standardizes on the 2026 resolver-centric
form; §9 maps the Nostr transport.

## 1. What ERC-7683 is (two versions)

ERC-7683 ("Cross Chain Intents") is an Ethereum ERC by Mark Toda (Uniswap),
Matt Rice and Nick Pai (Across). It exists in two materially different forms,
and any design that cites it has to say which one it means.

**7683-2024 (the one that shipped).** Created 2024-04-11, extended 2024-10-30,
last touched 2025-01-08. This is what Across, UniswapX and Eco deployed. It
defines:

- An outer envelope `GaslessCrossChainOrder` (signed off-chain by the user)
  and a lighter `OnchainCrossChainOrder` (sent by the user in a transaction).
  Envelope fields: `originSettler`, `user`, `nonce`, `originChainId`,
  `openDeadline`, `fillDeadline`, `orderDataType`, `orderData`.
- `orderDataType` is an **EIP-712 typehash** of the protocol's own sub-struct.
  `orderData` is that sub-struct, ABI-encoded, opaque to the standard.
- `IOriginSettler.open` / `openFor` (escrow the input, emit `Open`),
  `resolve` / `resolveFor` (a view that turns the opaque order into a
  `ResolvedCrossChainOrder` a filler can read without knowing the sub-struct),
  and `IDestinationSettler.fill(orderId, originData, fillerData)`.
- `ResolvedCrossChainOrder`: `user`, `originChainId`, `openDeadline`,
  `fillDeadline`, `orderId`, `maxSpent[]` (cap on what the filler gives),
  `minReceived[]` (floor on what the filler gets), `fillInstructions[]`.
- Foreign addresses are `bytes32`, not `address`. Permit2 is recommended,
  not required. Security of settlement is explicitly left to the settler.

Across's reference implementation is the cleanest example of the pattern:
`AcrossOrderData(address inputToken, uint256 inputAmount, address outputToken,
uint256 outputAmount, uint32 destinationChainId, address recipient, address
exclusiveRelayer, uint32 exclusivityPeriod, bytes message)`, with
`ACROSS_ORDER_DATA_TYPE_HASH = keccak256(that type string)` and a Permit2
witness type that nests the envelope and the sub-struct.

**7683-2026 (the redesign, Draft).** On 2026-05-13 the ERC text on
`ethereum/ERCs` master was replaced ("Redesign around resolvers", Francisco
Giordano, with the original authors). It keeps the name and number but drops
the envelope, the settler interfaces and `ResolvedCrossChainOrder`. Instead a
protocol exposes an opaque `payload` and an `IResolver.resolve(bytes)` view
that returns `ResolvedOrder { bytes[] steps; bytes[] variables; bytes[]
payments; Assumption[] assumptions; }`. Steps are `Call`s with attributes
(`NeedsStep`, `NeedsVariable`, `SpendsERC20`, `SpendsGas`, `RevertPolicy`,
`TimingBounds`); variables carry roles (`PaymentRecipient`, `PaymentChain`,
`StepCaller`, `ExecutionOutput`, `Witness`, `Query`, `QueryEvents`); payments
are ERC-20 transfers tied to a step. Addresses use ERC-7930 interoperable
addresses. The stated reason for the rewrite: the 2024 envelope was "only
superficially standardized", since solvers still needed per-protocol code.

Which to ground on: the 2024 form is what wallets, relayers and the
deployed settlers understand today, and its discipline (typed sub-struct,
typehash, resolve view, order id, two deadlines) is exactly what BECOME
lacks. The 2026 form is where the standard is heading, and its `assumptions`
list is a good fit for BECOME's honesty labels. This note grounds the
**data** on the 2024 sub-struct pattern and keeps the **naming** compatible
with a 2026 resolver, so neither path is closed.

## 2. Role mapping

| ERC-7683 term | BECOME today | BECOME target |
|---|---|---|
| user | the host wallet (it funds itself) | the client (a person, an agent, or a fiat relayer) |
| filler / solver | BECOME | BECOME, later any prover |
| originSettler | PayHook on Sepolia | PayHook (one per guest + verifier pair) |
| open | `open(jobId, claimant, deadline)` with `value = bid` | same, callable by the user or by a relayer `openFor` |
| fill | `fill(jobId, publicValues, proof)` | same; the "delivery" is a verified proof, not tokens |
| Open event | none | emit `Open(orderId, resolved)` from the PayHook |
| maxSpent | the bid | the bid (cap the user locks) |
| minReceived | the bid | the bid minus nothing (BECOME receives the stake) |
| destination chain | same chain | same chain; cross-chain is out of scope |
| orderId | `BecomeJobHash/v0` | `jobHash`, see §4 |

Two things are different from a token swap and should stay explicit:

- There is one chain and one leg. `fillInstructions` collapses to "call
  `fill` on this PayHook with a proof whose first public value is `jobHash`".
- The filler's "output" is a proof, so `maxSpent` on the filler side is zero
  tokens. What the filler spends is compute. That is fine under the standard;
  Across's `maxSpent` is likewise "a cap on filler liabilities".

## 3. `BecomeOrderData` v1

The sub-struct that goes in `orderData`. The EIP-712 type string is the
schema; `orderDataType = keccak256(BECOME_ORDER_DATA_TYPE)`.

```solidity
bytes constant BECOME_ORDER_DATA_TYPE = abi.encodePacked(
    "BecomeOrderData(",
    "uint16 schemaVersion,",       // 1
    "bytes32 jobHash,",            // names the question, first field of publicValues
    "bytes32 specRoot,",           // hash of the canonical manifest (0x0 while v0 hashing is used)
    "bytes32 acceptanceRuleHash,", // hash of (target FQN, permitted axiom set, coq version, flags)
    "address payHook,",            // the settler; pins verifier + programVKey by construction
    "address claimant,",           // BECOME's fill wallet; address(0) = open to any prover
    "address refundTo,",           // where an expired stake returns; never the relayer
    "uint256 bidWei,",             // the stake; 0 is legal (anonymous tier)
    "uint8 tier,",                 // 0 = smoke (no chain), 1 = certified
    "bytes32 payloadHash,"         // sha256 of the encrypted request bundle
    "bytes32 payloadLocator)"      // nostr event id / CID / S3 key hash; 0x0 if inline
);
bytes32 constant BECOME_ORDER_DATA_TYPE_HASH = keccak256(BECOME_ORDER_DATA_TYPE);
```

Field notes, and why each is in the order rather than somewhere else:

- **schemaVersion** first so the PayHook can refuse a struct it does not
  decode. The typehash already changes with the type string; the explicit
  version is for humans and for the JSON mirror.
- **jobHash** is the order id. It must equal the first `bytes32` of the
  guest's committed `publicValues`; that is the existing on-chain check and
  the one invariant the whole design rests on.
- **specRoot** and **acceptanceRuleHash** are the two halves of "the question
  is the tree plus the acceptance rule". Today `jobHash = sha256(Question.v)`
  (BecomeJobHash/v0) and there is no manifest, so `specRoot` is zero and
  `acceptanceRuleHash` covers only the pinned Coq version and flags. When the
  spec's manifest lands, `jobHash = H(specRoot, salt)` and both fields fill.
  Carrying them from v1 means the field set does not change at that point.
- **payHook** replaces `sp1Verifier` and `programVKey` in the order. Those two
  are immutables of the hook, and a different guest is a different hook.
  Letting an order name a verifier lets it name a weak one.
- **claimant** mirrors Across's `exclusiveRelayer`. `address(0)` is how a
  second prover appears later without a schema change.
- **refundTo** is the field the fiat path needs. A card relayer opens the
  order; the stake must not fall back to the relayer's treasury on expiry.
- **bidWei** is the stake and is what the wallet displays. Token is native
  ETH on Sepolia today; an ERC-20 variant is a v2 field, not a v1 one.
- **tier** makes the smoke path an order too, so one schema describes both
  paths and the client can tell from the order what "Settled" will mean.
- **payloadHash / payloadLocator**: the development, salt and target travel
  encrypted off-chain, as in the spec. The order carries only a commitment
  to that bundle and a pointer. `bytes32` rather than a `string` URI keeps
  the struct static and the hash cheap to verify.

Deliberately absent: `programVKey`, `sp1Verifier` (pinned in the hook), the
target text (in the payload), the deadline (in the envelope), `chainId` (in
the envelope), any secret.

## 4. Envelope

The envelope is the 2024 `GaslessCrossChainOrder` unchanged:

| Field | BECOME meaning |
|---|---|
| originSettler | the PayHook address |
| user | the funder; the host wallet in the anonymous tier |
| nonce | fresh per order; also the salt input once `H(specRoot, salt)` is used |
| originChainId | 11155111 (Sepolia) today |
| openDeadline | last time the stake may be locked |
| fillDeadline | last time `fill` may land; the PayHook's `deadline` |
| orderDataType | `BECOME_ORDER_DATA_TYPE_HASH` |
| orderData | ABI-encoded `BecomeOrderData` |

`orderId = jobHash`. The two deadlines are separate on purpose: an order the
client signed but nobody funded is a different failure from a funded job the
prover missed. Today the host collapses both, and the stuck-job finding in the
audit is a symptom of not having the second one.

## 5. Resolved form (what a prover reads)

`resolve(order)` returns the 2024 `ResolvedCrossChainOrder` with:

- `orderId = jobHash`
- `maxSpent = [ Output{ token: native, amount: bidWei, recipient: claimant, chainId } ]`
- `minReceived = [ same ]`
- `fillInstructions = [ FillInstruction{ destinationChainId: same,
  destinationSettler: payHook, originData: abi.encode(jobHash, tier,
  payloadHash, payloadLocator) } ]`

Under the 2026 resolver form the same order resolves to one `Call` step
(`payHook.fill(jobHash, publicValues, proof)`) with a `TimingBounds` on
`fillDeadline`, one native payment on that step to a `PaymentRecipient`
variable, and named assumptions. The assumptions are where BECOME's honesty
labels belong, as data rather than prose:

```
sp1-verifier:      <address>     (fixed implementation, not the gateway)
program-vkey:      <bytes32>
tcb:               T2CERT0       (guest checker verdict, not Coq-in-guest)
jobhash-scheme:    BecomeJobHash/v0
network:           sepolia       (test network)
question-class:    nat-sum       (answer known before proof)
```

## 6. Public values

Unchanged and restated because everything else hangs on it: the guest commits
`abi.encode(bytes32 jobHash, ...)` and the hook checks
`abi.decode(publicValues)[0] == jobs[jobHash]`. `jobHash` first, always. Any
future public value (visible numerals, tier, acceptanceRuleHash) is appended,
never prepended.

## 7. JSON mirror (what the MCP returns)

The MCP responses already carry most of this under other names. The mirror
below is the mapping, so a client that reads the JSON and a wallet that reads
the typed data see the same order.

| JSON today | Schema field | Note |
|---|---|---|
| `job_hash` | `jobHash` | same value |
| `job_id` (`orch-...`) | none | host-local handle; not part of the order |
| `bid_wei` | `bidWei` | |
| `certify` | `tier` | false = 0, true = 1 |
| `chain` | `originChainId` | string today, id in the order |
| `fill_tx`, `receipt_url` | none | settlement evidence, not order fields |
| `owner_token` | none | MCP session credential; never on chain |
| `run_url` | none | filler-side progress; a `Witness` variable in 2026 terms |
| (missing) | `orderDataType`, `schemaVersion` | add so clients can detect the shape |
| (missing) | `specRoot`, `acceptanceRuleHash` | zero until the manifest exists |
| (missing) | `refundTo`, `openDeadline`, `fillDeadline` | needed before any third party funds a job |
| (missing) | `payloadHash`, `payloadLocator` | needed when the request stops being a bare target string |

The existing advanced tool `become_encode_service_request` is the natural
producer of the ABI blob and the EIP-712 typed data; it is the only tool that
would change, and it is not on the default path.

## 8. Decision: standardize on the 2026 form

Owner decision 2026-09-10: BECOME DVM requests standardize on the 2026
resolver-centric ERC-7683. Concretely:

- **The payload is `BecomeOrderData` v1**, ABI-encoded (§3). The 2026 form
  does not constrain payload shape; the resolver defines it. Nothing in §3
  changes.
- **BECOME publishes a resolver.** `IResolver.resolve(payload)` returns the
  `ResolvedOrder` below. Until a resolver contract is deployed, the same
  function is provided off-chain by the host and documented as the reference;
  the on-chain one must return byte-identical output.
- **Client authorization stays EIP-712 over `BecomeOrderData`**, used for
  `open` / `openFor` on the PayHook. The 2026 draft defines no signing, so
  this layer is ours either way.
- **The anonymous MCP tier produces the same order.** When the host is both
  client and prover there is no Nostr hop, but every job still gets a
  canonical order and a resolved form, so a job created by tool call, by
  Nostr event, or by on-chain `open` is the same object with the same id.

### Resolved order for a BECOME job

| Part | Content |
|---|---|
| steps[0] | `Call` target = PayHook, selector = `fill(bytes32,bytes,bytes)`, arguments = [`jobHash`, var publicValues, var proof]; attributes `NeedsVariable(proof)`, `NeedsVariable(publicValues)`, `TimingBounds(block.timestamp, 0, fillDeadline)`, `SpendsGas(estimate)`, `RevertPolicy(abort, "already settled")`, `RevertPolicy(abort, "deadline")` |
| variables | `StepCaller(0)` (must equal `claimant` when non-zero; stated as an assumption), `PaymentRecipient` (the stake goes to the caller; PayHook pays `msg.sender`), `Witness(kind="sp1-groth16", data=abi.encode(programVKey, payloadLocator), variables=[jobHash])` producing `proof`, `Witness(kind="sp1-public-values", ...)` producing `publicValues`, `Query(PayHook.jobs(jobHash))` for open/claimant/amount/deadline/status |
| payments[0] | native stake `bidWei` from PayHook to `PaymentRecipient`, `onStepIdx = 0`, `estimatedDelaySeconds = 0` (paid in the fill transaction) |
| assumptions | `sp1-verifier`, `program-vkey`, `tcb=T2CERT0`, `jobhash-scheme=BecomeJobHash/v0`, `network=sepolia`, `question-class=nat-sum`, `native-payment` (the draft's payment type is ERC-20; native ETH is declared), `exclusive-claimant` when set |

Two rules of the draft bind us usefully. A resolver "MUST guarantee that an
order may only abort as explicitly specified in revert policies", so every
PayHook revert reason has to be enumerated in the resolved order. And named
assumptions "require solver validation before execution", which is exactly
the standing BECOME wants for its honesty labels.

Known costs: the draft is four months old and may move; payments are ERC-20
only (hence the `native-payment` assumption, or a WETH PayHook later); it
requires ERC-7930 interoperable addresses in the resolved form.

## 9. Nostr: transport and delivery

Three layers, none overlapping: **Nostr** carries the request and the
delivery, **the chain** holds the stake and settles, **the resolver** tells a
prover what to do. `jobHash` ties all three.

What exists in the server repo today: an outbound-only NIP-90 listener with
NIP-42 auth and silent ignore on intake failure; NIP-44 v2 encryption to
BECOME's key; a placeholder "logical kind" 51000 wrapped as NIP-59 gift-wrap
(kind 1059) because the Buzz relay allowlist has no DVM kind; a JSON
`BecomeServiceRequest/v0` order blob whose `encryptedPayloadUri` is
`nostr-event:<id>`; an atomic `screen_request` gate that issues a token
`enqueue_job` requires; a result publisher. The shape is right. Three things
are placeholders: the kind, the JSON order, and the payment tags.

### NIP-90 mapping

| NIP-90 element | BECOME use |
|---|---|
| job request kind (5000-5999) | **5700** (owner decision 2026-09-10): "ERC-7683 order request". Result kind **6700**. The kind means "resolve and fill this ERC-7683 order"; what the job is about lives in the resolver, not the kind. 51000 was a test placeholder outside NIP-90. Registry text: [`nostr-kind-5700.yaml`](nostr-kind-5700.yaml) |
| `p` tag | BECOME's pubkey; mirrors `claimant` (absent when `claimant = address(0)`) |
| `encrypted` + content | NIP-44 v2 to BECOME's key: the development, salt, `specRoot`, target, and the full `BecomeOrderData`; NIP-90 text names NIP-04, which is superseded and should be noted as a deliberate deviation |
| clear tags | routing and money only, per the DVM spec §4: `param payhook`, `param jobHash`, `param orderDataType`, `param cap` (wei), `relays`; no `i` tag with source text |
| `bid` tag | not used: NIP-90 `bid` is millisats for Lightning; the stake is on chain |
| `relays` tag | where BECOME publishes feedback and results |
| feedback kind 7000 `status` | `processing` = Proving, `success` = Settled, `error` = Failed; `payment-required` is never sent because funding precedes the request; `amount` tag carries the fill tx hash instead of a bolt11 |
| result kind (request + 1000) | encrypted delivery: answer file and compiled form, `e` = request event, `p` = client, plus settlement tx reference |
| NIP-89 kind 31990 | BECOME's handler announcement: supported job kind (`k`), plus PayHook address, resolver address, `programVKey`, `orderDataType` in content. This fills the resolver-discovery gap the 2026 draft leaves open |

### Intake order (unchanged from the DVM spec)

Addressed to BECOME and encrypted, else ignore. PayHook shows the job open
with claimant BECOME, cap sufficient, deadline ahead, else ignore. Decrypt,
rebuild the manifest, confirm `jobHash`, else ignore. Failures stay silent;
the listener never sends NIP-90 error feedback. Only then does `resolve` run
and the prover start.

### What standardizing changes on the Nostr side

- Register 5700 / 6700 in the protocol's registry of kinds (a pull request
  from the owner's or Grok Bot's GitHub account; the text is in
  `nostr-kind-5700.yaml`) and get 5700 allowlisted on Buzz, retiring the
  gift-wrap fallback.
- Tag layout for 5700: `p` = exclusive claimant (optional), `R` = resolver as
  an ERC-7930 interoperable address (carries the chain id), `x` = order id
  (`jobHash`), `expiration` = fill deadline (NIP-40), `relays`. No `i`,
  `output` or `bid` tags. Content = the ERC-7683 payload, NIP-44 encrypted
  when `p` is present, clear when the order is open to any solver.
- Tag layout for 6700: `e` = the 5700 request, `p` = the customer, `x` =
  order id, `settlement` = fill transaction hash. Content = the encrypted
  deliverable. Kind 7000 feedback unchanged, with `expired` and `canceled`
  borrowed from NIP-69's status set.
- The listener filters on `#p` and `#R`, never on the kind alone, so BECOME
  does not download every intent on a relay.
- Replace the JSON `BecomeServiceRequest/v0` with the ABI `BecomeOrderData`
  v1 inside the encrypted content, and derive the clear `param` tags from it.
- Publish the kind 31990 announcement with the resolver and PayHook pins.
- Emit kind 7000 feedback from the same state machine that drives
  `agent_status`, so an MCP poller and a Nostr subscriber see the same
  transitions.

## 10. What this does not decide

- Whether to emit `Open` from the PayHook now (it costs a contract change).
- Whether `claimant = address(0)` orders are ever accepted (the market future).
- ERC-20 stakes, cross-chain funding, fiat relayers: each is a v2 field or an
  envelope choice, none changes v1.
- Who submits the registry pull request for 5700 / 6700 (owner or Grok Bot).
- Whether the resolver ships first as an on-chain contract or as the host's
  reference implementation.

## Sources

- ERC-7683 text at the last 2024-form revision:
  https://github.com/ethereum/ERCs/blob/f9fb91e0d7e7649d5bf47c0c82ac7d10ab12d90a/ERCS/erc-7683.md
- ERC-7683 current master (2026-05-13 resolver redesign):
  https://github.com/ethereum/ERCs/blob/master/ERCS/erc-7683.md
- File history showing the redesign commit:
  https://github.com/ethereum/ERCs/commits/master/ERCS/erc-7683.md
- Across reference implementation (`ERC7683Across.sol`, type strings and Permit2 witness):
  https://github.com/across-protocol/contracts/commit/108be77c29a3861c64bdf66209ac6735a6a87090
- ERC-7930 interoperable addresses (required by the 2026 form):
  https://github.com/ethereum/ERCs/blob/master/ERCS/erc-7930.md
- BECOME job hash rule in force: `BecomeJobHash/v0 = sha256(Question.v)`;
  PayHook ABI in use: `open(bytes32,address,uint64)`, `fill(bytes32,bytes,bytes)`,
  `jobs(bytes32) -> (client, claimant, amount, deadline, status)`.
