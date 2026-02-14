
# SealedProof Docs

**A verifiable evidence layer for AI agent actions**  
Offline verifiable · Non-repudiable · Minimal disclosure · Evidence handoff

Agents are moving from prototypes into production workflows. That shift changes the security question from *“what did we log?”* to *“what can we prove?”*  
Logs help you **observe**. They don’t reliably let a third party **verify** what an agent actually did—especially after incidents, disputes, or vendor handoff.

SealedProof defines a pragmatic, open evidence format and verification workflow for **agent action receipts**—so security teams, auditors, and customers can validate agent behavior *offline* and *after the fact*, without needing your internal systems.

---

## What SealedProof provides

### Evidence receipts for agent actions
A “receipt” is a cryptographically verifiable statement about an agent action (tool call, decision, output artifact), designed for **independent verification**.

### Evidence packs for handoff
Evidence can be bundled into a portable **Evidence Pack** that you can hand to:
- a customer security team for acceptance
- an internal audit function
- incident response / forensics
- a third-party assessor

### Offline verification with reason codes
Verification produces a deterministic result plus **reason codes** that explain *why* a pack is valid/invalid—useful for audits and procurement acceptance criteria.

---

## Core guarantees (what we optimize for)

- **Offline verifiable**: verification does not require network access or live dependencies.  
- **Non-repudiable**: signatures and domain-separated hashes bind “what was claimed” to “who signed it.”  
- **Minimal disclosure**: prove integrity and provenance without exposing sensitive payloads when not required.  
- **Evidence handoff**: a pack is portable, complete, and verifiable by third parties.  
- **Explainability for auditors**: stable reason codes → consistent acceptance criteria.

---

## How it works (high-level)

1. **Capture**: the agent runtime emits receipts at critical boundaries (tool calls, artifacts, policy decisions).  
2. **Normalize**: receipts are canonicalized for cross-language determinism (JSON canonicalization profile).  
3. **Seal**: domain-separated hashes + signatures create a verifiable proof envelope.  
4. **Bundle**: receipts, manifests, and metadata are exported as an Evidence Pack.  
5. **Verify**: an offline verifier checks the pack and outputs a report + reason codes.

SealedProof is designed to **complement** observability (traces/metrics/logs). You can keep OpenTelemetry for runtime visibility, and use SealedProof for *proof-grade* third-party verification.

---

## Who uses this

- **Security & GRC teams**: audit trails you can hand to a third party without “trust us.”  
- **Platform teams shipping agents**: acceptance tests for customers and regulated buyers.  
- **Agent framework & tool vendors**: a standard export format that makes integrations easier.  
- **Incident response**: integrity + provenance when something goes wrong.

---

## Start here

- **Quickstart (5 min)**: [quickstart.md](quickstart.md)  
- **Demo Evidence Pack**: [demo-evidence-pack.md](demo-evidence-pack.md)  
- **Specs (v0.1 draft)**: [specs/overview.md](specs/overview.md)  
- **Offline Verifier CLI**: [specs/offline-verifier-cli.md](specs/offline-verifier-cli.md)  
- **Reason Codes**: [specs/reason-codes.md](specs/reason-codes.md)  
- **Suite Registry** (international + national crypto suites): [specs/suite-registry.md](specs/suite-registry.md)

---

## Open standards alignment (selected)

SealedProof is opinionated, but not isolated. We align with widely used standards and industry practice, including:

- **NIST AI RMF 1.0** (risk framing for trustworthy AI)  
- **Microsoft Responsible AI Standard** (operational requirements and governance expectations)  
- **OpenTelemetry GenAI semantic conventions** (observability vocabulary for GenAI/agent spans)  
- **IETF RFC 8785 (JCS)** for deterministic JSON canonicalization

---

## Open-source posture

We treat open source as the default distribution path:
- Specs and documentation are public and versioned.
- Reference verifier and conformance tests are designed to be reusable.
- We welcome PRs that improve correctness, interoperability, and clarity.

If you want to help shape the “receipt layer” for the agent era, start by opening an issue on the docs/spec repos with:
- a missing use case
- an edge case that breaks verification determinism
- a compliance/audit requirement we should encode into reason codes

---

*SealedProof focuses on verifiability and evidence handoff. It is not a policy engine, and it does not replace runtime security controls.*
