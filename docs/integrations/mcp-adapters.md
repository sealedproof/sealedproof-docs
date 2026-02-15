# MCP Adapters (Agent/MCP Evidence Adapters) (v0.1)

**Path:** `integrations/mcp-adapters.md`  
**Status:** Draft (public)  
**Audience:** agent platform engineers, tool providers, security teams  
**Goal:** make Agent tool calls (including MCP tools) **evidence-producing by default** with minimal friction and clear disclosure control.

This document defines:
- what an “adapter” is,
- what must be captured for tool-call evidence,
- how to package it into evidence packs,
- how to correlate with telemetry and audits,
- how to avoid leaking secrets/PII.

Related docs:
- Evidence pack format: `specs/evidence-pack-format.md`
- Domain separation: `specs/signing-hash-domain-separation.md`
- Suite Registry: `specs/suite-registry.md`
- Data handling: `security/data-handling.md`
- Validation workflow: `conformance/validation-workflow.md`
- OpenTelemetry integration: `integrations/opentelemetry.md`

---

## 1. Why MCP needs evidence

Agent ecosystems scale by composition:
- tools call tools,
- agents call tools,
- workflows fan out across services.

This expands the attack surface:
- tool output tampering,
- replay of old tool results,
- “prompt-only audits” that cannot be verified,
- data leakage via tool interfaces.

Evidence adapters provide a consistent, verifiable envelope around:
- inputs (what was asked),
- outputs (what was returned),
- context (who/what/when),
- policy decisions (what was redacted),
- cryptographic bindings (hash/sign).

**Outcome:** “tool calls” become **audit-grade events**.

---

## 2. Adapter definition (normative)

An **adapter** is a component that sits at a trust boundary and produces evidence for each tool invocation.

Adapters MAY be implemented as:
- **in-process middleware** (inside the agent runtime),
- **sidecar / gateway** (between agent and tool),
- **tool-native** (inside the tool server itself).

Regardless of placement, an adapter MUST:
1) capture a minimal evidence record (§3),
2) canonicalize + bind it cryptographically (per suite/profile),
3) emit an Evidence ID and pack linkage,
4) enforce minimum disclosure (no secrets in cleartext).

---

## 3. Minimal tool-call evidence record (MUST)

For each tool invocation, the adapter MUST produce a **ToolCall Statement** with at least:

### 3.1 Identity & context
- `evidence_id` (stable)
- `tool.name`
- `tool.version` (or build digest)
- `actor.type` (agent/service/user) — policy-defined
- `actor.id` (pseudonymous ID; avoid PII)
- `timestamp.start`, `timestamp.end` (or duration)
- `trace.trace_id`, `trace.span_id` (if available)

### 3.2 Input binding (digest-first)
- `input.digest` — hash of canonicalized input representation
- `input.disclosure` — `none|redacted|encrypted|full`
- optionally `input.redacted_artifact_ref` (if redacted view is included)

### 3.3 Output binding (digest-first)
- `output.digest` — hash of canonicalized output representation
- `output.disclosure` — `none|redacted|encrypted|full`
- optionally `output.redacted_artifact_ref`

### 3.4 Policy and verification hints
- `profile` (policy/profile ID)
- `suite_id`
- `reason.hints` (optional, non-sensitive)
- `chain.prev_evidence_digest` (optional, for intra-run continuity)

> **Default posture:** store raw inputs/outputs only when policy allows; otherwise store digests and redacted views.

---

## 4. Canonicalization and signing (MUST)

All ToolCall Statements MUST be:
- canonicalized using the SealedProof JCS profile (`specs/canonicalization-jcs-profile.md`),
- hashed and signed using a suite from `specs/suite-registry.md`,
- domain-separated as `SPv1|statement|attest|<suite_id>|` (see domain separation spec).

This prevents:
- reinterpreting a tool call signature as a different object,
- cross-tool replay under different semantics.

---

## 5. Packaging into evidence packs

### 5.1 Per-run packing (recommended)
A typical agent “run” produces one evidence pack containing:
- all tool-call statements,
- the manifest and Merkle commitments (if used),
- optional anchors (TSA / transparency log),
- trust bundle snapshot references.

### 5.2 Per-call mini-packs (optional)
In some architectures, each tool call MAY produce a mini-pack for streaming verification.  
If used, it MUST still link into run-level continuity (e.g., prev/next digest linkage).

---

## 6. MCP mapping guidance (practical)

MCP message shapes vary by implementation, but adapters should treat MCP as:
- **request** (tool name + parameters),
- **response** (tool output),
- **transport** metadata (session, server identity).

Recommended mapping:
- tool identifier: `tool.name` = MCP tool name (fully qualified if available)
- server identity: include `tool.server_id` (non-sensitive)
- parameters: canonicalize a **normalized** parameters object (stable key ordering, consistent types)
- response: canonicalize normalized output (prefer structured JSON over free-form strings when possible)

If the tool output is non-JSON (binary/text), represent it as:
- an artifact file with digest, referenced from the statement, or
- a structured wrapper with `content_type` + `digest`.

---

## 7. Secrets, PII, and minimum disclosure (MUST)

Adapters are a data boundary. They MUST enforce `security/data-handling.md`:

- never record secrets in cleartext,
- never copy raw PII into statements/telemetry unless policy allows,
- prefer digests and redacted artifacts,
- record `disclosure` level explicitly.

**Redaction strategy:** produce deterministic redacted views, and bind them in the manifest so a verifier can validate integrity of what was disclosed.

---

## 8. Failure modes and verifier expectations

Adapters MUST produce deterministic failure evidence:
- if canonicalization fails → emit a reason code and fail closed (strict profiles)
- if tool returns malformed output → record output digest of the raw bytes and mark parsing failure
- if policy detects forbidden data → fail validation under strict profiles

Verifiers SHOULD return reason codes aligned with `conformance/validation-workflow.md`.

---

## 9. Correlation with OpenTelemetry (SHOULD)

Adapters SHOULD:
- emit `sealedproof.evidence_id` on the tool call span,
- emit `sealedproof.pack_digest` at run completion,
- keep digests/IDs in telemetry, keep payload inside packs.

See `integrations/opentelemetry.md`.

---

## 10. Implementation checklist

**Agent runtime**
- [ ] Intercept tool call boundary.
- [ ] Normalize + canonicalize inputs/outputs (digest-first).
- [ ] Apply redaction policy deterministically.
- [ ] Produce ToolCall Statement + Evidence ID.
- [ ] Add statement and artifacts into pack; update manifest coverage.
- [ ] Sign/anchor per suite/profile; emit pack digest.

**Tool provider**
- [ ] Expose stable tool identity + version/build hash.
- [ ] Prefer structured output (JSON) where possible.

**Security**
- [ ] Validate that adapters never leak secrets/PII in telemetry.
- [ ] Enforce strict profiles for production-critical tools.

---

**Status:** v0.1 baseline. Designed to be compatible with evolving MCP and agent ecosystems without locking to a single runtime vendor.

