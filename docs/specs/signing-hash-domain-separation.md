# Signing & Hash Domain Separation (v0.1)

**Status:** Draft (public)  
**Audience:** implementers, security engineers, auditors  
**Goal:** prevent cross-context hash/signature reuse by defining **explicit namespaces** and **domain separation labels** for all SealedProof cryptographic bindings.

> If you only remember one line: **the same bytes MUST NOT be valid in two different security meanings.**  
> Domain separation makes “meaning” cryptographically explicit.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative.

---

## 1. Why domain separation exists

Modern evidence systems bind many different objects:
- a statement about a run,
- a manifest of artifacts,
- a Merkle root,
- a transparency-log entry,
- a TSA timestamp token,
- an export to another ecosystem (in-toto/SCITT/etc).

Without domain separation, two objects might accidentally (or maliciously) share the same hash input bytes, enabling:
- **signature replay** across object types,
- **hash collision-by-context** (not cryptographic collision, but “same hash used for a different claim”),
- **confused deputy** issues where a verifier interprets a signature under the wrong semantics,
- **algorithm agility bypass** if suite selection is not bound to what is signed.

Domain separation ensures each signed/hash-bound object carries:
1) a **type** (what it is),  
2) a **purpose** (why it’s being bound), and  
3) a **suite** (how it’s bound).

---

## 2. Design principles

### 2.1 Mandatory, deterministic, canonical
All domain separation input MUST be:
- deterministic,
- canonicalized (see `specs/canonicalization-jcs-profile.md`),
- bound into the signed bytes, not carried “out-of-band”.

### 2.2 Strongly typed binding
Each signed object MUST bind:
- `object_type` (e.g., `statement`, `manifest`, `receipt`, `bundle`),
- `purpose` (e.g., `attest`, `timestamp`, `export`, `anchor`),
- `suite_id` (cryptographic algorithm suite).

### 2.3 Forward-compatible namespace versioning
Domain labels MUST include a version prefix to enable evolution without ambiguity.

---

## 3. Domain label format (normative)

SealedProof uses a structured **domain label** prepended to hash/sign inputs:

```
SPv1|<object_type>|<purpose>|<suite_id>|
```

- `SPv1` is the domain namespace version.
- `object_type` and `purpose` are ASCII lower-kebab-case.
- `suite_id` is the canonical suite identifier from `specs/suite-registry.md`.
- The trailing `|` is required.

The domain label MUST be encoded as UTF-8 bytes.  
No BOM. No whitespace.

### 3.1 Object type registry (baseline)
The following `object_type` values are reserved:

- `statement` — an attestation/claim object
- `manifest` — pack manifest / coverage list
- `merkle-root` — Merkle root commitment
- `log-entry` — transparency log entry body
- `tsa-token` — RFC3161 timestamp token binding
- `export` — derived export objects (e.g., in-toto mapping)
- `trust-bundle` — trust material snapshot
- `policy` — policy/profile declaration

Deployments MAY define additional types under an organization prefix (see §8).

### 3.2 Purpose registry (baseline)
Reserved `purpose` values:

- `attest` — signing an attestation statement
- `anchor` — anchoring a commitment (e.g., Merkle root)
- `timestamp` — binding to time authority / log
- `export` — signing a derived export form
- `verify` — verifier-produced reports
- `rotate` — binding key rotation events

---

## 4. Canonical “to-be-signed” bytes (normative)

A signature MUST be computed over:

```
to_be_signed = domain_label || canonical_bytes(payload)
```

Where:
- `domain_label` follows §3
- `canonical_bytes(payload)` is produced by the SealedProof JCS Profile (`specs/canonicalization-jcs-profile.md`)

**Forbidden:** signing a language-level string representation, or signing non-canonical JSON.

### 4.1 Payload canonicalization boundaries
Each object spec MUST define which fields are included in `payload` for signing and which are excluded.
At minimum:
- signature fields MUST be excluded from the canonicalization target,
- any “derived” digest fields MUST be either recomputable or explicitly included as commitments (avoid redundant “self-hash” traps).

---

## 5. Hash namespace and sub-domains

SealedProof uses hash functions in multiple places:
- file digests,
- statement digests,
- manifest digests,
- Merkle tree nodes and roots,
- log anchoring.

To avoid mix-ups, each hash role MUST have its own domain label:

### 5.1 File digest domain
```
SPv1|digest|file|<suite_id>|
hash( domain || file_bytes )
```

### 5.2 JSON object digest domain
```
SPv1|digest|json|<suite_id>|
hash( domain || canonical_json_bytes )
```

### 5.3 Merkle node domain
```
SPv1|digest|merkle-node|<suite_id>|
hash( domain || left_digest || right_digest )
```

### 5.4 Merkle root domain
```
SPv1|digest|merkle-root|<suite_id>|
hash( domain || root_node_digest )
```

**Note:** leaf/node encoding MUST be fixed-length digest bytes (not hex strings). Higher-level specs define exact concatenation rules.

---

## 6. Preventing signature reuse vulnerabilities

A verifier MUST reject a signature if:
- `object_type` does not match the expected type at that verification step,
- `purpose` does not match the expected purpose,
- `suite_id` does not match or is not allowed under the applicable profile,
- the payload canonical bytes differ.

This prevents:
- using a valid `statement|attest` signature as a `manifest|anchor` signature,
- reinterpreting a TSA token binding as a log entry binding,
- using a signature produced under one suite as if it were under another.

---

## 7. Suite binding (algorithm agility correctness)

Suite selection MUST be bound into `domain_label` (`suite_id`).
A verifier MUST use `suite_id` from the signed object to select:
- signature algorithm,
- hash algorithm,
- canonicalization profile version (if suite defines it),
- key type constraints.

A verifier MUST NOT “guess” suite behavior based on key type alone.

---

## 8. Extensibility (organization namespaces)

Organizations MAY define custom object types and purposes using a prefix:

- `object_type`: `org.<reverse-domain>.<name>`
- `purpose`: `org.<reverse-domain>.<name>`

Example:
- `org.com.acme.build-attestation`

Rules:
- must remain ASCII lowercase and use `.` and `-` only.
- must not collide with reserved baseline values.

---

## 9. Conformance tests (recommended)

Implementations SHOULD include test vectors proving:
- same payload under different domain labels produces different hashes/signatures
- same domain label but different canonicalization inputs fails verification
- suite mismatch yields deterministic failure reason codes

Reason codes SHOULD align with `conformance/validation-workflow.md`.

---

## 10. Security considerations

- Domain separation is not optional. It is a primary defense against “semantic confusion” and signature replay.
- Changing the domain label format is a breaking change; bump `SPv1` to `SPv2`.
- Profiles SHOULD define allowed suites and enforce them strictly.

---

**Status:** v0.1 baseline. Expect refinements as Suite Registry and export mappings mature.

