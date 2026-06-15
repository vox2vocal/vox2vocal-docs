# Vox2Vocal MVP Technical Requirements Document

문서 버전: v0.2
작성일: 2026-06-15
상태: 초안
적용 skill: `trd-writer`
기준 문서:

- `pm/vox2vocal-mvp-prd.md` v0.12
- `pm/vox2vocal-mvp-feature-definition.md` v0.5
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.3
- `pm/vox2vocal-mvp-api-data-contract-plan.md` v0.1

## Technical Summary

Vox2Vocal P0는 학습자가 Ken Kamikita - `Mist`를 선택하고, `intro 0:00-0:28` section을 선택한 뒤, 앱에서 본인 목소리를 녹음하고, section-limited self-voice preview를 앱 안에서 재생해 평가하는 end-to-end 흐름을 구현한다.

P0 기술 목표는 상업 품질의 full-song generation이 아니라, 다음 조건을 안전하게 검증하는 것이다.

- 본인 voice input만 recorder 또는 fallback upload로 수집한다.
- song package, section, consent snapshot, rights flag snapshot이 없으면 job을 만들지 않는다.
- canonical job state는 하나의 owner가 관리한다.
- engine stage는 표준 StageResult로 정규화한다.
- preview는 app-only signed playback으로만 제공한다.
- rights, consent, audit, deletion 상태가 차단되면 processing 또는 playback을 fail closed한다.
- contact follow-up은 P0에서 gate만 정의하고 기본 disabled로 둔다. core preview/rating flow의 blocking dependency가 아니다.

현행 엔진 구성은 최종 목표를 위해 유지한다. 다만 P0 metric-eligible preview의 첫 경로는 `partial_real` lineaged preview로 둔다. 즉 `audio-ingest`, section validation, `voice-pitch`, target pitch mapping, constrained preview generation/render, lightweight `evaluation`, `safety-rights`를 우선하고, full `singing-synthesis`, full `vocoder-render`, full `mix-master`, `voice-analysis`, `phoneme-alignment`, `rhythm-timing`, `expression`, `voice-conversion`은 필요 시 제한적으로 사용하거나 Later/shadow path로 둔다.

## Product Requirement Traceability

| Product requirement | Technical implementation area | Primary systems | Validation |
| --- | --- | --- | --- |
| Account creation and login | Auth GraphQL/gRPC, token handling, role read model | App, BFF, API Gateway, User Service | Signup/login success, `me` query, role redirect |
| Role-based access | Role/status check and surface routing | App, BFF, API Gateway, User Service | learner/admin/reviewer/governance access matrix |
| First-login required consent and job consent snapshot | Versioned consent records and immutable job snapshot | App, BFF, API Gateway, DB, Safety Rights | missing/outdated/withdrawn consent blocks recorder submit and `completeAudioUpload` |
| Admin song package for `Mist` | Song package, section map, reference asset, rights/risk record | Internal web, BFF/API Gateway, DB, object storage | package cannot publish without required metadata or risk acceptance |
| Song selection then section selection | Separate song and section read models | App, BFF/API Gateway, DB | selected song id and section id are required for job |
| In-app recorder primary path | Recorder, mic permission, take review, local take state | App | record, replay, retake, submit available |
| Fallback upload | Presigned upload session and object storage path | App, BFF/API Gateway, object storage | `.wav`/`.mp3`, TTL, size guard, idempotency |
| Audio ingest and section validation | Canonical audio conversion and validation StageResult | engine-audio-ingest, worker, storage | canonical artifact, duration, loudness, silence, section mismatch |
| Target pitch comparison | F0 extraction and target note mapping | engine-voice-pitch, melody mapping, evaluation | confidence thresholds, disputed ranges, low-confidence display |
| Self-voice section preview | Minimal partial-real preview generation and render path | engine-audio-ingest, engine-voice-pitch, melody mapping, constrained preview render, storage | `preview_available=true`, `section_limited=true`, lineage to user voice input |
| App-only playback | Short-lived signed GET and playback audit | App, BFF/API Gateway, storage, audit | no download/share/export, URL denied when blocked |
| Job status and partial artifact handling | Canonical job state owner and StageResult ledger | worker, DB, BFF, App | no BFF-invented final state, timeout terminal state |
| Rating and failure tags | Playback telemetry, rating, failure tag taxonomy | App, BFF/API Gateway, DB/analytics | rating only after meaningful preview playback |
| Educator/expert review | Separate web review queue and detail surface | Internal web, BFF/API Gateway, DB, storage | only consented jobs, no raw audio by default |
| Retention and deletion | Artifact retention deadline, deletion jobs, evidence ledger | worker, storage, DB, audit | raw audio 30-day default, deletion evidence, playback block on failure |
| Contact follow-up | Disabled capability status only | App/internal web, BFF/API Gateway, User Service | no contact UI, value storage, send, decrypt, or export in P0 |

## Architecture / Approach

### System Boundary

P0 uses the existing workspace shape and does not introduce a standalone Conversion Job Orchestrator service.

- `vox2vocal-app`: learner mobile/web surface for auth, song selection, section selection, recording, result, rating, deletion settings, and hidden/disabled contact follow-up capability status.
- Internal web surface: P0 defines role-gated web routes in the same app/web structure, such as `/internal`, for admin song package, educator/expert review, governance evidence, deletion/audit review. A separate admin project is Later; the security and API boundary must still be separate from learner routes.
- `vox2vocal-bff-server`: GraphQL BFF for app and internal web surfaces. It shapes app-facing schemas and forwards authenticated requests to API Gateway.
- `vox2vocal-api-gateway`: internal orchestration boundary. It forwards auth/user context and exposes internal service calls through gRPC or equivalent internal contracts.
- `vox2vocal-user-service`: account, auth, user identity, role/status.
- `vox2vocal-worker`: BullMQ-based worker today; P0 adds `conversion-job-state` bounded module for canonical job state, timeout handling, stage ledger adaptation, deletion jobs, and outbox publishing.
- `engine-*`: engine pipeline services. Engine composition remains as documented for the final product direction.
- Object storage: raw voice input, canonical audio, reference audio, generated preview, reports, deletion evidence references.
- PostgreSQL: source-of-truth records for users, consent, song package, upload sessions, jobs, stage results, artifacts, ratings, audits, deletion evidence, and disabled contact capability state.
- NATS JetStream: durable engine pipeline event stream.
- Redis/BullMQ: app-facing or operational async jobs such as email, scheduled deletion worker dispatch, small notifications, and retryable app tasks.

