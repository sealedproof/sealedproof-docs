# Validation Workflow (Third‑Party Verification)

This document defines a **repeatable, offline‑verifiable** validation workflow for SealedProof evidence packs. It is written for **security engineers, SRE/ops, platform teams, and independent auditors** who need a deterministic way to answer one question:

> **Is this agent action trace trustworthy enough to be relied upon—without trusting the producer’s infrastructure?**

---

## 1) Why “logs” are not enough in the Agent era

Traditional application logs are:
- **mutable** (admins, compromised hosts, or a pipeline bug can rewrite history),
- **context‑ambiguous** (who wrote the log? under what key? from what code path?),
- **non‑portable** (verification often requires direct access to internal systems),
- **hard to explain** (auditors get “it looked fine” instead of deterministic reasons).

SealedProof introduces a **portable evidence pack** designed for:
- **offline verification** (no dependency on producer systems),
- **non‑repudiation** (a signer cannot plausibly deny the attested event),
- **minimal disclosure** (reveal only what is required),
- **handoff‑ready** (auditable chain‑of‑custody).

---

## 2) Roles and trust boundaries

**Producer**  
The system that generates evidence for an agent action (tool calls, outputs, environment claims).

**Holder**  
The party who stores/transports the evidence pack (may be the producer, user, or a third party).

**Verifier (Third Party)**  
An independent validator (customer security, auditor, regulator, partner) running an offline verifier.

**Relying Party**  
The party that consumes the final verdict/report to make decisions (release approvals, incident triage, compliance sign‑off).

**Trust boundary**  
The verifier **does not trust**:
- producer runtime, storage, CI/CD, logs, or internal clocks,
- any unanchored “it says so” claims inside the pack.

The verifier **may trust**:
- cryptographic primitives,
- published suite definitions,
- explicit trust anchors (public keys/certs) and policy profiles.

---

## 3) Inputs and outputs

### Required inputs
1. **Evidence Pack** (the artifact under validation)
2. **Suite Registry** (which algorithms/encodings are allowed for a suiteId)
3. **Policy Profile** (the validation policy; strictness, required claims, disclosure rules)
4. **Trust Anchors** (public keys/certificates/allowlists; optional revocation metadata)

### Recommended inputs
- **Known-good time source** (optional; for strict timestamp policies)
- **Reference test vectors** (for deterministic verification across environments)

### Outputs
**Validation Report** (portable, machine-readable)
- `verdict`: PASS / FAIL / PASS_WITH_WARNINGS
- `reason_codes[]`: ordered, stable codes (human-explainable)
- `commitments`: digests/identifiers that can be compared across systems
- `policy_profile`: which policy was enforced
- `suite_id`: cryptographic suite used
- `pack_id`: unique pack identifier (derived from canonical commitments)

---

## 4) Conformance levels

Validation is profile-driven. A relying party should publish which level they require.

- **SP‑C0 (Basic Integrity)**  
  Pack integrity, manifest coverage, signatures valid, format compliant.

- **SP‑C1 (Security Baseline)**  
  SP‑C0 + domain separation checks + replay/rollback protections + key hygiene constraints.

- **SP‑C2 (Audit‑Grade)**  
  SP‑C1 + stricter timestamp policy + stronger provenance requirements + reproducibility guarantees.

> In practice: **C1** is the default enterprise baseline; **C2** is for regulated, high‑assurance pipelines.

---

## 5) End‑to‑end validation pipeline

### Stage 0 — Preflight (format & version gate)
**Goal:** ensure the verifier can interpret the pack deterministically.

Checks:
- Pack schema version supported
- `suiteId` resolvable in Suite Registry
- Policy profile found and applicable
- Required top-level files present

Fail conditions:
- Unknown schema version
- Unknown suiteId
- Missing mandatory pack components

---

### Stage 1 — Canonicalization & hashing (determinism gate)
**Goal:** the same semantic content must hash to the same commitment.

Checks:
- Canonical JSON rules applied (e.g., JCS profile required by suite)
- No non-deterministic fields in signed payloads
- Hash computations match suite specification

Fail conditions:
- Canonicalization mismatch
- Hash mismatches

---

### Stage 2 — Manifest coverage (completeness gate)
**Goal:** manifest must cover **all** pack files, and nothing is “outside the manifest”.

Checks:
- Every file in pack is listed in manifest
- No extra/untracked files
- File digests match manifest entries
- Manifest itself is included in the signing scope

Fail conditions:
- Missing file entry
- Digest mismatch
- Manifest not signed

---

### Stage 3 — Signature verification (non‑repudiation gate)
**Goal:** validate that the attestation is cryptographically bound to a known key identity.

Checks:
- Signature valid over the correct canonical bytes
- Public key resolves to an allowed trust anchor
- Key constraints satisfied (algorithm, key size, suite requirements)
- Optional: revocation / rotation rules applied

Fail conditions:
- Invalid signature
- Untrusted signer
- Forbidden algorithm/key parameters

---

### Stage 4 — Domain separation (anti‑cross‑protocol gate)
**Goal:** prevent a signature/hash from being replayed across contexts.

Checks:
- Each signature and hash includes a **domain tag** (purpose/context)
- Domain tag matches pack type and policy profile
- No ambiguous “generic signing” without context binding

Fail conditions:
- Missing/incorrect domain separation
- Reuse of commitments across contexts where forbidden

---

