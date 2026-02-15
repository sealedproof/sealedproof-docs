# OpenTelemetry Integration (v0.1)

**Path:** `integrations/opentelemetry.md`  
**Status:** Draft (public)  
**Audience:** platform engineers, SRE/SOC, audit teams, SIEM integrators  
**Goal:** make SealedProof evidence **first-class telemetry**, so investigations can pivot from **trace/span → evidence pack** (and back) in seconds.

SealedProof evidence does not replace telemetry. It upgrades it: telemetry tells you **what happened**; an evidence pack lets a third party prove **it happened that way**.

Related docs:
- Evidence pack format: `specs/evidence-pack-format.md`
- Security baseline: `security/security-baseline.md`
- Data handling / minimum disclosure: `security/data-handling.md`
- Validation workflow: `conformance/validation-workflow.md`

---

## 1. What problem this solves (SOC reality)

In incident response, teams ask:

- “Which agent/tool call produced this effect?”
- “Was the output tampered with after the fact?”
- “Can we show an auditor *verifiable* proof, not screenshots?”

OTel already provides correlation for:
- distributed traces (`trace_id`)
- spans (`span_id`)
- logs correlation
- metrics dimensions

SealedProof adds:
- stable **Evidence IDs**
- immutable **digest commitments** (statement/manifest/pack)
- offline verification and deterministic failure reasons

**Result:** a SOC analyst can click a trace → retrieve the evidence pack → verify offline → attach a conformance report to the incident ticket.

---

## 2. Correlation model (normative)

### 2.1 Primary join keys
An integration MUST be able to correlate:
- **Span** ↔ **Evidence item**
- **Trace** ↔ **Evidence pack**

The canonical join keys are:

- `sealedproof.evidence_id` — stable identifier for a specific evidence item (e.g., a tool call statement)
- `sealedproof.pack_id` — identifier for the evidence pack (may be derived from digest)
- `sealedproof.pack_digest` — digest commitment of the pack (preferred for immutability)
- `sealedproof.statement_digest` — digest of the specific statement/attestation
- `sealedproof.verification_ref` — optional pointer/reference used by a verifier report

At minimum, producers SHOULD emit `sealedproof.evidence_id` and `sealedproof.pack_digest` into telemetry.

### 2.2 Evidence ID lifecycle
- Evidence IDs SHOULD be generated at the point of evidence creation.
- Evidence IDs MUST be stable for the lifetime of the pack and any exports.
- Evidence IDs MUST NOT embed PII or secrets (see `security/data-handling.md`).

---

## 3. OTel attribute conventions (recommended)

OTel semantic conventions evolve. Until a formal semantic convention exists upstream, SealedProof uses a vendor-style namespace:

### 3.1 Span attributes
Recommended span attributes:

- `sealedproof.evidence_id` (string)
- `sealedproof.pack_id` (string, optional)
- `sealedproof.pack_digest` (string, e.g., base64url or hex — profile-defined)
- `sealedproof.statement_digest` (string)
- `sealedproof.profile` (string, e.g., `sp.profile.enterprise.strict.v1`)
- `sealedproof.suite_id` (string, e.g., `sp.ietf.sha256.ed25519.v1`)
- `sealedproof.disclosure` (string enum: `none|redacted|encrypted|full`)

### 3.2 Resource attributes (service-level)
Recommended resource attributes:

- `sealedproof.deployment` (string, e.g., env/cluster identifier)
- `sealedproof.trust_bundle_id` (string, trust material snapshot)
- `sealedproof.policy_id` (string, profile/policy version)

### 3.3 Logs correlation
If your logging pipeline supports trace correlation, include:

- `trace_id`, `span_id` (standard OTel fields)
- `sealedproof.evidence_id`
- `sealedproof.pack_digest`

This enables: log line → span → evidence pack.

---

## 4. Mapping patterns

### 4.1 Agent tool call as a span
A typical mapping for agent systems:

- 1 agent “run” = 1 trace
- each tool call / model invocation = 1 span
- each span produces (or references) one evidence statement

Span name examples:
- `agent.tool.call`
- `agent.model.invoke`
- `mcp.tool.execute`

Attributes include `sealedproof.evidence_id` and optionally `sealedproof.statement_digest`.

### 4.2 Pack-level anchoring at trace end
At the end of a trace/run, the system:
1) finalizes a pack manifest and pack digest,
2) optionally anchors to TSA / transparency log,
3) emits `sealedproof.pack_digest` as a trace-level attribute or a terminal span attribute.

---

## 5. Query workflows (SOC + audit)

### 5.1 From trace to proof
1) Find trace in your APM/trace UI (e.g., “agent produced bad output”).
2) Locate the relevant span and copy `sealedproof.pack_digest` and/or `sealedproof.evidence_id`.
3) Use your internal retrieval service (or CLI) to fetch the evidence pack by digest.
4) Run the verifier (offline if required) and produce a conformance report.
5) Attach report to the incident ticket/audit case.

### 5.2 From evidence to telemetry
1) Start from an evidence pack received from a third party.
2) Extract `trace_id`/`span_id` correlation fields (if included by policy), or search by `sealedproof.pack_digest`.
3) Pull the related trace/spans to reconstruct operational context.

---

## 6. Sampling guidance (pragmatic)

Evidence and telemetry have different economics.

- Telemetry may be sampled aggressively.
- Evidence SHOULD NOT be sampled for regulated/security-critical actions.

Recommended approach:
- Always generate evidence for policy-defined critical actions (e.g., privilege changes, financial actions, production writes).
- Emit minimal correlation attributes to OTel for all actions.
- If telemetry sampling drops a span, the evidence still exists and remains verifiable.

---

## 7. Data handling & privacy (MUST)

OTel attributes and logs can leak sensitive data if misused.

Producers MUST:
- never place secrets or raw PII into OTel attributes,
- prefer digests/IDs over raw inputs/outputs,
- apply redaction levels (`sealedproof.disclosure`) consistently,
- keep “full payload” inside evidence packs under policy-controlled disclosure (see `security/data-handling.md`).

---

## 8. Interop and future standardization (non-normative)

SealedProof intentionally aligns with:
- OTel correlation primitives (trace/span IDs)
- SOC workflows (SIEM pivots, incident tickets)
- audit requirements (verifiable evidence, deterministic reason codes)

As the ecosystem matures, SealedProof may propose an OTel semantic convention upstream. Until then, treat these attributes as stable SealedProof conventions.

---

## 9. Implementation checklist

**Platform / Agent runtime**
- [ ] Generate an `evidence_id` per tool call / model invocation.
- [ ] Emit `sealedproof.evidence_id` on the corresponding span.
- [ ] Finalize pack at run end; emit `sealedproof.pack_digest`.
- [ ] Ensure suite/profile are bound and visible (`sealedproof.suite_id`, `sealedproof.profile`).

**SOC / Observability**
- [ ] Index `sealedproof.pack_digest` and `sealedproof.evidence_id` as searchable fields.
- [ ] Provide a “Fetch & Verify Evidence” action in your internal tooling (optional but powerful).

**Audit / Compliance**
- [ ] Require conformance reports to include trace correlation fields when allowed by policy.
- [ ] Validate that telemetry does not contain disallowed data classes (strict profiles).

