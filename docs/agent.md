# BECOME MCP Server — Agent Contract

**Live production host:** `https://mcp.belovedecosystem.com/mcp`  
**Contract version:** 2026-09-16 mainnet  
**Auth:** None required for default tier

---

## What BECOME is

BECOME is the **Gallina front door**: any AI agent sends a Gallina proposition (formal Coq statement), and the MCP server Coq-checks it in seconds. For certified results, the system proves the proposition on EC2 using Groth16 zero-knowledge proofs, then settles the verified result on **Ethereum mainnet** via a PayHook contract.

This is **production infrastructure**, not a demonstration:

- **Settlement chain:** Ethereum mainnet (not testnet)
- **Settlement mechanism:** PayHook contract with mainnet ETH
- **Trust boundary (TCB):** **T2CERT0** — guest certificate verification in the EC2 proving enclave
  - T2CERT0 does **not** include full `coqchk` re-verification inside the guest
  - The proving workflow trusts the EC2 attestation + certificate chain
- **Historical note:** Sepolia testnet was used for rehearsal and testing; production jobs settle on mainnet

---

## Production deployment model

### Smoke tier (anonymous, no auth)

- **Tool:** `become_orchestrate` with `certify: false`, `bid_wei: 0`
- **What runs:** `coqc` + `coqchk` on the MCP host
- **Latency:** Seconds
- **Cost:** Free
- **Output:** `agent_status: "Settled"`, `answer` (Coq numeral), `job_hash`
- **Trust:** The MCP host operator's Coq checker

Use this to validate Gallina syntax and get quick answers for simple arithmetic propositions.

### Certified tier (requires auth)

- **Tool:** `become_orchestrate` with `certify: true`
- **What runs:**
  1. Coq check on MCP host
  2. EC2 Groth16 proof generation (~60–90 minutes)
  3. Mainnet settlement via PayHook contract
- **Output:** All of smoke tier, plus:
  - `fill_tx`: Ethereum mainnet transaction hash
  - `receipt_url`: Etherscan link to settlement transaction
  - `answer.v`: Downloadable Coq proof artifact
- **Trust boundary (T2CERT0):**
  - EC2 Nitro Enclave attestation
  - Guest certificate verification
  - **Not included:** Full `coqchk` re-verification inside the proving guest
- **Cost:** Paid tier; mainnet gas costs apply

---

## Trust model & honesty

### What you can trust

1. **Job hash integrity:** `BecomeJobHash/v0` — cryptographically binds the target proposition, chain, bid, and timestamp
2. **Settlement verification:** Every certified job includes `fill_tx`; verify it yourself on Etherscan
3. **T2CERT0 attestation:** EC2 Nitro Enclave attestation proves the proof was generated in a measured, attested environment

### What you cannot trust without verification

- **`agent_status: "Settled"`** is server-asserted; always check the `fill_tx` on-chain for certified jobs
- **Smoke-tier results** (`certify: false`) trust the MCP host's Coq checker with no blockchain settlement
- **T2CERT0 limitations:** The proving guest performs certificate verification but does **not** re-run full `coqchk` on the Coq proof; the initial host-side check is trusted

### Mainnet vs testnet

- **Production:** All certified jobs (`certify: true`) settle on **Ethereum mainnet**
- **Historical:** Sepolia testnet was used during development and rehearsal (2026-06-15 through 2026-08-31)
- **No testnet mode:** There is no user-selectable testnet option in production; `chain: "sepolia"` in orchestrate calls is **ignored** and jobs settle on mainnet

---

## Tools

### `become_orchestrate` — Primary entry point

**Inputs:**

