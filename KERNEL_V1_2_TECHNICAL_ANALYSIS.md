# Kernel v1.2 — Cryptographic Design (Technical Analysis)

> **Scope note:** This document describes the **kernel-internal cryptographic and determinism design** for audit-log artifacts. It does **not** define audit sufficiency, legal compliance, authenticity, trusted time, or policy decisions. Those concerns are intentionally external and composable.

---

## Executive Summary

Kernel v1.2 implements a **deterministic, cryptographically chained audit artifact model** for event capture systems.

The design provides:

* **Tamper-evidence** — modifications break cryptographic continuity
* **Crash awareness** — partially captured logs remain verifiable up to the failure boundary
* **Explicit gap semantics** — missing data is recorded, not inferred
* **Determinism** — identical inputs produce identical hashes across environments
* **Versioned evolution** — forward compatibility via domain separation and additive fields

This document explains *how* these properties are achieved and *what they do and do not guarantee*.

---

## Core Cryptographic Primitive

### Hash Algorithm: SHA-256

Kernel v1.2 uses **SHA-256** consistently for all integrity-relevant hashes.

**Rationale:**

* Widely analyzed, industry-standard algorithm
* Available in all target environments (Node, browser, workers)
* Sufficient collision resistance for audit integrity
* Hardware acceleration on modern platforms

SHA-256 is used for:

* segment hashes (`h`)
* gap hashes (`h_gap`)
* chain links (`ch`)
* optional terminal seals (`terminal_ch`)

Algorithm agility is intentionally deferred to versioned evolution rather than runtime negotiation.

---

## Chain Hash Architecture

### Chain Initialization (Genesis)

```
ch_0 = SHA256(stableStringify([
  "audit_root_v1.2",
  run_id
]))
```

**Properties:**

* Deterministic: same `run_id` → same genesis hash
* Unique per run
* Domain-separated to prevent cross-protocol reuse

No zero values or implicit sentinels are used.

---

## Canonical Encoding

Deterministic hashing requires canonical serialization.

### Requirements

Determinism must hold across:

* JavaScript engines (V8, SpiderMonkey, JavaScriptCore)
* Node, browser, and worker contexts
* Operating systems

### Rules

**Key ordering**

* Object keys sorted lexicographically

**Whitespace independence**

* Formatting does not affect hash input

**Number normalization**

* Timestamps and counters represented as integers

**Encoding**

* JSON → UTF-8 bytes → SHA-256

These rules ensure identical semantic data produces identical hashes.

---

## Segment Hashing

Segments are hashed *only when sealed*.

```
h = SHA256(stableStringify([
  "segment_h_v1.2",
  canonicalSegmentBody(seg)
]))
```

The canonical body includes identifiers, bounds, counters, and events, and explicitly excludes `h` and `ch`.

This prevents recursive hashing and preserves determinism.

---

## Gap Handling

Kernel v1.2 supports **explicit persisted GAP records**.

Important constraints:

* GAPs are exported **only if they were persisted**
* The exporter never infers or synthesizes gaps
* Absence of a GAP implies no *persisted* evidence of discontinuity

### GAP Hashing

```
h_gap = SHA256(stableStringify([
  "gap_h_v1.2",
  { seg_id_start, seg_id_end, reason_code }
]))
```

Human-readable reason text may exist but is not hashed, allowing descriptive evolution without breaking integrity.

---

## Seal Records (Optional)

Kernel v1.2 supports an optional terminal `seal` record.

* The verifier can require seals in strict mode
* The kernel does not auto-emit seals by default

Seals bind the deterministic root hash and final chain state but do not add additional cryptographic layers in v1.2.

---

## Persistence and Recovery

Stored `ch` values act as **cryptographic checkpoints**.

On restart:

* storage is authoritative
* the chain resumes from the highest sealed segment
* unsealed data is discarded

This preserves integrity across crashes without rewriting history.

---

## Verification Semantics

Verification evaluates **artifact integrity only**.

Possible outcomes:

* **OK** — integrity and continuity preserved
* **PARTIAL** — integrity preserved, completeness unknown
* **INVALID** — integrity violated

These outcomes describe *what can be proven*, not *what should be accepted*.

---

## Security Properties

### Tamper Evidence

Any modification to events, ordering, gaps, or hashes breaks chain continuity and is detectable.

### Integrity Cascade

Each chain link depends on all prior links. A single divergence invalidates all subsequent verification.

### Explicit Absence

Persisted GAP records cryptographically commit the *position* and *extent* of missing data.

This model proves **what is present and what is absent**, not whether absence was legitimate.

---

## Performance Characteristics

* Hashing occurs at seal time, not per event
* Typical segment hashing latency is sub-millisecond
* Memory usage is bounded via segmenting and caching

The cryptographic overhead is negligible relative to I/O and event handling.

---

## Compatibility and Evolution

* v1.2 verifiers can detect and label v1.1 exports
* Migration requires replay or restart; in-place mutation is forbidden
* Future versions evolve via domain tags and additive fields

Version mismatches fail safely and visibly.

---

## Limitations

This design intentionally does **not** provide:

* authenticity or identity guarantees
* trusted timestamps
* non-repudiation
* inclusion/exclusion proofs beyond linear verification

Those guarantees must be layered externally if required.

---

## Conclusion

Kernel v1.2 defines a **deterministic, gap-aware, cryptographically chained audit artifact model**.

It is suitable as a **technical component** in higher-level systems that require:

* explicit integrity guarantees
* honest representation of failure
* deterministic replay and verification

It is not a complete audit, compliance, or logging solution on its own.
