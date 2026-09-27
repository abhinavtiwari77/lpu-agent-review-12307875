# B1 — Consent withdrawal is promised by the notice but no shipped path can exercise it

Status: SOURCE-VERIFIED (complete static trace). All line numbers refer to the snapshot at baseline commit `114cc2fa65016651f520c7ed48ceeb55be986784`.

## Part A — The promise

`submission/src/beep_agent/consent.py`, `consent_policy()`, the `notice` dict (lines 32–50):

```python
withdrawal=("You may withdraw AI or recording permission separately. Withdrawal pauses the review. "
            "Pause or finish stops new local capture immediately. It cannot retract material already received by a provider."),
```

This text is served via `GET /api/consent-policy` (api.py `participant_consent_policy`) and rendered to every participant by `ConsentForm.tsx` line 153: `<p className="fine-print">{notice?.withdrawal}</p>`.

The notice distinguishes withdrawal from pause/finish: "Pause or finish stops new local capture immediately" — i.e., pause/finish is NOT withdrawal.

## Part B — No shipped UI path can withdraw

1. **The only consent writer in the frontend is ConsentForm.** Exhaustive search of `web/src` for `mutate("consent"` and `onConsent`:
   - `ConsentForm.tsx:96–98` — `onConsent({ ai: true, recording: true, ... })` — both flags hard-coded `true`.
   - `Review.tsx:1095–1097` — `<ConsentForm onConsent={(value) => mutate("consent", value)...}` — the only caller.
   No other component posts to `/consent`.

2. **The form's checkboxes cannot produce a reduced decision.** `ConsentForm.tsx`:
   - lines 33–34: `const [ai, setAi] = useState(false); const [recording, setRecording] = useState(false);`
   - line 167: submit button `disabled={!ai || !recording || disabled}` — the form only ever submits when BOTH are checked.
   - line 94: `onSubmit` requires `ai && recording` before calling `onConsent`.

3. **The type forbids false.** `types.ts` lines 27–30:
   ```ts
   export interface ConsentDecision {
     ai: true;
     recording: true;
   ```
   A reduced decision is unrepresentable in the client's type system.

4. **The form unmounts once consented.** `Review.tsx` line 1092: the consent layout renders only when
   `isParticipant && !hasConsent(ownConsent) && !terminal`. After the participant consents, `hasConsent(ownConsent)` is true, so the form is gone. The remaining controls (dock footer) offer only Pause / Resume / Finish / Handover / Takeover — none of which post to `/consent`.

5. **Closing the tab does not withdraw.** Server side, nothing stops recording on participant disconnect:
   - `realtime.py` `assemble_native`: `close_on_disconnect=False` in `RoomOptions`.
   - `recording.py`: `RecordingService.stop` is invoked only from worker `recording_stop` jobs; those jobs are enqueued only by `store._fence` (consent withdrawal/reduction, finish, expiry, budget) and resume-handover paths — none of which fire on a participant leaving the room.
   - LiveKit Egress records the ROOM (`RoomCompositeEgressRequest` in `recording.py start()`), not a participant's tracks, so an individual cannot "stop sharing their way out" of the recording either.

Conclusion for Part B: the promised action ("You may withdraw AI or recording permission separately") has no reachable client affordance in any state.

## Part C — The backend withdrawal machinery is complete but unreachable

1. `store.py` `set_consent` → `_consent_receipt_context` (function spanning lines 377–416; the withdrawal branch sits at lines 396–412). When the new decision is a reduction of the previous one (`reducing_only`), it writes a receipt with:
   - `notice_binding="withdrawal_prior_notice"` and `policy=prior["policy"]` (the previous receipt's stored policy — which DID include the withdrawal sentence the participant saw), or
   - `notice_binding="withdrawal_notice_unavailable"` with `policy=None` when no prior policy exists.

2. This branch is fully implemented and correct as far as it goes — but it is **unreachable from any shipped client**: Part B proved no UI path can send a reduced decision. So the receipt bindings, the DB CHECK constraint in `initialize()` (`policy IS NOT NULL OR notice_binding='withdrawal_notice_unavailable'`), and the code comment ("No backfill of pre-receipt grants… Withdrawal remains possible, with the historical gap explicit") are all dormant machinery for a flow the product never wired up. That dormancy is itself the corroboration: the flow was designed and partially built, then the client affordance was never added.

3. The backend is otherwise READY for withdrawal: `POST /api/sessions/{id}/consent` (`api.py` line 229) accepts `ai: false` / `recording: false` (`ConsentBody` uses `StrictBool`, both optional directions), and `store.set_consent` handles reduction correctly — it fences the epoch (`_fence`), pauses the session, and writes the receipt. The missing piece is precisely and only the client affordance.

## Part D — Why this is one finding, not two

Parts A+B (promise without affordance) and Part C (dormant receipt machinery) share one root cause: the withdrawal flow was designed and partially built (notice text, types, API, store, receipts, DB constraints) but never completed end-to-end (no UI control). Per the scoring guide's "one root cause earns one award", this is filed once.

## Search commands used (reproducible)

```
grep -rn "mutate(\"consent\"" submission/web/src
grep -rn "onConsent" submission/web/src --include=*.tsx | grep -v test
grep -rn "ai: true" submission/web/src/types.ts
grep -rn "consent-receipts" submission/web/src --include=*.tsx --include=*.ts | grep -v test   # (used for I2)
sed -n '30,50p' submission/src/beep_agent/consent.py
sed -n '380,415p' submission/src/beep_agent/store.py
```
