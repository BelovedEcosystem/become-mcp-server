# Service Request as quads: SLA, SLO, SLI, SLM over ERC-7683

Status: design, 2026-09-10. Generalizes the BECOME order (see
[`BECOME-SCHEMA.md`](BECOME-SCHEMA.md)) into a full Service Request encoded
as **Context . Subject . Predicate . Object** quads, and maps that encoding
onto the ERC-7683 (2026) resolved order. Nothing here is implemented.

## 1. Why quads, and why they fit ERC-7683

A Service Request is an agreement (SLA) that names objectives (SLOs), each
measured by indicators (SLIs) through a stated measurement (SLM), with a
payment tied to each objective. That is four kinds of statement about a few
named things. Quads say exactly that: a named graph (Context) holding
Subject-Predicate-Object triples.

The 2026 ERC-7683 resolved order is the same shape from the solver's side:

| Service Request | ERC-7683 (2026) resolved order |
|---|---|
| **SLI** (what is measured) | a **Variable**: `Query`, `QueryEvents`, `ExecutionOutput` or `Witness` |
| **SLM** (how, where, how often it is measured) | the variable's procedure: target, selector, block, witness kind; frequency as `TimingBounds` |
| **SLO** (target, period, payment per unit, success criterion) | a **Payment** whose `amountFormula` references the SLI variable, gated by a **Step** with `TimingBounds` and `RevertPolicy` |
| **SLA** (title, effective date, deposit, guarantee, notes) | the order envelope and escrow; a solver bond; **Assumptions** for anything waived or not enforced |
| **SOW** (what work, which sections it links) | the **Steps** list and the required contexts |

So a quad dataset can be the ERC-7683 **payload**, and the resolver becomes
a deterministic translator from quads to steps, variables, payments and
assumptions. The old validator rules (counts, SLI to SLM to SLO linkage,
required SOW links) become the resolver's "MUST" checks: a dataset that fails
them resolves to a single abort.

## 2. Encoding rules

- **Format:** N-Quads (one quad per line, `<subject> <predicate> <object> <context> .`).
  Subjects, predicates and contexts are IRIs; objects are IRIs or typed
  literals. No blank nodes, so canonicalization is a sort.
- **Canonical hash:** `specRoot = sha256(canonical N-Quads)`, where canonical
  means sorted lines, UTF-8, `\n` terminated. This is the manifest hash the
  DVM spec asks for, and it now covers the SLA, not only the Coq tree.
- **Namespace:** `become:` = `https://mcp.belovedecosystem.com/ns/v1#`.
  Contexts are `become:sla`, `become:sow`, `become:sli`, `become:slm`,
  `become:slo`, `become:order`, `become:deployment`.
- **Subjects:** the job is `become:job/<jobHash>`; SLOs, SLIs, SLMs are
  `become:job/<jobHash>/slo/1`, `/sli/coqchk`, `/slm/coqchk`, and so on.
- **Order id:** `jobHash = sha256(specRoot || nonce)`. The quads are the
  question; the answer is never in them.

## 3. Predicate vocabulary (v1)

| Context | Predicates |
|---|---|
| `become:sla` | `title`, `effectiveDate`, `client`, `provider`, `securityDeposit` (wei), `guarantee` (wei, solver bond), `note`, `waives` |
| `become:sow` | `projectName`, `linksTo` (one per required context), `input` (IRI of the encrypted development), `target` (the proposition, as literal) |
| `become:sli` | `definition`, `basedOn`, `unit`, `kind` (`witness` / `query` / `event` / `execution`) |
| `become:slm` | `measures` (an SLI), `source`, `system`, `method`, `frequency`, `block` |
| `become:slo` | `name`, `usesSli`, `target`, `comparator` (`eq`, `gte`, `lte`), `periodStart`, `periodEnd`, `paymentPerUnit` (wei), `paymentCap` (wei), `successCriteria`, `recipient` |
| `become:order` | `schemaVersion`, `payHook`, `claimant`, `refundTo`, `bidWei`, `tier`, `openDeadline`, `fillDeadline`, `dueDate` (soft), `payloadHash`, `payloadLocator` |
| `become:deployment` | `payHook`, `verifier`, `programVKey`, `guest`, `elfSha256`, `chainId`, `acceptedAssumption` |

