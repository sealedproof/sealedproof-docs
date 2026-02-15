# Proof Envelope (Schema and Semantics)

**Status:** Draft (public)

The **Proof Envelope** binds an **Evidence Pack** to explicit verification semantics:
- which suite is used (`suiteId`),
- what is being verified (pack digest + coverage),
- who attests (signatures / identity constraints),
- optionally when/where it was anchored (timestamp/log anchors).

The envelope is designed for **handoff**: it MUST remain verifiable outside the producer environment.

> **Normative keywords**: **MUST**, **MUST NOT**, **SHALL**, **SHOULD**, **MAY** are used as in RFC 2119.

---

## 1. Design goals

The Proof Envelope MUST be portable, deterministic under canonicalization, explicit about suite/profile, compatible with minimal disclosure, and offline-verifiable by default.

---

## 2. Canonical representation

Unless a suite specifies otherwise, the envelope MUST be UTF‑8 JSON and MUST be canonicalized using the declared JCS profile (`canonicalization-jcs-profile.md`).

---

## 3. Minimal schema (normative)

```json
{
  "spVersion": "1.0",
  "type": "SealedProofEnvelope",
  "suiteId": "SP.SUITE.ED25519.JCS.SHA256",
  "profile": "SP.JCS.UTF8.V1",
  "packRef": {
    "packDigest": "sha256:...",
    "manifestDigest": "sha256:...",
    "ledgerDigest": "sha256:...",
    "packFormat": "SP.PACK.V1"
  },
  "claims": {
    "evidenceId": "sp_evi_...",
    "producer": {"id": "org.example", "kidHint": "kid:..."},
    "subject": {"type": "agent-action", "id": "trace:..."},
    "createdAt": "2026-02-15T00:00:00Z",
    "disclosure": {"level": "minimal"}
  },
  "anchors": [
    {"type": "rfc3161", "tokenDigest": "sha256:..."}
  ],
  "signatures": [
    {
      "kid": "kid:example:ed25519:01",
      "alg": "EdDSA",
      "sig": "base64url...",
      "signedFields": ["spVersion","type","suiteId","profile","packRef","claims","anchors"]
    }
  ]
}
```

### Required fields
- `spVersion` MUST be present.
- `type` MUST equal `"SealedProofEnvelope"`.
- `suiteId` MUST be present and resolvable via Suite Registry.
- `profile` MUST be present and supported.
- `packRef.packDigest` MUST be present.
- `signatures[]` MUST be present unless policy allows unsigned test mode.

### packRef semantics
- `packDigest` MUST deterministically identify the pack.
- `manifestDigest` SHOULD be present; if present it MUST match the pack manifest.
- `ledgerDigest` SHOULD be present when ledger semantics are used.
- `packFormat` MUST identify the pack format version.

### claims (minimal metadata)
Claims exist for indexing and interpretation; they MUST NOT embed raw evidence/PII/secret payloads.

### anchors (optional)
Anchors are optional but recommended for high-stakes audits. Anchors MUST be referenced by digest (not URL) and validated by policy.

### signatures
Signatures MUST cover all verification‑critical fields. Domain separation MUST follow `signing-hash-domain-separation.md`.

---

## 4. Extension model

Extensions MUST be namespaced. Unknown extensions MUST fail closed unless policy explicitly allows ignoring them.

---

## 5. Interoperability requirements

Producers MUST emit envelopes that pass `conformance/test-vectors.md`. Breaking changes MUST follow `conformance/compatibility-policy.md`.

