# Evidence Pack Format (v0.1-draft)

This document specifies the **public, interoperable format** of a SealedProof **Evidence Pack** (“pack”).  
A pack is a **portable, sealed bundle** that enables **offline third‑party verification** of AI agent actions, with **non‑repudiation**, **minimal disclosure**, and **handoff** across organizational boundaries.

> **Design intent:** Logs help you observe. Evidence Packs help you **prove**.

---

## 0. Conventions and normative terms

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are to be interpreted as described in RFC 2119.

---

## 1. Goals and non‑goals

### 1.1 Goals
A compliant Evidence Pack format MUST support:

- **Offline verification**: verifiers MUST validate integrity and signatures without contacting the producer.
- **Determinism**: canonicalization + hashing MUST be stable across languages/platforms.
- **Completeness**: a manifest MUST cover all files (no silent omission or insertion).
- **Non‑repudiation**: signatures MUST bind claims to signer identity (under an explicit suite).
- **Minimal disclosure (extensible)**: the format MUST allow redacted views that remain verifiable.
- **Handoff**: packs MUST be portable across tools and org boundaries.

### 1.2 Non‑goals (for v0.1)
This spec does NOT define:
- a full policy language (see `conformance/*` and policy profiles),
- a runtime enforcement system (this is evidence, not a gate),
- the “truth” of model outputs (this proves *what happened* and *what was sealed*),
- selective disclosure cryptography details (reserved for `specs/minimal-disclosure-*`).

---

## 2. Terminology

- **Evidence Pack (pack)**: the sealed bundle (files + metadata) presented to a verifier.
- **Envelope**: the signed top‑level JSON describing the pack and referencing the manifest.
- **Manifest**: a complete inventory of pack files and their digests.
- **Statement**: a structured claim about an agent action (e.g., tool call, approval).
- **Artifact**: a raw or derived byte object referenced by digest (request/response payloads, snapshots).
- **Commitment**: a digest/identifier that binds to bytes deterministically.
- **Suite (`suiteId`)**: a named cryptographic and canonicalization profile (see `specs/suite-registry.md`).
- **Trust bundle**: offline key material/certificates used by the verifier (optional but recommended).

---

## 3. Pack container and directory layout

### 3.1 Container form
An Evidence Pack MUST be representable as either:
- a directory tree, or
- a single archive file (e.g., `.zip`, `.tar`).

**Archive requirements (SHOULD):**
- deterministic file ordering,
- no platform-specific metadata required for verification,
- preserve exact bytes of each file.

### 3.2 Path normalization
All manifest paths MUST be:
- POSIX-style (`/` separators),
- relative (MUST NOT start with `/`),
- free of `..` segments,
- UTF‑8 encoded.

Verifiers MUST reject packs whose manifest violates these rules.

### 3.3 Canonical layout (recommended)
A pack SHOULD follow this layout:

```
/pack.json                # Signed envelope (REQUIRED)
/manifest.json            # Pack manifest (REQUIRED)
/statements/              # Structured claims (REQUIRED: >= 1 statement)
/artifacts/               # Raw/derived artifacts (OPTIONAL)
/trust/                   # Trust bundle for offline verification (OPTIONAL)
/anchors/                 # Timestamp / transparency proofs (OPTIONAL)
/disclosure/              # Redacted views / disclosure metadata (OPTIONAL)
/reports/                 # Verifier output reports (OPTIONAL, non-normative)
```

The verifier MUST NOT require this exact layout if the manifest correctly enumerates files; however, producers SHOULD follow it for ecosystem interoperability.

---

## 4. Required files

### 4.1 `pack.json` (Envelope) — REQUIRED
The envelope MUST:
- declare `spVersion` and `suiteId`,
- bind to the manifest via a digest commitment,
- carry signer identity references (`kid` / certificate references),
- include one or more signatures over deterministic bytes.

### 4.2 `manifest.json` — REQUIRED
The manifest MUST:
- enumerate **all files** in the pack (including nested directories),
- include digest and size for each entry,
- label each entry with a role (envelope / manifest / statement / artifact / trust / anchor / disclosure / report),
- be covered by the envelope signature scope (directly or indirectly).

