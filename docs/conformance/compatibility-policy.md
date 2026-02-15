# Compatibility Policy (Versioning & Deprecation) (v0.1)

**Path:** `conformance/compatibility-policy.md`  
**Status:** Draft (public)  
**Audience:** platform owners, implementers, auditors, integrators  
**Goal:** provide predictable, enterprise-grade rules for:
- version semantics,
- breaking changes,
- deprecation windows,
- interoperability guarantees.

This policy is designed for regulated environments where “silent changes” are unacceptable.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

---

## 1. Compatibility objectives

SealedProof prioritizes:
1) **Verifiability stability** — a pack verified today should verify tomorrow under stated policies.
2) **Explicitness** — all semantics that affect verification must be versioned and bound.
3) **Fail-closed safety** — unknown or ambiguous behavior must not be accepted in strict mode.
4) **Interop-first evolution** — changes ship with vectors and migration guidance.

---

## 2. Version surfaces (what is versioned)

Compatibility depends on multiple layers. Each layer MUST be versioned:

- **Pack format version** (evidence pack schema)
- **Canonicalization profile version** (e.g., `sp-jcs-v1`)
- **Suite ID version** (e.g., `...v1`, `...v2`)
- **Policy/profile version** (verification rules)
- **Verifier version** (implementation behavior)

A pack MUST declare (directly or indirectly via profile):
- pack format version,
- suite_id,
- canonicalization profile,
- policy/profile identifier.

---

## 3. Semantic versioning rules (normative)

SealedProof follows “verification-semantic versioning” for public specs.

### 3.1 Spec documents
- **Major**: breaking verification semantics (old verifiers cannot safely verify new artifacts)
- **Minor**: additive changes, optional fields, new suites with clear policy gating
- **Patch**: clarifications, editorial changes, non-semantic fixes

### 3.2 Pack format versions
Pack format versions are treated as **major semantic boundaries**:
- A verifier MUST reject unknown pack format versions in strict mode.
- A verifier MAY accept unknown versions only if configured explicitly (non-strict).

### 3.3 Suite versions
Suite versions (`...vN`) are semantic:
- Increasing suite version may change algorithms, encodings, or constraints.
- A verifier MUST NOT treat `v2` as compatible with `v1` unless registry/policy explicitly states so.

---

## 4. What counts as a breaking change

A change is **breaking** if it can cause:
- a previously valid pack to fail verification under the same policy, or
- a previously invalid pack to pass verification under the same policy, or
- ambiguous verification outcomes across implementations.

Examples (breaking):
- changing canonicalization rules
- changing digest encoding rules (hex vs base64url)
- changing domain label format without bumping namespace version
- changing manifest coverage semantics (e.g., strictness)
- changing reason code meaning

Non-breaking (minor) examples:
- adding optional metadata fields that are excluded from signing
- adding new suites as `allowed` with policy gating
- adding optional anchors

---

## 5. Deprecation windows (enterprise-friendly)

### 5.1 Deprecation phases
SealedProof uses four phases:

1) **Recommended** — default for new issuance
2) **Allowed** — permitted but not default
3) **Deprecated** — permitted for verification only (issuance discouraged or blocked by strict profiles)
4) **Forbidden** — must be rejected

These map directly to Suite Registry `status` levels.

### 5.2 Minimum windows (baseline)
Unless urgent security requires shorter timelines:
- **Deprecated window** SHOULD be ≥ 12 months for enterprise deployments.
- **Allowed grace** SHOULD be ≥ 6 months before deprecating a widely used suite.

Deployments MAY choose stricter windows.

### 5.3 Emergency deprecation
If a critical break is discovered, SealedProof MAY:
- mark a suite as deprecated immediately,
- provide a migration suite and vectors,
- publish a security advisory.

Strict profiles MAY enforce emergency deprecation instantly.

---

## 6. Interoperability guarantees

For each released spec version, SealedProof commits to:
- publishing conformance vectors for Core behaviors,
- maintaining reason code stability for defined failure classes,
- documenting migration paths for any breaking change.

Implementations claiming conformance MUST:
- state which spec/profile versions they implement,
- run and publish conformance results (recommended for transparency).

---

## 7. Migration rules (normative)

When migrating from `X` to `Y` where `Y` is a breaking boundary:

- Producers MUST be able to produce `Y` artifacts without ambiguity.
- Verifiers MUST support verifying `X` artifacts during the deprecation window (if policy requires).
- A migration guide MUST specify:
  - data transformations (if any),
  - anchoring continuity strategy,
  - how to audit mixed environments.

---

## 8. Reason code stability policy

Reason codes are part of interoperability.

- Reason code identifiers MUST be stable once published.
- New reason codes MAY be added, but existing codes MUST NOT change meaning.
- Strict mode SHOULD treat warnings as failure if the policy defines them as such.

---

## 9. Governance and publication (recommended)

Public changes SHOULD be published with:
- tagged spec versions,
- changelog entries,
- updated conformance vectors,
- migration notes,
- security advisories when applicable.

---

**Status:** v0.1 baseline. Enterprises can adopt this policy as an internal standard for accepting SealedProof evidence.

