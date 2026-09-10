# BECOME data model: the Service Request lifecycle in quads

Status: design v2, 2026-09-10. Owner decision: **all BECOME schema are
quads.** The quad dataset is the one data model; the ERC-7683 order struct,
the JSON `Job` on MCP, the Nostr event tags, the NIP-89 announcement and the
resolved order are projections of it. v2 consolidates the full lifecycle
and closes the gaps found against the DVM spec and the live host (§10).
Companions: [`BECOME-SCHEMA.md`](BECOME-SCHEMA.md) (projections),
[`ORDER-SCHEMA.md`](ORDER-SCHEMA.md) (ERC-7683 rationale),
[`nostr-kind-5700.yaml`](nostr-kind-5700.yaml) (registry text).

## 1. The four words

| Term | Means | Holds |
|---|---|---|
| **SLI** | a number that can be measured | definition, unit, kind |
| **SLM** | how that number is obtained | source, system, method, frequency |
| **SLO** | what the number must be, over what period | indicator, comparator, target, period |
| **SLA** | the parties, the terms, and what money moves when an SLO is met or missed | parties, dates, deposit, guarantee, waivers, consequences |

Objectives are money-free. Money lives only in SLA consequence clauses,
each pointing at one SLO with `onMet` or `onMissed`. The guarantee (solver
bond) is the `onMissed` amount.

## 2. Lifecycle

Nine stages. Each stage adds exactly one graph or one event; nothing is
edited in place.

| # | Stage | Actor | Adds | Event / transaction |
|---|---|---|---|---|
| 0 | Discover | provider | `deployment` (once) | NIP-89 kind 31990 |
| 1 | Draft | client (or its agent) | `sla`, `sow`, `manifest`, `sli`, `slm`, `slo`, `order` | none; validator runs (§7 rule 1) |
| 2 | Commit | client | `specRoot`, then `envelope` with `nonce`; `jobHash` fixed | none |
| 3 | Fund | funder | `auth` (order signature) | PayHook `open` |
| 4 | Request | client | none (graphs travel) | Nostr kind 5700 |
| 5 | Intake | provider | `request` | none (silent on reject) |
| 6 | Prove | provider | `status` (Proving), `proof` | kind 7000 `processing` |
| 7 | Settle | provider | `settlement`, `status` (Settled) | PayHook `fill`; kind 7000 `success` |
| 8 | Deliver | provider | `result` | kind 6700 |
| 9 | Close | either | `close` | refund tx / kind 7000 `expired` or `canceled` |

Stage 1 through 2 are the question. Stage 3 is the money. Stages 5 through 9
are the provider's account of what happened. A tier 0 (smoke) job skips
stages 3, 7 and the chain half of 9.

## 3. Encoding

- **Format:** N-Quads, one per line: `<subject> <predicate> <object> <graph> .`
  IRIs for subject, predicate, graph; IRIs or typed literals for objects. No
  blank nodes, so canonical form is sorted lines, UTF-8, `\n` terminated.
- **Namespace:** `become:` = `https://mcp.belovedecosystem.com/ns/v1#`.
  Borrowed: `dc:title`, `dc:date`, `xsd:` datatypes.
- **The job subject.** Client graphs are written before `jobHash` exists, so
  they use the fixed placeholder subject `<become:this>` (and
  `<become:this/slo/1>`, `/sli/coqchk`, `/slm/coqchk`, `/sla/c1`,
  `/file/Question.v`). `specRoot` is the hash of that canonical text. Once
  `jobHash` is known, every reader resolves `<become:this>` to
  `<become:job/<jobHash>>`. Provider graphs use the resolved subject
  directly. This removes the circularity of hashing a graph that names its
  own hash.
- **Addresses:** `eth:<chainId>:<address>`, `eth:<chainId>:tx/<hash>`,
  `nostr:pubkey/<hex>`, `nostr:event/<id>`.

## 4. Graphs

Three families.

**Client graphs** are the question. They are hashed into `specRoot`.

