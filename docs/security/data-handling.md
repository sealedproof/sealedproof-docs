# Data Handling & Minimum Disclosure (v0.1)

This document defines the **data classification and minimum disclosure baseline** for SealedProof Evidence Packs.  
It is written for enterprise security, privacy, audit, and platform/SRE teams, and is intended to be suitable for external review.

**Scope:** PII handling, secret/credential handling, confidential fields, redaction strategies, selective disclosure patterns, and verifier expectations.

Related docs:
- Security baseline checklist: `security/security-baseline.md`
- Replay/rollback/fork protections: `security/replay-rollback.md`
- Key lifecycle: `security/key-management.md`
- Evidence pack format: `specs/evidence-pack-format.md`
- Manifest coverage: `specs/pack-manifest.md`
- Validation workflow: `conformance/validation-workflow.md`

> **Core rule:** Evidence Packs are for **verifiable integrity** and **auditability**, not for bulk data export.  
> The default posture is **minimize, compartmentalize, and disclose only what’s necessary**.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative as in RFC 2119.

---

## 1. Principles

### 1.1 Data minimization by design
A producer MUST design Evidence Packs so that:
- sensitive data is **excluded by default**,
- verifiability is achieved via **digests and deterministic commitments**,
- disclosures are **explicit, intentional, and policy-bound**.

### 1.2 Separation of proof from payload
Evidence Packs SHOULD separate:
- **proof-critical metadata** (statements, digests, signatures, continuity claims), and
- **payload artifacts** (files, logs, records),
so that most verification can occur without exposing raw sensitive content.

### 1.3 Least disclosure across trust boundaries
When evidence crosses an organizational boundary, the producer MUST support:
- controlled disclosure (who sees what),
- a verifier-visible policy outcome (pass/fail + reason codes),
- a defensible audit trail for what was disclosed.

---

## 2. Data classification (baseline taxonomy)

Deployments MUST adopt a classification scheme. This document provides a baseline taxonomy:

### 2.1 Public
Information safe to disclose broadly (e.g., non-sensitive build metadata, public artifact hashes).

### 2.2 Internal
Non-public operational metadata that may assist attackers if leaked (e.g., internal hostnames, non-sensitive logs).

### 2.3 Confidential
Business-sensitive data (e.g., proprietary configs, internal architecture snapshots, customer-specific identifiers).

### 2.4 Restricted / Regulated
Data requiring strict controls:
- **PII** (personal data),
- credentials/secrets (API keys, tokens, passwords),
- regulated data (industry/region-specific),
- security-sensitive diagnostics (e.g., stack traces containing secrets).

> Policy profiles SHOULD define which classes are allowed in which pack types (e.g., “Public Conformance Pack” vs “Customer Audit Pack”).

---

## 3. Minimum disclosure requirements (MUST)

### 3.1 No secrets in cleartext
Evidence Packs MUST NOT include secrets in cleartext. This includes:
- access tokens, API keys, private keys,
- passwords, session cookies,
- unredacted environment variables containing secrets.

If a producer cannot guarantee this, the pack MUST fail validation under strict profiles.

### 3.2 PII handling
If PII is present, the producer MUST:
- classify it as Restricted/Regulated,
- apply explicit disclosure policy controls,
- provide redaction or selective disclosure mechanisms as required by profile.

Recommended posture: **avoid including PII at all**, and instead include:
- digests, pseudonyms, or tokenized identifiers,
- a separate disclosure path for authorized auditors.

### 3.3 Digest-first references
Where possible, statements MUST reference artifacts by digest, not by embedding raw content.  
Paths and filenames are treated as metadata and MUST NOT be used as a trust primitive.

### 3.4 Explicit disclosure intent
Packs that contain Restricted/Regulated content MUST:
- declare this in pack metadata (profile-defined),
- include a disclosure rationale or ticket reference (recommended),
- include an access control expectation (e.g., “auditors only”).

---

## 4. Redaction & selective disclosure patterns

This section describes standardized patterns. Policy profiles determine which are required.

### 4.1 Structural redaction (recommended baseline)
Producers SHOULD generate redacted views of artifacts:
- remove or mask sensitive fields,
- preserve stable structure for audit readability,
- compute digests over the **redacted** version included in the pack.

This is the simplest and most interoperable approach.

**Verifier expectation:** verify integrity of the redacted artifact against the manifest digests; do not infer content of removed fields.

