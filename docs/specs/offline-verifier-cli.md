# Offline Verifier CLI (sp-verify)

**Status:** Draft (public)

This document specifies the **Offline Verifier CLI** for SealedProof. It targets **enterprise audit, SOC workflows, DevSecOps, and third‑party verification**, where evidence must remain verifiable **without network access**.

> **Normative keywords**: **MUST**, **MUST NOT**, **SHALL**, **SHOULD**, **MAY** are to be interpreted as in RFC 2119.

---

## 1. Why an “offline verifier” exists

In the AI Agent era, many failures are not “bugs”—they are **unverifiable executions**:
- tool calls that cannot be proven,
- logs that can be edited,
- evidence that cannot be handed off across org boundaries,
- investigations that collapse because *the proof is not portable*.

The Offline Verifier CLI provides a **deterministic verification boundary**:
- validate integrity and authenticity,
- produce stable **reason codes** (machine‑readable + auditor‑friendly),
- output results attachable to tickets, IR reports, and compliance dossiers.

---

## 2. Scope

The Offline Verifier CLI covers:
- Verification of a **Proof Envelope** and its bound **Evidence Pack**.
- Validation of **pack manifest coverage**, **ledger semantics**, and **canonicalization rules**.
- Signature verification based on **Suite Registry** and configured trust roots.
- Emitting **stable reason codes** and structured outputs for automation.

Non‑goals:
- Performing live lookups to transparency logs (offline means offline).
- Enforcing business‑specific policy beyond the provided policy file.
- Providing UI/visualization beyond machine‑readable output.

---

## 3. Inputs and outputs

### 3.1 Inputs

The verifier MUST support these inputs:

- `--envelope <path>`: Proof Envelope file (required)
- `--pack <path>`: Evidence Pack directory or archive (required unless embedded)
- `--policy <path>`: Verification policy file (optional; defaults apply)
- `--trust <path>`: Trust roots bundle (required for signature verification unless policy allows unsigned test mode)
- `--fixtures <path>`: Optional fixtures for conformance testing (test‑vectors)

The verifier MUST accept:
- Evidence Pack as a directory (unpacked) **or** a single archive file.
- UTF‑8 paths; it MUST NOT rely on filesystem‑specific ordering.

### 3.2 Outputs

The verifier MUST produce:
- **Exit code** (for CI/automation)
- **Result JSON** (for SOC/audit systems)
- Optional **human summary** (operator readability)

Result JSON MUST include:
- `verdict` (pass/fail)
- `suiteId`
- `packDigest`
- `envelopeDigest`
- `reasonCodes[]` (stable IDs)
- `observations[]` (non‑fatal, structured)
- `timestamp` (verification time)
- `policyVersion` and `verifierVersion`

---

## 4. Command surface

The CLI command name is **`sp-verify`**.

### 4.1 `sp-verify verify`

Performs full offline verification.

```bash
sp-verify verify   --envelope ./proof.envelope.json   --pack ./pack/   --trust ./trust/roots.bundle.json   --policy ./policy/verify.policy.json   --out ./reports/verify.result.json
```

**Required behavior**
- MUST canonicalize inputs according to the configured canonicalization profile (see `canonicalization-jcs-profile.md`).
- MUST compute digests deterministically (no locale/time‑dependent behavior).
- MUST verify all signature(s) required by the envelope and policy.
- MUST validate manifest coverage over all required pack files.
- MUST validate ledger semantics and detect replay/rollback indicators when present.
- MUST emit stable reason codes on failure.

### 4.2 `sp-verify inspect`

Prints normalized metadata without making a pass/fail decision.

```bash
sp-verify inspect --envelope ./proof.envelope.json --pack ./pack/
```

### 4.3 `sp-verify explain`

Resolves reason codes into auditor‑facing explanations (no guessing; no free‑form).

```bash
sp-verify explain --code SPV-MANIFEST-MISSING
```

### 4.4 `sp-verify extract`

Extracts artifacts for handoff or downstream tooling.

```bash
sp-verify extract --pack ./pack/ --select manifest,ledger --out ./exports/
```

---

## 5. Exit codes

The verifier MUST use deterministic exit codes:

- `0`: PASS
- `1`: FAIL (verification did not meet policy)
- `2`: INVALID_INPUT (missing/invalid files, parse errors)
- `3`: UNSUPPORTED (suiteId/profile not supported)
- `4`: INTERNAL_ERROR (verifier bug)

Exit codes MUST NOT be overloaded with “warnings.” Warnings are conveyed via `observations[]`.

---

## 6. Verification phases (deterministic pipeline)

The verifier MUST run phases in order and MUST stop only when policy says “fail‑fast”.

1) **Parse & validate envelope schema**  
2) **Canonicalize & compute digests**  
3) **Signature verification** (suiteId → suite; trust roots; constraints)  
4) **Pack integrity & coverage** (manifest completeness; file digests)  
5) **Ledger semantics** (continuity; fork/rollback signals)  
6) **Policy evaluation** (required signers/timestamps/disclosure)  

**Determinism rule**: Given identical inputs, policy, and trust roots, the verifier MUST produce identical JSON output (except `timestamp`).

---

## 7. Policy model (verification.policy.json)

Policy is optional; defaults MUST exist.

Minimum policy fields SHOULD include:
- `requireSignatures: true|false`
- `allowedSuiteIds: [...]`
- `requiredSigners: [...]`
- `requireTimestamp: true|false`
- `failFast: true|false`
- `allowExtraFiles: true|false`
- `minDisclosureLevel: "public"|"internal"|"restricted"`

Policies MUST be versioned.

---

## 8. Security considerations

- MUST NOT attempt outbound network access.
- MUST treat pack contents as untrusted input (zip‑slip defense, traversal defense, size limits).
- MUST NOT execute embedded scripts.
- SHOULD support resource caps to prevent decompression bombs.

---

## Appendix A: Minimal example output (JSON)

```json
{
  "verdict": "pass",
  "suiteId": "SP.SUITE.ED25519.JCS.SHA256",
  "envelopeDigest": "sha256:...",
  "packDigest": "sha256:...",
  "reasonCodes": [],
  "observations": [
    {"code":"SPV-OBS-TIMESTAMP-ABSENT","severity":"low","message":"Timestamp token not present; policy allows."}
  ],
  "policyVersion": "1.0",
  "verifierVersion": "1.0.0",
  "timestamp": "2026-02-15T12:00:00Z"
}
```