### 4.3 `statements/*` — REQUIRED (at least one)
A pack MUST contain at least one statement describing:
- **what** action occurred (class/type),
- **what artifacts** are relevant (by digest commitment),
- **who** asserts it (bound via signature in the envelope or a statement-level signature).

Statement schemas are extensible; v0.1 requires only the minimal interface described in §6.

---

## 5. Digest and commitment format

### 5.1 Digest identifier
A digest MUST be encoded as:

- `"<algo>:<value>"`

Examples:
- `sha256:2cf24dba5fb0a...`
- `blake3:9a3f...`

The allowed `algo` values MUST be defined by the suite (`suiteId`).  
Verifiers MUST reject digests whose algorithm is not permitted by the suite/policy profile.

### 5.2 Content-addressing (recommended)
Artifacts SHOULD be content-addressed:
- artifact filenames MAY be derived from their digest (`artifacts/sha256/<hex>`), but this is not required.
- statements MUST reference artifacts by digest, not by file path, to avoid path ambiguity.

---

## 6. Statement model (minimal interface for v0.1)

Statements are JSON documents stored under `statements/`.  
A statement MUST include:

- `statementType` (string, namespaced)
- `statementVersion` (string)
- `issuedAt` (RFC 3339 timestamp; “claimed time” unless anchored)
- `subject` (what this statement is about)
- `claims` (structured fields; extensible)

### 6.1 `subject` structure (minimum)
`subject` MUST include:

- `actionId` (stable identifier for correlation across systems)
- `artifacts[]` (list of artifact commitments relevant to the action)

Each artifact reference MUST include:
- `digest` (required)
- `mediaType` (recommended)
- `name` (optional human label; non-normative)

### 6.2 Namespacing
`statementType` MUST be namespaced to avoid collisions, for example:
- `sealedproof.action.toolcall`
- `sealedproof.action.approval`
- `org.example.custom.<type>`

Verifiers MUST ignore unknown non-critical claim fields but MUST fail closed on unknown critical fields as defined by the policy profile.

---

## 7. Envelope (`pack.json`) structure (normative)

`pack.json` MUST be valid JSON and MUST be canonicalizable under the declared suite.

### 7.1 Top-level shape
The envelope MUST be a JSON object with:

- `spVersion` (string, e.g. `"0.1"`)
- `suiteId` (string)
- `packType` (string; MUST be `"evidence-pack"`)
- `packId` (string; derived, see §7.3)
- `createdAt` (RFC 3339 timestamp)
- `producer` (object; signer identity context)
- `manifest` (object; commitment to `manifest.json`)
- `signatures[]` (array; >= 1)
- `extensions` (object; optional)

### 7.2 `producer` (minimum)
`producer` MUST include:
- `org` (string; legal entity or org identifier)
- `system` (string; service/app/runtime name)
- `keyRef` (string; key id or certificate reference; typically `kid:...`)

### 7.3 `packId` derivation
`packId` MUST be a deterministic identifier derived from canonical commitments, at minimum:
- `suiteId`
- manifest digest
- envelope payload digest (see §7.4)

A recommended derivation is:

- `packId = "spk_" + base32( digest( domainTag || suiteId || manifestDigest || payloadDigest ) )`

The exact domainTag and digest function MUST be defined by the suite and documented in the suite registry.

### 7.4 Signature scope (what is signed)
The envelope MUST define a deterministic **signature scope**:
- The signature MUST cover all fields except `signatures[]` itself.
- Producers MUST sign the canonical bytes of a “signing view”:

`signing_view = { spVersion, suiteId, packType, packId, createdAt, producer, manifest, extensions? }`

Verifiers MUST reconstruct the signing view and validate signatures over canonical bytes according to the suite.

### 7.5 `signatures[]` (minimum)
Each signature entry MUST include:
- `role` (e.g., `"producer"`, `"approver"`, `"witness"`)
- `keyRef` (e.g., `kid:...` or certificate reference)
- `sig` (base64 or base64url; as defined by suite)
- `alg` (optional if fully implied by suite; otherwise MUST match suite registry)

