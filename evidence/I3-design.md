# I3 — Session preflight for room-token admission

Contribution group: I3. Evidence status: need SOURCE-VERIFIED; design only (not implemented).

## Problem (evidence in this snapshot)

`Review.tsx join()` requests `POST /api/sessions/{id}/room-token` and validates only the
returned identity/room/URL. Admission prerequisites are computed by
`api._require_room_admission` (role allowed, facilitator-only-in-introduction, session in
{active, introduction}, both consents) plus the runtime/recording freshness gates — but when
any check fails, the participant receives a bare 409/503, which the UI renders as a generic
"The media room could not connect. Check the connection and runtime configuration, then try
again." (`onRoomError`). The participant cannot tell whether the fix is "wait for the
recorder", "ask the facilitator to consent", or "the session was paused".

The information already exists at the exact moment of failure — the API just discards it.

## Proposed solution

Add a read-only preflight that returns the same checks as structured data:

```
GET /api/sessions/{id}/admission
→ { "eligible": false, "reasons": ["recording_not_fresh"] }
→ { "eligible": true, "reasons": [] }
```

- Fixed label set (no free text): `not_consented_client`, `not_consented_facilitator`,
  `not_in_joinable_state`, `recording_not_fresh`, `recorder_not_configured`,
  `session_time_exhausted`, `agent_dispatch_pending`.
- `Review.tsx`: call it when `connected === false` and gate the join button on `eligible`,
  rendering `reasons` (mapped to the existing human-copy style of the `nextStep` panel)
  instead of enabling a join that will fail.
- Keep `POST /room-token` as the authoritative gate; the preflight is advisory
  (poll every ~2 s alongside the existing session poll, or fold into the existing
  `useResource(sessionPath(sessionId), 2000)` cycle to avoid an extra request).

## Implementation approach

1. `api.py`: new endpoint reusing `_require_room_admission` logic — refactor its checks into
   a pure function returning `list[str]` (empty = pass) so the token endpoint and the
   preflight share one implementation (no drift).
2. Recording freshness: same check the token path performs indirectly
   (`recording_status == "recording"`, `egress_id` present, `recording_verified_at` fresh) —
   expose as `recording_not_fresh` when it would fail.
3. Frontend: extend the `nextStep` computation in `Review.tsx` — when reasons are
   non-empty, show them as the blocked join explanation.

## Acceptance criteria (measurable)

1. With a recorder not yet fresh, the endpoint returns `eligible:false` with
   `reasons` containing `recording_not_fresh`; the join button is disabled and the reason is
   displayed.
2. With consent missing from one party, `not_consented_*` is returned (and the existing
   consent panel already covers it — the endpoint must agree with the UI's own state).
3. When all prerequisites pass, `eligible:true`, and the existing join flow behaves exactly
   as before (no behavioural change on the happy path).
4. The endpoint performs no mutation (GET; no epoch fencing side effects) and leaks no
   internal identifiers beyond what the session view already returns.
5. `tests/test_api.py` covers cases 1–3.

## Testing strategy

API tests driving the shared check function directly plus through the endpoint; a
`Review.test.tsx` case asserting the disabled-join-with-reason rendering.

## Effort / risks / trade-offs

~2–3 h. Risks: label/UI drift from the token endpoint (mitigated: one shared function);
extra polling (mitigated: fold into the existing 2 s session poll). Trade-offs: none
material — the checks run on every token request already; this surfaces them earlier and
readably.

## Why this over alternatives

Mapping HTTP codes to guesses client-side (the alternative) duplicates the backend's logic
and drifts. The shared-function preflight keeps one source of truth and converts the most
common support question during a live review ("why can't I join?") into a self-serve answer.
