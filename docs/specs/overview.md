# SealedProof Specifications Overview

**Status:** Draft (public)

This document is the navigational overview of the SealedProof spec set: what each spec covers, how the parts fit, and what “compliant” means in enterprise verification terms.

> **Normative keywords**: **MUST**, **MUST NOT**, **SHALL**, **SHOULD**, **MAY** are used as in RFC 2119.

---

## 1. SealedProof in one line

SealedProof defines a **verifiable evidence layer for AI Agent actions**: **offline verifiable**, **non‑repudiable**, **minimal disclosure**, and **handoff‑ready** across organizations.

---

## 2. What we are standardizing

In many systems, the “proof” of an agent action is mutable or non-portable. SealedProof standardizes portable artifacts and verification procedures so any verifier can answer offline:
- Did the execution happen?
- Was it altered?
- Who attested to it?
- Is disclosure minimal?
- Can I re‑verify it outside the producer’s environment?

---

## 3. Spec-level architecture (artifacts, not implementation)

SealedProof defines three portable artifacts:
1) **Evidence Pack** — deterministic bundle with full coverage and digests.  
2) **Proof Envelope** — binding layer: suiteId, digests, signatures, optional anchors.  
3) **Verification Result** — deterministic verdict + stable reason codes for automation.

Implementation is intentionally out‑of‑scope; specs define **verifiable outputs** and **acceptance criteria**.

---

## 4. Spec map (high-level)

- Concepts: `concepts/trust-model.md`
- Core: `specs/evidence-pack-format.md`, `specs/pack-manifest.md`, `specs/ledger-semantics.md`, `specs/proof-envelope.md`
- Verification contract: `specs/offline-verifier-cli.md`, `specs/reason-codes.md`
- Security: `security/*`
- Conformance: `conformance/*`
- Interop: `specs/canonicalization-jcs-profile.md`, `specs/signing-hash-domain-separation.md`, `specs/suite-registry.md`, `integrations/*`
- Roadmap: `roadmap.md`

---

## 5. What “compliant” means

A producer/verifier pair is compliant if:
- Evidence Packs can be handed off to third parties,
- offline verification yields deterministic results,
- failures map to stable reason codes,
- suites/profiles are explicit and interoperable,
- breaking changes follow compatibility policy.

---

## 6. Non-goals

No requirement on runtime/storage/DB; no requirement on a specific log/TSA; no forced disclosure of sensitive data; no dependency on online verification.

---

## 7. Reading order

1) `concepts/trust-model.md`  
2) `security/security-baseline.md`  
3) `specs/evidence-pack-format.md` → `specs/pack-manifest.md` → `specs/proof-envelope.md`  
4) `specs/offline-verifier-cli.md` + `specs/reason-codes.md`  
5) `conformance/validation-workflow.md` + `conformance/test-vectors.md`  
6) Interop/integrations

---

## 8. Change control

Specs MUST evolve under `conformance/compatibility-policy.md`. Breaking changes SHALL increment major version, ship migration guidance, and update test vectors.
