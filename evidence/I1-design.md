# I1 — Expose transient recording/job failures in session state

Contribution group: I1. Evidence status: need SOURCE-VERIFIED from the snapshot; design only (not implemented).

## Problem (evidence in this snapshot)

- `worker.py` `_recording`: both failure paths wrap the store update in
  `with suppress(StoreError): await store_call(self.store.set_recording, tenant, sid, "failed", eid)`
  — the `RecordingError` reason (a fixed label such as "Egress failed; recording is not confirmed")
  is discarded at this boundary.
- `store.py` `observe_recording`: on a failed health readback it flips
  `recording_status="failed"` and pauses the session. No reason is recorded anywhere in the
  session document.
- `Operator.tsx` renders `statusLabel(session.status)` ("Review interrupted") and
  `session.recording_status` ("failed") — with no cause. The only ways to learn why are direct
  database queries (`beep_jobs.error`, `beep_jobs.result`) or reading worker logs.

## Current limitation

An operator seeing a paused/failed session cannot distinguish "transient Egress hiccup, retry
will heal it" from "recording never worked" without host access. During a live client session
this is the difference between "wait" and "abort and reschedule".

## Proposed solution

When the session transitions to a recording-failed state, persist a bounded failure record on
the session document and surface it to operators:

```python
s["last_recording_failure"] = {
    "reason": "<fixed RecordingError label>",   # from the existing fixed-label set
    "egress_id": eid,                            # may be None
    "reservation_id": reservation_id,            # may be None
    "at": utcnow(),
}
```

Render in `Operator.tsx` (session ledger row + review header) as a small warning with the
label; never render exception text or provider payloads (consistent with the pack's
privacy discipline in `telemetry.SDKPrivacyFilter`).

## Implementation approach

1. `store.set_recording(..., "failed", ...)`: accept an optional `reason: str | None`;
   validate it against a fixed label set (reuse `RecordingError` message constants or a
   `Literal`); write `last_recording_failure`.
2. `worker._recording` failure paths: pass the caught `RecordingError`'s fixed label through
   (it already carries `.args`/message labels — no new failure vocabulary needed).
3. `store.observe_recording` failure branch: write reason `"health_check_failed"`.
4. `api.py`: the field is already part of the session document returned by
   `GET /api/sessions/{id}` — no new endpoint needed. Strip it for non-operators
   (the same way `internal_opportunity` is popped) to keep participant responses minimal.
5. `Operator.tsx`: render when present.

## Acceptance criteria (measurable)

1. After any `set_recording(status="failed")` with a reason, `GET /api/sessions/{id}` (as
   operator) contains `last_recording_failure.reason` from the fixed label set and an ISO
   timestamp.
2. The operator session list shows a failure indicator for that session within one poll
   interval (10 s, per `useResource("/sessions", 10_000)`).
3. Participant (client/facilitator) responses never contain `last_recording_failure`.
4. The field holds exactly one entry (overwritten), and its total size is bounded
   (< 300 bytes).
5. `tests/test_api.py` gains a case asserting 1–4.

## Testing strategy

Unit: drive `set_recording`/`observe_recording` with a forced failure; assert the field and
its stripping for participants (store-level tests in `tests/test_store.py`, API-level in
`tests/test_api.py`). No new fixtures needed beyond an existing store session.

## Effort / risks / trade-offs

~1–2 h. Risks: leaking detail if free text slips in (mitigated: fixed labels only); session
document growth (bounded to one slot). Trade-off: a small amount of extra state versus
operational blindness — the pack's own `report_failure` precedent
(`_settle_report_failure` already does exactly this shape for report jobs) shows the pattern
is established; this extends it to recording.

## Why this over alternatives

Log-only requires host access; DB-only requires DB access; doing nothing leaves every
transient failure invisible. The `report_failure` field proves the codebase already trusts
this pattern for exactly this purpose.
