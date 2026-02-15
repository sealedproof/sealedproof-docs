# Replay / Rollback / Fork Defense (v0.1)

This document defines the **security baseline** for defending against **replay**, **rollback**, and **fork/conflict** in SealedProof deployments.  
It is written for **enterprise security, audit, SRE/ops**, and external reviewers (e.g., SCITT/IETF audiences).

This document is **policy‑aware**:
- The format capabilities are defined in `specs/*`.
- Conformance and third‑party verification flow is defined in `conformance/validation-workflow.md`.
- Reason codes are standardized in `specs/reason-codes.md`.

> **Goal:** When evidence crosses trust boundaries, the verifier must be able to detect “same proof reused”, “history rewritten”, and “two conflicting histories” — *offline*, with deterministic explanations.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative as in RFC 2119.

---

## 1. Problem statement

### 1.1 Replay 
An attacker reuses a previously valid proof (or parts of it) to falsely claim a new action occurred, or to pass a control gate without performing the required action.

### 1.2 Rollback 
An attacker presents an older, previously valid state as if it were the latest state, hiding newer events (e.g., failed checks, revoked approvals, patched vulnerabilities).

### 1.3 Fork / conflict 
Two incompatible histories are presented for the same continuity domain (same run/pipeline/tenant), enabling “choose your own history” attacks.

**Why logs alone fail:** logs are mutable by operators, often online‑only, and do not provide portable, offline verifiability. Evidence Packs create a **sealed, transferable proof boundary**.

---

## 2. Threat model scope

This document assumes:
- Verifiers may be offline and cannot trust the producer’s database.
- Producers may be honest-but-buggy or partially compromised.
- Network paths may be hostile (MITM, replay).
- Storage systems may be rolled back (snapshot restore), or selectively deleted.

This document does not attempt to prevent all attacks; it standardizes what evidence is required to **detect** them.

---

## 3. Definitions

- **Continuity domain (`chainId`)**: the scope in which order and non‑rollback are meaningful (e.g., “agent run”, “deployment pipeline”, “tenant ledger”).
- **Entry head (`head`)**: a deterministic commitment to an event/statement in that continuity domain.
- **Prev link (`prev`)**: pointer to the prior entry head.
- **Sequence (`seq`)**: monotonically increasing index within `chainId`.
- **Checkpoint**: a signed summary of (`chainId`, `seq`, `head`) at a point in time.
- **Anchor**: an external, append‑only or timestamp proof binding a checkpoint/head to a timeline (optional by policy).

Ledger semantics are defined in `specs/ledger-semantics.md`.

---

## 4. Security requirements (baseline)

### 4.1 Domain separation (replay across contexts)
Producers MUST implement **domain separation** for all commitments used in continuity:
- entry head derivation,
- packId derivation,
- signing scopes.

A verifier MUST fail if domain separation is missing where required by suite/policy.  
(See `specs/signing-hash-domain-separation.md` when published.)

### 4.2 Unique action correlation (replay within a domain)
Statements participating in continuity SHOULD include stable correlation identifiers:
- `actionId` (required by `specs/evidence-pack-format.md` minimal model),
- and (recommended) `runId` / `pipelineId` / `ticketId` depending on domain.

Policy MAY require uniqueness constraints (e.g., `actionId` must not repeat within `chainId`).

### 4.3 Monotonic sequencing and linkage (rollback/fork)
For audit‑grade profiles, continuity statements MUST include:
- `chainId`, `seq`, `prev`, `head` (see `specs/ledger-semantics.md`).
Verifiers MUST enforce:
- `seq` monotonicity under available history,
- `prev` linkage correctness,
- `head` determinism (recomputed vs claimed).

### 4.4 Checkpoints and anchoring (strong rollback defense)
If policy requires time‑binding or anti‑rollback beyond local history, producers MUST provide:
- **checkpoint objects** (signed),
- and one of: **timestamp tokens** or **transparency / append‑only inclusion proofs** as anchors.

If policy does not require anchoring, producers SHOULD still provide checkpoints to improve auditability and future proofing.

---

## 5. Detection strategies

### 5.1 Detecting cross‑context replay
**Threat:** A valid artifact or signature from Context A is reused in Context B.

**Detection MUST include:**
- Domain separation tags bound into all commitment derivations.
- Statement `subject` must bind to artifacts by digest, not path.
- Policy checks that disallow digest reuse across forbidden contexts (profile-defined).

**Verifier outputs:**
- `E301/E302` for missing/mismatched domain separation.
- `E303` for cross-context replay detected (policy-controlled).

### 5.2 Detecting intra-domain replay
**Threat:** A previously valid statement is replayed as a new event within the same chain.

**Detection SHOULD include (policy-dependent):**
- `seq` strictly increasing.
- Uniqueness of `actionId` within `chainId`.
- (Optional) nonces or per-run monotonic counters, recorded as artifacts and sealed.

**Verifier outputs:**
- `E701` (required claim missing), `E401` (continuity broken), or policy-specific replay codes.

### 5.3 Detecting rollback
**Threat:** An older chain head is presented as the current head.

