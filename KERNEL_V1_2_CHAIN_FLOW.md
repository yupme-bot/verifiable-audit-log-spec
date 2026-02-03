# Kernel v1.2 — Audit Chain Visual Reference (Accurate to this repo)

This file is a **visual / mental model** for how the v1.2 audit chain works **in the attached kernel ZIP**.

Key repo rules (locked here):
- Hashing uses **SHA-256 over stable JSON** (`stableStringify(...)`).
- **Segments get `h` + `ch` computed at seal time**.
- Exporter is **persisted-chain-only**: it exports only persisted chain records and their referenced persisted segments.
- Exporter **never infers/manufactures gaps**.
- `type:"seal"` is **supported by the verifier**, but the kernel **does not auto-emit** a seal record by default.

---

## 1) Chain in one picture

```
RUN METADATA (first NDJSON line)
 explains run_id, lens, keys, etc.

root_ch = SHA256(stableStringify(["audit_root_v1.2", run_id]))

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
`run_id, seg_id, start_ts, end_ts, count, sealed, events` (and excludes `h/ch`).

### Gap hash
```
h_gap = SHA256(stableStringify([
  "gap_h_v1.2",
  { seg_id_start, seg_id_end, reason_code }
]))
```

`reason_text` may exist for display, but is **not hashed**.

### Chain link hash
```
ch_next = SHA256(stableStringify([
  "link_v1.2",
  prev_ch,
  h_or_h_gap
]))
```

---

## 3) GAPs in this repo

A GAP is **not** “a missing segment found during export”.

Instead:
- GAPs exist **only if they were persisted as chain records**.
- The watchdog path emits a GAP **exactly once** on stall:
  - `reason_code = 2` (worker_failure)
  - range is `{ seg_id_start, seg_id_end }` with `seg_id_end` exclusive

If the persisted chain references a segment that is missing from storage, export throws:
`missing_segment_referenced_by_chain:<seg_id>`.

---

## 4) Export format snapshot (accurate shapes)

Export is NDJSON with **`v: "1.1"` on every record** (v1.2 is additive via fields like `h/ch`). The exporter emits:

1) `type:"run"` (metadata)
2) zero or more **persisted chain records**:
   - `type:"segment"` lines (contain `seg: { ... h, ch, events ... }`)
   - `type:"gap"` lines (contain `seg_id_start`, `seg_id_end`, `reason_code`, `h`, `ch`)
   - optionally `type:"seal"` lines (contain `algo`, `root_ch`, `terminal_ch`)
3) `type:"trace"` lines

---

## 5) Verifier outcomes (what you’ll see)

The verifier returns `{ ok, status, ... }`:

- If `allow_partial: true`:
  - missing seal => `status: "partial"` (but still returns `last_ch`)
  - truncated final JSON line => `status: "partial"`
- If `allow_partial: false`:
  - missing seal => `status: "invalid"`, `error: "missing_seal_line"`

Any hash mismatch / reordering / tampering => `status: "invalid"`.

---


**Why the active segment is excluded from export:**

Exporting unsealed data creates unverifiable output—records that appear in the file but were never sealed into the hash chain. Persisted-chain-only export avoids this failure mode.