Multi-signature packs are allowed; policy profiles determine which roles are required.

---

## 8. Manifest (`manifest.json`) structure (normative)

### 8.1 Top-level shape
The manifest MUST be a JSON object with:
- `spVersion`
- `manifestVersion`
- `packId`
- `generatedAt`
- `entries[]` (array; >= 2, at least envelope + manifest)

### 8.2 `entries[]` (required fields)
Each entry MUST include:
- `path` (normalized, see §3.2)
- `role` (one of: `envelope`, `manifest`, `statement`, `artifact`, `trust`, `anchor`, `disclosure`, `report`, `other`)
- `digest` (see §5)
- `size` (integer; bytes)
- `mediaType` (string; recommended)
- `required` (boolean; whether a verifier MUST require presence under the policy profile)

### 8.3 Completeness rules
Verifiers MUST enforce:
- **no extra files** beyond manifest entries,
- **no missing files** referenced by entries with `required=true`,
- digest match for all present files.

Policy profiles MAY allow optional files; they MUST still be digest-checked if present.

---

## 9. Trust bundle and anchors (optional)

### 9.1 `trust/`
A pack MAY include a trust bundle for offline verification, such as:
- signer certificates,
- key allowlists,
- suite registry snapshot,
- revocation metadata.

If included, trust bundle files MUST be listed and digest‑checked via the manifest.

### 9.2 `anchors/`
A pack MAY include external anchoring proofs:
- timestamp tokens,
- transparency/append-only log inclusion proofs,
- witnessed checkpoints.

Anchors MUST be treated as optional unless required by policy profile (e.g., C2).

---

## 10. Minimal disclosure (optional in v0.1)

A pack MAY be presented in a redacted form. If so:

- the redacted pack MUST remain internally consistent and verifiable,
- redacted bytes MUST be represented by commitments (not silent deletion),
- the disclosure model MUST be declared (e.g., `disclosure/disclosure.json`).

v0.1 defines the container capability only; cryptographic disclosure proofs are specified elsewhere.

---

## 11. Validation requirements (format-level)

A verifier implementing this spec MUST, at minimum:

1. Resolve `suiteId` via suite registry (policy-controlled).
2. Canonicalize and hash `manifest.json`; compare to envelope manifest commitment.
3. Enforce manifest completeness (no extra/missing required files).
4. Verify all file digests.
5. Reconstruct signing view and validate signature(s).
6. Emit deterministic verdict and reason codes (see `specs/reason-codes.md`).

---

## 12. Extensibility

### 12.1 `extensions`
Both envelope and statements MAY include `extensions` objects.  
Extensions MUST be namespaced:

- `extensions["org.example.*"] = ...`

Verifiers MUST ignore unknown extensions unless a policy profile marks them as critical.

### 12.2 New roles and statement types
New signature roles and statement types are allowed. Interoperability requires:
- public documentation,
- stable schemas,
- and conformance tests where possible.

---

## 13. Security considerations

- **Ambiguity kills verifiability**: the canonicalization profile and signature scope must be deterministic.
- **Manifest coverage is non-negotiable**: any file outside manifest is an integrity failure.
- **Time is subtle**: unanchored timestamps are claimed time; treat accordingly in policy.
- **Do not embed secrets**: default evidence capture must prevent accidental inclusion of API keys/tokens/credentials.
- **Reason codes are part of the standard**: pass/fail without explainability is not audit-grade.

---

## 14. Minimal example (illustrative)

```
pack.json
manifest.json
statements/action-0001.json
artifacts/req.bin
artifacts/resp.bin
```

`manifest.json` lists all five files with digests.  
`statements/action-0001.json` references `req.bin`/`resp.bin` by digest.  
`pack.json` signs the signing view and binds the manifest digest.

---

**Status:** v0.1-draft. This spec will evolve alongside Suite Registry, Reason Codes, and Conformance Profiles.