### Stage 5 — Replay / rollback protection (history integrity gate)
**Goal:** detect “looks valid but is not the same history” attacks.

Checks (profile-dependent):
- Event sequence numbers are monotonic (if present)
- Optional chain/ledger proofs are continuous (no missing links)
- Anchor proofs (if provided) are well-formed and consistent
- Conflicting history detection rules enforced

Fail conditions:
- Broken continuity
- Conflicting chain heads without a valid reconciliation proof
- Pack claims “latest” but proves an older state (policy-dependent)

---

### Stage 6 — Timestamp policy (time integrity gate)
**Goal:** make “when” meaningful without trusting producer clocks blindly.

Checks:
- Timestamp fields exist where required
- Internal ordering is consistent (start <= end, monotonic constraints)
- If an external time proof is required, validate it
- If no external proof exists, downgrade confidence as policy specifies

Fail conditions:
- Missing required timestamps
- Invalid time ordering
- Required time proof missing/invalid (C2 policies)

---

### Stage 7 — Minimal disclosure validation (privacy gate)
**Goal:** the pack reveals only what it must.

Checks:
- Redactions are structurally valid (commitments preserved)
- Disclosure policy satisfied (required fields revealed, optional fields hidden)
- Sensitive fields encrypted or omitted per profile

Fail conditions:
- Over-disclosure (policy violation)
- Under-disclosure (required fields absent)

---

### Stage 8 — Policy evaluation (semantic gate)
**Goal:** verify the pack meets relying-party policy requirements.

Checks:
- Required claims present (tool call identity, environment, version, policy id)
- Allowed tool-call classes only (if policy restricts)
- Required evidence types included (e.g., input hash, output hash, model id commitments)
- “No-blanket-trust” rules satisfied (no self-asserted unverifiable claims used for PASS)

Fail conditions:
- Missing required claims
- Forbidden claims/behaviors present
- Attempt to pass using unverifiable self-assertions

---

### Stage 9 — Verdict synthesis (explainability gate)
**Goal:** produce a stable result with reasons that humans can act on.

Rules:
- **FAIL** if any *hard* rule fails under the profile
- **PASS_WITH_WARNINGS** if only soft rules triggered
- **PASS** if no violations and all required checks pass

Output requirements:
- Deterministic ordering of reason codes (most severe first)
- A short, actionable explanation per code
- Include commitments/hashes needed for reproducible comparisons

---

## 6) Reason codes: how failures are explained

Reason codes are the contract between verifier and humans. They must be:
- **stable** (do not change meaning over time),
- **actionable** (tell an engineer what to fix),
- **auditable** (map to a check in a published policy).

Recommended structure:
- `E###` hard failures (verification invalid)
- `W###` warnings (verification valid but reduced assurance)
- `I###` informational notices (non-gating metadata)

Examples (illustrative; see `specs/reason-codes.md` for the canonical list):
- `E101` ManifestDigestMismatch  
- `E201` SignatureInvalid  
- `E221` SignerNotTrusted  
- `E301` DomainSeparationMissing  
- `E401` ChainContinuityBroken  
- `W501` TimestampNotExternallyProven  
- `W601` MinimalDisclosurePolicySoftViolation

---

## 7) What “counts as passing”

A pack **passes** only if:
1. **Integrity**: all referenced bytes match manifest digests
2. **Authenticity**: signatures validate against trusted anchors
3. **Context binding**: domain separation prevents cross-context replay
4. **Policy compliance**: required claims present, forbidden behaviors absent
5. **Explainability**: verifier emits a deterministic report with reason codes

A pack may be **accepted with warnings** if:
- policy allows warnings, and
- warnings do not undermine the specific reliance decision.

A pack **must fail** if:
- it can be validly interpreted in multiple ways, or
- it relies on unverifiable self-assertions to claim trust.

---

## 8) Reproducibility expectations

To be “third‑party verifiable”, verification must be reproducible:
- Two independent verifiers, same inputs ⇒ same `verdict`, same `reason_codes`, same `commitments`.
- Any differences must be surfaced as explicit reason codes (not silent divergence).

Recommended practice:
- Publish verifier version and policy profile id in every report
- Provide test vectors and negative tests (tampered pack cases)

---

## 9) Example validation report (JSON)

```json
{
  "pack_id": "spk_...",
  "suite_id": "SP-INTL-ED25519-JCS-2026",
  "policy_profile": "SP-C1-BASELINE",
  "verdict": "PASS_WITH_WARNINGS",
  "reason_codes": ["W501", "I010"],
  "commitments": {
    "manifest_digest": "sha256:...",
    "payload_digest": "sha256:...",
    "signer_key_id": "kid:..."
  },
  "verifier": {
    "name": "sealedproof-verifier",
    "version": "0.1.0",
    "build": "git:..."
  },
  "verified_at": "2026-02-15T10:00:00Z"
}
```

---

## 10) Operational guidance (CI / incident / audit)

- **CI Gate**: run validation on every agent-run export before promotion to production.
- **Incident Response**: require packs for suspicious actions; validate offline; attach report to the ticket.
- **Audit Readiness**: store packs + reports in an immutable bucket; index by pack_id and signer key id.

---

## 11) Implementation notes (non-normative)

- Profiles decide strictness. Some environments prefer “warnings don’t block”; regulated environments often require “warnings block”.
- Keep your **suite registry and policy profiles versioned**. Verification is only as good as the policy you publish.


