# BECOME schema

One document. Status: design v2, 2026-09-10. The live host still returns
the interim JSON in `/agent.md`; everything below is the target the next
code pass builds toward. Companion files (`SERVICE-REQUEST-QUADS.md`,
`BECOME-SCHEMA.md`, `ORDER-SCHEMA.md`) now point here.

Contents: Part 0 explains the schema in plain words. Part 1 is the data
model (quads). Part 2 is the projections, field by field, for the chain,
Nostr, the prover and MCP. Part 3 is why ERC-7683, and which version.
Appendix A is the Nostr registry entry. Appendix B lists the decisions.

## Part 0: the schema in plain words

**The one idea.** A BECOME job is a growing pile of statements. Each has
four parts: which section it belongs to, what it is about, which property,
what value. "In the agreement section, this job's client is wallet X." That
four-part form is a quad. The pile is the only truth. The blockchain struct,
the JSON a bot reads, the Nostr message: each is a view generated from the
pile, never written by hand.

**The sections are the contract's chapters.** The agreement (SLA): parties,
dates, deposit, guarantee, and the consequence clauses that move money when
an objective is met or missed. The statement of work (SOW): the question.
The manifest: fingerprints of every file. The indicators (SLI): the numbers
that decide the outcome. The measurements (SLM): how each number is
obtained, which is where trust lives. The objectives (SLO): what each number
must be and by when, with no money in them. The order: the money terms in
the shape the chain needs.

**Two fingerprints.** The client's chapters are hashed into `specRoot`,
which is the question. A fresh random number is added to make `jobHash`,
which is this particular asking. The proof commits to `jobHash` first, and
the settlement contract checks it first. That one link is what lets the
chain pay only for a proof of exactly this question, without anyone
trusting BECOME. After that only the provider adds statements: received,
proving, proof evidence, settled, delivered, or expired or cancelled. Those
are signed, and never change the fingerprint.

**Projections.** A projection is a view of the same facts reshaped for one
reader: a ten-field struct for the chain, a 38-field JSON object for a bot,
tags plus an encrypted body for a Nostr relay, an announcement for
discovery, and a resolved order (steps, values, payments, assumptions) for a
prover. A projection may leave facts out; it may never add any. To add a
field to a projection, add the predicate to the model first. The one
declared exception is the MCP session credential (`owner_token`, `job_id`),
which is how a session proves it may read a job and says nothing about the
job.

**The anonymous tier, honestly.** For a free job the pile says client equals
provider, deposit and guarantee are zero and waived, and the consequence
clauses pay zero. Any solver reading it sees an agreement with no teeth,
which is the truth of the interim system, as data.

**Nine stages.** Discover, draft, commit, fund, request, intake, prove,
settle, deliver, close. Each adds one chapter or sends one message. Nothing
is edited; the pile only grows. A free smoke job skips fund and settle.

## Part 1: the data model (quads)

### 1. The four words

| Term | Means | Holds |
|---|---|---|
| **SLI** | a number that can be measured | definition, unit, kind |
| **SLM** | how that number is obtained | source, system, method, frequency |
| **SLO** | what the number must be, over what period | indicator, comparator, target, period |
| **SLA** | the parties, the terms, and what money moves when an SLO is met or missed | parties, dates, deposit, guarantee, waivers, consequences |

Objectives are money-free. Money lives only in SLA consequence clauses,
each pointing at one SLO with `onMet` or `onMissed`. The guarantee (solver
bond) is the `onMissed` amount.

### 2. Lifecycle

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

### 3. Encoding

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

### 4. Graphs

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

### 5. Worked example A: certified 45+8 (tier 1)

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

### 6. Worked example B: smoke 5+1 (tier 0)

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

### 7. Resolver: graphs to ERC-7683 (2026)

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

### 8. Projections

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

### 9. Decisions (2026-09-10)

- All schema are quads; projections are derived, never authored.
- SLA holds the money; SLO is money-free; `guarantee` is the `onMissed` amount.
- Vocabulary self-contained, borrowing only `dc:title`, `dc:date`, `xsd:`.
- Client graphs travel inside the 5700 event; the development by locator,
  pinned by the `manifest` graph.
