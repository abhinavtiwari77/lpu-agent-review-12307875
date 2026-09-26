# I2 — Consent-notice versioning and participant-visible receipts

Contribution group: I2 (distinct from B1: B1 is the missing withdrawal flow; I2 is provenance
visibility for the consent that DID happen). Evidence status: SOURCE-VERIFIED; design
only (not implemented).

## Problem (evidence in this snapshot)

- The backend already builds a rigorous consent provenance chain:
  `consent.py` stamps `notice_version`, `policy_id = content_hash(policy)`,
  `provider_configuration_hash`; `store._append_consent_receipt` writes immutable receipts
  (DB trigger `beep_consent_receipts_immutable`, `store.py` lines 93–105) with `epoch_before`/`epoch_after`,
  `principal_kind`, `token_digest`, `notice_binding`, and the accepted `policy` JSON.
- None of this reaches the participant:
  - `ConsentForm.tsx` shows the notice text but never which version/policy_id was accepted;
  - `GET /api/sessions/{id}/consent-receipts` exists (`api.py`, role-scoped in
    `list_consent_receipts`) but has **no frontend consumer** (verified: no non-test
    reference in `web/src`).
- A participant asked "what exactly did I agree to?" has no in-product answer; only the
  facilitator's word or a database query.

## Proposed solution

1. **Post-consent acknowledgement**: after a successful `POST /consent`, render the accepted
   `notice_version` and `policy_id` (both present in the session document returned by the
   endpoint) in the consent panel: "You accepted notice <version> (<policy_id prefix>…) at
   <time>."
2. **"My consent receipts" section**: in the participant's `ReportView.tsx` (or a small new
   tab), call the existing receipts endpoint and list the participant's own receipts:
   `seq`, `role`, `ai`, `recording`, `occurred_at`, `notice_binding`, `policy_id`.
   Do NOT render the full `policy` blob (noise); render the binding label
   (`accepted_current_notice` / `withdrawal_prior_notice` / …) as plain text.
3. Backend change: none strictly required — the endpoint exists and is already session- and
   role-scoped. Optional nicety: include `policy_id`/`notice_version` directly on the receipt
   row (currently inside the `policy` JSON) to avoid client-side extraction.

## Implementation approach

- `ConsentForm.tsx`: accept the post-consent session from `mutate("consent", ...)` (already
  returns `{session}`), display the two fields.
- `ReportView.tsx`: `useResource(`${sessionPath(sessionId)}/consent-receipts`, 0)` — fetch
  once on mount (receipts are immutable; no polling needed), paginate with `next_after_seq`
  if the list grows.
- Types: add `ConsentReceipt` to `types.ts`.

## Acceptance criteria (measurable)

1. After consenting, the participant can see the exact `notice_version` and `policy_id` they
   accepted, sourced from the server response (not local state).
2. The receipts list shows at least the participant's own receipt(s) with role, both flags,
   timestamp and notice binding.
3. A facilitator's receipt list contains no client rows and vice versa (server already
   scopes via `role=None if who["role"] == "operator" else who["role"]`; assert in
   `tests/test_api.py`).
4. `ConsentForm.test.tsx` gains a case asserting the version/policy_id render after consent.
5. No receipt content is rendered to the other participant role or to operators through this
   UI path (operators already have their own view).

## Testing strategy

Component tests with the existing `consent-test-fixture.ts` policy fixture; API test for
role scoping (extend `tests/test_consent_receipts.py`, which already covers the store and
endpoint thoroughly).

## Effort / risks / trade-offs

~2–3 h, frontend-mostly. Risks: displaying too much (mitigate: fixed fields, no policy blob);
no new auth surface (endpoint already exists and is scoped). Trade-offs: one more panel in
the report view — justified because the pack's own product philosophy ("A review you
control", receipts bound to epochs) is otherwise invisible to the person it protects.

## Why this over alternatives

Export-only (download receipts as JSON) is weaker: participants rarely export. Rendering the
provenance where the participant already looks (the report) makes the consent artifact
checkable in place — which is the entire point of the receipt machinery the backend already
maintains.
