# Kernel v1.2 — Audit Chain Visual Reference

**(Accurate to this kernel implementation)**

> **Scope note:** This document describes *kernel‑internal audit‑chain mechanics*. It does **not** define audit sufficiency, compliance guarantees, trusted time, identity/authenticity, or policy decisions. Those concerns are intentionally external.

This file is a **visual and mental model** for how the Kernel v1.2 audit chain works in the attached kernel artifacts.

---

## Key rules (locked for this implementation)

* Hashing uses **SHA‑256** over stable JSON (`stableStringify(...)`).
* Segments compute `h` and `ch` **at seal time**.
* Export is **persisted‑chain‑only**: only persisted chain records and their referenced persisted segments are exported.
* The exporter **never infers or manufactures gaps**.
* `type:"seal"` is supported by the verifier, but the kernel does **not** auto‑emit a seal record by default.

---

## 1) Chain in one picture

**RUN METADATA** (first NDJSON line)

* explains `run_id`, lens, keys, etc.

```
root_ch = SHA256(stableStringify([
  "audit_root_v1.2",
  run_id
]))

      ┌───────────────┐
      │  root_ch (R)  │
      └───────┬───────┘
              │  link(prev_ch, h)
              ▼
   ┌───────────────────────┐
   │ SEGMENT chain record   │  (persisted)
   │  h = segmentHash(...)  │
   │  ch = chainHash(R, h)  │
   └──────────┬────────────┘
              │
              ▼
   ┌───────────────────────┐
   │ GAP chain record       │  (persisted, optional)
   │  h = gapHash(...)      │
   │  ch = chainHash(prev,h)│
   └──────────┬────────────┘
              │
              ▼
   ┌───────────────────────┐
   │ SEGMENT chain record   │  (persisted)
   └──────────┬────────────┘
              │
              ▼
   ┌───────────────────────┐
   │ SEAL chain record      │  (persisted, optional)
   │  root_ch, terminal_ch  │
   └───────────────────────┘
```

**Note:** GAP records are exceptional. They are emitted only when a failure boundary is explicitly recorded (for example, a watchdog‑triggered persistence failure), not during normal operation.

---

## 2) Exact hash formulas (as implemented)

### Segment hash

```
h = SHA256(stableStringify([
  "segment_h_v1.2",
  canonicalSegmentBody(seg)
]))
```

`canonicalSegmentBody(seg)` includes:

* `run_id`
* `seg_id`
* `start_ts`, `end_ts`
* `count`
* `sealed`
* `events`

It explicitly **excludes** `h` and `ch`.

---

### GAP hash

```
h_gap = SHA256(stableStringify([
  "gap_h_v1.2",
  { seg_id_start, seg_id_end, reason_code }
]))
```

* `reason_text` may exist for display or diagnostics but is **not hashed**.

---

### Chain link hash

```
ch_next = SHA256(stableStringify([
  "link_v1.2",
  prev_ch,
  h_or_h_gap
]))
```

---

## 3) GAPs in this implementation

A GAP is **not** “a missing segment discovered during export”.

Instead:

* GAPs exist **only if they were persisted** as chain records.
* The watchdog path emits **exactly one GAP** on a stall or failure boundary.
* `reason_code = 2` indicates `worker_failure`.
* The GAP range is `{ seg_id_start, seg_id_end }` with `seg_id_end` exclusive.
* If the persisted chain references a segment that is missing from storage, export fails with:
  `missing_segment_referenced_by_chain:<seg_id>`.

**Important clarification:**
Absence of a GAP does **not** imply uninterrupted capture. It only implies that **no discontinuity was successfully persisted**.

This model distinguishes **what was persisted** from **what may have occurred**. It does not infer intent, completeness, or correctness beyond the chain itself.

---

## 4) Export format snapshot (record shapes)

Export is **NDJSON** with `v: "1.1"` on every record. (Kernel v1.2 integrity is additive via fields such as `h` and `ch`.)

The exporter emits:

* `type:"run"` — run metadata
* zero or more persisted chain records:

  * `type:"segment"` — contains `seg: { ... h, ch, events ... }`
  * `type:"gap"` — contains `seg_id_start`, `seg_id_end`, `reason_code`, `h`, `ch`
  * optional `type:"seal"` — contains `algo`, `root_ch`, `terminal_ch`
* `type:"trace"` — diagnostic records (non‑chain)

---

## 5) Verifier outcomes (observable behavior)

The verifier returns `{ ok, status, ... }`.

If `allow_partial: true`:

* missing seal → `status: "partial"` (with `last_ch`)
* truncated final JSON line → `status: "partial"`

If `allow_partial: false`:

* missing seal → `status: "invalid"`, error: `missing_seal_line`
* any hash mismatch, reordering, or tampering → `status: "invalid"`

These outcomes describe **verifier behavior**, not audit policy. Whether `PARTIAL` is acceptable is an external decision.

---

## Why the active segment is excluded from export

Exporting unsealed data produces unverifiable output — records that appear in the file but were never sealed into the hash chain.

Persisted‑chain‑only export avoids this failure mode and ensures that every exported record is verifiable within the chain.