P0 runtime dependency decision:

- NATS JetStream is a required P0 runtime dependency in every local/internal environment.
- If a runtime does not already provide NATS JetStream, `vox2vocal-infra` must include provisioning, stream setup, durable consumer setup, and healthcheck work before engine pipeline testing.

### Core Flow

```text
login/signup
-> song selection
-> section selection
-> recorder entry gate
-> record/replay/retake
-> createAudioUploadSession
-> upload recorder take or fallback file to object storage
-> completeAudioUpload
-> conversion-job-state creates canonical job
-> transactional outbox publishes audio.ingest.requested
-> engine pipeline emits stage events
-> worker adapter normalizes events into StageResult
-> BFF reads job projection
-> app shows processing/result
-> signed preview playback
-> rating/failure tags
```

### Job State Ownership

Enterprise-style long-running jobs are usually managed through a workflow/state-machine model or through `job table + stage ledger + transactional outbox`. P0 chooses the second option because it fits the current repo shape and avoids introducing a new orchestration service too early.

The `conversion-job-state` bounded module owns:

- job id and canonical state
- upload completion commit boundary
- stage transition and StageResult upsert
- idempotency handling
- partial artifact interpretation
- timeout after `completeAudioUpload.committed_at + 60 minutes`
- retention deadline assignment
- app-facing read projection
- transactional outbox rows for engine requests

BFF and API Gateway must not invent final states. They may only expose projections from the canonical job state owner.

### Event And StageResult Approach

Engine services may emit engine-specific typed events, but the app-facing job state uses normalized StageResult records.

```text
engine event
-> worker event consumer / adapter
-> validation and idempotency check
-> StageResult upsert
-> job projection update
```

The adapter belongs to `worker` for P0. Engines should not write directly into job-state tables. This keeps engine evolution independent while preserving one canonical projection.

P0 NATS event contract defaults:

- Stream: `VOX2VOCAL_AUDIO`.
- Subject prefix: `audio`.
- Required ingest subjects: `audio.ingest.requested`, `audio.ingest.started`, `audio.ingest.completed`, `audio.ingest.failed`, `audio.ingest.dead_letter`.
- Required envelope fields: `schema_version`, `event_id`, `trace_id`, `job_id`, `audio_asset_id`, `target_section_id`, `attempt`, `occurred_at`, `producer`, `payload`.
- `audio.ingest.requested` payload must include the source object reference, expected content constraints, target song package id, target section id, and idempotency key reference.
- `audio.ingest.completed` payload must include canonical artifact reference, duration, sample rate, channel count, loudness/silence summary, validation confidence, and warnings.
- `audio.ingest.failed` and `audio.ingest.dead_letter` payloads must include machine-readable failure code, user-safe reason category, retryability, attempt count, and source event id.
- Downstream engine events after ingest may keep engine-specific payloads, but must keep the same envelope fields and be normalized into StageResult before reaching app-facing job state.

### Upload And Playback Approach

Recorder take and fallback upload both use presigned direct object storage upload.

- `createAudioUploadSession` issues a short-lived PUT URL.
- App uploads the local recorder take or fallback file to the issued path.
- `completeAudioUpload` validates session ownership, expiry, object HEAD, size, content type, consent snapshot, rights state, and audit allow decision.
- Only `completeAudioUpload` creates the conversion job and outbox event.
- Preview playback uses short-lived signed GET URL.
- Reference pre-listen uses a separate signed URL path and is denied unless `reference_prelisten_allowed=true`, scope allows the selected section, no active recording is in progress, and audit write succeeds.
- P0 object storage backend is MinIO for local/internal operation. Keep the storage interface S3-compatible so later migration to managed S3-compatible storage does not change app/BFF contracts.
- In-app recorder may upload a platform-native recording format when necessary, but `audio-ingest` is the authoritative normalization boundary. A job is not considered ingested until the input is validated and normalized into the accepted canonical audio contract.

### Engine Pipeline Approach

Engine composition remains current because it supports the final product goal. P0 depth is constrained around a partial-real, lineaged preview:

| Option | Path | P0 Use | Counts For P0 Self-voice Success |
| --- | --- | --- | --- |
| `mock` | `upload -> job state -> static/fake preview -> playback/rating UI` | UI, loading/error/result layout testing only | No |
| `partial_real` | `audio-ingest -> section validation -> user pitch extraction -> target pitch mapping -> constrained self-voice preview -> lightweight render/evaluation -> safety-rights` | Recommended alpha path | Yes, if lineage and playback criteria pass |
| `real_synthesis` | `audio-ingest -> voice-pitch -> melody-mapping -> singing-synthesis -> vocoder-render -> mix-master -> evaluation -> safety-rights` | Preferred long-term path; P0 feature flag or shadow path after partial-real stabilizes | Yes |

P0 recommendation is `partial_real`. It must be derived from the committed user voice input, app-playable, section-limited, and machine-checkable against `source_audio_asset_id`. Pitch-only success never counts as self-voice preview success.

- P0 critical path: `audio-ingest`, section validation, `voice-pitch`, target pitch mapping, constrained preview generation/render, lightweight `evaluation`, `safety-rights`.
- P0 limited/supporting: `voice-analysis` for energy/silence/quality hints if useful.
- P0 non-blocking or Later depth: `phoneme-alignment`, `rhythm-timing`, full `singing-synthesis`, full `vocoder-render`, full `mix-master`, `expression`, `voice-conversion`, unless the selected preview path specifically requires them.

## Frontend Changes

### Learner App

Implement or extend these screens:

- Login / Signup
- Song Selection
- Section Selection
- Record Take / Take Review
- Processing Status
- Result / Preview / Rating
- Data / Consent / Deletion Settings
- Contact follow-up opt-in UI is out of P0 and remains hidden/disabled.