| Graph | Predicates |
|---|---|
| `become:sla` | `dc:title`, `dc:date`, `client` (eth), `clientPubkey` (nostr), `provider` (eth), `providerPubkey` (nostr), `securityDeposit`, `guarantee`, `waives`, `note`; on `…/sla/cN`: `onMet` \| `onMissed` (an SLO), `pays`, `to`, `cap` |
| `become:sow` | `projectName`, `questionClass`, `target` (Gallina literal), `coqVersion`, `flag`, `permittedAxiom`, `input` (development locator), `linksTo` |
| `become:manifest` | on `…/file/<path>`: `path`, `sha256`, `bytes`; on `<become:this>`: `fileCount`, `developmentHash` (sha256 over the sorted `path:sha256` lines) |
| `become:sli` | `definition`, `unit`, `kind` (`witness` \| `query` \| `event` \| `execution`) |
| `become:slm` | `measures`, `source`, `system`, `method`, `frequency`, `block`, `attested` (boolean) |
| `become:slo` | `usesSli`, `comparator` (`eq` \| `gte` \| `lte`), `target`, `periodStart`, `periodEnd` |
| `become:order` | `schemaVersion`, `payHook`, `claimant`, `funder`, `refundTo`, `bidWei`, `tier`, `openDeadline`, `fillDeadline`, `dueDate` |

**Envelope graph** is client-asserted but outside `specRoot`, because it
holds what must be fresh per job or cannot be known before hashing.

| Graph | Predicates |
|---|---|
| `become:envelope` | `specRoot`, `nonce`, `jobHash`, `relay` (multiple), `encryptedTo` (nostr pubkey or none) |
| `become:auth` | `orderSignature` (EIP-712 over `BecomeOrderData`), `signer`, `signedAt` |

`jobHash = sha256(specRoot || nonce)`. The same question asked twice has
the same `specRoot` and different `jobHash`es; duplicate-target policy is a
provider rule on `specRoot`, not on `jobHash` (§7 rule 6).

**Provider graphs** are the account. They are signed by the provider's
Nostr key and never hashed.

| Graph | Predicates |
|---|---|
| `become:deployment` | `payHook`, `verifier`, `programVKey`, `guest`, `elfSha256`, `chainId`, `resolver`, `nostrPubkey`, `questionClass` (multiple), `acceptsAnonymous`, `maxBidWei`, `createRatePerHour`, `acceptedAssumption`, `status` |
| `become:request` | `event` (the 5700 id), `receivedAt`, `screened` (`accepted` \| `rejected`), `screenReason` (never published when rejected) |
| `become:status` | `agentStatus`, `terminal`, `pollAfterMs`, `how`, `code`, `error`, `stage`, `createdAt`, `updatedAt`, `late` |
| `become:proof` | `runId`, `runUrl`, `provingStartedAt`, `proofPresent`, `proofSha256`, `publicValues` (hex), `vkey` |
| `become:settlement` | `openTx`, `fillTx`, `receiptUrl`, `settledAt`, `paid` (wei), `paidTo` |
| `become:result` | `answer`, `inbox`, `answerV`, `certified`, `deliveredEvent` (6700 id), `deliveredAt` |
| `become:close` | `closedAs` (`expired` \| `canceled` \| `refunded`), `closedAt`, `refundTx`, `reason` |

`agentStatus` is one of `Queued`, `Proving`, `Settled`, `Failed`,
`Expired`, `Cancelled`. `terminal` is true for the last four.

**Exempt from the projection rule:** MCP transport credentials
(`owner_token`, `job_id`) exist only in the MCP projection and in no graph.
They are how a session proves it may read a job; they say nothing about the
job. This is the one declared exception to "a projection never carries what
the graphs do not".

## 5. Worked example A: certified 45+8 (tier 1)

