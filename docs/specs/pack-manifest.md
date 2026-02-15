# Pack Manifest (v0.1)

This document specifies the **Pack Manifest** (`manifest.json`) for SealedProof Evidence Packs.  
The manifest is the **non‑negotiable completeness and integrity boundary**: if a byte is not in the manifest, it is not part of the proof.

> **Rule:** No manifest coverage → no audit value.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative as in RFC 2119.

---

## 1. Goals

The manifest MUST enable a verifier to:
1. Enumerate **all files** included in the pack.
2. Validate **integrity** (digest + size).
3. Enforce **no insertion, no omission**.
4. Apply **role-aware policy** (e.g., statements required, reports optional).
5. Produce deterministic **reason codes** on failure (see `specs/reason-codes.md`).

---

## 2. Manifest file identity

- Manifest filename: `manifest.json` (recommended; other names MAY be allowed if referenced by envelope and policy).
- The manifest MUST be referenced by the envelope (`pack.json`) via a **digest commitment**.
- The manifest MUST itself be included as an entry in `entries[]` and MUST be protected by the envelope signature scope.

---

## 3. Path rules (MUST)

All `path` values in entries MUST:
- use POSIX separators (`/`),
- be relative (MUST NOT start with `/`),
- contain no `.` or `..` segments,
- be UTF‑8,
- be unique within `entries[]`.

Verifiers MUST reject manifests violating these rules.

---

## 4. Digest rules

### 4.1 Digest format
`digest` MUST be encoded as:

- `"<algo>:<value>"`  
  e.g., `sha256:...`

Allowed algorithms MUST be defined by the `suiteId` and enforced by policy.

### 4.2 Size and digest pairing
Every entry MUST include:
- `size` (bytes, integer)
- `digest`

Verifiers MUST verify both:
- `size` matches actual file length,
- `digest` matches actual file bytes.

---

## 5. Roles and semantics

Each entry MUST include a `role`:

- `envelope`  — the signed pack header (`pack.json`)
- `manifest`  — the manifest itself
- `statement` — structured claims (`statements/*.json`)
- `artifact`  — raw/derived bytes referenced by statements
- `trust`     — certificates/keys/allowlists for offline trust
- `anchor`    — timestamp / append-only proof objects
- `disclosure`— redaction/disclosure metadata
- `report`    — verifier-produced outputs (non-normative)
- `other`     — reserved for extensions (policy-controlled)

Policy profiles decide which roles are required and which are allowed.

---

## 6. Manifest schema (normative)

Top-level `manifest.json` MUST be a JSON object:

```json
{
  "spVersion": "0.1",
  "manifestVersion": "0.1",
  "packId": "spk_...",
  "generatedAt": "2026-02-15T10:00:00Z",
  "entries": [ /* ... */ ],
  "extensions": { /* optional */ }
}
```

### 6.1 Required fields
- `spVersion` (string)
- `manifestVersion` (string)
- `packId` (string; MUST match envelope)
- `generatedAt` (RFC3339 timestamp)
- `entries` (array; MUST include at least envelope + manifest)

### 6.2 Entry object
Each entry MUST be a JSON object:

```json
{
  "path": "statements/action-0001.json",
  "role": "statement",
  "digest": "sha256:...",
  "size": 1234,
  "mediaType": "application/json",
  "required": true,
  "labels": ["toolcall"],
  "extensions": { }
}
```

Required entry fields:
- `path` (string; path rules apply)
- `role` (string; one of allowed roles)
- `digest` (string)
- `size` (integer)
- `required` (boolean)

Recommended:
- `mediaType` (string)
- `labels` (array of strings; non-normative hints)
- `extensions` (object; namespaced)

---

## 7. Completeness and exclusion rules

### 7.1 Completeness
Verifiers MUST enforce:
- **No extra files**: any file present in the pack but absent from `entries[]` → FAIL.
- **No missing required files**: any entry with `required=true` absent in the pack → FAIL.
- **Digest/size match**: mismatch → FAIL.

### 7.2 Optional files
Entries with `required=false` MAY be absent.  
If present, they MUST still be digest/size verified.

### 7.3 Deterministic sorting (recommended)
Producers SHOULD sort `entries` deterministically by `path` ascending (byte-wise / Unicode codepoint).  
Verifiers MUST NOT depend on ordering, but deterministic ordering improves diffing and reproducibility.

---

## 8. Archives and packaging notes (non-normative)

If distributing as an archive:
- Prefer a deterministic archive build (stable ordering, stable timestamps if possible).
- Verification MUST be performed against the extracted file bytes and the manifest digests; archive metadata MUST NOT be used as proof.

---

## 9. Validation steps (minimum)

A verifier MUST:
1. Load `pack.json` and resolve `suiteId`.
2. Compute digest of `manifest.json` canonical bytes and compare with envelope commitment.
3. Parse manifest and validate path rules.
4. Enumerate all files in the container and ensure each appears in `entries`.
5. For each present file, verify size + digest.
6. Enforce required file presence.
7. Emit deterministic reason codes on any failure.

---

## 10. Failure modes (examples)

- Extra file found → `E110 ExtraFileNotInManifest`
- Missing required statement → `E111 MissingRequiredFile`
- Digest mismatch → `E101 ManifestDigestMismatch` or `E120 FileDigestMismatch` (profile-dependent)

Canonical codes are defined in `specs/reason-codes.md`.

---

**Status:** v0.1. Manifest rules are intentionally strict because ambiguity is the enemy of verification.

