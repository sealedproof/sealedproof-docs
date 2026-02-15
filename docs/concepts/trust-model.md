# Trust Model: Why Logs Aren’t Enough in the Agent Era

> **Thesis:** In an agent-driven world, *execution* scales faster than *oversight*. Traditional “log-centric” auditing is not built to withstand autonomous tool-use, multi-actor delegation, and adversarial incentives. **SealedProof** is designed for a different world: **evidence-first** verification—offline verifiable, non-repudiable, minimal disclosure, and transferable.

---

## 1. The Shift: From “Human Actions” to “Agent Operations”

Agents don’t just “generate text.” They:
- call tools (APIs, databases, wallets, cloud consoles),
- execute workflows across multiple services,
- act continuously and in parallel,
- operate with delegated authority (keys, tokens, service accounts),
- and can be chained (agent → agent → subcontracted agent).

**The problem:** once agents exist, *the number of consequential actions explodes*—and the cost of a single compromised action becomes catastrophic.

---

## 2. What We Are Protecting

We protect the ability to answer, **after the fact**, with high confidence:

1) **What happened?**  
2) **Who/what authorized it?**  
3) **What inputs led to it?**  
4) **What tool was invoked, with what parameters, and what response came back?**  
5) **Was anything omitted, altered, or replayed?**  
6) **Can an independent verifier confirm it offline?**  
7) **Can we share proof without over-sharing sensitive data?**

This is not “observability.”  
This is **accountability under adversarial conditions**.

---

## 3. Threats Unique to the Agent Era

### 3.1 Incentive Misalignment and Plausible Deniability
When agents act:
- operators can deny intent (“the model did it”),
- vendors can deny responsibility (“tooling did it”),
- and the system can lose a clear chain of custody.

**We need cryptographic accountability, not narrative accountability.**

### 3.2 Toolchain Tampering
Attackers can:
- modify logs at rest,
- truncate logs (“selective memory”),
- rewrite history after the fact,
- or inject fake entries to create plausible cover stories.

### 3.3 Replay, Rollback, and Equivocation
In distributed systems, attackers can:
- replay a previously valid action,
- roll back state to erase evidence,
- or present different “truths” to different parties (**equivocation**).

### 3.4 Partial Disclosure vs. Total Exposure
Traditional audit approaches often force a tradeoff:
- either share full raw logs (leaking secrets / user data),
- or share redacted summaries (not independently verifiable).

We need **minimal disclosure proofs** with **verifiable completeness**.

### 3.5 Third-Party Verification is Now Mandatory
Agents will be deployed in regulated contexts, sensitive supply chains, or high-risk automation.
You cannot rely on a vendor’s internal statements.
You need **portable evidence** that a third party can verify.

---

## 4. Why “Logs” Are Not Evidence

Logs are:
- **mutable** (by privileged actors),
- **context-fragile** (missing inputs/outputs or correlation),
- **non-transferable** (hard to “hand off” to another verifier),
- **non-reproducible** (verification depends on internal systems),
- **privacy-hostile** (raw logs leak too much),
- and typically **not cryptographically bound** to the artifacts they describe.

### In other words:
**Logs are stories. Evidence is a proof.**

---

## 5. The SealedProof Trust Model (At a Glance)

### 5.1 Actors
- **Producer**: the system that generates a proof record for an agent action  
- **Verifier**: an offline verifier (human or tool) that checks integrity and policy compliance  
- **Relying Party**: any third party who needs to trust the result (auditor, customer, regulator, partner)  
- **Storage**: where evidence packs live (can be untrusted)

### 5.2 Trust Boundaries
We assume:
- storage can be compromised,
- transport can be observed,
- producers can be pressured or attacked,
- and the verifier must work offline.

We do **not** assume:
- access to internal databases,
- access to cloud consoles,
- or trust in the producer’s runtime environment.

### 5.3 What Must Be True
To “trust” a claim, the relying party must be able to verify:

- **Integrity**: data has not been modified  
- **Completeness**: required artifacts are present (no silent omission)  
- **Authenticity**: evidence originated from the claimed producer key  
- **Non-repudiation**: producer cannot later deny having produced it  
- **Context binding**: inputs/outputs are bound to the tool call and timing  
- **Anti-rollback / anti-replay**: the pack has continuity defenses  
- **Minimal disclosure**: sensitive fields can be selectively disclosed without breaking verification

---

## 6. Evidence Pack vs. Log Stream

### Evidence Pack (SealedProof)
A **sealed bundle** that is:
- cryptographically signed,
- content-addressed (hashes),
- and complete by manifest.

It can be:
- exported,
- stored anywhere,
- emailed,
- or uploaded to an auditor,
then verified **offline**.

### Log Stream (Traditional)
A server-side append stream that is:
- dependent on centralized custody,
- hard to transfer,
- hard to verify without privileged access,
- and typically unverifiable as a whole by third parties.

---

## 7. Minimal Disclosure: “Share Proof, Not Raw Data”

The trust model supports:
- disclosing only what is necessary for a verifier to check a claim,
- while proving that what remains hidden has not been tampered with,
- and that nothing required by policy is missing.

**Goal:** auditors verify *facts* without receiving *everything*.

---

## 8. Design Goals and Non-Goals

### Goals
- **Offline verification** as a first-class guarantee
- **Non-repudiation** for evidence producers
- **Transferable proof** across org boundaries
- **Policy-driven verification** with explainable failure reasons
- **Interoperable suites** (international + national crypto stacks)
- **Open ecosystem posture** (SDKs, schemas, verifiers)

### Non-Goals
- Real-time monitoring dashboards (observability ≠ verification)
- Full attestation of runtime security (we can integrate with TEEs, but don’t depend on them)
- Replacing all existing logging (we complement, not duplicate, operational logs)

---

## 9. A Simple Mental Model

**Log World:**  
> “Trust us, here’s what happened.”

**Proof World (SealedProof):**  
> “Here is a portable evidence pack. Verify it independently. Offline.”

---

## 10. Outcome: A New Baseline for Agent Accountability

In the agent era, “audit logs” become a UI for a deeper primitive:
**verifiable evidence**.

SealedProof’s trust model is built on a single idea:

> If an action matters, the system must be able to **prove** it—  
> **offline**, to **anyone**, with **minimal disclosure**, and without relying on internal trust.

