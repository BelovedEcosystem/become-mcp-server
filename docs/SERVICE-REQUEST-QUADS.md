# BECOME data model: quads

Status: design, 2026-09-10. Owner decision: **all BECOME schema are quads.**
The quad dataset is the one data model. The ERC-7683 order struct, the JSON
`Job` on MCP, the Nostr event tags, the NIP-89 announcement and the
resolved order are projections of it, never sources. Companion documents:
[`BECOME-SCHEMA.md`](BECOME-SCHEMA.md) (the projections, field by field),
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
each of which points at one SLO with `onMet` or `onMissed`. The guarantee
(solver bond) is the `onMissed` amount, which gives it a home it did not
have before.

## 2. Encoding

- **Format:** N-Quads: `<subject> <predicate> <object> <graph> .` one per
  line. IRIs for subject, predicate, graph; IRIs or typed literals for
  objects. No blank nodes.
- **Canonical form:** sorted lines, UTF-8, `\n` terminated. Deterministic
  without a canonicalization algorithm because there are no blank nodes.
- **Namespace:** `become:` = `https://mcp.belovedecosystem.com/ns/v1#`.
  Borrowed: `dc:title`, `dc:date` (Dublin Core), `xsd:` datatypes. Nothing
  else is borrowed in v1 (decision, §8).
- **Subjects:** the job is `become:job/<jobHash>`. Sub-entities are
  `become:job/<jobHash>/slo/1`, `/sli/coqchk`, `/slm/coqchk`,
  `/sla/c1`. Chain addresses are `eth:<chainId>:<address>`; Nostr events
  are `nostr:event/<id>`.

## 3. Graphs

Two families. **Client-asserted** graphs are the question and are hashed.
**Provider-asserted** graphs are the answer and progress; they are signed by
the provider's Nostr key and never hashed into the order.

| Graph | Family | Predicates |
|---|---|---|
| `become:sla` | client | `dc:title`, `dc:date`, `client`, `provider`, `securityDeposit`, `guarantee`, `waives`, `note`; consequences on `…/sla/cN`: `onMet` / `onMissed` (an SLO), `pays` (wei), `to` (address), `cap` (wei) |
| `become:sow` | client | `projectName`, `target` (Gallina literal), `input` (development locator), `linksTo` |
| `become:sli` | client | `definition`, `unit`, `kind` (`witness` \| `query` \| `event` \| `execution`) |
| `become:slm` | client | `measures`, `source`, `system`, `method`, `frequency`, `block` |
| `become:slo` | client | `usesSli`, `comparator` (`eq` \| `gte` \| `lte`), `target`, `periodStart`, `periodEnd` |
| `become:order` | client | `schemaVersion`, `payHook`, `claimant`, `refundTo`, `bidWei`, `tier`, `openDeadline`, `fillDeadline`, `dueDate`, `nonce` |
| `become:deployment` | provider (published once) | `payHook`, `verifier`, `programVKey`, `guest`, `elfSha256`, `chainId`, `resolver`, `acceptedAssumption`, `status` |
| `become:status` | provider | `agentStatus`, `terminal`, `pollAfterMs`, `how`, `code`, `error`, `runUrl`, `stage`, `proofPresent`, `late` |
| `become:settlement` | provider | `openTx`, `fillTx`, `receiptUrl`, `settledAt` |
| `become:result` | provider | `answer`, `inbox`, `answerV`, `certified` |

Hashes: `specRoot = sha256(canonical client graphs)`;
`jobHash = sha256(specRoot || nonce)`. The `become:order` graph is hashed
too, so the money terms are part of the question. Provider graphs reference
`become:job/<jobHash>` as subject and are appended, never merged into the
hash.

## 4. Worked example: the 45+8 certified job