- One SLO per order in v1.
- Nostr kinds 5700 request, 6700 result; kind 7000 reused.
- Transport credentials (`owner_token`, `job_id`) are exempt from the
  projection rule.

### 10. Gaps closed in v2 (against the DVM spec and the live host)

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

### 11. Open

- Cancellation on Nostr: a client-signed kind 5 deletion of the 5700 event
  versus a `become:cancel` request predicate; the provider honours either
  only before Proving.
- Per-unit payment formulas with the ERC-7683 authors.
- Whether provider graphs are published whole as Nostr content per
  transition, or only as the 7000 status tag.

## Part 2: projections, field by field

One job, one id. The order id is `jobHash`, and it appears in every layer:

```
Nostr (NIP-90)          Ethereum (ERC-7683 + PayHook)        MCP (JSON)
kind 5700  ──x tag───►  jobHash ◄──── publicValues[0]  ◄──── job_hash
kind 7000  status       jobs(jobHash).status                  agent_status
kind 6700  settlement   fill tx                               fill_tx / receipt_url
```

### Layer 0: `Deployment` (the pins)

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

### Layer 1: the order (ERC-7683 payload)

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

### Layer 2: the request on Nostr (kind 5700)

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

### Layer 3: the chain (PayHook)

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

### Layer 1b: the resolved order (what a prover reads)

`IResolver.resolve(orderData)` per ERC-7683 (2026 form):

| Part | Value |
|---|---|
| steps[0] | `Call` payHook.`fill(bytes32,bytes,bytes)` with [`jobHash`, var `publicValues`, var `proof`]; `NeedsVariable(proof)`, `NeedsVariable(publicValues)`, `TimingBounds(block.timestamp, 0, fillDeadline)`, `SpendsGas(est)`, `RevertPolicy(abort, "settled")`, `RevertPolicy(abort, "deadline")`, `RevertPolicy(abort, "bad proof")` |
| variables | `StepCaller(0)`; `PaymentRecipient` (= caller); `Witness("sp1-groth16", abi.encode(programVKey, developmentHash), [jobHash]) -> proof`; `Witness("sp1-public-values", same) -> publicValues`; `Query(payHook.jobs(jobHash))` |
| payments[0] | native `bidWei` from payHook to `PaymentRecipient`, on step 0, delay 0 |
| assumptions | `sp1-verifier=0xb69f2584CBcFf99a58C4e7002E8b89Af54a6f4e2`, `program-vkey=0x00116101c20b687297ae11e1fea4bd4bd003ef3320da147a3db1f65bc81f388c`, `tcb=T2CERT0`, `jobhash-scheme=BecomeJobHash/v0`, `network=sepolia`, `question-class=nat-sum`, `native-payment`, `exclusive-claimant=<addr>` when set |

### Layer 2b: progress and delivery on Nostr

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

### Layer 4: MCP

The MCP surface is a JSON view of the same job. One object, `Job`, is
returned by every job tool; the create tools also accept a `Request`. Field
names are snake_case mirrors of Layers 1 to 3. "live" marks what the host
returns today (`become-mcp-orchestrate/v0`); "v1" marks fields the schema adds.

#### Tools

| Tool | Input | Returns | Anonymous | Notes |
|---|---|---|---|---|
| `become_orchestrate` | `Request` | `Job` | yes, creates bind ownership | the default ask |
| `become_status` | `job_id`, `owner_token` | `Job` | yes, with token | reads never rate-limited |
| `become_result` | `job_id`, `owner_token` | `Job` + `answer_v` | yes, with token | same object, adds the Coq source |
| `become_cancel` | `job_id`, `owner_token` | `Job` (terminal) | yes, with token | best effort; never un-settles |
| `become_ask` | `Request` with `bid_wei` required | `Job` | yes | advanced alias; will be retired once `Request` carries the order |
| `become_encode_service_request` | `job_id` | `Order` (Layer 1) + EIP-712 typed data | yes | the only tool that emits the ERC-7683 payload; unsigned, no broadcast |
| `become_spec_check`, `become_engine_query` | | | | operator-only; not part of the data model |

#### `Request` (input to create)

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


#### Deadlines

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

#### `Job` (output of every job tool): closed schema

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

#### `Order` (output of `become_encode_service_request`)

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

#### One model, four views

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

### Invariants

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

