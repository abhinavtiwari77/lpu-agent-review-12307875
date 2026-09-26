# Hypothesis investigated and NOT confirmed — recording-stop CAS "drop"

This file documents a candidate bug that was investigated in depth and then **rejected** during final verification. It is NOT in the bug register. It is kept here because the investigation was substantial and the reasoning is part of an honest audit trail.

## The hypothesis

`store.set_recording`'s recording_stop CAS (`store.py` ~line 938):

```python
if job["kind"] == "recording_stop" and (
    job["payload"].get("egress_id") != s.get("egress_id")
    or egress_id not in {None, s.get("egress_id")}
    or status not in {"stopping", "stopped", "failed"}
):
    raise StoreError("Recording cleanup does not own the current egress")
```

hypothesis: during a pause → (withdraw) → re-consent race, the stale stop job's final
`set_recording(status="stopped", egress_id=old)` raises this StoreError *after* the external
stop succeeded but *before* `complete_recording_cleanup` settles the old reservation, leaving
`recording_cleanup_pending` semantics wedged for the session (every later report forced to
recording-incomplete partial).

## Why it was rejected

Walking every interleaving with the actual guards, in order:

1. **Serial path (no interleaving).** `set_consent` (withdrawal or any reduction) and
   `control(pause/finish)` call `_fence()`, which increments `consent_epoch` and enqueues the
   `recording_stop` job *with the reservation + cleanup signature*. The stop worker then runs
   while no other mutation is in flight: `set_recording("stopping")` persists (guard passes:
   `payload["egress_id"]` still equals `s["egress_id"]` because nothing else changed it), the
   external `reconcile_stop` runs, then `set_recording("stopped", eid)` passes the same guard
   (status "stopped" IS in the accepted set, egress still matches), and
   `complete_recording_cleanup` settles the reservation. No drop.

2. **Concurrent re-consent while the stop job holds the lease.** A second consent-reduction or
   resume during the stop job's execution can only change state via `set_consent`/`control`,
   both of which serialize on the session row lock (`FOR UPDATE`) — the same lock
   `set_recording` takes. The stop worker's `set_recording` calls and the intervening
   mutation interleave at transaction boundaries, but:
   - If the mutation lands *between* "stopping" and "stopped": at the final
     `set_recording("stopped", eid)` the CAS *does* reject (egress cleared/changed by the
     fence) — the hypothesised moment. However, that same fence (`_fence`) is exactly the path
     that **re-enqueues** cleanup: `_fence` runs `self._enqueue_recording_cleanup(...)` when
     `reservation and s.get("recording_cleanup_pending")`, or enqueues a fresh `recording_stop`
     carrying the reservation when egress/recording status is live. So the obligation is
     re-armed by the very mutation that invalidated the old job; the old job's rejection does
     not strand it.
   - If instead the re-consent *grants* (resume path): `control(resume)` enqueues a new
     `recording_start` for the current epoch; a `recording_start` cannot proceed to
     `set_recording("starting")` while `recording_cleanup_pending` is true
     (`store.py` ~line 949: "Previous recording outcome requires reconciliation"), and
     `claim_job`'s `eligible` filter blocks queued recording jobs for a session that already
     has a running job (`beep_jobs_one_running` unique partial index on
     `session_id WHERE status='running'`). The start job waits; the stop job still owns the
     lease and completes normally.

3. **Worker crash / lease expiry between "stopping" and settle.** This is a real stranded-state
   candidate, but it is handled by the explicit ambiguity machinery: `claim_job` marks expired
   jobs `lease_expired_ambiguous`; `_settle_report_failure` and the reservation's unsettled
   state keep the report partial and the session non-deletable (`delete_session` refuses while
   any reservation is unsettled). The system *fails closed, visibly* (partial report, deletion
   blocked) rather than silently. That is a designed-for outcome, not a defect; and reaching it
   requires a worker crash, not ordinary UI actions.

4. **Direct `request_recording_cleanup` / administrative reconciliation** exists
   (`store.request_recording_cleanup`) precisely to re-arm an obligation whose owning start
   expired — the "an expired owning start may retain a late ID" docstring.

Conclusion: in every interleaving, either the stop completes, or the invalidating mutation
re-arms cleanup itself, or the state fails closed with explicit reconciliation requirements.
The hypothesised "silent drop" is intercepted by `beep_jobs_one_running`, the
`recording_cleanup_pending` gate on "starting", and `_fence`'s re-enqueue. **No confirmed
defect.** Not filed.

## What remains true

- This path is only statically analysed; a live DB+Docker reproduction could still surface an
  interleaving the static walk missed (recorded as a limitation, not a claim).
- The *observability* gap around these failures is real and filed separately as I1
  (`worker.py` suppresses the StoreError around `set_recording(..., "failed")`), which is what
  makes this area hard to diagnose operationally — that improvement stands on its own need.