```json
{
  "target": "exists n : nat, 5 + 1 = n",
  "certify": false,
  "bid_wei": "0",
  "wait": false
}
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `target` | string | required | Gallina proposition (Coq syntax) |
| `certify` | boolean | `false` | `false` = smoke check only; `true` = EC2 prove + mainnet settlement |
| `bid_wei` | string | `"0"` | Wei amount as decimal string; zero-bid allowed on default tier |
| `chain` | string | — | **Ignored in production**; all certified jobs settle on mainnet |
| `wait` | boolean | `false` | `true` = block until settled; `false` = return job_id immediately |
| `due_date` | string | — | ISO 8601 deadline hint (not enforced) |

**Outputs:**

```json
{
  "job_id": "j_3a8f9c2b1d4e5f6a",
  "owner_token": "tok_9x8y7z6w5v4u3t2s",
  "agent_status": "Settled",
  "terminal": true,
  "answer": "6",
  "inbox": "( 6 )",
  "job_hash": "0xabcd...",
  "fill_tx": "0x1234...",
  "receipt_url": "https://etherscan.io/tx/0x1234..."
}
```

| Field | Type | Description |
|---|---|---|
| `job_id` | string | Unique job identifier; required for polling |
| `owner_token` | string | Session-bound secret; required for `become_status` / `become_result` |
| `agent_status` | enum | `Queued`, `Checking`, `Proving`, `Settling`, `Settled`, `Failed` |
| `terminal` | boolean | `true` when job is complete (Settled or Failed) |
| `answer` | string | Coq numeral result; **only trust when `agent_status: "Settled"`** |
| `inbox` | string | S-expression form of answer |
| `job_hash` | string | `BecomeJobHash/v0` of this job |
| `fill_tx` | string | (Certified jobs only) Mainnet settlement transaction hash |
| `receipt_url` | string | (Certified jobs only) Etherscan link |

**Agent rules:**

1. **Never trust `answer` or `inbox` until `agent_status: "Settled"`**
2. Keep both `job_id` **and** `owner_token`; you cannot poll without both
3. If `terminal: false`, call `become_status` after `poll_after_ms`
4. For certified jobs, **independently verify** `fill_tx` on Etherscan; do not trust the server's Settled assertion alone

---

### `become_status` — Poll job progress

**Inputs:**

```json
{
  "job_id": "j_3a8f9c2b1d4e5f6a",
  "owner_token": "tok_9x8y7z6w5v4u3t2s"
}
```

Both `job_id` and `owner_token` are required. Jobs are bound to the creating session; polling without the correct token returns `auth_required`.

**Outputs:** Same schema as `become_orchestrate`, plus:

| Field | Type | Description |
|---|---|---|
| `poll_after_ms` | integer | Milliseconds to wait before next poll |
| `progress` | string | Human-readable status (e.g., "Proof generated, awaiting settlement") |
| `how` | string | Detailed execution notes |
| `run_url` | string | GitHub Actions run URL (if `auto_generate` workflow triggered) |

---

### `become_result` — Fetch final result

**Inputs:**

```json
{
  "job_id": "j_3a8f9c2b1d4e5f6a",
  "owner_token": "tok_9x8y7z6w5v4u3t2s"
}
```

**Outputs:** Same as `become_status`, plus:

| Field | Type | Description |
|---|---|---|
| `answer_v` | string | (Optional) URL to download full Coq proof artifact |
| `visible_result` | string | Human-readable summary |

Only call this after `agent_status: "Settled"`. Calling on incomplete jobs returns partial data.

---

### `become_ask` (Advanced)

Alias for `become_orchestrate` that **requires** an explicit `bid_wei` value. Prefer `become_orchestrate` unless the user explicitly specifies a non-zero bid.

---

## Gallina target format

BECOME accepts **Gallina propositions** (Coq formal syntax), not natural-language questions.

### Supported patterns

**Arithmetic with existential:** `exists n : nat, A + B = n`

```gallina
exists n : nat, 5 + 1 = n
exists n : nat, 42 * 7 = n
exists n : nat, 1000 - 137 = n
```

**Arithmetic with universal:** `forall n : nat, n + 0 = n`

```gallina
forall n : nat, n + 0 = n
forall n m : nat, n + m = m + n
```

**Boolean/decidable propositions:**

```gallina
5 + 1 = 6
17 < 100
exists p : bool, p = true
```

### Unsupported

- **Natural-language questions:** `"What is 5 + 1?"` → Use `exists n : nat, 5 + 1 = n`
- **Non-Coq syntax:** Python, JavaScript, or other languages
- **Underspecified propositions:** `5 + 1` (not a proposition; needs `= n` or similar)

If you send natural language, the server may return `gallina_required` with a suggested `target`; rewrite and retry.

---

## Error handling

| Error code | Meaning | Resolution |
|---|---|---|
| `auth_required` | Certify or non-zero bid needs authentication | Obtain API key or enable OAuth |
| `gallina_required` | Target is not valid Gallina | Check syntax; use `exists n : nat, ...` pattern |
| `jobid_already_settled` | This exact target was already certified on mainnet | Change the proposition or accept the prior receipt |
| `not_found` | Job not found or owner_token mismatch | Verify `job_id` and `owner_token` |
| `rate_limit` | Anonymous tier quota exceeded | Wait or authenticate for higher limits |

---

## Historical context (for reference only)

### NatToy era (pre-2026-09)

Early versions of BECOME documentation referenced **NatToy** as the canonical example target. NatToy was a pedagogical Coq module for natural-number arithmetic. Production BECOME supports arbitrary Gallina propositions, not just NatToy-specific syntax.

**If you encounter NatToy references in older docs:** Treat them as historical. The modern BECOME contract is language-agnostic within Coq/Gallina.

### Sepolia testnet era (2026-06 through 2026-08)

During development, certified jobs settled on Sepolia testnet. This was **rehearsal infrastructure only**. Since 2026-09-01, all production certified jobs settle on **Ethereum mainnet**.

**Migration notes:**

- Old receipts on Sepolia remain accessible but are not production artifacts
- `chain: "sepolia"` parameter in orchestrate calls is ignored; mainnet is used
- Test/demo workflows should use `certify: false` (smoke tier) instead of testnet settlement

---

## Rate limits & quotas

### Anonymous tier (no auth)

- **Smoke checks (`certify: false`):** 30 jobs/hour/IP
- **Certified jobs (`certify: true`):** Not allowed; returns `auth_required`

### Authenticated tier (API key or OAuth)

- **Smoke checks:** Unlimited
- **Certified jobs:** 10 concurrent, 100/day (subject to change; check with operator)

---

## Verification recipe

To independently verify a certified job settled on mainnet:

1. **Get the receipt:** `become_result` returns `fill_tx` (e.g., `0x1234abcd...`)
2. **Check Etherscan:** Open `https://etherscan.io/tx/0x1234abcd...`
3. **Verify contract interaction:** Transaction should call the PayHook contract (`0x...` — contact operator for canonical address)
4. **Decode input data:** The `fill_tx` input includes the `job_hash`; verify it matches `BecomeJobHash/v0(target, chain, bid, timestamp)`

**BecomeJobHash/v0 recipe:**

```
keccak256(
  "BecomeJobHash/v0",
  keccak256(abi.encode(target)),
  chain_id,
  bid_wei,
  unix_timestamp
)
```

This is cryptographic proof that the settlement transaction corresponds to your exact job.

---

## Support & contact

- **Documentation:** This file (`/agent.md`) is the canonical contract
- **Repository:** `https://github.com/BelovedEcosystem/become-mcp-server`
- **Security disclosures:** (Add SECURITY.md contact here)
- **Operator:** Beloved Ecosystem

---

## License

This MCP server and documentation are licensed under Apache 2.0. See `LICENSE` in the repository.

---

**Last updated:** 2026-09-16  
**Contract version:** mainnet-production  
**Settlement chain:** Ethereum mainnet  
**TCB:** T2CERT0 (EC2 Nitro Enclave attestation + certificate verification; not full coqchk-in-guest)
