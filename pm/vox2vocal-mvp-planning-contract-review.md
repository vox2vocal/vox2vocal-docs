# Vox2Vocal MVP Planning / API Contract Review

리뷰일: 2026-06-15
적용 skill: `trd-reviewer`
리뷰 대상:

- `pm/vox2vocal-mvp-prd.md` v0.12
- `pm/vox2vocal-mvp-feature-definition.md` v0.5
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.3
- `pm/vox2vocal-mvp-trd.md` v0.2
- `pm/vox2vocal-mvp-api-data-contract-plan.md` v0.1

## Verdict

Status: **Revise before full implementation ticketing**

UI/page planning is strong enough to hand off to a UI design agent for wireframes. API/data contracts are much more concrete than the previous TRD, but implementation ticketing should still wait for a small number of contract closures: proto service names, downstream engine event schemas, playback telemetry tolerance, and rights/risk approval owner.

## Requirement Coverage

- PRD learner flow is covered: auth, song selection, section selection, recorder/take review, upload, processing, preview, playback telemetry, rating, failure tags, deletion.
- Internal surfaces are covered: limited admin, review queue/detail, governance evidence.
- TRD now covers shared contracts for `JobProjection`, `PreviewArtifact`, `PlaybackSession`, and `PlaybackEvent`.
- API/data contract plan covers endpoint behavior, request/response shape, validation, errors, compatibility, data model, migration, indexing, and events/analytics.
- Contact follow-up scope is now consistent: P0 stores disabled capability status only and does not collect contact values.

## Blocking Issues

### 1. Proto/service ownership is still not concrete enough for backend tickets

- Issue: The API contract defines GraphQL operations and internal boundary intent, but not exact proto package/service/message names.
- Why it matters: BFF, API Gateway, User Service, and Worker can still diverge on service naming and generated client ownership.
- Severity: High.
- Required fix: Add a small proto contract appendix or follow-up doc naming P0 services, messages, and ownership.

### 2. Downstream engine event schemas remain open

- Issue: Audio ingest envelope is concrete, but `voice-pitch`, `target_pitch_mapping`, `preview_synthesis`, `render`, `preview_evaluation`, and `safety_rights` event schemas are still open.
- Why it matters: StageResult normalization depends on source event fields. Without event schemas, worker adapter tickets will contain too much guesswork.
- Severity: High for engine/worker tickets.
- Required fix: Define minimum event payload for each downstream stage before `spec-to-tickets`.

### 3. Rights/risk approval owner remains a launch blocker

- Issue: The docs require rights/risk evidence and risk acceptance, but the temporary P0 approver and operational evidence location are still open.
- Why it matters: Real song usage is the highest policy risk. Internal operation should not expose `Mist` to learners until the DB source of truth has an approver and evidence ref.
- Severity: Blocker for learner exposure, not for UI wireframes.
- Required fix: Pick temporary P0 approver, evidence location, allowed group, re-review deadline, and kill-switch owner.

## Important Risks

- Playback tolerance is underspecified. The contract says impossible progress should be rejected, but the tolerance window is still open. This can affect `preview_played=true`.
- `needs_review` has no SLA or owner. Low-confidence/disputed jobs may pile up and make internal review unreliable.
- The API contract is large enough that implementation tickets may need contract tests before feature tickets.
- Page plan is implementation-ready, but visual design still needs component-level decisions for recorder, player, rating, failure tags, and evidence tables.
- Contact follow-up is now scoped correctly, but future references must keep "disabled capability only" to avoid accidental P0 scope creep.

## Recommended Fixes

- Add `proto-service-contract` section to the API/data contract plan:
  - `VoxAuthService`, `VoxSongPackageService`, `VoxUploadService`, `VoxJobService`, `VoxPlaybackService`, `VoxReviewService`, `VoxGovernanceService`, or final equivalents.
  - message names and owning repo.
- Add downstream engine event contract appendix:
  - required fields per stage
  - success/failure payload
  - retryable flag
  - artifact refs
  - confidence summary
  - user-safe reason mapping
- Add rights/risk launch checklist:
  - approver
  - evidence ref
  - allowed users/groups
  - allowed sections
  - prohibited uses
  - re-review deadline
  - kill-switch owner
- Add playback telemetry constants:
  - late flush grace window
  - wall-clock tolerance
  - max accepted event batch size
  - stale session rejection rule
- Add review SLA:
  - `needs_review` owner
  - response target
  - fallback terminal state if unresolved

## Missing Tests / Observability

- Contract tests should be required before feature tickets:
  - GraphQL schema to proto mapping
  - proto to worker JobProjection mapping
  - enum fallback and unknown major schema version rejection
- Engine event adapter tests are still missing until downstream event schemas exist.
- Playback tests should include:
  - stale event after session expiry
  - event id replay with changed payload
  - impossible timeline range
  - background/muted events not counting
  - rights/consent/deletion changes between playback and rating
- Rights/risk observability should include:
  - package exposure decision
  - risk acceptance id
  - kill-switch owner
  - latest rights state at playback URL issuance

## Rollout / Rollback Concerns

- Rollout sequence is improved because contract freeze is now step 1.
- Rollback is mostly safe, but signed URL revocation remains a known limitation. Short TTL and playback-blocked state are acceptable for P0.
- Migration plan is additive and safe. However, physical choice for `PreviewArtifact` table versus typed `artifact_refs` row is still open and should be decided before schema tickets.
- Contact follow-up rollback is safe because no contact values should exist in P0.

## Questions Before Build

- What exact proto service and message names will be used for P0 contracts?
- Will `PreviewArtifact` be a separate table or a typed `artifact_refs` row with JSON metadata?
- What is the flush grace window for late playback progress events?
- What wall-clock tolerance should reject impossible playback progress?
- Who is the temporary P0 rights/risk approver for `Mist`?
- Where is the operational evidence ref stored before a full admin project exists?
- What SLA applies to `needs_review` jobs?

## Go / Revise / No-go

Recommendation:

- **Go** for UI design wireframes using `pm/vox2vocal-mvp-page-flow-plan.md`.
- **Revise** before backend/API/worker/engine implementation ticketing.
- **No-go** for learner exposure until rights/risk approval evidence exists in the DB source of truth.

Conditions to proceed to `spec-to-tickets`:

- Close proto service/message naming.
- Close downstream engine event schemas or explicitly split them into separate engine-contract tickets first.
- Close rights/risk launch checklist.
- Close playback telemetry tolerance constants.
- Decide `PreviewArtifact` physical schema.
