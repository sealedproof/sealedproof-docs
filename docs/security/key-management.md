# Key Management (Lifecycle) (v0.1)

This document defines the **key management baseline** for SealedProof deployments: key generation, storage, HSM usage, rotation, revocation, and audit events.

It is written for enterprise security teams, platform/SRE, and external reviewers. It aims to be **implementation-agnostic** while still being **audit-grade**.

Related documents:
- Threat and trust assumptions: `concepts/trust-model.md`
- Security baseline checklist: `security/security-baseline.md`
- Validation workflow: `conformance/validation-workflow.md`
- Reason codes: `specs/reason-codes.md`
- Evidence pack format & manifest coverage: `specs/evidence-pack-format.md`, `specs/pack-manifest.md`

> **Principle:** Verification MUST NOT require trusting the producer’s runtime or database. Keys and trust material must be managed so that third parties can verify **offline** with deterministic outcomes.

---

## 0. Normative language

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are normative as in RFC 2119.

---

## 1. Goals

Key management MUST enable:
1. **Non-repudiation**: signatures are attributable to a controlled key.
2. **Offline verifiability**: verifiers can validate signatures without live calls to the producer.
3. **Rotation without breakage**: old packs remain verifiable; new packs move forward.
4. **Revocation with explanation**: deterministic reasons for “not trusted” cases.
5. **Auditability**: lifecycle operations are logged as security events.

---

## 2. Threats (what key management must resist)

- Key exfiltration (insider, malware, supply chain).
- Unauthorized signing (privilege escalation, CI/CD compromise).
- Rollback of trust state (presenting old allowlists / old cert chains).
- Confusion attacks (wrong suite, wrong domain, wrong key usage).
- Weak entropy during generation.
- Silent key replacement (key swapped while keeping same identifier).

---

## 3. Key model: roles and types

### 3.1 Roles (recommended)
- **Root / Trust Anchor**: offline, rarely used.
- **Issuer / Intermediate**: endorses operational signing keys.
- **Pack Signer**: signs envelope and/or statements per suite.
- **Witness / Approver** (optional): signs checkpoints, approvals, releases.
- **Verifier**: consumes trust bundles; no private keys.

### 3.2 Proof-critical keys
This document focuses on keys used to produce verifiable signatures/commitments.

---

## 4. Key identifiers and metadata

### 4.1 `kid` requirements
Every proof signature MUST reference a stable key identifier (`kid`) that:
- uniquely identifies the public key in the trust bundle,
- is collision-resistant,
- is bound to suite usage.

Recommended `kid` encoding:
- `kid = base64url(sha256(publicKeyBytes))` or X.509 SKI derived from the public key.

### 4.2 Binding to suite and usage
A key MUST be scoped by:
- `suiteId` compatibility (algorithm + canonicalization + signing scope),
- intended usage (e.g., `pack-envelope-signing`, `checkpoint-witness`).

Verifiers MUST fail if:
- key algorithm does not match suite (suite/profile rule),
- key usage is not authorized by policy,
- `kid` cannot be resolved in trust material (`E221 SignerNotTrusted`).

---

## 5. Key generation

### 5.1 Entropy and RNG
Key generation MUST use a cryptographically secure RNG backed by the OS or HSM.  
Deployments MUST NOT generate keys using application-level pseudo-random sources.

### 5.2 Approved algorithms (profile-defined)
Allowed algorithms are defined by `suiteId` and policy profiles.

### 5.3 Environment and tenant separation
- dev / staging / prod keys MUST NOT mix.
- multi-tenant separation MUST be explicit.

---

## 6. Key storage & access control

### 6.1 General rules
Private keys MUST:
- never be stored in source code repositories,
- never be embedded in client apps,
- never be logged or exported in plaintext.

Access MUST be least-privilege and enforced by IAM/OS/HSM boundaries.

### 6.2 Storage options (policy-driven)
From strongest to weakest:
1. **HSM / Cloud HSM**
2. **Cloud KMS with HSM-backed keys**
3. **Cloud KMS (software-backed)**
4. **OS keystore / encrypted file vault** (dev/sandbox only unless explicitly allowed)

Policy MUST define which is acceptable per deployment tier.

### 6.3 HSM requirements (when used)
If HSM is used, the system MUST ensure:
- private keys are non-exportable (where supported),
- signing operations are authorized (workload identity/MFA/approvals),
- key usage is restricted to expected operations,
- auditing is enabled for all key operations.

