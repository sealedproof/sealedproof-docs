# Suite Registry (Algorithm Suite Registry) (v0.1)

**Status:** Draft (public)  
**Audience:** implementers, security engineers, platform owners, auditors  
**Goal:** enable **algorithm agility without ambiguity** by standardizing how SealedProof names, validates, and migrates cryptographic suites.

A **suite** is a versioned bundle of:
- hash algorithm (for digests, Merkle, manifest coverage),
- signature algorithm (for statements and bindings),
- canonicalization profile (if pinned),
- key type requirements,
- optional anchoring modes (TSA / transparency log) compatibility.

> “Agility” without a registry becomes guesswork.  
> The registry makes the meaning of `suite_id` stable, reviewable, and enforceable.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

---

## 1. Suite ID format (normative)

Suite IDs are ASCII, lowercase, and deterministic.

```
sp.<family>.<hash>.<sig>.<v>
```

Where:
- `sp` — SealedProof prefix
- `<family>` — suite family (e.g., `ietf`, `cn`, `hybrid`)
- `<hash>` — hash identifier (e.g., `sha256`, `sha512`, `sm3`)
- `<sig>` — signature identifier (e.g., `ed25519`, `p256`, `rsa3072`, `sm2`)
- `<v>` — integer version (e.g., `v1`, `v2`)

Examples:
- `sp.ietf.sha256.ed25519.v1`
- `sp.ietf.sha256.p256.v1`
- `sp.ietf.sha512.ed25519.v1`
- `sp.cn.sm3.sm2.v1`

**Rules:**
- MUST match regex: `^sp\.[a-z0-9]+\.[a-z0-9]+\.[a-z0-9]+\.v[0-9]+$`
- MUST NOT include whitespace
- MUST be treated as case-sensitive (and only lowercase is valid)

---

## 2. Suite registry entry schema

A registry entry MUST define the following fields (conceptual JSON):

```json
{
  "suite_id": "sp.ietf.sha256.ed25519.v1",
  "status": "recommended | allowed | deprecated | forbidden",
  "hash": {
    "name": "SHA-256",
    "oid": "2.16.840.1.101.3.4.2.1",
    "output_bytes": 32
  },
  "signature": {
    "name": "Ed25519",
    "key_type": "ed25519",
    "public_key_encoding": "raw | spki",
    "signature_encoding": "raw"
  },
  "canonicalization": {
    "profile": "sp-jcs-v1",
    "spec": "specs/canonicalization-jcs-profile.md"
  },
  "constraints": {
    "min_key_strength_bits": 128,
    "allow_rfc3161_tsa": true,
    "allow_transparency_log": true
  },
  "migration": {
    "supersedes": [],
    "superseded_by": ["sp.ietf.sha256.ed25519.v2"]
  }
}
```

This document specifies required semantics; a concrete machine-readable registry file MAY exist (see §9).

---

## 3. Status levels and enforcement

### 3.1 Status definitions
- **recommended**: default for new deployments under strict profiles
- **allowed**: usable, but not the default recommendation
- **deprecated**: allowed only for legacy verification / grace periods
- **forbidden**: MUST be rejected

### 3.2 Verification behavior
A verifier MUST:
- read `suite_id` from the signed object’s domain label (see `specs/signing-hash-domain-separation.md`),
- look up the suite entry,
- enforce profile-specific rules (e.g., “no deprecated suites”),
- fail closed if suite is unknown or forbidden.

---

## 4. Compatibility strategy (interop)

### 4.1 Cross-ecosystem exports
When exporting to other ecosystems (e.g., in-toto statements), SealedProof MUST preserve:
- the original `suite_id`,
- the original digests/signature bindings,
- a mapping layer that does not alter the cryptographic meaning.

### 4.2 Multi-suite verification
A verifier SHOULD support verifying evidence packs produced under multiple suites, provided:
- the pack’s profile allows them,
- the suite IDs are known and allowed,
- mixed-suite packs follow explicit rules (see §5).

---

## 5. Mixed suites (policy-driven)

Mixed-suite packs are risky because they can create “weakest-link” confusion.

**Baseline rule:** A pack SHOULD use a single suite.  
If mixing is allowed, deployments MUST define:

- which objects may differ (e.g., file digests vs statement signature),
- which suite governs top-level validity,
- how verifiers decide “overall pass”.

Recommended approach:
- a top-level **pack suite** governs the envelope signature and manifest,
- subordinate suites MAY exist only for embedded legacy artifacts, explicitly labeled and policy-approved.

---

## 6. Migration strategy (agility without chaos)

### 6.1 Migration triggers
Suites are migrated due to:
- algorithm deprecation,
- compliance requirements,
- ecosystem alignment,
- performance/compatibility constraints.

### 6.2 Backward verification
Verifiers MUST be able to verify deprecated suites for a defined grace period, if policy requires.
Verification outputs SHOULD clearly indicate:
- `suite_id`,
- `status` at verification time,
- whether the suite is deprecated.

### 6.3 Forward issuance
Producers MUST NOT issue new packs with deprecated suites under strict profiles, unless explicitly overridden.

### 6.4 Dual-anchoring (recommended)
During migrations, producers SHOULD:
- sign with the new suite,
- maintain verifiable linkage to prior packs (e.g., by embedding prior pack digest commitments),
- optionally anchor both via TSA/log for continuity.

---

## 7. CN vs International suites (policy-ready)

Many enterprises require both “international” and “domestic” crypto options.

The registry supports parallel families:
- `sp.ietf.*` — international suites aligned with common IETF ecosystems
- `sp.cn.*` — domestic suites (e.g., SM3/SM2) for regulated environments
- `sp.hybrid.*` — hybrid strategies if required by policy

Verification workflow remains identical; only `suite_id` changes, which is explicitly bound by domain separation.

---

## 8. Conformance requirements

A conforming implementation MUST:
- treat `suite_id` as an explicit input to hashing/signing verification,
- reject unknown suites in strict mode,
- emit stable reason codes for:
  - unknown suite,
  - forbidden suite,
  - deprecated suite under disallowing profile,
  - suite/key type mismatch.

Reason code conventions SHOULD align with `conformance/validation-workflow.md`.

---

## 9. Registry publication (recommended)

SealedProof SHOULD publish:
- a human-readable registry doc (this file),
- a machine-readable `registry/suites.json` (optional), signed by a SealedProof Registry key,
- conformance test vectors per suite.

Registry updates SHOULD be versioned and changelogged.

---

## 10. Security considerations

- Algorithm agility is a security feature only if suite meaning is explicit and verifiable.
- Avoid “silent fallback”. Unknown suite MUST not be guessed.
- Mixed suites should be used only with explicit policy and clear verification semantics.

---

**Status:** v0.1 baseline. This registry is expected to evolve with ecosystem feedback and conformance testing.