```
# SLA: parties, terms, consequences
<become:job/0x3b32…af2> <dc:title>                "BECOME certified sum 45+8"          <become:sla> .
<become:job/0x3b32…af2> <dc:date>                 "2026-09-10"^^xsd:date              <become:sla> .
<become:job/0x3b32…af2> <become:client>           <eth:11155111:0x11bD…2Eb>           <become:sla> .
<become:job/0x3b32…af2> <become:provider>         <eth:11155111:0x11bD…2Eb>           <become:sla> .
<become:job/0x3b32…af2> <become:securityDeposit>  "0"^^xsd:integer                    <become:sla> .
<become:job/0x3b32…af2> <become:guarantee>        "0"^^xsd:integer                    <become:sla> .
<become:job/0x3b32…af2> <become:waives>           "deposit"                            <become:sla> .
<become:job/0x3b32…af2> <become:waives>           "guarantee"                          <become:sla> .
<become:job/0x3b32…af2> <become:note>             "zero bid; self-funded; anonymous tier" <become:sla> .
<…/sla/c1> <become:onMet>  <…/slo/1>                                                   <become:sla> .
<…/sla/c1> <become:pays>   "0"^^xsd:integer                                            <become:sla> .
<…/sla/c1> <become:to>     <eth:11155111:0x11bD…2Eb>                                   <become:sla> .
<…/sla/c2> <become:onMissed> <…/slo/1>                                                 <become:sla> .
<…/sla/c2> <become:pays>   "0"^^xsd:integer                                            <become:sla> .
<…/sla/c2> <become:to>     <eth:11155111:0x11bD…2Eb>                                   <become:sla> .

# SOW
<become:job/0x3b32…af2> <become:projectName> "nat-sum"                                 <become:sow> .
<become:job/0x3b32…af2> <become:target>      "exists n : nat, 45 + 8 = n"              <become:sow> .
<become:job/0x3b32…af2> <become:input>       <nostr:event/<5700 id>>                    <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:sli>                               <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:slm>                               <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:slo>                               <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:order>                             <become:sow> .

# SLI: what is measured
<…/sli/coqchk>   <become:definition> "coqchk accepts answer.v against Question.v; assumptions within permitted set" <become:sli> .
<…/sli/coqchk>   <become:kind>       "witness"                                          <become:sli> .
<…/sli/settled>  <become:definition> "PayHook.jobs(jobHash).status"                     <become:sli> .
<…/sli/settled>  <become:kind>       "query"                                            <become:sli> .

# SLM: how
<…/slm/coqchk>   <become:measures>  <…/sli/coqchk>                                     <become:slm> .
<…/slm/coqchk>   <become:system>    "SP1 guest T2CERT0, vkey 0x00116101…388c"           <become:slm> .
<…/slm/coqchk>   <become:method>    "Groth16 proof; publicValues[0] == jobHash"         <become:slm> .
<…/slm/coqchk>   <become:frequency> "once"                                              <become:slm> .
<…/slm/settled>  <become:measures>  <…/sli/settled>                                    <become:slm> .
<…/slm/settled>  <become:system>    <eth:11155111:0x8FA8…4B8>                           <become:slm> .
<…/slm/settled>  <become:method>    "eth_call jobs(bytes32)"                             <become:slm> .
<…/slm/settled>  <become:frequency> "on fill"                                            <become:slm> .

# SLO: money-free objective
<…/slo/1> <become:usesSli>     <…/sli/settled>                                          <become:slo> .
<…/slo/1> <become:comparator>  "eq"                                                     <become:slo> .
<…/slo/1> <become:target>      "2"^^xsd:integer                                         <become:slo> .
<…/slo/1> <become:periodStart> "1789000000"^^xsd:integer                                <become:slo> .
<…/slo/1> <become:periodEnd>   "1789086400"^^xsd:integer                                <become:slo> .

# Order (money terms; hashed)
<become:job/0x3b32…af2> <become:schemaVersion> "1"^^xsd:integer                        <become:order> .
<become:job/0x3b32…af2> <become:payHook>       <eth:11155111:0x8FA8…4B8>               <become:order> .
<become:job/0x3b32…af2> <become:claimant>      <eth:11155111:0x11bD…2Eb>               <become:order> .
<become:job/0x3b32…af2> <become:refundTo>      <eth:11155111:0x11bD…2Eb>               <become:order> .
<become:job/0x3b32…af2> <become:bidWei>        "0"^^xsd:integer                        <become:order> .
<become:job/0x3b32…af2> <become:tier>          "1"^^xsd:integer                        <become:order> .
<become:job/0x3b32…af2> <become:fillDeadline>  "1789086400"^^xsd:integer               <become:order> .
<become:job/0x3b32…af2> <become:dueDate>       "1789050000"^^xsd:integer               <become:order> .

# Provider graphs (appended after the fact; signed, not hashed)
<become:job/0x3b32…af2> <become:agentStatus> "Settled"                                 <become:status> .
<become:job/0x3b32…af2> <become:terminal>    "true"^^xsd:boolean                       <become:status> .
<become:job/0x3b32…af2> <become:late>        "true"^^xsd:boolean                       <become:status> .
<become:job/0x3b32…af2> <become:fillTx>      <eth:11155111:tx/0xf80d…60f>              <become:settlement> .
<become:job/0x3b32…af2> <become:receiptUrl>  <https://sepolia.etherscan.io/tx/0xf80d…60f> <become:settlement> .
<become:job/0x3b32…af2> <become:answer>      "53"                                      <become:result> .
<become:job/0x3b32…af2> <become:certified>   "true"^^xsd:boolean                       <become:result> .
```

## 5. Projections

Every other schema is a function of the dataset.