### 4.2 Dual artifacts: full + redacted (policy-driven)
For certain enterprise audits, a pack MAY include both:
- a redacted artifact (default visible), and
- a full artifact encrypted for a restricted audience.

Requirements:
- both artifacts MUST be listed in the manifest,
- disclosure policy MUST define who may access the full artifact,
- envelope signing scope MUST bind both to prevent “swap attacks”.

### 4.3 Field-level encryption / compartmentalization (advanced)
Producers MAY encrypt sensitive subcomponents:
- encrypt individual files or sections,
- include key-wrapping metadata consistent with key management policy,
- keep verification meaningful via digests and signatures.

This pattern requires strong key lifecycle controls (`security/key-management.md`) and clear operational procedures.

### 4.4 Predicate / proof-of-properties (future profiles)
Some ecosystems may require proving properties without revealing raw data (e.g., “policy passed”, “threshold met”).  
SealedProof profiles MAY later standardize such mechanisms; until then, deployments SHOULD treat them as out-of-scope.

---

## 5. Handling common sensitive sources

### 5.1 Logs
- Logs MUST be treated as at least Internal; often Confidential or Restricted.
- Producers SHOULD implement log scrubbing (secrets, PII) before packaging.
- Consider splitting logs into:
  - a redacted summary (pack),
  - a full encrypted log (optional, policy-defined).

### 5.2 Configuration & environment
- Environment snapshots frequently contain secrets; do not package raw env dumps.
- If configuration must be evidenced, prefer:
  - config hashes,
  - redacted config with secrets removed,
  - policy-attested configuration state.

### 5.3 Identifiers
- User identifiers, device identifiers, and customer identifiers are often PII or Confidential.
- Prefer tokenization/pseudonymization with separately controlled mapping.

### 5.4 Attachments and binary blobs
- Always classify and scan artifacts before packaging.
- Apply content-type allowlists under policy profiles.

---

## 6. Storage, retention, and deletion (baseline)

### 6.1 Retention policy
Deployments MUST define retention for:
- packs,
- trust bundles,
- audit logs related to disclosure.

Retention MUST account for:
- regulatory requirements,
- incident response timelines,
- customer contracts.

### 6.2 Encryption at rest and in transit
- Packs stored server-side SHOULD be encrypted at rest.
- Transfer SHOULD use secure transport.
- If packs are distributed offline, producers SHOULD provide guidance for secure storage (vaults, access controls).

### 6.3 Deletion and “right to erasure”
Where applicable, policy MUST define:
- how deletion requests are handled,
- how integrity/audit requirements are preserved (e.g., retaining digests while deleting raw payload).

---

## 7. Verification expectations & reason codes

### 7.1 Verifier responsibilities
A verifier MUST:
- validate manifest coverage and digests,
- apply policy rules about forbidden data classes (profile-defined),
- emit deterministic reason codes and avoid vague errors.

### 7.2 Data handling failure signals (baseline)
Policy profiles SHOULD map failures to reason codes, such as:
- forbidden claim present → `E702 ForbiddenClaimPresent` (or profile-specific),
- sensitive data detected where disallowed → profile-specific `E7xx`,
- malformed redaction declaration → profile-specific `E7xx`.

> Exact codes are standardized in `specs/reason-codes.md`. This document defines the categories; profiles pin exact mappings.

---

## 8. Operational checklist

### 8.1 Producer checklist (security/privacy)
- [ ] Data classification applied to every artifact source.
- [ ] Secrets are scrubbed; no cleartext credentials in packs.
- [ ] PII is excluded by default; if included, explicit policy and rationale exist.
- [ ] Artifacts referenced by digest; paths are non-authoritative metadata.
- [ ] Redaction pipeline is deterministic and tested.
- [ ] Optional encrypted full artifacts follow key policy and are manifest-bound.
- [ ] Retention, encryption, and deletion procedures are documented.

### 8.2 Verifier checklist
- [ ] Enforce forbidden data classes per profile.
- [ ] Fail closed on “secrets present” in strict profiles.
- [ ] Emit stable reason codes and preserve audit traces of verification.
- [ ] Treat “redacted” as an explicit state; do not attempt inference.

---

## 9. What this document does NOT do

This document does not prescribe:
- a specific redaction algorithm implementation,
- a specific encryption scheme or KMS vendor,
- jurisdiction-specific legal advice.

It defines **what must be true** for audit-grade, enterprise-safe evidence handling.

---

**Status:** v0.1 baseline. Will evolve with profile definitions and conformance test vectors.