```
# --- client graphs (hashed into specRoot); subject <become:this> ---
<become:this> <dc:title>                 "BECOME certified sum 45+8"                       <become:sla> .
<become:this> <dc:date>                  "2026-09-10"^^xsd:date                           <become:sla> .
<become:this> <become:client>            <eth:11155111:0x11bD…2Eb>                        <become:sla> .
<become:this> <become:clientPubkey>      <nostr:pubkey/<client hex>>                       <become:sla> .
<become:this> <become:provider>          <eth:11155111:0x11bD…2Eb>                        <become:sla> .
<become:this> <become:providerPubkey>    <nostr:pubkey/<BECOME hex>>                       <become:sla> .
<become:this> <become:securityDeposit>   "0"^^xsd:integer                                 <become:sla> .
<become:this> <become:guarantee>         "0"^^xsd:integer                                 <become:sla> .
<become:this> <become:waives>            "deposit"                                         <become:sla> .
<become:this> <become:waives>            "guarantee"                                       <become:sla> .
<become:this> <become:note>              "zero bid; self-funded; anonymous tier"           <become:sla> .
<become:this/sla/c1> <become:onMet>      <become:this/slo/1>                               <become:sla> .
<become:this/sla/c1> <become:pays>       "0"^^xsd:integer                                  <become:sla> .
<become:this/sla/c1> <become:to>         <eth:11155111:0x11bD…2Eb>                         <become:sla> .
<become:this/sla/c2> <become:onMissed>   <become:this/slo/1>                               <become:sla> .
<become:this/sla/c2> <become:pays>       "0"^^xsd:integer                                  <become:sla> .
<become:this/sla/c2> <become:to>         <eth:11155111:0x11bD…2Eb>                         <become:sla> .

<become:this> <become:projectName>       "nat-sum"                                         <become:sow> .
<become:this> <become:questionClass>     "nat-sum"                                         <become:sow> .
<become:this> <become:target>            "exists n : nat, 45 + 8 = n"                      <become:sow> .
<become:this> <become:coqVersion>        "8.20.1"                                          <become:sow> .
<become:this> <become:flag>              "-Q . Become"                                     <become:sow> .
<become:this> <become:input>             <nostr:event/<development bundle id>>             <become:sow> .
<become:this> <become:linksTo>           <become:manifest>                                 <become:sow> .
<become:this> <become:linksTo>           <become:sli>                                      <become:sow> .
<become:this> <become:linksTo>           <become:slm>                                      <become:sow> .
<become:this> <become:linksTo>           <become:slo>                                      <become:sow> .
<become:this> <become:linksTo>           <become:order>                                    <become:sow> .

<become:this/file/Question.v> <become:path>   "Question.v"                                <become:manifest> .
<become:this/file/Question.v> <become:sha256> "0x<sha256 of Question.v>"                  <become:manifest> .
<become:this/file/Question.v> <become:bytes>  "412"^^xsd:integer                          <become:manifest> .
<become:this> <become:fileCount>         "1"^^xsd:integer                                  <become:manifest> .
<become:this> <become:developmentHash>   "0x<sha256 over sorted path:sha256 lines>"        <become:manifest> .

<become:this/sli/coqchk>  <become:definition> "coqchk accepts answer.v against the manifest; Print Assumptions within permittedAxiom" <become:sli> .
<become:this/sli/coqchk>  <become:kind>       "witness"                                    <become:sli> .
<become:this/sli/settled> <become:definition> "PayHook.jobs(jobHash).status"               <become:sli> .
<become:this/sli/settled> <become:kind>       "query"                                      <become:sli> .

<become:this/slm/coqchk>  <become:measures>  <become:this/sli/coqchk>                       <become:slm> .
<become:this/slm/coqchk>  <become:system>    "SP1 guest T2CERT0 vkey 0x00116101…388c"        <become:slm> .
<become:this/slm/coqchk>  <become:method>    "Groth16; publicValues[0] == jobHash"           <become:slm> .
<become:this/slm/coqchk>  <become:frequency> "once"                                          <become:slm> .
<become:this/slm/coqchk>  <become:attested>  "true"^^xsd:boolean                             <become:slm> .
<become:this/slm/settled> <become:measures>  <become:this/sli/settled>                      <become:slm> .
<become:this/slm/settled> <become:system>    <eth:11155111:0x8FA8…4B8>                       <become:slm> .
<become:this/slm/settled> <become:method>    "eth_call jobs(bytes32)"                         <become:slm> .
<become:this/slm/settled> <become:frequency> "on fill"                                        <become:slm> .
<become:this/slm/settled> <become:attested>  "true"^^xsd:boolean                             <become:slm> .

<become:this/slo/1> <become:usesSli>     <become:this/sli/settled>                           <become:slo> .
<become:this/slo/1> <become:comparator>  "eq"                                                <become:slo> .
<become:this/slo/1> <become:target>      "2"^^xsd:integer                                    <become:slo> .
<become:this/slo/1> <become:periodStart> "1789000000"^^xsd:integer                           <become:slo> .
<become:this/slo/1> <become:periodEnd>   "1789086400"^^xsd:integer                           <become:slo> .

<become:this> <become:schemaVersion>     "1"^^xsd:integer                                  <become:order> .
<become:this> <become:payHook>           <eth:11155111:0x8FA8…4B8>                         <become:order> .
<become:this> <become:claimant>          <eth:11155111:0x11bD…2Eb>                         <become:order> .
<become:this> <become:funder>            <eth:11155111:0x11bD…2Eb>                         <become:order> .
<become:this> <become:refundTo>          <eth:11155111:0x11bD…2Eb>                         <become:order> .
<become:this> <become:bidWei>            "0"^^xsd:integer                                  <become:order> .
<become:this> <become:tier>              "1"^^xsd:integer                                  <become:order> .
<become:this> <become:openDeadline>      "1789003600"^^xsd:integer                         <become:order> .
<become:this> <become:fillDeadline>      "1789086400"^^xsd:integer                         <become:order> .
<become:this> <become:dueDate>           "1789050000"^^xsd:integer                         <become:order> .

# --- envelope (client, not in specRoot); subject resolved ---
<become:job/0x3b32…af2> <become:specRoot>  "0x<sha256 of the block above>"                <become:envelope> .
<become:job/0x3b32…af2> <become:nonce>     "0x<32 bytes>"                                  <become:envelope> .
<become:job/0x3b32…af2> <become:jobHash>   "0x3b3208bf…869af2"                             <become:envelope> .
<become:job/0x3b32…af2> <become:relay>     <wss://buzz.belovedecosystem.com>               <become:envelope> .
<become:job/0x3b32…af2> <become:encryptedTo> <nostr:pubkey/<BECOME hex>>                   <become:envelope> .
<become:job/0x3b32…af2> <become:orderSignature> "0x<eip712 sig>"                           <become:auth> .
<become:job/0x3b32…af2> <become:signer>    <eth:11155111:0x11bD…2Eb>                       <become:auth> .

# --- provider graphs (signed, not hashed) ---
<become:job/0x3b32…af2> <become:event>       <nostr:event/<5700 id>>                       <become:request> .
<become:job/0x3b32…af2> <become:receivedAt>  "1789000120"^^xsd:integer                     <become:request> .
<become:job/0x3b32…af2> <become:screened>    "accepted"                                     <become:request> .
<become:job/0x3b32…af2> <become:agentStatus> "Settled"                                      <become:status> .
<become:job/0x3b32…af2> <become:terminal>    "true"^^xsd:boolean                            <become:status> .
<become:job/0x3b32…af2> <become:createdAt>   "1789000100"^^xsd:integer                      <become:status> .
<become:job/0x3b32…af2> <become:late>        "true"^^xsd:boolean                            <become:status> .
<become:job/0x3b32…af2> <become:runUrl>      <https://github.com/…/actions/runs/…>          <become:proof> .
<become:job/0x3b32…af2> <become:publicValues> "0x3b3208bf…869af2…"                          <become:proof> .
<become:job/0x3b32…af2> <become:proofSha256> "0x<sha256 of proof.bin>"                      <become:proof> .
<become:job/0x3b32…af2> <become:openTx>      <eth:11155111:tx/0x…>                          <become:settlement> .
<become:job/0x3b32…af2> <become:fillTx>      <eth:11155111:tx/0xf80d…60f>                   <become:settlement> .
<become:job/0x3b32…af2> <become:receiptUrl>  <https://sepolia.etherscan.io/tx/0xf80d…60f>   <become:settlement> .
<become:job/0x3b32…af2> <become:paid>        "0"^^xsd:integer                               <become:settlement> .
<become:job/0x3b32…af2> <become:answer>      "53"                                           <become:result> .
<become:job/0x3b32…af2> <become:certified>   "true"^^xsd:boolean                            <become:result> .
<become:job/0x3b32…af2> <become:deliveredEvent> <nostr:event/<6700 id>>                     <become:result> .
```

