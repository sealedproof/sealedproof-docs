# Roadmap (MVP → Interop → Audit Ecosystem → Standardization) (v0.1)

**Path:** `roadmap.md`  
**Status:** Public roadmap (living document)  
**Audience:** enterprise buyers, security teams, integrators, open-source community  
**Goal:** communicate a credible execution path toward an industry-grade evidence layer—without overpromising.

This roadmap is intentionally oriented around **verifiability milestones** and **ecosystem adoption**, not vanity features.

---

## Guiding principles

- **Evidence-first:** every milestone improves third-party verifiability, not just internal observability.
- **Interop > lock-in:** publish specs, vectors, and adapters to reduce integration friction.
- **Fail closed by default:** strict profiles and deterministic reason codes.
- **Open ecosystem:** OSS-friendly interfaces; keep core trust guarantees stable.

---

## Phase 0 — Foundation (Now → MVP)

**Objective:** ship a usable evidence pack pipeline with offline verification and strict safety defaults.

Deliverables:
- Evidence pack format v0.1 (manifest coverage, digests, statements)
- Canonicalization JCS profile v0.1
- Domain separation v0.1
- Suite Registry v0.1 (at least one recommended suite)
- Verifier CLI v0.1 (offline verification, reason codes, reports)
- Security baseline + key management + data handling + replay/rollback docs
- Validation workflow (3rd-party verifier path)

Success criteria:
- A third party can validate a pack offline and produce a deterministic report.
- Packs fail closed under strict mode; warnings do not silently pass.
- Minimal disclosure works: redacted artifacts can be validated.

---

## Phase 1 — Interoperability (MVP+)

**Objective:** make SealedProof “easy to integrate” into real agent stacks and security tooling.

Deliverables:
- OpenTelemetry correlation conventions (trace/span ↔ evidence/pack)
- MCP adapters guidance + reference adapter(s)
- Export mapping bridge(s) (e.g., in-toto statement mapping) where appropriate
- Conformance test vectors published as tagged releases
- Compatibility policy and migration guidance (no surprises)

Success criteria:
- Two independent implementations can verify the same vectors and packs.
- Integrators can add evidence with minimal code changes.
- SOC teams can pivot trace → evidence → report.

---

## Phase 2 — Audit & Compliance Ecosystem

**Objective:** make evidence packs operationally useful for audits, incident response, and vendor due diligence.

Deliverables:
- Standard verifier report formats (JSON + human-readable summary)
- Evidence pack handoff patterns (secure sharing, access control profiles)
- Policy profiles for common environments (enterprise strict, regulated, offline-first)
- Integration recipes for SIEM / ticketing / GRC workflows (guides, not hard dependencies)

Success criteria:
- Auditors can request evidence packs instead of screenshots.
- Vendor assessments can verify claims using stable reason codes and vectors.
- Organizations can standardize “evidence requirements” in procurement.

---

## Phase 3 — Standardization & Industry Alignment

**Objective:** move from “a product spec” to “a shared industry practice”.

Deliverables:
- Participate in relevant communities and working groups (where applicable)
- Publish drafts focusing on interoperability contracts (formats, profiles, reason codes)
- Align with adjacent standards and ecosystems (OTel, supply chain, transparency log practices)
- Reference implementation(s) and test suites to validate interoperability

Success criteria:
- External parties cite the spec/vectors as a baseline for verifiable agent audits.
- Multi-vendor ecosystems can exchange packs and verify results.
- Adoption grows through integration, not marketing claims.

---

## What we will not do

To keep trust, we explicitly avoid:
- “magic AI security” claims without verification contracts
- opaque “black box trust”—verification must work offline and deterministically
- forcing lock-in through proprietary-only formats

---

## How to track progress

Each phase ships:
- spec updates (versioned)
- vector releases (tagged)
- verifier releases
- migration notes

If you’re an enterprise or tool vendor and want to align early, start with:
- `conformance/validation-workflow.md`
- `security/security-baseline.md`
- `specs/evidence-pack-format.md`