## Part 3: why ERC-7683, and which version

### 1. What ERC-7683 is (two versions)

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

### 2. Role mapping

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

### 3. Decision: standardize on the 2026 form

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

#### Resolved order for a BECOME job

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

### Sources

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

## Appendix A: Nostr registry entry (kinds 5700 / 6700)

Paste into `schema.yaml` of `nostr-protocol/registry-of-kinds`; the pull request must come from the owner's or Grok Bot's GitHub account.

```yaml
# Registry entry for Nostr kinds 5700 / 6700 (NIP-90 job request / result).
#
# Paste these two blocks into schema.yaml of
# https://github.com/nostr-protocol/registry-of-kinds and open a pull request
# titled "Add kinds 5700/6700: ERC-7683 order request and result".
# Kind 7000 (job feedback) is already defined by NIP-90 and is reused as is.
#
# Both numbers were free in the archived DVM registry
# (nostr-protocol/data-vending-machines, read-only since 2026-09-07) and in
# registry-of-kinds as of 2026-09-10.

  5700:
    description: >-
      ERC-7683 order request (NIP-90 job request). The event asks a solver to
      resolve and fill an ERC-7683 order. What the order is about is defined
      by the resolver named in the R tag, not by this kind. Content is the
      ERC-7683 payload consumable by IResolver.resolve: NIP-44 encrypted to
      the claimant when a p tag is present, plain when the order is open to
      any solver. Funding and payment are on chain via the resolver's
      settlement contract; this kind uses no i, output or bid tags and never
      emits a payment-required status.
    in_use: true
    content:
      type: free
    required:
      - R
      - x
    tags:
      - name: p
        next:
          type: pubkey
          required: true
        description: exclusive claimant; when present the content is encrypted to this key
      - name: R
        next:
          type: string
          required: true
        description: resolver contract as an ERC-7930 interoperable address (carries chain id)
      - name: x
        next:
          type: string
          required: true
        description: order id (32-byte hex); for BECOME this is the job hash
      - name: expiration
        next:
          type: integer
          required: true
        description: fill deadline as a unix timestamp (NIP-40)
      - name: relays
        next:
          type: string
          required: true
        description: relays where the solver should publish feedback and the result
    multiple:
      - relays

  6700:
    description: >-
      ERC-7683 order result (NIP-90 job result for kind 5700). References the
      request and the settlement transaction that paid the solver. Content is
      the deliverable, NIP-44 encrypted to the customer.
    in_use: true
    content:
      type: free
    required:
      - e
      - p
      - x
    tags:
      - name: e
        next:
          type: eventid
          required: true
        description: the kind 5700 request
      - name: p
        next:
          type: pubkey
          required: true
        description: the customer
      - name: x
        next:
          type: string
          required: true
        description: order id, same value as in the request
      - name: settlement
        next:
          type: string
          required: true
        description: settlement transaction hash on the resolver's chain
```

## Appendix B: decisions (2026-09-10)

1. Standardize BECOME DVM requests on ERC-7683, the 2026 resolver-centric
   form; the client-signed order stays an EIP-712 struct.
2. Nostr kinds 5700 (ERC-7683 order request) and 6700 (result); kind 7000
   reused; the kind means "resolve and fill this order", the resolver
   defines what the job is.
3. All BECOME schema are quads; projections are derived, never authored.
4. SLA holds the money; SLO is money-free; `guarantee` is the `onMissed`
   amount.
5. Vocabulary self-contained, borrowing only Dublin Core `title`/`date` and
   `xsd:` datatypes; ODRL alignment is named future work.
6. Client graphs travel inside the 5700 event; the development by locator,
   pinned by the manifest graph.
7. One SLO per order in v1.
8. `Deployment` is a first-class object, one live per chain, referenced by
   `payhook`; one source file in the server repo feeds docs, NIP-89 and MCP.
9. The client `Job` is a closed schema; operator data only under
   `diagnostics` with an operator key; host paths and staging recipes never
   reach a client; no prose summary field.
10. `due_date` is soft (in the order graph, informational); `fillDeadline`
    is hard (the timing bound and PayHook deadline); `late` is provider
    status.
11. Transport credentials (`owner_token`, `job_id`) are exempt from the
    projection rule.
