# SealedProof Security Baseline (v0.1)

> **Baseline goal:** make AI agent actions **provable**, not merely “logged”.  
> **Four pillars:** **offline verifiability** · **non-repudiation** · **minimal disclosure** · **evidence handoff**

This document defines the **minimum security baseline** for any system that produces and verifies **evidence packages** for AI agent actions (tool calls, API calls, database writes, code changes, workflow approvals, etc.).  
It is written as a practical **industry checklist**. Use it as a gate for MVPs, vendors, and audits.

---

## 1. Scope

**In scope**

- Evidence generation for agent actions (requests, responses, side effects, approvals).
- Evidence packaging, storage, export, and third‑party verification.
- Key management, canonicalization, integrity protection, replay/rollback resistance.
- Minimal‑disclosure redaction and “handoff” between parties.

**Out of scope (for this baseline)**

- Full product UX, billing, policy engines, AI governance workflow design.
- “Truth of content” (whether a model hallucinated). This baseline proves **what happened**, **who signed**, **under which policy**, and **what was disclosed**.

---

## 2. Security objectives

A compliant system MUST meet these objectives:

1. **Offline verifiability**  
   A verifier can validate an evidence package **without network access** (except optional trust anchor refresh).

2. **Non‑repudiation**  
   The producing party cannot plausibly deny producing the package (subject to defined key custody and compromise rules).

3. **Minimal disclosure**  
   Only the minimum required fields are disclosed to the verifier; sensitive fields are redacted or selectively disclosed with proofs.

4. **Evidence handoff**  
   Packages can be transferred across organizations and remain verifiable, with clear chain‑of‑custody semantics.

---

## 3. Threat model (baseline)

### 3.1 Adversaries (examples)

- A developer/ops operator trying to rewrite history after an incident.
- A compromised agent runtime attempting to forge “clean” receipts.
- A malicious integrator swapping files inside an exported bundle.
- A verifier being misled by ambiguity (“what exactly was signed?”).
- A passive observer inferring secrets from metadata.

### 3.2 Trust assumptions (explicit)

- At least one signing key (or hardware-backed identity) remains uncompromised during the interval the evidence is produced.
- The verifier possesses (or can obtain) the public material required to validate signatures for the selected crypto suite(s).
- Time is “best-effort” unless a trusted timestamp/anchor is configured (see §7.5).

---

## 4. Normative language

- **MUST / MUST NOT**: required for baseline compliance.
- **SHOULD / SHOULD NOT**: strongly recommended; deviations require written rationale.
- **MAY**: optional.

---

## 5. Checklist: Evidence package correctness

### 5.1 Canonicalization & deterministic hashing

- **SP-BL-001**: All signed objects MUST be canonicalized deterministically (e.g., **JCS** for JSON or a fixed CBOR profile).  
- **SP-BL-002**: Hashing MUST be stable across platforms and locales (no floating ambiguity, timezone drift, key order variance).
- **SP-BL-003**: The system MUST define **hash domain separation** (different contexts produce different hash domains even with same bytes).

### 5.2 Manifest coverage (no “missing files”)

- **SP-BL-010**: Every evidence package MUST contain a **manifest** that enumerates **all files** in the package (including nested attachments), with strong digests.
- **SP-BL-011**: The manifest itself MUST be covered by a signature (directly or via an enclosing signed object).
- **SP-BL-012**: Verifiers MUST reject packages with:
  - extra files not in the manifest,
  - missing files referenced by the manifest,
  - digest mismatches.

### 5.3 Versioning & schema safety

- **SP-BL-020**: Every signed structure MUST carry an explicit **schema version**.
- **SP-BL-021**: Backward compatibility MUST be defined (allowed changes, forbidden changes).
- **SP-BL-022**: Unknown critical fields MUST fail closed (verifier rejects), not fail open.

---

## 6. Checklist: Cryptography & suite registry

### 6.1 Algorithm agility with a bounded registry

- **SP-BL-030**: The system MUST be algorithm‑agile (multiple suites), but only via a **named suite registry** (no ad‑hoc mixing).
- **SP-BL-031**: Every package MUST declare `suiteId` and MUST NOT silently change crypto primitives.
- **SP-BL-032**: Suite registry entries MUST define:
  - signature algorithm,
  - digest algorithm,
  - canonicalization rules,
  - key representation,
  - verifier behavior.

### 6.2 Signature semantics (“what exactly is signed?”)

- **SP-BL-040**: The signed bytes MUST be unambiguous and reproducible by verifiers.
- **SP-BL-041**: Multi‑signature support SHOULD exist for multi‑party workflows (producer + approver + witness).

---

## 7. Checklist: Key management & identity

### 7.1 Key custody

- **SP-BL-050**: Signing keys MUST be uniquely attributable to an organization/service identity (not shared across unrelated services).
- **SP-BL-051**: Production signing keys SHOULD be hardware‑backed (HSM/KMS/secure enclave) where feasible.

### 7.2 Rotation & revocation

