# Ledger Semantics (Append‑Only Continuity) (v0.1)

This document defines the **ledger semantics** used by SealedProof to detect replay, rollback, and conflicting histories.  
It does **not** mandate a specific storage backend. The goal is to standardize what it means for evidence to be **append‑only**, **continuous**, and **auditable**.

> **Key idea:** verification should not depend on trusting a database. It should depend on *provable continuity*.

---

## 0. Scope

Ledger semantics apply when a pack claims continuity across events, such as:
- ordered agent actions,
- approvals and releases,
- state transitions that must not be rolled back silently.

v0.1 defines:
- minimal continuity fields,
- chain linkage rules,
- fork/conflict handling expectations,
- optional anchoring integration points.

---

## 1. Terminology

- **Entry**: a logical record representing one event/action in a chain.
- **Chain head**: the latest entry commitment.
- **Prev**: the prior entry commitment referenced by a new entry.
- **Checkpoint**: a signed snapshot of chain head + metadata.
- **Anchor**: an external proof (timestamp / append‑only log inclusion) for a checkpoint/head.

---

## 2. Minimal ledger fields (v0.1)

A statement that participates in continuity SHOULD include `ledger` fields inside `claims`:

```json
"ledger": {
  "chainId": "agent-run:2026-02-15:abc",
  "seq": 42,
  "prev": "sha256:...",
  "head": "sha256:...",
  "checkpoint": "sha256:..." 
}
```

Field requirements:
- `chainId` (string) — identifies a continuity domain (e.g., a run, a pipeline, a tenant)
- `seq` (integer) — monotonically increasing sequence number within chainId
- `prev` (digest) — commitment to previous entry (MAY be null for genesis)
- `head` (digest) — commitment of current entry (MUST be derivable deterministically)
- `checkpoint` (digest, optional) — commitment of an endorsed checkpoint object

Policy profiles may require or relax these fields.

---

## 3. Entry commitment (`head`) derivation

Each entry `head` MUST be derived deterministically from canonical bytes under the suite.  
A recommended derivation is:

- `head = digest( domainTag || chainId || seq || prev || statementDigest || artifactsDigestSet )`

Where:
- `domainTag` is defined by suite and prevents cross-context replay,
- `statementDigest` is the digest of the canonical statement,
- `artifactsDigestSet` is a deterministic aggregation (e.g., sorted digests).

The exact derivation MUST be specified per suite or per published profile, and MUST be reproducible by independent verifiers.

---

## 4. Continuity rules

### 4.1 Monotonic sequence
For a given `chainId`, `seq` MUST be strictly increasing by 1 unless policy explicitly allows gaps (rare).  
If gaps are allowed, they MUST be justified by policy and surfaced in warnings.

### 4.2 Linkage
If `seq > 0`, `prev` MUST equal the prior entry’s `head`.  
A verifier MUST fail if:
- `prev` is missing when required,
- `prev` does not match the prior validated head.

### 4.3 Single head (no silent forks)
For a given `chainId` and `seq`, there MUST NOT be two distinct valid heads.  
If multiple heads exist, the verifier MUST emit a conflict code (`E402`) unless a reconciliation proof is provided and accepted by policy.

---

## 5. Forks and conflicts

Forks can happen due to:
- concurrent writers,
- partial outages,
- malicious attempts to rewrite history.

v0.1 handling:
- If two packs claim the same `chainId` and `seq` but different `head`, the verifier MUST treat it as a conflict.
- Policy may allow a **reconciliation statement** that:
  - references both heads,
  - is signed by an authorized role,
  - declares the canonical branch.
- Without reconciliation, verdict MUST be FAIL for audit-grade profiles.

---

## 6. Rollback detection

Rollback is presenting an older chain head as the “latest.”  
Verifiers SHOULD detect rollback by:
- comparing presented `seq` and known highest validated `seq` (if history is available), or
- validating anchors/checkpoints that bind a later head to an external timeline.

If policy requires “latest,” and the pack proves only an older state, the verifier MUST fail (`E403`).

---

## 7. Checkpoints (optional but recommended)

A checkpoint is a signed object that summarizes continuity at a point in time:

```json
{
  "chainId": "agent-run:...",
  "head": "sha256:...",
  "seq": 42,
  "issuedAt": "2026-02-15T10:00:00Z",
  "signatures": [ ... ]
}
```

Benefits:
- faster verification (verify checkpoint + recent delta),
- better anchoring (timestamp/checkpoint logging),
- clearer audit artifacts.

If included in a pack, checkpoint files MUST be listed in the manifest and integrity-checked.

---

## 8. Anchoring integration (optional)

Anchoring strengthens “when” and “cannot be retroactively rewritten,” via:
- timestamp tokens,
- transparency / append-only logs,
- witnessed external attestations.

v0.1 does not require anchoring by default. Policies (e.g., C2) may require it.

---

## 9. Validation workflow (ledger-specific)

If ledger semantics are in scope, verifiers MUST:
1. Validate statement integrity/signature as usual.
2. Recompute `head` derivation and compare to claimed `head`.
3. Enforce monotonic `seq` and `prev` linkage (given available history).
4. Detect conflicts (multiple heads).
5. Validate checkpoint/anchor proofs if provided/required.
6. Emit deterministic reason codes (`E401/E402/E403`, etc.).

---

## 10. Security considerations

- **Domain separation** is mandatory for entry commitments; otherwise cross-chain replay becomes plausible.
- **Concurrency** must be explicit: either prevent forks or provide reconciliation rules.
- **No hidden state**: continuity must be explainable with explicit commitments and signed objects.

---

**Status:** v0.1. Ledger semantics are defined at the proof layer; implementations may use any storage/transport as long as they produce verifiable continuity artifacts.

