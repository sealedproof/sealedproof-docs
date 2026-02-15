# Reason Codes (v0.1)

Reason codes are the **human‑operational contract** of SealedProof verification.  
They enable deterministic failure analysis, audit traceability, and tooling integration (SIEM/SOAR, CI gates, compliance workflows).

> A verdict without stable reasons is not audit-grade.

---

## 0. Principles

Reason codes MUST be:
- **Stable**: meaning does not change across versions.
- **Deterministic**: same inputs → same ordered codes.
- **Actionable**: each code maps to a specific check and remediation.
- **Compositional**: multiple codes may be emitted; severity ordering is defined.

Reason codes MUST NOT:
- leak sensitive content (no raw secrets, no full payload dumps),
- be tied to UI text (messages can evolve; codes must not).

---

## 1. Code format

A reason code is a short identifier:

- `E###` — Error / hard failure (verdict MUST be FAIL)
- `W###` — Warning (verdict MAY be PASS_WITH_WARNINGS if policy allows)
- `I###` — Informational notice (non-gating)

Optional subcode (MAY) for specialization:
- `E201.1`, `W501.2` (meaning must be documented and stable)

---

## 2. Severity ordering

Verifiers MUST output codes in a deterministic order:
1. Errors (`E`) ascending by numeric value
2. Warnings (`W`) ascending
3. Info (`I`) ascending

Policy profiles MAY override ordering for specific operational workflows, but the default ordering above MUST be supported.

---

## 3. Code record (machine-readable report)

Validation reports SHOULD include for each code:
- `code` (e.g., `E201`)
- `title` (short stable label)
- `message` (human-readable explanation; versioned text allowed)
- `evidenceRef` (optional pointers to manifest paths/digests; MUST NOT include secrets)
- `remediation` (optional; actionable next step)

Example:

```json
{
  "code": "E201",
  "title": "SignatureInvalid",
  "message": "Envelope signature verification failed under suite SP-INTL-ED25519-JCS-2026.",
  "evidenceRef": { "path": "pack.json", "keyRef": "kid:..." },
  "remediation": "Ensure pack.json signing view is canonicalized per suite and re-sign with the correct key."
}
```

---

## 4. Code taxonomy (v0.1 baseline)

### 4.1 Pack structure & versioning (E0xx)
- `E001 UnsupportedPackVersion` — `spVersion` unsupported
- `E002 UnsupportedManifestVersion` — `manifestVersion` unsupported
- `E003 UnknownSuiteId` — suiteId not found in suite registry
- `E004 PolicyProfileNotFound` — required policy profile missing

### 4.2 Manifest & file integrity (E1xx)
- `E101 ManifestCommitmentMismatch` — manifest digest does not match envelope commitment
- `E110 ExtraFileNotInManifest` — file exists in pack but not listed
- `E111 MissingRequiredFile` — required=true entry absent
- `E120 FileDigestMismatch` — file bytes do not match entry digest
- `E121 FileSizeMismatch` — file length does not match entry size
- `E130 InvalidPathInManifest` — path normalization rule violated
- `E131 DuplicatePathInManifest` — duplicate `path` entries

### 4.3 Canonicalization & hashing (E2xx)
- `E201 SignatureInvalid` — signature verification failed
- `E202 SigningScopeMismatch` — verifier reconstructed signing view differs
- `E210 CanonicalizationMismatch` — canonical bytes differ from suite rules
- `E211 DigestAlgorithmNotAllowed` — digest algo not permitted by suite/policy
- `E221 SignerNotTrusted` — signer key not in trust anchors / allowlist
- `E222 SignerRevoked` — key/cert revoked under policy inputs

### 4.4 Domain separation & context binding (E3xx)
- `E301 DomainSeparationMissing` — no domain tag where required
- `E302 DomainSeparationMismatch` — domain tag does not match pack type/suite
- `E303 CrossContextReplayDetected` — commitment reused where forbidden

### 4.5 Ledger / continuity / rollback (E4xx)
- `E401 ChainContinuityBroken` — missing link / discontinuity detected
- `E402 ConflictingChainHeads` — conflicting heads without reconciliation proof
- `E403 RollbackSuspected` — older state presented as latest under policy

### 4.6 Timestamp / anchoring (E5xx, W5xx)
- `E501 RequiredTimeProofMissing` — policy requires external time proof
- `E502 TimeProofInvalid` — provided proof fails validation
- `E503 TimeOrderingInvalid` — internal time ordering invalid
- `W501 TimestampNotExternallyProven` — timestamps present but not externally anchored

### 4.7 Minimal disclosure / privacy (E6xx, W6xx)
- `E601 DisclosureModelInvalid` — disclosure metadata malformed
- `E602 RequiredFieldRedacted` — required claims hidden
- `W601 OverDisclosurePolicySoftViolation` — extra fields revealed vs recommended policy
- `W602 RedactionPresent` — informational: pack is a redacted view

### 4.8 Policy & semantics (E7xx, W7xx)
- `E701 RequiredClaimMissing` — mandatory claim absent
- `E702 ForbiddenClaimPresent` — forbidden claim present
- `E703 UnverifiableSelfAssertionUsed` — attempted PASS based on unverifiable self-claims
- `W701 OptionalEvidenceMissing` — recommended evidence absent (policy allows)

### 4.9 Informational (I0xx)
- `I001 VerifiedOffline` — verification performed offline
- `I010 AnchorNotProvided` — no anchoring proof present
- `I020 MultiSignatureObserved` — multiple signatures found

---

## 5. Backward compatibility and governance

### 5.1 Adding codes
New codes MAY be added. Once published, their meaning MUST NOT change.

### 5.2 Deprecation
Codes MUST NOT be removed. If obsolete, mark as deprecated and provide replacement guidance.

### 5.3 Mapping to checks
Each code MUST map to:
- a unique check identifier in the verifier,
- a policy rule section (profile),
- and, where relevant, a spec section (format / suite / ledger).

---

## 6. Implementation guidance (non-normative)

- Do not emit “generic failure.” Always emit at least one `E###`.
- Prefer the most specific code available.
- Avoid leaking secrets: reason codes should reference digests/paths, not payload content.

---

**Status:** v0.1 baseline. The canonical list evolves by addition only, under strict stability rules.