- **SP-BL-052**: Key rotation MUST be supported without breaking verification for historical packages.
- **SP-BL-053**: Verifier behavior for revoked/compromised keys MUST be defined (soft fail vs hard fail; policy‑driven).

### 7.3 Key discovery (offline-friendly)

- **SP-BL-054**: A package MUST embed enough information for offline verification, such as:
  - signer public key material, or
  - certificate chain, or
  - stable key reference resolvable from a provided trust bundle.

### 7.4 Attestation (recommended)

- **SP-BL-055**: Where available, the system SHOULD support build/runtime **attestation** (e.g., signed provenance), to separate “key misuse” from “trusted runtime”.

### 7.5 Time anchors (optional but powerful)

- **SP-BL-056**: If time matters (compliance, incident response), the system SHOULD support:
  - trusted timestamping, or
  - periodic anchoring to an external immutable log, or
  - witnessed checkpoint signatures.
- **SP-BL-057**: Verifiers MUST treat unanchored timestamps as “claimed time”, not guaranteed time.

---

## 8. Checklist: Tamper-evident history (replay/rollback resistance)

If your product maintains a ledger/history of evidence (recommended):

- **SP-BL-060**: The history MUST be append‑only in semantics (logical immutability).
- **SP-BL-061**: Entries MUST be chained with forward integrity (hash‑link).
- **SP-BL-062**: Rollback/rewrite attempts MUST be detectable by third‑party verification (checkpointing, anchors, or monotonic counters).
- **SP-BL-063**: Replay attacks MUST be mitigated (nonce/sequence/correlation ID rules).

---

## 9. Checklist: Minimal disclosure & privacy

### 9.1 Data classification

- **SP-BL-070**: Each field MUST have a classification (public / internal / sensitive / secret).
- **SP-BL-071**: Packages intended for third parties MUST support a **redacted view**.

### 9.2 Redaction rules

- **SP-BL-072**: Redaction MUST be verifiable: redacted fields are replaced with commitments/digests, not silently removed.
- **SP-BL-073**: The verifier MUST be able to confirm “this redacted bundle corresponds to that original signed structure” within the allowed disclosure model.

### 9.3 PII and secrets

- **SP-BL-074**: Secrets MUST NOT be embedded in evidence by default (API keys, tokens, credentials).
- **SP-BL-075**: If payloads can contain PII, the system MUST provide:
  - configurable scrubbing,
  - retention controls,
  - safe export policies.

---

## 10. Checklist: Verifier experience & explainability

A verifier that “just says invalid” is not audit‑grade.

- **SP-BL-080**: Verification MUST return **structured reason codes** (machine‑readable) plus human explanations.
- **SP-BL-081**: Reason codes MUST distinguish at least:
  - malformed package,
  - missing/extra file,
  - digest mismatch,
  - signature invalid,
  - unsupported suite,
  - schema mismatch,
  - policy failure (e.g., key revoked),
  - timestamp/anchor failure (if enabled).
- **SP-BL-082**: Verifier outputs SHOULD be exportable (JSON) for audit systems.

---

## 11. Checklist: Operational hardening

- **SP-BL-090**: Evidence generation MUST be protected from casual bypass (authZ, least privilege, separate service identity).
- **SP-BL-091**: Evidence storage MUST be access‑controlled and monitored (RBAC + audit logs).
- **SP-BL-092**: Evidence export MUST record chain‑of‑custody metadata (who exported, when, for whom).
- **SP-BL-093**: Supply chain hardening SHOULD be applied to the build pipeline of the evidence components (reproducible builds, provenance, dependency pinning).
- **SP-BL-094**: Incident response MUST define what happens when keys are suspected compromised (freeze, rotate, mark affected time window, re‑issue trust bundle).

---

## 12. Compliance evidence (what you should be able to show)

To claim baseline compliance, you SHOULD be able to provide:

- A public **suite registry** document.
- Canonicalization test vectors + cross-language verifier parity tests.
- Sample evidence packages (valid + intentionally broken) demonstrating each reason code.
- A written key management policy (rotation, custody, revocation).
- A minimal disclosure spec (what can be redacted, how it’s proven).
- A verifier CLI (or equivalent) that runs offline and produces auditable outputs.

---

## 13. Quick self-audit (10 questions)

1. Can a third party verify your package on an air‑gapped laptop?  
2. If someone adds a file into the zip, do you catch it?  
3. If someone deletes a file, do you catch it?  
4. Is the signed data canonical and deterministic?  
5. Do you define exactly what is signed (and what isn’t)?  
6. Can you rotate keys without breaking old receipts?  
7. If a key is compromised, can you explain what is affected?  
8. Can you disclose “just enough” without leaking secrets/PII?  
9. Do you return structured reason codes, not vague errors?  
10. Can your story survive a hostile audit?

---

**Status:** v0.1 is a baseline. Future versions will add stronger requirements for selective disclosure proofs, attestation, and cross‑suite interoperability.

