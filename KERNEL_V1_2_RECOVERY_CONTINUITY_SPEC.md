## Recovery & Continuity Specification (Kernel v1.2)

> **Scope note:** This document defines **kernel-internal recovery and continuity semantics**. It does **not** define audit sufficiency, authenticity, trusted time, or policy decisions. Those concerns are external to the kernel and this specification.

This section describes how the kernel behaves when failures occur, how continuity is preserved *when possible*, and how recovered audit artifacts should be interpreted.

---

## 6. Recovery Decision Matrix

The table below summarizes recovery behavior under different failure conditions. It describes **mechanical outcomes**, not judgments about correctness or acceptability.

| Condition                         | Degraded Mode | GAP Persisted | Recovery Allowed | Resulting Chain                   |
| --------------------------------- | ------------- | ------------- | ---------------- | --------------------------------- |
| Normal operation                  | No            | No            | No-op            | Continuous                        |
| Watchdog fired                    | Yes           | Yes           | Yes              | `[Seg N] → [GAP] → [Seg N+1]`     |
| Watchdog fired, GAP not persisted | Yes           | No            | No               | Chain ends at last sealed segment |
| Storage missing                   | Yes           | Yes           | No               | Chain frozen                      |
| Storage corrupted                 | Yes           | N/A           | No               | Chain terminal                    |

**Interpretation:**

* Recovery is allowed **only** when failure boundaries were explicitly persisted.
* Absence of a persisted GAP prevents recovery, even if a failure occurred.

---

## 7. Non-Goals of Recovery

Recovery is intentionally conservative. It **does not** attempt to repair or conceal failures.

Specifically:

* Recovery does **not** reconstruct lost events.
* Recovery does **not** infer or synthesize missing segments.
* Recovery does **not** reorder, compact, or rewrite history.
* Recovery does **not** guarantee continuity of time — only continuity of proof.
* Recovery does **not** attempt to hide or soften operational failures.

These constraints preserve interpretability and prevent post hoc manipulation.

---

## 8. Forensic Interpretation

Recovered chains must be interpreted carefully:

* A recovered chain indicates **controlled continuation**, not uninterrupted operation.
* Absence of a GAP implies **no persisted evidence of discontinuity**, not proof of perfect capture.
* Timestamp discontinuities after recovery are expected and **must not** be treated as chain corruption.

This model distinguishes **what is cryptographically verifiable** from **what may have occurred operationally**.

---

## 9. Verifier Expectations

A verifier evaluating a recovered artifact should:

* Validate continuity using only chained `ch` values.
* Treat time gaps as informational, not integrity failures.
* Fail verification **only** if chain order, linkage, or cryptographic integrity is violated.

Verification outcomes describe **artifact integrity**, not operational health or audit acceptability.

---

## 10. Optional Recovery Telemetry

Implementations may emit **non-hashed diagnostic records** indicating that recovery occurred (for example: recovery time, previous terminal `seg_id`, new starting `seg_id`).

Such telemetry:

* is informational only
* must not influence hash chaining
* must not alter verification outcomes

This allows operational context to be preserved without weakening cryptographic guarantees.
