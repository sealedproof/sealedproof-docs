# JCS Canonicalization Profile (Sealedproof)

**Status:** Draft (public)  
**Audience:** implementers, security engineers, auditors  
**Goal:** deterministic, cross-platform canonical JSON bytes for hashing/signing inside Sealedproof evidence workflows.

This document defines the **Sealedproof JCS Profile**: a strict, interoperable subset of JSON canonicalization rules aligned with **RFC 8785 (JCS)** and **RFC 7493 (I-JSON)**, plus Sealedproof-specific conformance requirements and validation errors.

---

## 1. Normative references

- RFC 8785 — JSON Canonicalization Scheme (JCS)  
- RFC 7493 — The I-JSON Message Format (I-JSON)  
- RFC 8259 — The JSON Data Interchange Syntax  
- ECMA-262 (ES6+) — ECMAScript (for JSON primitive serialization behavior referenced by RFC 8785)

---

## 2. Terms

- **Canonical JSON bytes**: UTF-8 encoded canonical JSON text (no BOM) produced by this profile.
- **Producer**: a component that emits evidence data and/or computes hashes/signatures.
- **Verifier**: a component that recomputes canonical bytes and validates hashes/signatures.

---

## 3. Profile scope

This profile applies wherever Sealedproof needs **byte-for-byte stability**:

- hashing payloads into Merkle trees / manifests
- signing statements (attestations) and derived claims
- verifying third-party evidence packs offline

> The canonicalization target MUST be clearly defined by the calling spec (e.g., “canonicalize object X excluding field Y”). This profile defines **how** canonicalization is performed, not **what** fields are included.

---

## 4. Input constraints (I-JSON alignment)

A Producer MUST reject input that violates I-JSON/JCS constraints:

### 4.1 Duplicate property names
- JSON objects MUST NOT contain duplicate property names.

### 4.2 Unicode validity and stability
- JSON strings MUST be expressible as Unicode.
- Parsed JSON string data MUST NOT be altered during subsequent processing/serialization (no normalization, no rewriting).
- Lone surrogates MUST be rejected (e.g., an isolated U+DEAD surrogate).

### 4.3 Numeric constraints
- JSON numbers MUST be expressible as IEEE 754 double-precision values.
- Non-finite values (NaN, +Infinity, -Infinity) MUST be rejected.
- If higher precision or big integers are required, they MUST be represented as **strings** (with an application-level convention).

---

## 5. Canonicalization rules

### 5.1 Whitespace
- Whitespace between JSON tokens MUST NOT be emitted.

### 5.2 Literals
- `null`, `true`, `false` MUST be serialized exactly as shown (lowercase).

### 5.3 Strings
String canonicalization MUST follow RFC 8785 / ECMAScript rules:

- Control characters U+0000..U+001F MUST be escaped.
  - U+0008, U+0009, U+000A, U+000C, U+000D MUST be escaped as `\b`, `\t`, `\n`, `\f`, `\r`.
  - All other control characters MUST be escaped as `\uhhhh` using **lowercase hex**.
- `"` and `\` MUST be escaped as `\"` and `\\`.
- Other characters MUST be emitted “as is”.
- The resulting sequence MUST be enclosed in `"`.

**Forbidden / reject:**
- invalid Unicode (including lone surrogates)
- re-encoding that changes code points
- upper-case hex in `\u` escapes (MUST use lowercase)

### 5.4 Numbers
Numbers MUST be serialized according to the JSON number serialization rules referenced by RFC 8785 (ECMAScript `JSON.stringify()` compatible serialization).

**Operational requirements:**
- A Producer MUST use a serializer known to match RFC 8785 number formatting (or a verified implementation based on V8/Ryu-style algorithms).
- A Verifier MUST treat any numeric formatting mismatch as canonicalization failure.

**Key implications (non-exhaustive):**
- No leading `+`.
- No superfluous leading zeros.
- Use exponent form when required by the referenced algorithm.
- Beware of rounding: different internal numeric types can produce divergent JSON texts if not normalized.

### 5.5 Sorting object properties
Object property sorting MUST follow RFC 8785:

- Objects MUST be sorted recursively (child objects too).
- Arrays MUST be scanned for embedded objects (sort their properties), but array element order MUST NOT change.
- Sorting is applied to property names in their **raw (unescaped)** form.
- Property names MUST be compared as arrays of **UTF-16 code units**, compared as unsigned integers, independent of locale.
- Lexicographic order is determined by first differing code unit; if identical prefix, shorter string precedes longer.

**Important interop note:** sorting by UTF-8 or locale-aware collation is NOT equivalent and can break verification.

### 5.6 UTF-8 output bytes
- The final canonical JSON text MUST be encoded as UTF-8.
- UTF-8 BOM MUST NOT be emitted.

---

## 6. Conformance checklist (implementation)

### 6.1 Producer MUST
- Validate I-JSON constraints (no dup keys, valid Unicode, IEEE-754 representable numbers).
- Canonicalize using the rules in §5.
- Hash/sign the **canonical UTF-8 bytes**, not a language-specific string object.

### 6.2 Verifier MUST
- Recompute canonical bytes using the same rules.
- Reject if canonicalization fails or bytes differ.

### 6.3 Determinism tests (minimum)
Implementations SHOULD include a test suite covering:
- property sorting with non-ASCII keys and surrogate pairs
- control character escaping (`\n`, `\u000f`, etc.)
- negative zero (`-0`) behavior and rounding edge cases
- rejection of NaN/Infinity
- rejection of lone surrogates
- duplicate-key detection

---

## 7. Reference vectors (small)

### 7.1 Property sorting + control escaping
**Input (conceptual):**
```json
{"b":1,"a":[{"d":true,"c":"\u000A"}]}
```

**Canonical UTF-8 text:**
```json
{"a":[{"c":"\n","d":true}],"b":1}
```

---

## 8. Sealedproof integration notes (non-normative)

- Canonicalization is performed **before** hashing and signing.
- Higher-level specs define canonicalization boundaries (e.g., exclude signature fields, include declared “hash domain separation” labels).
- When exporting to other ecosystems (in-toto, SCITT, etc.), Sealedproof preserves canonicalization determinism by continuing to anchor hashes to canonical UTF-8 bytes.

---

## 9. Failure handling (recommended)

A Verifier SHOULD emit a stable reason code on failure (examples):

- `SP-JCS-001` duplicate object property name
- `SP-JCS-010` invalid Unicode (lone surrogate)
- `SP-JCS-020` non-finite number (NaN/Infinity)
- `SP-JCS-030` number serialization mismatch
- `SP-JCS-040` property sorting mismatch
- `SP-JCS-050` UTF-8 encoding/BOM violation

(Reason code registry is defined in `conformance/validation-workflow.md`.)

---

## 10. Security considerations (non-normative)

Canonicalization prevents “semantic-equivalent / byte-different” ambiguity attacks. However:

- Canonicalization does not make malicious data safe; it only makes it **stable** for cryptographic binding.
- Treat canonicalization failures as hard errors; do not “best-effort” normalize untrusted input.
- Be explicit about canonicalization scope to avoid signing unintended fields.