Recorder requirements:

- mic permission prompt
- count-in
- section timer
- input level meter
- record/stop
- own take replay
- retake
- submit
- fallback upload when enabled
- reference pre-listen before recording only when allowed
- lyric cue only when allowed
- stop reference playback before active recording starts

The app should perform cheap preflight checks where possible:

- recording duration candidate
- local take availability
- obvious silence/too-low input indicator
- clipping warning from input meter
- file extension/MIME pre-check for fallback upload

Authoritative validation remains server/engine owned.

### Internal Web Surface

Internal operators use a separate web surface, not the learner mobile app.

Required internal surfaces:

- Limited Admin Song Package / Rights Gate
- Educator / Expert Review Queue
- Review Detail / Internal Reviewer Mode
- Governance Evidence / Audit / Deletion

Admin web must support direct reference audio upload for P0. Provider automation remains Later.

## Backend / Service Changes

### BFF

BFF exposes app/internal web GraphQL operations and does not own domain state.

Required changes:

- GraphQL schema for song package, section selection, recorder upload session, job status, result, playback URL, rating, failure tags, deletion, disabled contact follow-up capability status, admin package management, review queue.
- Auth context extraction and forwarding to API Gateway.
- No logging of upload body, signed URL secrets, playable preview URLs, tokens, full lyrics, or contact plaintext.

### API Gateway

API Gateway coordinates internal calls and enforces authenticated user context forwarding.

Required changes:

- Extend existing upload session contract to support P0 metadata.
- Add `completeAudioUpload`.
- Add job status/result/playback internal contracts.
- Add song package and section contracts.
- Add rights/risk/admin contracts.
- Add consent/deletion contracts and disabled-by-default contact gate contracts.

### User Service

User Service remains owner of account/auth/user identity.

Required changes:

- Role/status field must support learner, educator/expert, admin, engine developer, product QA, security/ops.
- Internal surface access uses role allowlist and audit in P0. MFA/step-up is a Later requirement for high-risk admin/review/governance actions unless already implemented elsewhere.

### Worker

Worker gains P0 canonical job responsibilities:

- `conversion-job-state` bounded module
- `jobs` projection update
- StageResult adapter/consumer
- transactional outbox publisher
- timeout handling
- retention deadline assignment
- deletion job orchestration
- BullMQ jobs for contact or notification tasks when enabled

### Engines

Engines keep their current final-target composition. P0 implementation must define event contracts per stage and emit typed events with stable ids. Engine event payloads should include job id, stage, attempt, status, artifact refs, error codes, timing, confidence summary when relevant, and engine version.

### Safety Rights

Safety Rights must be invoked before:

- song package exposure
- recording/upload completion
- analysis/preview request
- reference pre-listen URL issuance
- generated preview playback URL issuance
- deletion and rights complaint state changes
- contact decrypt/send actions when contact follow-up is enabled

## API Contract

This section defines TRD-level contract boundaries. Field-level request/response, validation, migration, indexing, and analytics contracts are expanded in `pm/vox2vocal-mvp-api-data-contract-plan.md`.

P0 contract defaults:

- API Gateway internal contracts should be proto-first where a service boundary exists; BFF GraphQL derives app-facing schemas from those contracts.
- BFF and API Gateway are facades. They do not own canonical job state and must not write foreign domain tables directly.
- Every write API must define auth scope, idempotency behavior, validation rules, audit behavior, state transition effects, and user-safe error codes before ticketing.
- Standard user-safe error codes start with `auth_required`, `role_denied`, `consent_missing`, `consent_outdated`, `rights_blocked`, `rights_under_review`, `deletion_running`, `audit_failed`, `upload_expired`, `upload_invalid`, `idempotency_conflict`, `stage_failed`, `preview_unavailable`, `step_up_required`.
- Client-provided consent or rights refs are not authoritative. Server-side commit paths must compute and lock the current consent, rights, risk, song package, section, BPM/key, and reference asset snapshots.
- Job creation snapshots should include immutable ids or hashes for `song_package_revision_id`, `section_revision_id`, `reference_asset_id`, `reference_asset_checksum`, `rights_record_revision_id`, `risk_acceptance_id`, and `consent_snapshot_id/hash`.
- Ratings should be accepted only after a playback session or playback audit proves the metric-eligible preview was issued and played.

P0 contract freeze gate before `spec-to-tickets`:

- `api-data-contract-planner` must turn the object-level contracts below into GraphQL and proto field-level schemas before backend/frontend tickets are split.
- Generated contracts must define field names, nullability, enum values, pagination shape, idempotency behavior, error mapping, and schema version.
- Ticketing must not start from endpoint names alone.

Contract versioning:

- Every request/response, engine event, StageResult, JobProjection, PreviewArtifact, and PlaybackEvent includes `schema_version`.
- P0 uses additive minor changes only after a contract is implemented.
- Unknown major versions are rejected with a user-safe `unsupported_schema_version` or internal equivalent.
- Unknown enum values are stored as `unknown` plus original raw value for internal review; user-facing state must fall back to a safe reason.
- BFF GraphQL response version and API Gateway proto version must be traceable to the same contract revision.

Minimum shared objects:

`JobProjection`:

- `job_id`, `user_id`, `song_package_id`, `section_id`, `canonical_state`, `output_flags`, `user_safe_reason`, `created_at`, `updated_at`.
- `committed_at`, `timeout_at`, `terminal_at`, `blocked_at`, `deletion_state` when applicable.
- snapshot refs: `consent_snapshot_id/hash`, `rights_record_revision_id`, `risk_acceptance_id`, `song_package_revision_id`, `section_revision_id`, `reference_asset_id`.
- stage summary: latest StageResult per stage with status, error code, retryable flag, and timing.
- artifact summary: preview artifact id, pitch report artifact id, quality report artifact id, and playback eligibility.

`PreviewArtifact`:

- `artifact_id`, `job_id`, `source_audio_asset_id`, `source_audio_checksum`, `render_artifact_ref`, `pipeline_mode`, `generation_mode_version`.
- `mock_fixture_used`, `section_limited`, `section_id`, `section_start_ms`, `section_end_ms`, `preview_duration_ms`, `section_coverage_ratio`.
- lineage fields: `lineage_verification_status`, `lineage_source_event_id`, `source_user_id`, `consent_snapshot_id/hash`.
- quality fields: `quality_status`, `clipping_detected`, `loudness_summary`, `artifact_warning_codes`, `confidence_summary`.
- policy fields: `playback_blocked`, `rights_snapshot_id`, `risk_acceptance_id`, `deletion_state`, `retention_deadline`.
- Metric-eligible preview requires `pipeline_mode` in `partial_real` or `real_synthesis`, `mock_fixture_used=false`, `lineage_verification_status=verified`, `section_limited=true`, `section_coverage_ratio >= 0.8`, and `playback_blocked=false`.

`PlaybackSession`:

- `playback_session_id`, `artifact_id`, `job_id`, `user_id`, `issued_at`, `expires_at`, `signed_url_audit_id`, `playback_blocked_at`.
- `preview_duration_ms`, `unique_timeline_coverage_ms`, `unique_timeline_coverage_ratio`, `foreground_started`, `muted_seen`, `severe_error_seen`, `preview_played`.
- `preview_played_computed_at`, `rating_unlocked_at`, and `last_event_sequence`.

`PlaybackEvent`:

- `event_id`, `schema_version`, `playback_session_id`, `artifact_id`, `client_sequence`, `occurred_at`, `received_at`, `event_type`.
- `position_ms`, `duration_ms`, `played_range_start_ms`, `played_range_end_ms`, `muted`, `app_foreground`, `error_code`.

### Auth

- `signUp(email, password, displayName)`
- `login(email, password)`
- `refreshSession(refreshToken)`
- `logout(refreshToken)`
- `me()`

Existing gRPC contracts cover these partially.

### Song And Section

- `listSongPackages()`
- `getSongPackage(songPackageId)`
- `listSongSections(songPackageId)`
- `getSelectedSectionGuide(songPackageId, sectionId)`

Must return rights state, exposure decision, section metadata, BPM/key defaults, and recording guide flags.

### Upload And Job Creation

- `createAudioUploadSession(originalFilename, contentType, contentLength, songPackageId, sectionId)`
- `completeAudioUpload(uploadSessionId, idempotencyKey, songPackageId, sectionId, takeId?)`

`completeAudioUpload` is the job creation boundary. It must return either an existing idempotent job projection or a new job id/projection. It verifies the upload session and object HEAD, computes authoritative consent, rights, risk, song package, section, BPM/key, and reference asset snapshots server-side, commits the job and outbox row in one transaction, and rejects conflicting idempotency keys. The client must not provide authoritative consent or rights refs.

### Job Status And Result

- `getJobStatus(jobId)`
- `subscribeJobStatus(jobId)`
- `getJobResult(jobId)`

Job status must include canonical state, output flags, stage summaries, user-safe reason, and timeout/deletion indicators.

Final decision policy:

| Technical condition | Canonical state | Rating eligibility |
| --- | --- | --- |
| Metric-eligible preview artifact exists, lineage is verified, playback is allowed, and required reports are present or warning-only | `completed` | Allowed after `preview_played=true` |
| Preview artifact is playable but a non-critical report failed | `failed_with_partial_artifacts` | Allowed after `preview_played=true`; job completion success does not count |
| Pitch/report artifacts exist but no playable self-voice preview exists | `failed_with_partial_artifacts` | Not allowed |
| Critical engine failure, no useful user-facing artifact, or timeout without preview | `failed` | Not allowed |
| Consent, rights, deletion, or audit gate blocks processing or playback | `blocked` or deletion-specific state | Not allowed |
| Low confidence, disputed target notes, suspicious artifact, or ambiguous rights need human review | `needs_review` | Not allowed until review resolves playback eligibility |

### Playback

- `issuePreviewPlaybackUrl(jobId, artifactId)`
- `issueReferencePrelistenUrl(songPackageId, sectionId)`
- `recordPreviewPlaybackEvent(playbackSessionId, artifactId, eventId, clientSequence, schemaVersion, occurredAt, eventType, positionMs, durationMs, muted, appForeground, playedRangeStartMs?, playedRangeEndMs?, errorCode?)`

Preview playback URL issuance returns a `playback_session_id` and requires audit write success. Reference pre-listen must fail if active recording is in progress or scope does not allow selected section.

Playback event defaults:

- Event types: `playback_started`, `playback_progress`, `playback_paused`, `playback_seeked`, `playback_ended`, `playback_error`.
- Every playback event must include `event_id`, `schema_version`, `occurred_at`, and a monotonically increasing `client_sequence` within the playback session.
- The app should send `playback_progress` every 1 second while foreground, unmuted playback is active.
- The app must also flush playback progress on pause, seek, end, background, mute, and error.
- The server computes `unique_timeline_coverage` by merging distinct played ranges. Client-reported `total_played_ms` is not trusted for metric eligibility.
- Duplicate or out-of-order playback events must be idempotently merged by `playback_session_id`, `artifact_id`, event id, and played range.

Playback anti-abuse and race rules:

- Events are accepted only for the authenticated user, issued playback session, artifact, and job tuple.
- `occurred_at` must be inside the playback session validity window. Late delivery may be accepted only when the event time was valid and the server receives it within a small flush grace window.
- Played ranges must satisfy `0 <= played_range_start_ms < played_range_end_ms <= preview_duration_ms`; invalid ranges are rejected and logged with a safe reason.
- The server must reject or ignore impossible progress, such as played duration that exceeds wall-clock elapsed time plus tolerance, negative jumps without a seek event, or payload changes for an existing `event_id`.
- Muted or background playback events may be stored for diagnostics but must not increase metric-eligible coverage.
- Seek-loop replay of the same short segment must not increase unique timeline coverage beyond the distinct merged ranges.
- If consent, rights, deletion, audit, or artifact status changes after URL issuance but before rating, `preview_played` and rating unlock must be recomputed against the latest blocking state.
- Expired sessions, playback sessions issued for blocked artifacts, and severe playback errors cannot unlock rating.

`preview_played=true` requires all of the following:

- `preview_artifact_id` exists.
- `pipeline_mode` is `partial_real` or `real_synthesis`.
- `mock_fixture_used=false`.
- `section_limited=true`.
- `playback_blocked=false`.
- `playback_session_id` exists.
- signed playback URL issuance audit succeeded.
- playback started while app was foregrounded.
- player was not muted.
- no severe playback error occurred.
- unique timeline coverage is at least 80% of preview duration.

For `Mist intro` at 28 seconds, this means at least 22.4 seconds of distinct preview timeline must be heard. Replaying the same short segment repeatedly must not inflate `unique_timeline_coverage`.

### Rating And Review

- `submitPreviewRating(jobId, artifactId, playbackSessionId, rating, ratingPromptVersion)`
- `submitFailureTags(jobId, artifactId, tags, otherText?)`
- `reportPlaybackProblem(jobId, artifactId, reason)`
- `listReviewableJobs(filter)`
- `getReviewDetail(jobId)`
- `submitReviewComment(jobId, comment)`
- `submitTechnicalTags(jobId, tags)`

User perception tags and internal technical tags must be stored separately. Rating submission is allowed only after `preview_played=true`. Ratings below 4 require at least one failure tag or `other`.

### Consent, Deletion, Contact

- `getConsentStatus()`
- `grantConsent(consentType, version, scope)`
- `withdrawConsent(consentType)`
- `requestDeletion(jobId?, artifactId?)`
- `getDeletionStatus(requestId)`
- `getContactFollowupCapability()`

P0 contact follow-up is disabled capability only. `getContactFollowupCapability()` may return a disabled state and reason, but contact value collection, contact preference writes, decrypt, send, export, and contact UI are out of P0.

### Admin / Governance

- `createSongPackage(input)`
- `uploadReferenceAudioSession(songPackageId, contentType, contentLength)`
- `updateSectionMap(songPackageId, sections)`
- `updateRightsState(songPackageId, rightsState, evidenceRef)`
- `createRiskAcceptance(input)`
- `blockSongPackage(songPackageId, reason)`
- `listAuditEvidence(filter)`
- `listDeletionEvidence(filter)`

Admin writes must fail closed if audit write fails.

## Data Model / Migration

P0 requires additive schema changes. Exact Prisma models and migrations should be defined in the API/data contract planning step.

### P0 Data Ownership Matrix

Use schema-bounded ownership instead of one giant owner. PostgreSQL may remain one cluster/database in P0, but write ownership and migration ownership must be separated.

| Domain / table group | Owning repo/service | Write path | Read path |
| --- | --- | --- | --- |
| `users`, auth/session data, role/status | `vox2vocal-user-service` | User Service Prisma/migrations | API Gateway -> User Service -> BFF/App |
| account-level `consent_records` | `vox2vocal-user-service` | User Service computes current account consent | API Gateway/BFF read current consent status |
| contact follow-up capability | `vox2vocal-user-service` | disabled capability status only in P0 | no contact value writes, no plaintext UI, no send/export |
| song packages, section maps, reference assets, rights/risk records | `vox2vocal-worker` `conversion-job-state` bounded module for P0 | admin seed/admin-write path through API Gateway; separate admin project Later | BFF/API Gateway read projections |
| upload sessions, audio assets, conversion jobs, job snapshots | `vox2vocal-worker` `conversion-job-state` bounded module | `completeAudioUpload` and worker-owned module | BFF/API Gateway read app-facing projection |
| `stage_results`, `artifact_refs`, `outbox_events` | `vox2vocal-worker` `conversion-job-state` bounded module | worker event consumer / adapter | BFF/API Gateway read projection; engines never write tables directly |
| playback sessions, ratings, failure tags, technical tags | `vox2vocal-worker` `conversion-job-state` bounded module | app/reviewer APIs through API Gateway | product/QA/review projections |
| deletion requests/evidence hooks | `vox2vocal-worker` with platform/storage integration | deletion worker and audit write | governance/review projections |
| audit schema baseline, PostgreSQL grants, storage/NATS/Redis resources | `vox2vocal-infra` / platform owner | infra migrations/provisioning | services use least-privilege credentials |

BFF and API Gateway must remain facades. Engines emit typed stage events only and must not write job-state tables directly.

Core tables or model groups:

- `users`: existing user identity plus role/status compatibility.
- `consent_records`: type, version, scope, required flag, policy document version/hash, granted/withdrawn timestamps.
- `job_consent_snapshots`: immutable snapshot id/hash, selected song/section, rights/display flags, consent refs.
- `song_packages`: title, artist, language, BPM, key, status, default section, retention/deletion owner.
- `section_maps`: section id, label, start/end timestamp, duration, representative cue, BPM/key snapshot.
- `reference_assets`: reference audio artifact, checksum, uploader, provenance, retention deadline.
- `rights_records`: allowed/prohibited uses, approver, evidence location, expiry/re-review, complaint owner.
- `risk_acceptance_records`: allowed users/groups, exact section, duration, prohibited uses, kill-switch owner.
- `upload_sessions`: owner, object key, content type, expiry, idempotency key, selected song/section.
- `audio_assets`: source object, owner, source type, duration candidate, checksum.
- `conversion_jobs`: canonical state, selected song/section, source asset, BPM/key snapshot, consent snapshot, rights snapshot.
- `stage_results`: normalized stage ledger.
- `artifact_refs`: storage pointer, data class, status, rights state, playback allowed, retention deadline.
- `outbox_events`: pending engine requests after job commit.
- `playback_sessions`: issued preview playback session, audit refs, playback progress, muted/app foreground flags, severe error status, unique timeline coverage.
- `ratings`: artifact rating, prompt version, playback eligibility.
- `failure_tags`: user perception tags and optional `other`.
- `technical_tags`: reviewer/engine developer tags.
- `audit_records`: allow/deny decisions and governance actions.
- `deletion_requests` and `deletion_evidence`: deletion workflow and evidence.
- `contact_preferences`: Later/gate-ready model only. P0 must not migrate or write contact values unless a later contact collection gate explicitly enables it.

Migration principles:

- Use additive migrations first.
- Backfill only when needed for existing users.
- Avoid destructive column changes during P0.
- Store enums in a backward-compatible way or plan enum migration carefully.
- Keep playable URLs out of DB; store artifact identity and storage path only.
- Keep raw audio, generated preview, full lyrics, tokens, secrets, and contact plaintext out of logs.
- Use expand/contract migrations: add nullable structures first, deploy compatible code, backfill/validate, then tighten constraints in a later migration.
- Prefer text values plus validation or lookup tables for unstable state enums until P0 state names settle.
- Add unique constraints for upload completion idempotency, job per upload session, stage event dedupe, and outbox dedupe.
- Job creation and outbox insert must be one database transaction.
- Rollback disables feature flags and outbox publishing; it must not drop P0 data tables or delete audit/deletion evidence.
- DB roles should enforce ownership: BFF has no direct write role, engines have no job table write role, worker writes conversion schema, and user-service writes user/consent/contact schema.

## Auth / Permissions

Role model:

- `learner`: own jobs, own previews, own reports, own deletion/consent settings.
- `educator_or_expert`: consented reviewable jobs only.
- `admin`: song package and rights/risk management.
- `engine_developer`: limited debug metadata and technical tags.
- `product_qa`: pseudonymous metrics and quality reports.
- `security_or_ops`: audit, deletion, rights complaint, governance evidence.

Owner metadata that is not necessarily a login role:

- `policy_or_rights_owner`
- `platform_storage_owner`
- `kill_switch_owner`
- `complaint_owner`

Critical permission rules:

- Learner cannot upload reference audio.
- Learner cannot access another user's job or preview.
- Educator/expert cannot access raw audio by default.
- Engine developer cannot receive unrestricted raw audio access.
- Contact plaintext reveal is disabled in P0 unless a separate approver/key process exists.
- Break-glass raw/canonical audio access is disabled in P0 1-person operation.
- Any action involving rights, consent, playback URL, deletion, contact decrypt/send, or risk acceptance requires audit success.

P0 auth/security defaults:

- Learner auth starts with email/password plus email verification. Unverified users cannot submit uploads, create jobs, or issue playback URLs.
- Internal routes use a separate RBAC guard, allowlist, feature flag, and audit requirement even when they live in the same app/web structure.
- All sensitive API decisions must be rechecked server-side: `user_id`, role/status, consent snapshot, rights state, deletion state, and audit write success.
- MFA means a second authentication factor beyond password, such as OTP, passkey, or hardware security key.
- Step-up authentication means requiring that extra verification again immediately before high-risk actions even if the user is already logged in.
- P0 does not require learner step-up for normal recording/playback. Later admin/review/governance step-up candidates are reference audio upload/replace/delete, song publish/block, rights state change, risk acceptance, role elevation, retention policy change, deletion override, contact decrypt/send, and raw/canonical audio break-glass.
- Because P0 has no separate second reviewer/security owner, contact plaintext reveal and raw/canonical audio human break-glass remain disabled.

## Observability

Observability must make P0 success metrics and safety gates measurable.

Required identifiers:

- `trace_id`
- `request_id`
- `job_id`
- `stage_result_id`
- `artifact_id`
- `audio_asset_id`
- `user_id` or pseudonymous user id where appropriate
- `song_package_id`
- `section_id`
- `engine_name`
- `engine_version`

Required events/metrics:

- auth success/failure
- song selected
- section selected
- recorder ready
- take submitted
- upload session created/expired/completed
- `completeAudioUpload` committed/denied
- engine stage queued/running/succeeded/failed/blocked
- preview ready/completed/failed
- playback URL issued/denied
- playback started/ended/error
- playback progress with distinct timeline coverage
- `preview_played=true` threshold reached
- rating submitted
- failure tags submitted
- consent granted/withdrawn
- deletion requested/running/completed/failed
- rights state changed
- contact follow-up capability queried/denied

Logs must exclude raw audio, generated preview URLs, signed URL secrets, access tokens, refresh tokens, passwords, provider secrets, full lyrics, contact plaintext, and sensitive free text.

Reason code and alert defaults:

- Safety gates must emit standardized allow/deny reason codes, including `consent_missing`, `consent_outdated`, `rights_blocked`, `rights_under_review`, `audit_failed`, `deletion_running`, `role_denied`, `step_up_required`, and `playback_blocked`.
- Success-metric dashboard: funnel events, section job completion, metric-eligible preview playback, rating distribution, failure tags, pitch confidence, and time-to-preview.
- Safety/operations dashboard: audit write failures, playback URL deny spikes, rights state changes, deletion failures, outbox backlog, engine timeout P95, queue depth, signed URL issuance errors.
- Alerts should start for `audit_failed` spikes, any `deletion_failed`, rights block/unblock changes, oldest queued job over threshold, outbox backlog growth, and P95 timeout regression.
- Observability stores `artifact_id` and audit ids, not signed URLs or playable preview URLs.

## Performance / Security Considerations

Performance targets:

- P0 prioritizes stable section-level success over speed.
- P50 10 minutes and P95 30 minutes are observation targets.
- Jobs over 60 minutes become timeout candidates.
- Upload URL TTL starts at 15 minutes.
- Playback URL TTL starts at 5 minutes.
- `P0_MAX_UPLOAD_BYTES` starts at 50 MB.

Alpha capacity defaults:

| Area | P0 Default |
| --- | --- |
| Concurrent P0 conversion jobs | `2` active jobs globally |
| Heavy preview/synthesis concurrency | `1` active preview synthesis/render job |
| Ingest/pitch/evaluation worker concurrency | `2` per stage in a small internal runtime |
| Job-state/outbox workers | `1` outbox publisher and `2` StageResult consumers |
| Queue depth warning | waiting jobs `>= 4` for 5 minutes or oldest queued job `>= 10 minutes` |
| Queue depth critical/throttle | stop accepting non-admin P0 jobs at waiting jobs `>= 8` or oldest queued job `>= 20 minutes` |
| Per-stage soft timeout | ingest `5m`, pitch/mapping `5m`, partial-real preview synthesis `25m`, render/mix `10m`, evaluation `5m` |
| End-to-end timeout | terminalize after `60m` from `completeAudioUpload.committed_at` |
| User upload duration | hard max `60s`; do not silently process full-song input |
| Generated preview coverage | `section_coverage_ratio >= 0.8` or duration within section tolerance |