---

## 7. Signing operations (operational controls)

Signing systems SHOULD be isolated as a “signing service” with:
- constrained egress,
- minimal dependencies,
- explicit request authn/authz,
- rate limiting and anomaly detection.

Policy MAY require dual-control (two-person rule) for high-impact actions.

---

## 8. Rotation (planned and event-driven)

### 8.1 Rotation must not break verification
Existing evidence packs MUST remain verifiable after rotation:
- trust bundles retain old public keys for a retention window,
- policy defines acceptance windows and revocation rules.

### 8.2 Rotation modes
- scheduled rotation (e.g., 90/180/365 days),
- event-driven rotation (suspected compromise, role change, policy change, algorithm migration).

### 8.3 Overlap window (recommended)
During rotation:
- new packs use the new key,
- verifiers still accept old keys for historical packs,
- optionally dual-sign for critical transitions.

### 8.4 Evidence for rotation (recommended)
Rotation SHOULD produce verifiable records:
- “key activated” checkpoint,
- “key retired” statement,
- policy profile version update.

---

## 9. Revocation

### 9.1 Revocation triggers
MUST support:
- confirmed compromise,
- unauthorized signing detected,
- decommissioning,
- policy violation.

### 9.2 Offline verification considerations
Offline verification cannot rely solely on online OCSP/CRL. Therefore:
- packs or verification environments SHOULD carry a **trust bundle snapshot** (keys + revocation inputs),
- if time sensitivity matters, policy MUST require anchors/timestamps to bind “when the pack was signed”.

### 9.3 Revocation representations (policy-defined)
- allowlist-only bundles (remove keys to revoke),
- explicit revocation lists (denylist) with effective timestamps,
- CRL/OCSP when using PKI.

Verifiers MUST emit deterministic reason codes:
- revoked → `E222 SignerRevoked`
- not trusted/unresolved → `E221 SignerNotTrusted`

---

## 10. Audit events (MUST log)

Key lifecycle operations MUST be recorded with:
- who (identity),
- what (operation),
- which key (`kid`),
- when (time),
- outcome,
- change ticket / justification (recommended).

Minimum event set:
- `KeyCreated`, `KeyImported` (if allowed), `KeyActivated`
- `KeyUsedForSigning` (aggregation acceptable)
- `KeyRotationStarted`, `KeyRotationCompleted`
- `KeyRevoked`, `KeyRetired`
- `TrustBundlePublished`, `PolicyProfileUpdated`
- `UnauthorizedSigningAttemptDetected`

Audit logs SHOULD be tamper-resistant and retained per policy.

---

## 11. Trust material in Evidence Packs

When offline verification is required, packs (or the delivery channel) MUST provide sufficient trust inputs:
- public keys / trust anchors,
- suite registry snapshot (recommended),
- revocation inputs (allowlist/denylist snapshot),
- certificates and chains (if using PKI),
- (optional) attestations (e.g., “HSM-backed”, “non-exportable”).

All trust files MUST be:
- listed in the manifest (`role=trust`),
- integrity-checked (digest + size),
- bound to the envelope signing scope.

---

## 12. Conformance and reason codes (baseline)

Recommended mappings:
- key not found / not trusted → `E221 SignerNotTrusted`
- key revoked → `E222 SignerRevoked`
- signature invalid → `E201 SignatureInvalid`
- signing scope mismatch → `E202 SigningScopeMismatch`
- suite mismatch → `E003 UnknownSuiteId` (or profile-specific)

---

## 13. Operational checklist

### 13.1 Producer checklist
- [ ] Keys generated with CSPRNG; no app RNG.
- [ ] Private keys never in code, logs, or client apps.
- [ ] Key usage bound to suiteId + role.
- [ ] HSM/KMS least privilege + auditing enabled.
- [ ] Rotation plan + overlap window tested.
- [ ] Revocation procedure documented and exercised.
- [ ] Trust bundle snapshot mechanism exists for offline verifiers.

### 13.2 Verifier checklist
- [ ] Trust material validated by manifest + envelope signature scope.
- [ ] Policy enforces suite/usage constraints.
- [ ] Revocation inputs applied deterministically.
- [ ] Time-binding enforced when “when” matters.

---

**Status:** v0.1 baseline. This document will evolve with Suite Registry profiles and conformance test vectors.