`become:order` is `BecomeOrderData` written as quads; the ABI struct in
`BECOME-SCHEMA.md` is its binary form for the chain. Both hash to the same
`jobHash`, because `jobHash` is computed from the quads.

## 4. Worked example: the 45+8 certified job

```
# SLA
<become:job/0x3b32…af2> <become:title>           "BECOME certified sum 45+8"        <become:sla> .
<become:job/0x3b32…af2> <become:effectiveDate>   "2026-09-10"^^xsd:date            <become:sla> .
<become:job/0x3b32…af2> <become:client>          <eth:11155111:0x11bD…2Eb>         <become:sla> .
<become:job/0x3b32…af2> <become:provider>        <eth:11155111:0x11bD…2Eb>         <become:sla> .
<become:job/0x3b32…af2> <become:securityDeposit> "0"^^xsd:integer                  <become:sla> .
<become:job/0x3b32…af2> <become:guarantee>       "0"^^xsd:integer                  <become:sla> .
<become:job/0x3b32…af2> <become:waives>          "deposit"                          <become:sla> .
<become:job/0x3b32…af2> <become:note>            "zero bid; self-funded; anonymous tier" <become:sla> .

# SOW
<become:job/0x3b32…af2> <become:projectName> "nat-sum"                              <become:sow> .
<become:job/0x3b32…af2> <become:target>      "exists n : nat, 45 + 8 = n"           <become:sow> .
<become:job/0x3b32…af2> <become:input>       <nostr:event/<5700 id>>                 <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:sli>                            <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:slm>                            <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:slo>                            <become:sow> .
<become:job/0x3b32…af2> <become:linksTo>     <become:order>                          <become:sow> .

# SLIs: what is measured
<…/sli/coqchk>   <become:definition> "coqchk accepts answer.v against Question.v; Print Assumptions within permitted set" <become:sli> .
<…/sli/coqchk>   <become:kind>       "witness"                                       <become:sli> .
<…/sli/verified> <become:definition> "PayHook.jobs(jobHash).status == 2"             <become:sli> .
<…/sli/verified> <become:kind>       "query"                                         <become:sli> .

# SLMs: how each SLI is measured
<…/slm/coqchk>   <become:measures>  <…/sli/coqchk>                                  <become:slm> .
<…/slm/coqchk>   <become:system>    "SP1 guest T2CERT0, vkey 0x00116101…388c"        <become:slm> .
<…/slm/coqchk>   <become:method>    "Groth16 proof; publicValues[0] == jobHash"      <become:slm> .
<…/slm/coqchk>   <become:frequency> "once"                                           <become:slm> .
<…/slm/verified> <become:measures>  <…/sli/verified>                                <become:slm> .
<…/slm/verified> <become:system>    <eth:11155111:0x8FA8…4B8>                        <become:slm> .
<…/slm/verified> <become:method>    "eth_call jobs(bytes32)"                          <become:slm> .
<…/slm/verified> <become:frequency> "on fill"                                         <become:slm> .

# SLO: the objective and its payment
<…/slo/1> <become:name>            "certified answer settled"                        <become:slo> .
<…/slo/1> <become:usesSli>         <…/sli/verified>                                  <become:slo> .
<…/slo/1> <become:comparator>      "eq"                                              <become:slo> .
<…/slo/1> <become:target>          "2"^^xsd:integer                                  <become:slo> .
<…/slo/1> <become:periodStart>     "2026-09-10T18:00:00Z"^^xsd:dateTime              <become:slo> .
<…/slo/1> <become:periodEnd>       "2026-09-11T18:00:00Z"^^xsd:dateTime              <become:slo> .
<…/slo/1> <become:paymentPerUnit>  "0"^^xsd:integer                                  <become:slo> .
<…/slo/1> <become:paymentCap>      "0"^^xsd:integer                                  <become:slo> .
<…/slo/1> <become:successCriteria> <…/sli/coqchk>                                    <become:slo> .
<…/slo/1> <become:recipient>       <eth:11155111:0x11bD…2Eb>                         <become:slo> .

# Order (BecomeOrderData as quads)
<become:job/0x3b32…af2> <become:payHook>      <eth:11155111:0x8FA8…4B8>              <become:order> .
<become:job/0x3b32…af2> <become:claimant>     <eth:11155111:0x11bD…2Eb>              <become:order> .
<become:job/0x3b32…af2> <become:bidWei>       "0"^^xsd:integer                       <become:order> .
<become:job/0x3b32…af2> <become:tier>         "1"^^xsd:integer                       <become:order> .
<become:job/0x3b32…af2> <become:fillDeadline> "1789086400"^^xsd:integer              <become:order> .
<become:job/0x3b32…af2> <become:dueDate>      "1789050000"^^xsd:integer              <become:order> .
```