Storage and audio normalization defaults:

- P0 local/internal object storage is MinIO.
- Storage contracts must remain S3-compatible.
- Fallback upload accepts `.wav` and `.mp3` only.
- In-app recorder may submit platform-native audio when the app cannot reliably export `.wav` or `.mp3`, but `audio-ingest` must validate and normalize it before `audio_ingest=succeeded`.
- Canonical audio for downstream engines is normalized mono WAV/PCM.

Security and privacy controls:

- All audio-bearing object storage is private by default.
- Encrypt raw audio, canonical audio, generated preview, reference audio, and, if later enabled, contact ciphertext at rest.
- Use short-lived signed access for upload and playback.
- Use least-privilege service access to object storage.
- Deny playback if consent is withdrawn, rights are blocked/expired/under review, deletion is running, audit fails, or artifact is blocked/deleted.
- Keep contact follow-up behind an explicit gate: authenticated encryption, keyed hash, key management service, audit write success, no plaintext UI, no CSV export.
- Use rights state and risk acceptance on every selection, job creation, engine request, and playback issuance path.

Runtime policy changes:

- If required consent is withdrawn while a job is `created`, `queued`, `processing`, `preview_ready`, or `completed`, block new engine requests and new playback URLs immediately. Processing jobs should move to `blocked` unless deletion is requested.
- If rights become `blocked`, `under_review`, or `expired`, block song selection, new job creation, engine requests, reference pre-listen, and generated preview playback. Preserve artifacts for policy review unless deletion is also requested.
- If deletion is requested, mark related artifacts `deletion_pending`, block new playback URLs, enqueue deletion, and write deletion evidence. If deletion fails, mark `deletion_failed` and require platform/storage owner review.
- If audit write fails for a rights-sensitive action, fail closed and do not create the job, publish the engine request, issue playback, or change rights state.

## Rollout / Rollback Plan

### Rollout

1. Freeze field-level GraphQL/proto contracts with `api-data-contract-planner`.
2. Add additive database tables, DB grants, and read models behind feature flags.
3. Implement consent records and job consent snapshot.
4. Implement song package, section package, rights/risk evidence source of truth, and read path.
5. Implement recorder/take review UI and upload session path.
6. Implement `completeAudioUpload` and `conversion-job-state`.
7. Implement transactional outbox and StageResult schema validation.
8. Integrate `audio-ingest` and StageResult adapter.
9. Add partial-real pitch and preview pipeline with PreviewArtifact lineage verification.
10. Add result playback, playback audit/session, rating, and failure tags.
11. Add same-structure role-gated internal web routes for admin, review, governance.
12. Keep contact follow-up disabled. Do not collect, store, send, decrypt, or export contact values in P0.
13. Run internal allowlist P0 with `Mist intro`.

### Rollback

Rollback must be operationally safe rather than only code rollback.

- Disable song package exposure through rights state or feature flag.
- Disable `completeAudioUpload` for new jobs.
- Keep existing job status readable.
- Block new playback URL issuance when rights/consent/audit/deletion issues occur.
- Retain deletion jobs and audit evidence even if UI is rolled back.
- Keep contact follow-up disabled capability off; no contact values should exist in P0 rollback scope.
- If engine pipeline is unstable, stop outbox publishing for new engine requests and preserve existing jobs as failed/blocked/needs_review with user-safe reasons.
- Use explicit kill switches: `disable_complete_audio_upload`, `disable_playback_url_issue`, `disable_reference_prelisten`, `disable_internal_admin_writes`, `stop_engine_outbox_publish`.
- Already issued signed URLs cannot be reliably revoked; mitigation is short TTL, artifact blocked state, and denying all new URL issuance.
- Rollback must not drop P0 tables, erase audit evidence, or erase deletion evidence. Use feature flags and outbox pause/drain/replay instead.
- Worker rollback must define how pending outbox events and partially completed jobs are marked, retried, or moved to `needs_review`.

## Test Plan

### Unit Tests

- consent version/scope/currentness checks
- rights state allow/deny matrix
- upload session expiry and ownership checks
- `completeAudioUpload` idempotency
- StageResult normalization and duplicate handling
- canonical job state transitions
- playback URL eligibility
- deletion evidence creation
- contact follow-up capability remains disabled and rejects preference writes
- user-safe error code mapping
- StageResult schema validation and enum compatibility
- JobProjection, PreviewArtifact, PlaybackSession, and PlaybackEvent schema validation
- playback anti-abuse range validation and stale session rejection

### Integration Tests

- App -> BFF -> API Gateway signup/login/me
- song selection and section selection read path
- recorder/fallback upload session creation
- object HEAD validation after upload
- `completeAudioUpload` creates job and outbox event
- engine event -> StageResult -> job projection
- result playback URL issue/deny
- rating/failure tag submission
- consent withdrawal blocks playback and review access
- deletion request blocks playback and creates deletion job
- GraphQL to internal proto/gRPC contract compatibility
- BFF GraphQL -> API Gateway proto -> worker JobProjection contract tests
- transactional job creation plus outbox insert rollback on failure
- NATS duplicate/replay event handling
- playback progress interval, event id/schema version validation, duplicate/out-of-order event merge, stale/expired session rejection, impossible played range rejection, and unique timeline coverage calculation
- consent/rights/deletion changes between playback URL issuance, playback start, `preview_played=true`, and rating submission
- PreviewArtifact lineage validation: `mock_fixture_used=false`, source audio checksum, source user id, section id, duration, and section coverage
- object storage lifecycle and retention/deletion flow
- DB ownership/grant test: BFF and engines cannot directly write owned domain tables

### E2E Tests

- learner creates account, selects `Mist`, selects `intro`, records/upload take, gets processing status, plays preview, rates.
- reference pre-listen hidden when flag false.
- reference pre-listen stops before active recording when flag true.
- rights blocked package cannot be selected, processed, or played.
- failed preview with pitch report does not ask for primary self-voice rating.
- educator sees only consented reviewable jobs.
- internal admin web can block package and immediately prevent playback URL issuance.