**Detection MUST include one of:**
- Verifier maintains local memory of the highest validated `seq/head` for the domain (when applicable), OR
- Anchored checkpoints proving a later head existed at a later time, OR
- Cross-pack reconciliation logic (third-party sources) under policy.

**Verifier outputs:**
- `E403 RollbackSuspected` when policy requires “latest” and a downgrade is provable or strongly indicated.
- `W501 TimestampNotExternallyProven` when only claimed timestamps exist.

### 5.4 Detecting forks / conflicting heads
**Threat:** Two different valid heads exist for the same `chainId` and `seq` (or “latest”).

**Detection MUST include:**
- `head` derivation check (deterministic recomputation).
- Conflict detection when multiple heads exist for same position.
- (Optional) reconciliation statement acceptance only under policy, signed by authorized roles.

**Verifier outputs:**
- `E402 ConflictingChainHeads` unless reconciliation is present and policy allows.
- `E401 ChainContinuityBroken` for missing links.

---

## 6. Mitigation patterns (how producers should build evidence)

This section is **implementation‑agnostic**. It standardizes what producers MUST be able to show.

### 6.1 Single-writer chain (recommended where feasible)
- One authoritative producer writes the chain for a `chainId`.
- Each new statement includes `prev/head/seq`.
- Periodic checkpoints are signed by an approver/witness role.

Pros: simplest, strongest guarantees.  
Cons: requires clear ownership.

### 6.2 Multi-writer chain (allowed, requires reconciliation)
- Multiple writers may emit entries for the same `chainId`.
- Producers MUST include fork evidence and a reconciliation statement when conflicts occur.
- Reconciliation MUST be signed by an authorized role and MUST reference conflicting heads.

Pros: supports distributed systems.  
Cons: more complex; policy must be explicit.

### 6.3 Checkpoint + anchor (strong anti-rollback)
- Produce checkpoints at meaningful boundaries (e.g., “end of run”, “approval granted”, “release to prod”).
- Anchor checkpoints using timestamp authority tokens or transparency log inclusion.
- Include anchor proof files under `anchors/` and cover them by manifest.

---

## 7. Required evidence artifacts (what goes into the pack)

For profiles that claim rollback/fork resistance, a pack MUST contain:

1. **Continuity claims** inside statements:
   - `chainId`, `seq`, `prev`, `head` (as required by policy).
2. **Deterministic derivation inputs**:
   - statement digests (canonical form),
   - artifact digest set referenced by the statement.
3. **Checkpoint objects** (if policy requires/producer provides):
   - signed checkpoint JSON under a manifest entry (`role=anchor` or `role=other` per profile).
4. **Anchor proofs** (if policy requires):
   - timestamp token files or transparency inclusion proofs.
5. **Trust material** (if offline trust is required):
   - cert chain / allowlist / suite registry snapshot under `trust/`.

All files MUST be listed in the manifest and digest-checked.

---

## 8. Verification procedure (minimum)

A verifier performing replay/rollback/fork checks MUST:

1. Perform base pack validation (manifest, digests, signatures) as per `specs/evidence-pack-format.md`.
2. Recompute each continuity statement’s `head` deterministically and compare.
3. Enforce sequencing/linkage rules under available history:
   - if history is available, enforce strict `prev` linkage and monotonic `seq`;
   - if history is not available, enforce internal consistency and require anchors if policy demands stronger guarantees.
4. Validate checkpoint signatures and anchor proofs when present/required.
5. Detect conflicts across packs when the verifier has multiple packs for the same `chainId`.
6. Emit deterministic verdict and reason codes.

---

## 9. Reason codes mapping (baseline)

This document relies on standard codes from `specs/reason-codes.md`, including:

- Domain separation: `E301`, `E302`, `E303`
- Continuity: `E401`, `E402`, `E403`
- Timestamp/anchors: `E501`, `E502`, `W501`
- Signature scope: `E201`, `E202`

Producers SHOULD test against these codes using conformance test vectors.

---

## 10. Operational checklist (for security & SRE)

### 10.1 Producer checklist
- [ ] Commitments use suite-defined domain separation.
- [ ] Every pack contains a complete manifest; no out-of-manifest bytes.
- [ ] Continuity-enabled statements carry `chainId/seq/prev/head`.
- [ ] Checkpoints exist at clear boundaries.
- [ ] Anchors are included when policy requires anti-rollback guarantees.
- [ ] Secrets are excluded by default; redaction model is explicit when used.

### 10.2 Verifier checklist
- [ ] Enforce policy profile strictly (fail closed on critical items).
- [ ] Do not accept claimed time as proof unless anchored (profile-dependent).
- [ ] Detect and report conflicting heads; do not “pick one” silently.
- [ ] Persist highest validated heads when operating in continuous mode (if allowed).
- [ ] Emit stable reason codes; never “unknown error”.

---

## 11. Common pitfalls (avoid)

- **Signing the wrong view**: signatures must cover the correct signing scope; otherwise replay becomes possible.
- **Path-based artifact references**: always reference by digest; paths are metadata.
- **No checkpoint discipline**: without checkpoints/anchors, “latest” claims become weak.
- **Fork tolerance without policy**: allowing multi-head without reconciliation invites history manipulation.

---

**Status:** v0.1 baseline. This document will be refined alongside Suite Registry profiles and conformance test vectors.