## 5. Resolver: quads to ERC-7683 (2026)

Deterministic rules, in order. Any failure resolves to one abort with the
rule name.

1. **Validate the dataset.** Every `become:sow linksTo` context is present.
   Every SLO `usesSli` and `successCriteria` names an SLI. Every SLI has
   exactly one SLM `measures` it. `securityDeposit` or `guarantee` of zero
   requires a `waives` quad (the old "zero must have a waiver note" rule).
2. **Variables from SLIs.** `kind=witness` becomes `Witness(kind=slm.system,
   data=slm.method, variables=[jobHash])`. `kind=query` becomes
   `Query(target=slm.system, selector from slm.method, block=slm.block)`.
   `kind=event` becomes `QueryEvents`. `kind=execution` becomes
   `ExecutionOutput`.
3. **Steps from the SOW.** For BECOME one step: `payHook.fill(jobHash,
   publicValues, proof)` with `NeedsVariable` on every witness the SLOs use,
   `TimingBounds(block.timestamp, periodStart, min(periodEnd, fillDeadline))`,
   `RevertPolicy(abort, …)` for each PayHook revert.
4. **Payments from SLOs.** One payment per SLO: `amountFormula =
   paymentPerUnit × units(sliVariable)` capped at `paymentCap`, `sender` =
   the escrow, `recipientVarIdx` = `PaymentRecipient`, `onStepIdx` = the
   step whose `TimingBounds` covers the SLO period. The `comparator` and
   `target` become the step's success condition; when unmet, the revert
   policy aborts and no payment fires.
5. **Assumptions from the SLA and deployment.** Every `waives` quad, every
   `become:deployment acceptedAssumption`, plus `self-funded` when
   `client == provider`, `native-payment`, and the deployment pins.

Limitation to note: the 2026 draft's `amountFormula` is a constant or a
variable. Per-unit payment therefore needs the units to be a variable the
solver can obtain (a `Query` or `ExecutionOutput`), or the formula is
precomputed at order time. For BECOME today every SLO is binary and
`paymentPerUnit` equals the bid, so the limitation does not bite.

## 6. What this changes for BECOME

- `specRoot` stops being zero. It is the hash of the canonical quads, and the
  Coq development is referenced from the SOW rather than hashed alone.
- The acceptance rule (permitted axioms, Coq version, flags) is quads in
  `become:sli`, so `acceptanceRuleHash` is derivable and no longer separate.
- The English-to-formal gap gets a home: an agent drafts quads from a
  sentence, the validator in step 1 rejects malformed requests before any
  stake is locked, and `become:target` is the one literal that still has to
  be Gallina.
- The Nostr 5700 event does not change. Its encrypted content becomes the
  quads instead of the JSON bundle; the tags stay `p`, `R`, `x`,
  `expiration`, `relays`.
- The MCP `Request` gains an optional `quads` input; when present it is the
  ask and `target` must equal `become:sow target`. The MCP `Job` gains
  `spec_root` filled and `sla` (a compact rendering of the SLA and SLOs).

## 7. Open decisions

- Whether `become:` predicates reuse existing vocabularies (Dublin Core for
  `title`/`date`, the W3C ODRL policy vocabulary for permissions and duties)
  or stay self-contained. Self-contained is simpler; reuse is more legible to
  tools that already speak RDF.
- Whether the quads travel in full inside the 5700 content or only
  `specRoot` plus a locator, with the dataset fetched from the locator.
- Where multi-SLO orders settle: one PayHook job per SLO, or one job with a
  vector of public values. The latter needs a PayHook change.