## 6. Worked example B: smoke 5+1 (tier 0)

Only the lines that differ from A. No PayHook, no envelope signature, no
settlement graph, no 7000 `success` with a transaction.

```
<become:this> <become:target>            "exists n : nat, 5 + 1 = n"                       <become:sow> .
<become:this> <become:tier>              "0"^^xsd:integer                                  <become:order> .
<become:this> <become:bidWei>            "0"^^xsd:integer                                  <become:order> .

<become:this/sli/coqchk>  <become:kind>       "execution"                                  <become:sli> .
<become:this/slm/coqchk>  <become:system>    "host coqc/coqchk 8.20.1"                       <become:slm> .
<become:this/slm/coqchk>  <become:method>    "coqchk over manifest + answer.v; Print Assumptions" <become:slm> .
<become:this/slm/coqchk>  <become:attested>  "false"^^xsd:boolean                            <become:slm> .

<become:this/slo/1> <become:usesSli>     <become:this/sli/coqchk>                            <become:slo> .
<become:this/slo/1> <become:comparator>  "eq"                                                <become:slo> .
<become:this/slo/1> <become:target>      "accepted"                                          <become:slo> .

<become:job/…> <become:agentStatus> "Settled"                                              <become:status> .
<become:job/…> <become:answer>      "6"                                                    <become:result> .
<become:job/…> <become:certified>   "false"^^xsd:boolean                                   <become:result> .
```