| Projection | From graphs | Rule |
|---|---|---|
| `BecomeOrderData` (ABI) | `order` (+ `jobHash`, `specRoot`) | field per predicate; `acceptanceRuleHash = sha256(canonical sli + slm)` |
| ERC-7683 resolved order | all client graphs + `deployment` | §6 |
| Nostr 5700 tags | `order`, `sla` | `p` = `claimant`; `R` = `deployment resolver`; `x` = `jobHash`; `expiration` = `fillDeadline` |
| Nostr 5700 content | all client graphs | the canonical N-Quads, NIP-44 encrypted when `claimant` is set |
| Nostr 7000 / 6700 | `status`, `settlement`, `result` | `status` → `status` tag; `fillTx` → `settlement` tag; `result` → encrypted content |
| NIP-89 31990 | `deployment` | content is the live deployment graph as JSON |
| MCP `Job` (JSON) | `order`, `status`, `settlement`, `result`, `deployment` | snake_case of each predicate; the closed 38-field list in `BECOME-SCHEMA.md` |
| MCP `Request` (JSON) | `sow`, `order` | inverse projection: the host builds the client graphs from it |

A projection never carries information the graphs do not. If a field is
wanted in a projection, it is added as a predicate first.

## 6. Resolver: graphs to ERC-7683 (2026)

Deterministic, in order. Any failure resolves to one abort naming the rule.

1. **Validate.** Every `sow linksTo` graph is present. Every `slo usesSli`
   names an SLI with exactly one `slm measures` it. Every `sla` consequence
   names an SLO. `securityDeposit` or `guarantee` of zero requires a
   matching `waives`. `sow target` is non-empty. In v1, exactly one SLO
   (decision, §8).
2. **Variables from SLIs**, procedure from the matching SLM: `witness` →
   `Witness(kind = slm.system, data = slm.method, variables = [jobHash])`;
   `query` → `Query(target = slm.system, selector from slm.method, block =
   slm.block)`; `event` → `QueryEvents`; `execution` → `ExecutionOutput`.
3. **Steps from the SOW.** For BECOME one step: `payHook.fill(jobHash,
   publicValues, proof)`; `NeedsVariable` on every witness; `TimingBounds
   (block.timestamp, slo.periodStart, min(slo.periodEnd, order.fillDeadline))`;
   a `RevertPolicy(abort, …)` per PayHook revert reason; the SLO's
   comparator and target are the step's success condition.
4. **Payments from SLA consequences.** One payment per `onMet` clause:
   `amountFormula = pays`, `sender` = escrow, recipient = `to`,
   `onStepIdx` = the step of that SLO. One bond claim per `onMissed`
   clause, from `guarantee`, when the step aborts. (The draft's
   `amountFormula` is a constant or a variable; per-unit payment needs the
   unit count as a variable, or a precomputed amount. Not needed while
   objectives are binary.)
5. **Assumptions.** Every `waives`; every `deployment acceptedAssumption`;
   `self-funded` when `client == provider`; `native-payment`; the deployment
   pins (`verifier`, `programVKey`, `guest`); `late-allowed` when the SLA has
   no `onMissed` with a non-zero amount.

## 7. What this means for BECOME today

- `specRoot` and `acceptanceRuleHash` are no longer zero: both are hashes
  of client graphs.
- The anonymous tier is stated honestly: `client == provider`, both amounts
  zero and waived, consequences that pay zero, `self-funded` in the
  assumptions.
- `dueDate` (soft, in `order`) and `fillDeadline` (hard, the `TimingBounds`
  upper bound and the PayHook deadline) are different predicates. `late` is
  a provider-asserted status.
- The English-to-formal gap gets a home: an agent drafts the client graphs
  from a sentence; rule 1 rejects malformed requests before any stake is
  locked; `sow target` is the one literal that must still be Gallina.

## 8. Decisions (2026-09-10)

- **All schema are quads.** Projections are derived, never authored.
- **SLA holds the money; SLO is money-free.** Consequences point at SLOs
  with `onMet` / `onMissed`; `guarantee` is the `onMissed` amount.
- **Vocabulary is self-contained**, borrowing only Dublin Core `title` and
  `date` and the `xsd` datatypes. Alignment with ODRL is named future work,
  to be done as a documented translation if a second provider needs it.
- **Client graphs travel inside the 5700 event; the development travels by
  locator** (`sow input`). The agreement is small by construction and is
  verifiable against `specRoot` before anything is fetched.
- **One SLO per order in v1.** Multi-SLO settlement (one PayHook job per
  SLO, or a vector of public values in a new PayHook) is decided when a real
  second objective exists, by whether the objectives share a proof.

## 9. Open

- Per-unit payment formulas: raise with the ERC-7683 authors; a product of
  two variables is a small extension to `amountFormula`.
- Whether provider graphs are also published as a Nostr event per
  transition (kind 7000 already carries `status`; a full graph would be a
  content payload).
