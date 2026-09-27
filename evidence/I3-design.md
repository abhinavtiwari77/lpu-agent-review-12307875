# I3 — Session preflight for room-token admission

Contribution group: I3. Evidence status: SOURCE-VERIFIED; design only (not implemented).

## Problem (evidence in this snapshot)

`Review.tsx join()` (lines 582–613) fires `POST /api/sessions/{id}/room-token` only after the
client's own pre-gate (`eligible`/`admissionReady`, lines 384–396: both consents, runtime
readiness, `recording_status === "recording"` with `egress_id`). That pre-gate is necessarily
partial: the server-only prerequisites — agent-dispatch presence, runtime configuration, and
the session-unchanged epoch re-check — are discovered only at press time, as one-off HTTP
errors surfaced verbatim by `errorMessage` (`api.ts` line 100 passes `ApiError.message`
through): 503 "Agent dispatch is pending or requires reconciliation" (`api.py`
`_room_admission`), 503 "Room providers are not configured" (`api.py` line 273), 409
"Session changed during room admission" (`_require_room_admission`, line 349). RTC-level
failures collapse further into the generic "The media room could not connect. Check the
connection and runtime configuration, then try again." (`onRoomError`, `Review.tsx` line
419). The "Your next step" panel (lines 741–930) already explains every client-known blocking
condition; the server-known ones arrive as a single transient error line — if the participant
misses it, they are back to a generic "Join the call" that will fail again.

The information already exists at the exact moment of failure — the API just discards it
instead of exposing it as structured, pollable state.

## Proposed solution

Add a read-only preflight that returns the same checks as structured data:

```
GET /api/sessions/{id}/admission
→ { "eligible": false, "reasons": ["recording_not_fresh"] }
→ { "eligible": true, "reasons": [] }
```

- Fixed label set (no free text): `not_consented_client`, `not_consented_facilitator`,
  `not_in_joinable_state`, `facilitator_outside_introduction`, `session_changed`,
  `room_providers_not_configured`, `agent_dispatch_pending`, and `recording_not_fresh`
  (already client-visible; included so the endpoint is one source of truth).
- `Review.tsx`: call it when `connected === false` and gate the join button on `eligible`,
  rendering `reasons` (mapped to the existing human-copy style of the `nextStep` panel)
  instead of enabling a join that will fail.
- Keep `POST /room-token` as the authoritative gate; the preflight is advisory
  (poll every ~2 s alongside the existing session poll, or fold into the existing
  `useResource(sessionPath(sessionId), 2000)` cycle to avoid an extra request).

## Implementation approach

1. `api.py`: new endpoint reusing the token endpoint's checks — refactor
   `_require_room_admission` plus `_room_admission`'s runtime/dispatch steps into a pure
   function returning `list[str]` (empty = pass) so the token endpoint and the preflight
   share one implementation (no drift).
2. Recording freshness: mirror the client's existing check (`recording_status ==
   "recording"` with `egress_id` present — `Review.tsx` lines 390–391) so the endpoint agrees
   with the panel the participant already sees; expose as `recording_not_fresh` when it fails.
3. Frontend: extend the `nextStep` computation in `Review.tsx` — when reasons are
   non-empty, show them as the blocked join explanation.

## Acceptance criteria (measurable)

1. With agent dispatch pending (the token endpoint's 503 case), the endpoint returns
   `eligible:false` with `agent_dispatch_pending`; the join affordance is suppressed and the
   reason is displayed.
2. With consent missing from one party, `not_consented_*` is returned and the endpoint
   agrees with the client's own pre-gate (it must never contradict what the panel already
   shows for client-known conditions).
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