### Security / Governance Tests

- no raw audio, tokens, signed URLs, full lyrics, contact plaintext in logs.
- audit write failure causes fail-closed behavior.
- role-based access denies cross-user job access.
- contact follow-up disabled capability rejects UI writes, preference writes, send, decrypt, and export.
- break-glass raw/canonical audio access disabled without second reviewer/security owner.
- internal routes require role allowlist and never fall through to learner access.
- playback rating cannot be submitted without metric-eligible playback audit/session.
- metric-eligible rating is revoked or blocked if latest consent, rights, deletion, audit, or artifact state becomes blocking before submission.

## Risks and Tradeoffs

- Keeping engines in their final-target composition preserves long-term architecture but increases perceived P0 scope. Mitigation: constrain P0 implementation depth and ticketing.
- `worker` as canonical job state owner avoids a new orchestrator service but can grow beyond simple queue responsibilities. Mitigation: keep `conversion-job-state` as a bounded module with clear tables and outbox.
- Presigned upload/playback reduces BFF file handling cost but adds object storage validation and signed URL leakage risk. Mitigation: short TTL, object HEAD verification, no signed URL logging.
- App-side audio normalization can reduce server cost but may vary by platform. Mitigation: app may pre-normalize only when output contract is reliable; ingest remains authoritative.
- Contact follow-up is useful for internal learning but introduces sensitive data handling. Mitigation: keep it gate-only and disabled by default in P0; require explicit opt-in, encrypted storage, audit, no plaintext UI, and no CSV export before enabling.
- `partial_real` can accelerate validation but risks overstating product readiness. Mitigation: require machine-checkable user voice lineage and exclude mock from success metrics.

## Alternatives Considered

- Standalone Conversion Job Orchestrator: better long-term separation, not required for P0 because existing worker can host a bounded module.
- Temporal or Step Functions: strong workflow event history and long-running orchestration, deferred until P0 validates the preview loop and orchestration complexity justifies adoption.
- Kafka for engine pipeline: enterprise-grade stream processing, but heavier than needed for current P0. NATS JetStream remains the preferred P0 engine event stream.
- BFF-owned job state: rejected because BFF should not be source of truth for final processing states.
- Engine-owned final job state: rejected because different engines would compete over app-facing final state.
- Server-only upload path through BFF: rejected for cost and scalability; presigned direct object storage is preferred.
- Fully enabled contact follow-up in P0: rejected. P0 keeps disabled capability status only; contact UI, value storage, send, decrypt, and export remain out of scope until encryption/key/audit requirements pass in a later decision.

## Resolved P0 Technical Decisions From Review

- DB/schema ownership uses schema-bounded ownership: User Service owns identity/account consent and disabled contact capability state; Worker `conversion-job-state` owns P0 song package, rights/risk, upload, job, stage, artifact, rating/tag, and deletion workflow tables; Infra owns PostgreSQL grants and platform resources.
- API Gateway internal contracts are proto-first where a service boundary exists; BFF GraphQL derives app-facing contracts and stays a facade.
- NATS JetStream is a required P0 runtime dependency. Infra must provision it wherever it is missing.
- P0 ingest event defaults use `VOX2VOCAL_AUDIO` stream, `audio` subject prefix, required ingest subjects, and a shared envelope with `schema_version`, `event_id`, `trace_id`, `job_id`, `audio_asset_id`, `target_section_id`, `attempt`, `occurred_at`, `producer`, and `payload`.
- First metric-eligible preview path is `partial_real` lineaged preview. `real_synthesis` is preferred long-term and may run as a feature-flagged or shadow path. `mock` and pitch-only success do not count for P0 self-voice success.
- Internal admin/review/governance surfaces use the same app/web structure for P0 with separate role-gated routes, RBAC, feature flags, and audit. A separate admin project is Later.
- Rights evidence and risk acceptance source of truth lives in DB from P0 through seed/admin-write paths before the separate admin project exists.
- Learner MFA/step-up is not required for normal P0 recording/playback. Later admin/review/governance step-up is required for high-risk actions. P0 break-glass and contact plaintext reveal remain disabled without a second reviewer/security owner.
- Alpha capacity defaults start at 2 concurrent P0 conversion jobs globally, 1 heavy preview/render job, 60-minute end-to-end timeout, 15-minute upload URL TTL, 5-minute playback URL TTL, and 50 MB upload size guard.
- P0 object storage backend is MinIO with an S3-compatible contract.
- Ingest owns authoritative audio normalization. The app may pre-normalize only when reliable; fallback upload accepts `.wav` and `.mp3`, while recorder output can be normalized by `audio-ingest`.
- `preview_played=true` requires a real `partial_real` or `real_synthesis` preview, successful playback audit/session, foreground unmuted playback, no severe playback error, and at least 80% unique timeline coverage. For `Mist intro`, that is at least 22.4 seconds of distinct timeline coverage.
- Playback telemetry uses 1-second foreground/unmuted progress events with `event_id`, `schema_version`, `occurred_at`, and session-scoped `client_sequence`; the server merges distinct played ranges and does not trust client total playback time.
- Contact follow-up is disabled capability status only in P0; value collection, storage, send, decrypt, and export are out of scope.

## Open Technical Questions

- What is the exact event schema for each downstream engine stage before StageResult normalization, beyond the P0 audio-ingest envelope?
- If break-glass or contact plaintext reveal is ever enabled after P0, who becomes the second reviewer or security owner?

## References

- `vox2vocal-docs/pm/vox2vocal-mvp-prd.md`
- `vox2vocal-docs/pm/vox2vocal-mvp-feature-definition.md`
- `vox2vocal-docs/pm/vox2vocal-mvp-page-flow-plan.md`
- `vox2vocal-docs/pm/vox2vocal-mvp-api-data-contract-plan.md`
- `vox2vocal-docs/engine/README.md`
- `vox2vocal-docs/engine/audio-ingest/README.md`
- `vox2vocal-docs/engine/logging/README.md`
- `vox2vocal-bff-server/README.md`
- `vox2vocal-api-gateway/proto/gateway.proto`
- `vox2vocal-worker/README.md`
- `vox2vocal-user-service/README.md`
