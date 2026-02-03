

---

## 6. Recovery Decision Matrix

| Condition | Degraded Mode | GAP Persisted | Recovery Allowed | Resulting Chain |
|---|---|---|---|---|
| Normal operation | No | No | No-op | Continuous |
| Watchdog fired | Yes | Yes | Yes | `[Seg N] → [GAP] → [Seg N+1]` |
| Watchdog fired | Yes | No | Yes | `[Seg N] → [Seg N+1]` (timestamp jump) |
| Storage missing | Yes | Yes | No | Chain frozen |
| Storage corrupted | Yes | N/A | No | Chain terminal |

---

## 7. Non-Goals of Recovery

- Recovery does not reconstruct lost events.
- Recovery does not infer or synthesize missing segments.
- Recovery does not re-order, compact, or rewrite history.
- Recovery does not guarantee continuity of time—only continuity of proof.
- Recovery does not attempt to hide or soften operational failures.

---

## 8. Forensic Interpretation

- A recovered chain indicates controlled continuation, not uninterrupted operation.
- Absence of a GAP implies no persisted evidence of discontinuity, not proof of perfect capture.
- Timestamp jumps are expected after recovery and must not be treated as chain corruption.

---

## 9. Verifier Expectations

A verifier evaluating a recovered log should:

- Validate continuity using only chained `ch` values.
- Treat time gaps as informational.
- Fail verification only if chain order, linkage, or integrity is violated.

---

## 10. Optional Recovery Telemetry

Implementations may emit non-hashed diagnostic events indicating recovery invocation
(e.g., recovery time, previous terminal `seg_id`, new starting `seg_id`).

Such telemetry must never affect cryptographic integrity.