`attested = false` on the SLM is what makes a tier 0 "Settled" readable as
"checked on the host, nobody else can verify it". The projection to the MCP
`labels` and to the resolved-order assumptions carries it as `unattested`.

## 7. Resolver: graphs to ERC-7683 (2026)

Deterministic, in order. Any failure resolves to one abort naming the rule.

1. **Validate.** All `sow linksTo` graphs present. Every `slo usesSli` names
   an SLI with exactly one `slm measures`. Every `sla` consequence names an
   SLO. Zero `securityDeposit` or `guarantee` requires a matching `waives`.
   `sow target`, `coqVersion` non-empty. `manifest fileCount` equals the
   number of `…/file/*` subjects and `developmentHash` recomputes. Exactly
   one SLO in v1.
2. **Commit check.** `envelope specRoot` equals the hash of the canonical
   client graphs; `jobHash = sha256(specRoot || nonce)`.
3. **Variables from SLIs**, procedure from the matching SLM: `witness` →
   `Witness(kind = slm.system, data = slm.method, variables = [jobHash])`;
   `query` → `Query(target = slm.system, selector from slm.method, block)`;
   `event` → `QueryEvents`; `execution` → `ExecutionOutput`. `attested =
   false` adds assumption `unattested:<sli>`.
4. **Steps from the SOW.** Tier 1: one step `payHook.fill(jobHash,
   publicValues, proof)`, `NeedsVariable` on each witness,
   `TimingBounds(block.timestamp, slo.periodStart, min(slo.periodEnd,
   order.fillDeadline))`, one `RevertPolicy(abort, …)` per PayHook revert
   reason (`settled`, `deadline`, `not claimant`, `bad proof`). Tier 0: no
   on-chain step; the resolved order has an empty steps list and the SLO is
   evaluated by the provider alone (assumption `unattested`).
5. **Payments from SLA consequences.** One payment per `onMet`:
   `amountFormula = pays`, sender = escrow, recipient = `to`, `onStepIdx` =
   the SLO's step. One bond claim per `onMissed` from `guarantee` when the
   step aborts. Per-unit amounts need the unit count as a variable; not
   needed while objectives are binary.
6. **Duplicate policy.** If `deployment` records a prior `Settled` job with
   the same `specRoot`, abort `already_certified` (the re-certify guard,
   keyed by `specRoot` because `jobHash` is salted).
7. **Assumptions.** Every `waives`; every `deployment acceptedAssumption`;
   `self-funded` when `sla client == sla provider`; `native-payment`; the
   deployment pins (`verifier`, `programVKey`, `guest`); `unattested:<sli>`
   per rule 3; `late-allowed` when no `onMissed` pays more than zero.

## 8. Projections

