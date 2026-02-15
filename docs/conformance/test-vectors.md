# Conformance Test Vectors (v0.1)

**Path:** `conformance/test-vectors.md`  
**Status:** Draft (public)  
**Audience:** implementers, verifier authors, auditors, integrators  
**Goal:** define a **minimal, high-signal** set of test vectors that prove interoperability and prevent regressions.

This document is intentionally implementer-oriented: it tells you what to test, how to structure vectors, and what “pass” means—without disclosing product-specific internals.

Related:
- Validation workflow & reason codes: `conformance/validation-workflow.md`
- Evidence pack format: `specs/evidence-pack-format.md`
- JCS profile: `specs/canonicalization-jcs-profile.md`
- Domain separation: `specs/signing-hash-domain-separation.md`
- Suite registry: `specs/suite-registry.md`
- Replay/rollback: `security/replay-rollback.md`

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

---

## 1. Why test vectors matter (interop reality)

A “verifiable evidence layer” fails if different implementations:
- canonicalize JSON differently,
- compute different digests for the same payload,
- accept signatures under the wrong semantics,
- diverge on replay/rollback detection,
- produce inconsistent reason codes.

Test vectors provide:
- **interoperability contracts** across languages/runtimes
- **regression fences** for security-critical behaviors
- **auditor confidence** through reproducible verification outcomes

---

## 2. Vector set scope (minimal but complete)

A conforming implementation MUST pass all vectors in the **Core Set** and SHOULD pass the **Extended Set**.

### 2.1 Core Set (MUST)
1) **JCS canonicalization**: UTF-8, number formats, escaping, ordering  
2) **Digest roles**: file digest, JSON digest, Merkle node/root digest  
3) **Domain separation**: same payload under different labels must differ  
4) **Signature verification**: at least one `recommended` suite  
5) **Manifest coverage**: missing file / extra file detection  
6) **Replay & rollback**: monotonic sequence / chain checks  
7) **Minimum disclosure**: redacted view integrity + disclosure markers  
8) **Reason codes**: stable failure classification

### 2.2 Extended Set (SHOULD)
- multi-suite verification (allowed + deprecated behaviors under policy)
- TSA binding verification (RFC3161) when enabled
- transparency log anchoring verification when enabled
- export mapping integrity (e.g., in-toto) when enabled
- large-pack streaming verification (resource constraints)

---

## 3. Vector format (normative)

Vectors MUST be machine-readable and deterministic.

Recommended directory layout:

```
conformance/
  vectors/
    v0.1/
      README.md
      manifest.json
      cases/
        <case-id>/
          input/
          expected/
          artifacts/
          notes.md
```

### 3.1 Required case files
Each test case MUST include:

- `input/`  
  - `object.json` (or `pack/` directory)  
  - `suite_id` and `profile` metadata (may be embedded or adjacent)
- `expected/`  
  - `canonical.json` (expected canonical output, if applicable)  
  - `digest.txt` (expected digest, encoding defined by profile)  
  - `verify.json` (expected verification result object)
- `notes.md` (human explanation, non-normative)

### 3.2 Expected verification result schema
`expected/verify.json` MUST include:

```json
{
  "pass": true,
  "reason_code": "OK",
  "suite_id": "sp.ietf.sha256.ed25519.v1",
  "profile": "sp.profile.enterprise.strict.v1",
  "artifacts_checked": 12,
  "statements_checked": 7
}
```

On failure (`pass=false`), it MUST include:
- `reason_code` (stable)
- optional `details` (non-sensitive)

Reason code naming SHOULD align with `conformance/validation-workflow.md`.

---

## 4. Core test cases (normative)

### 4.1 JCS canonicalization vectors
Cases MUST cover:
- UTF-8 multi-byte characters (including emoji, CJK)
- escaping rules (`\uXXXX`, `\n`, quotes)
- number normalization (`1`, `1.0`, `1e0`, `-0`, large integers)
- object member ordering and duplicate key handling (reject)
- whitespace insensitivity (input) vs strict output

**Pass condition:** byte-for-byte equality against `expected/canonical.json` and matching digest in `expected/digest.txt`.

### 4.2 Digest role vectors
Must include separate cases for:
- file digest over raw bytes
- JSON digest over canonical bytes
- Merkle node digest (left/right order)
- Merkle root digest binding

**Pass condition:** expected digests match, and swapping left/right changes node digest.

### 4.3 Domain separation vectors
Same payload, different domain labels:
- `SPv1|statement|attest|...|`
- `SPv1|manifest|anchor|...|`
- `SPv1|digest|json|...|`

**Pass condition:** all resulting hashes/signatures differ; verifier rejects semantic mismatch.

### 4.4 Signature verification vectors
At minimum:
- one passing signature vector under a `recommended` suite
- one failure vector (wrong key / wrong suite / tampered payload)

**Pass condition:** deterministic pass/fail and reason codes.

### 4.5 Manifest coverage vectors
Cases MUST cover:
- missing file referenced by manifest
- extra file not referenced by manifest (strict profile should fail)
- wrong digest for referenced file

**Pass condition:** failure reason codes correspond to coverage errors.

### 4.6 Replay/rollback vectors
Cases MUST cover:
- monotonic sequence expectation (if profile uses it)
- chain linkage mismatch (`prev_digest` broken)
- fork detection (two competing successors)

**Pass condition:** failures map to replay/rollback reason codes.

### 4.7 Minimum disclosure vectors
Cases MUST include:
- redacted artifacts + their digests
- statement disclosing only digests while hiding payload
- policy markers (`disclosure=redacted|encrypted`)

**Pass condition:** verifier can validate integrity of disclosed artifacts without needing hidden payload.

### 4.8 Reason-code stability vectors
Provide a set of “known-bad” cases that MUST map to specific reason codes:
- unknown suite
- forbidden suite
- canonicalization failure
- signature invalid
- manifest mismatch
- anchor missing (if required)

**Pass condition:** consistent reason_code strings across implementations.

---

## 5. Interoperability guidance (recommended)

To avoid “pass locally, fail elsewhere”:
- publish vectors as immutable releases (tagged)
- include multi-language reference implementations only as optional helpers
- use deterministic encodings (base64url vs hex fixed by profile)
- avoid platform-dependent line endings in artifacts

---

## 6. Regression and CI policy (recommended)

Implementations SHOULD run:
- full Core Set on every commit
- Extended Set on release candidates and nightly builds

A suite/profile update MUST:
- add new vectors for new behavior
- keep old vectors for backward verification windows (see compatibility policy)

---

## 7. Security considerations

Test vectors are part of the security perimeter:
- treat them as normative references
- require review for any change
- ensure “strict mode” cases remain strict (warnings must fail)

---

**Status:** v0.1 baseline. This spec defines the contract; concrete vector files are published separately as tagged releases.

