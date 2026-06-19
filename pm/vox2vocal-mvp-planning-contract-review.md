# Vox2Vocal MVP 기획 / API 계약 재리뷰

리뷰일: 2026-06-15
적용 skill: `trd-reviewer`
리뷰 대상:

- `pm/vox2vocal-mvp-prd.md` v0.14
- `pm/vox2vocal-mvp-feature-definition.md` v0.6
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.4
- `pm/vox2vocal-mvp-trd.md` v0.4
- `pm/vox2vocal-mvp-api-data-contract-plan.md` v0.3

## Verdict

Status: **Go for contract-first spec-to-tickets, Revise before feature implementation, No-go for learner exposure until DB evidence exists**

이전 리뷰의 핵심 blocker였던 stage naming 불일치, Safety Rights 역할 혼재, rights/risk evidence ref 모호성, playback session coverage 정책 부족, `PreviewArtifact` index/constraint 부족, stale feature/page-flow 기준 문서는 문서상 보완되었다.

이제 `spec-to-tickets`는 진행 가능하다. 단, 곧바로 learner feature 티켓부터 만들면 안 된다. 첫 epic은 반드시 contract foundation이어야 하며, proto/schema/test/seed/admin-write path를 먼저 쪼개야 한다.

## Requirement Coverage

- PRD의 learner flow는 account, song selection, section selection, recorder/take review, upload, processing, preview playback, rating/failure tag, deletion request까지 추적된다.
- Feature definition은 PRD v0.14, TRD v0.4, API/data contract v0.3 기준으로 동기화되었다.
- Page flow는 feature definition v0.6과 API/data contract v0.3 기준으로 업데이트되었다.
- API/data contract는 proto service, shared types, endpoint contracts, engine event contract, rights/risk evidence contract, migration/indexing, events/analytics를 포함한다.
- TRD는 architecture, frontend/backend changes, API/data boundaries, auth/security, observability, rollout/rollback, tests, risks/tradeoffs를 유지한다.

## Blocking Issues

### 1. Learner exposure remains blocked until DB evidence seed exists

- 문서상 `evidence_ref`는 `governance_evidence_records.evidence_ref`로 닫혔다.
- 그러나 실제 DB source of truth에 `Mist` rights/risk record와 governance evidence row가 없으면 learner selection, job creation, playback URL issuance, rating unlock은 No-go다.
- Required before exposure: seed/admin-write path, audit record link, allowed users/groups, exact section, re-review deadline, kill-switch owner.

### 2. Contract foundation must precede feature tickets

- 문서 계약은 충분하지만 실제 proto files, generated clients, schema validators, event adapter tests는 아직 없다.
- If skipped, BFF, API Gateway, Worker, engines can still diverge even though the docs are clearer.
- Required before feature implementation tickets: proto/service files, GraphQL-proto mapping tests, EngineStageEvent validation, StageSummary status mapping tests.

### 3. Internal surface scope still needs product/design decision

- Page flow leaves mobile/web split for admin/review/governance open.
- This does not block learner flow ticketing, but it can affect route structure, permissions, and UI design prompts.
- Recommended default: learner app mobile-first plus web-compatible; internal admin/review/governance web-first inside the same route/auth structure for P0.

## Important Risks

- `Mist` still uses real copyrighted reference material. `unlicensed_internal_risk_accepted` reduces exposure but does not clear rights.
- `preflight_safety_rights` and `post_render_safety_rights` are now separated, but implementation must enforce the preflight gate before publishing outbox events.
- Playback telemetry is now single-session scoped. Actual mobile foreground/background behavior can still lose events and should be tested on real devices.
- `PreviewArtifact` physical table and indexes are defined, but migration order and backfill behavior must be additive and rollback-safe.
- Feature and page docs are now synchronized, but future PRD/TRD changes must bump versions together to avoid stale ticket generation.

## Recommended Fixes

- Create a **Contract Foundation** epic before feature tickets:
  - proto package and message files
  - shared enums and error codes
  - GraphQL-proto mapping tests
  - generated client publishing
  - unknown enum and unknown major schema tests
- Create an **Engine Event Contract** epic:
  - `voice_pitch`, `target_pitch_mapping`, `preview_synthesis`, `render`, `preview_evaluation`, `post_render_safety_rights` validators
  - EngineStageEvent to StageResult mapping
  - duplicate/replay event handling
- Create a **Governance Evidence Seed/Admin Path** epic:
  - `governance_evidence_records`
  - `rights_records`
  - `risk_acceptance_records`
  - audit link and kill-switch owner
- Create a **Playback Telemetry Gate** epic:
  - single-session coverage calculation
  - 15-second late flush grace
  - 2-second wall-clock tolerance
  - severe error taxonomy
  - rating unlock expiry

## Missing Tests / Observability

- GraphQL schema to proto request/response mapping tests.
- Proto `JobProjection`, `PreviewArtifact`, `PlaybackSession`, and `PlaybackEvent` contract tests.
- EngineStageEvent payload validation tests per stage.
- Stage status mapping tests: `started -> running`, `needs_review -> needs_review`, `blocked -> blocked`.
- Preflight safety tests proving `completeAudioUpload` cannot publish outbox events when rights/consent/audit fails.
- Playback tests for duplicate event id, changed payload replay, stale event after grace window, severe error, muted/background exclusion, single-session coverage, and rating unlock expiry.
- Governance observability for exposure decision, evidence ref, risk acceptance id, kill-switch owner, and latest rights state at playback URL issuance.

## Rollout / Rollback Concerns

- Rollout is now safer because contract freeze, evidence seed, and playback validation are explicit first steps.
- Rollback remains acceptable for P0 if feature flags can disable package exposure, upload completion, outbox publishing, reference pre-listen, and playback URL issuance.
- Already issued signed URLs remain a known limitation; short TTL plus playback-blocked state is acceptable for P0.
- Rollback must not drop governance evidence, audit records, deletion evidence, or artifact metadata.

## Questions Before Build

- Which exact repo owns each engine stage payload version after implementation starts?
- Will internal admin/review/governance be web-first only for P0, or also fully supported in mobile UI?
- If break-glass raw/canonical audio access or contact plaintext reveal is enabled after P0, who becomes the required second reviewer/security owner?
- Should P0 keep single-event playback telemetry only, and defer batch endpoint decisions until beta?

## Go / Revise / No-go

- **Go**: contract-first `spec-to-tickets`.
- **Go**: UI design wireframe prompt using page-flow v0.4.
- **Revise**: learner feature implementation tickets must wait until contract foundation tickets are defined first.
- **No-go**: `Mist` learner exposure until `governance_evidence_records`, `rights_records`, and `risk_acceptance_records` are present in DB source of truth with audit links.

Recommended next step:

1. Run `spec-to-tickets` on the updated PRD/TRD/API/page-flow documents.
2. Make the first epic `Contract Foundation`.
3. Only then split learner app, worker pipeline, playback/rating, review/admin/governance tickets.