| Projection | From graphs | Rule |
|---|---|---|
| `BecomeOrderData` (ABI) | `order`, `envelope` | field per `order` predicate; `jobHash`, `specRoot` from `envelope`; `acceptanceRuleHash = sha256(canonical sow + manifest + sli + slm)`; no payload fields (they were circular) |
| ERC-7683 resolved order | client graphs + `envelope` + `deployment` | §7 |
| Nostr 5700 tags | `order`, `envelope`, `deployment` | `p` = `sla providerPubkey`; `R` = `deployment resolver`, else `deployment payHook` until a resolver exists; `x` = `jobHash`; `expiration` = `fillDeadline`; `relays` = `envelope relay` |
| Nostr 5700 content | client graphs + `envelope` | canonical N-Quads; NIP-44 to `envelope encryptedTo` when set |
| Nostr 7000 | `status`, `settlement`, `close` | `status` tag from `agentStatus` (`processing`, `success`, `error`, `expired`, `canceled`); `amount` tag = `paid` + `fillTx` |
| Nostr 6700 | `result`, `settlement` | `e` = `request event`; `p` = `sla clientPubkey`; `x` = `jobHash`; `settlement` = `fillTx`; content = `result` encrypted to `clientPubkey` |
| NIP-89 31990 | `deployment` | `k` = 5700; content = the live deployment graph as JSON |
| MCP `Job` | `order`, `envelope`, `status`, `proof`, `settlement`, `result`, `close`, `deployment` | snake_case of each predicate; closed list in `BECOME-SCHEMA.md`; plus the exempt `owner_token`, `job_id` |
| MCP `Request` | `sow`, `order`, `sla` (parties) | inverse: the host drafts the client graphs; `quads` may be supplied directly and then `target` must equal `sow target` |

## 9. Decisions (2026-09-10)

- All schema are quads; projections are derived, never authored.
- SLA holds the money; SLO is money-free; `guarantee` is the `onMissed` amount.
- Vocabulary self-contained, borrowing only `dc:title`, `dc:date`, `xsd:`.
- Client graphs travel inside the 5700 event; the development by locator,
  pinned by the `manifest` graph.
- One SLO per order in v1.
- Nostr kinds 5700 request, 6700 result; kind 7000 reused.
- Transport credentials (`owner_token`, `job_id`) are exempt from the
  projection rule.

## 10. Gaps closed in v2 (against the DVM spec and the live host)

| Gap | Fix |
|---|---|
| `payloadHash` / `payloadLocator` were inside the hashed order but depend on the published event | removed from `order`; the 5700 id is recorded by the provider in `request event` |
| development referenced by locator only; files could be swapped | `manifest` graph with per-file `sha256` and `developmentHash`, hashed into `specRoot` |
| acceptance rule (Coq version, flags, permitted axioms) had no predicates | `sow coqVersion`, `flag`, `permittedAxiom`; `acceptanceRuleHash` defined over them |
| graphs named their own hash | placeholder subject `<become:this>`, resolved after hashing |
| nonce inside `specRoot` made the same question a different spec | `nonce` moved to `envelope`; `specRoot` identifies the question, `jobHash` the job |
| re-certify guard keyed by salted `jobHash` never fires | rule 6, keyed by `specRoot` |
| `owner_token` had no source graph | declared exempt (transport credential) |
| client and provider Nostr identities absent | `sla clientPubkey`, `providerPubkey`; `deployment nostrPubkey` |
| `relays` tag had no source | `envelope relay` |
| `R` tag with null resolver | falls back to `payHook` until a resolver is deployed |
| expiry / reclaim / cancel unmodelled | `close` graph; `Expired`, `Cancelled` statuses; kind 7000 `expired` / `canceled` |
| proof evidence unrecorded | `proof` graph: `publicValues`, `proofSha256`, `runId`, `vkey` |
| signatures had nowhere to live | `auth` graph, unhashed |
| funder distinct from client (fiat path) | `order funder` |
| host acceptance policy invisible to solvers | `deployment acceptsAnonymous`, `maxBidWei`, `createRatePerHour` |
| `questionClass`, `createdAt` missing | added to `sow` / `deployment` and `status` |
| tier 0 indistinguishable from tier 1 in the model | `slm attested`; example B; assumption `unattested` |

## 11. Open

- Cancellation on Nostr: a client-signed kind 5 deletion of the 5700 event
  versus a `become:cancel` request predicate; the provider honours either
  only before Proving.
- Per-unit payment formulas with the ERC-7683 authors.
- Whether provider graphs are published whole as Nostr content per
  transition, or only as the 7000 status tag.
