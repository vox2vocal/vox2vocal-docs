# Vox2Vocal MVP Technical Requirements Document

문서 버전: v0.1
작성일: 2026-06-15
상태: 초안
적용 skill: `trd-writer`
기준 문서:

- `pm/vox2vocal-mvp-prd.md` v0.11
- `pm/vox2vocal-mvp-feature-definition.md` v0.4
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.2

## Technical Summary

Vox2Vocal P0는 학습자가 Ken Kamikita - `Mist`를 선택하고, `intro 0:00-0:28` section을 선택한 뒤, 앱에서 본인 목소리를 녹음하고, section-limited self-voice preview를 앱 안에서 재생해 평가하는 end-to-end 흐름을 구현한다.

P0 기술 목표는 상업 품질의 full-song generation이 아니라, 다음 조건을 안전하게 검증하는 것이다.

- 본인 voice input만 recorder 또는 fallback upload로 수집한다.
- song package, section, consent snapshot, rights flag snapshot이 없으면 job을 만들지 않는다.
- canonical job state는 하나의 owner가 관리한다.
- engine stage는 표준 StageResult로 정규화한다.
- preview는 app-only signed playback으로만 제공한다.
- rights, consent, audit, deletion 상태가 차단되면 processing 또는 playback을 fail closed한다.
- contact follow-up은 P0 포함 기능이지만 core preview/rating flow의 blocking dependency가 아니다.

현행 엔진 구성은 최종 목표를 위해 유지한다. 다만 P0 구현 깊이는 `audio-ingest`, `voice-pitch`, `melody-mapping`, `singing-synthesis`, `vocoder-render`, `mix-master`, `evaluation`, `safety-rights`의 최소 end-to-end path를 우선하고, `voice-analysis`, `phoneme-alignment`, `rhythm-timing`, `expression`, `voice-conversion`은 현행 엔진 문서상 구성은 유지하되 P0 필수 경로에서는 제한적으로 사용하거나 Later 깊이로 둔다.

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
| Self-voice section preview | Minimal preview generation and render path | singing synthesis, vocoder render, mix master, storage | `preview_available=true`, `section_limited=true`, lineage to user voice input |
| App-only playback | Short-lived signed GET and playback audit | App, BFF/API Gateway, storage, audit | no download/share/export, URL denied when blocked |
| Job status and partial artifact handling | Canonical job state owner and StageResult ledger | worker, DB, BFF, App | no BFF-invented final state, timeout terminal state |
| Rating and failure tags | Playback telemetry, rating, failure tag taxonomy | App, BFF/API Gateway, DB/analytics | rating only after meaningful preview playback |
| Educator/expert review | Separate web review queue and detail surface | Internal web, BFF/API Gateway, DB, storage | only consented jobs, no raw audio by default |
| Retention and deletion | Artifact retention deadline, deletion jobs, evidence ledger | worker, storage, DB, audit | raw audio 30-day default, deletion evidence, playback block on failure |
| Contact follow-up | Optional opt-in, encrypted contact storage, audit | App/internal web, BFF/API Gateway, DB, KMS/Vault or OS keychain | disabled unless contact collection gate passes |

## Architecture / Approach

### System Boundary

P0 uses the existing workspace shape and does not introduce a standalone Conversion Job Orchestrator service.

- `vox2vocal-app`: learner mobile/web surface for auth, song selection, section selection, recording, result, rating, deletion settings, and contact follow-up opt-in.
- Internal web surface: admin song package, educator/expert review, governance evidence, deletion/audit review. This is separate from the learner app.
- `vox2vocal-bff-server`: GraphQL BFF for app and internal web surfaces. It shapes app-facing schemas and forwards authenticated requests to API Gateway.
- `vox2vocal-api-gateway`: internal orchestration boundary. It forwards auth/user context and exposes internal service calls through gRPC or equivalent internal contracts.
- `vox2vocal-user-service`: account, auth, user identity, role/status.
- `vox2vocal-worker`: BullMQ-based worker today; P0 adds `conversion-job-state` bounded module for canonical job state, timeout handling, stage ledger adaptation, deletion jobs, and outbox publishing.
- `engine-*`: engine pipeline services. Engine composition remains as documented for the final product direction.
- Object storage: raw voice input, canonical audio, reference audio, generated preview, reports, deletion evidence references.
- PostgreSQL: source-of-truth records for users, consent, song package, upload sessions, jobs, stage results, artifacts, ratings, audits, deletion evidence, and contact preferences.
- NATS JetStream: durable engine pipeline event stream.
- Redis/BullMQ: app-facing or operational async jobs such as email, scheduled deletion worker dispatch, small notifications, and retryable app tasks.

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

### Upload And Playback Approach

Recorder take and fallback upload both use presigned direct object storage upload.

- `createAudioUploadSession` issues a short-lived PUT URL.
- App uploads the local recorder take or fallback file to the issued path.
- `completeAudioUpload` validates session ownership, expiry, object HEAD, size, content type, consent snapshot, rights state, and audit allow decision.
- Only `completeAudioUpload` creates the conversion job and outbox event.
- Preview playback uses short-lived signed GET URL.
- Reference pre-listen uses a separate signed URL path and is denied unless `reference_prelisten_allowed=true`, scope allows the selected section, no active recording is in progress, and audit write succeeds.

### Engine Pipeline Approach

Engine composition remains current because it supports the final product goal. P0 depth is constrained:

- P0 critical path: `audio-ingest`, `voice-pitch`, `melody-mapping`, `singing-synthesis`, `vocoder-render`, lightweight `mix-master`, `evaluation`, `safety-rights`.
- P0 limited/supporting: `voice-analysis` for energy/silence/quality hints if useful.
- P0 non-blocking or Later depth: `phoneme-alignment`, `rhythm-timing`, `expression`, `voice-conversion`, unless a selected preview path specifically requires them.

`partial_real` can count for P0 self-voice success only when the preview artifact is derived from the committed user voice input and machine-checkable lineage exists.

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
- Contact follow-up opt-in UI when gate is enabled

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

- GraphQL schema for song package, section selection, recorder upload session, job status, result, playback URL, rating, failure tags, deletion, contact follow-up, admin package management, review queue.
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
- Add consent/deletion/contact contracts.

### User Service

User Service remains owner of account/auth/user identity.

Required changes:

- Role/status field must support learner, educator/expert, admin, engine developer, product QA, security/ops.
- Internal surface access may require stronger authentication later. P0 MFA mechanism is an open question unless already implemented elsewhere.

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

This section defines TRD-level contracts. Exact request/response schemas should be expanded with `api-data-contract-planner`.

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
- `completeAudioUpload(uploadSessionId, idempotencyKey, songPackageId, sectionId, consentSnapshotRef, rightsFlagSnapshotRef, takeId?)`

`completeAudioUpload` is the job creation boundary. It must return either an existing idempotent job projection or a new job id/projection.

### Job Status And Result

- `getJobStatus(jobId)`
- `subscribeJobStatus(jobId)`
- `getJobResult(jobId)`

Job status must include canonical state, output flags, stage summaries, user-safe reason, and timeout/deletion indicators.

### Playback

- `issuePreviewPlaybackUrl(jobId, artifactId)`
- `issueReferencePrelistenUrl(songPackageId, sectionId)`

Both require audit write success. Reference pre-listen must fail if active recording is in progress or scope does not allow selected section.

### Rating And Review

- `submitPreviewRating(jobId, artifactId, rating, ratingPromptVersion)`
- `submitFailureTags(jobId, artifactId, tags, otherText?)`
- `reportPlaybackProblem(jobId, artifactId, reason)`
- `listReviewableJobs(filter)`
- `getReviewDetail(jobId)`
- `submitReviewComment(jobId, comment)`
- `submitTechnicalTags(jobId, tags)`

User perception tags and internal technical tags must be stored separately.

### Consent, Deletion, Contact

- `getConsentStatus()`
- `grantConsent(consentType, version, scope)`
- `withdrawConsent(consentType)`
- `requestDeletion(jobId?, artifactId?)`
- `getDeletionStatus(requestId)`
- `upsertContactFollowupPreference(contactType, contactValue, allowedPurposes, consentVersion)`
- `withdrawContactFollowup()`

Contact endpoints must be disabled unless the contact collection gate passes.

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
- `ratings`: artifact rating, prompt version, playback eligibility.
- `failure_tags`: user perception tags and optional `other`.
- `technical_tags`: reviewer/engine developer tags.
- `audit_records`: allow/deny decisions and governance actions.
- `deletion_requests` and `deletion_evidence`: deletion workflow and evidence.
- `contact_preferences`: encrypted contact value, masked value, keyed hash, consent and purpose fields.

Migration principles:

- Use additive migrations first.
- Backfill only when needed for existing users.
- Avoid destructive column changes during P0.
- Store enums in a backward-compatible way or plan enum migration carefully.
- Keep playable URLs out of DB; store artifact identity and storage path only.
- Keep raw audio, generated preview, full lyrics, tokens, secrets, and contact plaintext out of logs.

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
- rating submitted
- failure tags submitted
- consent granted/withdrawn
- deletion requested/running/completed/failed
- rights state changed
- contact opt-in granted/withdrawn
- contact send/decrypt denied or performed by service path

Logs must exclude raw audio, generated preview URLs, signed URL secrets, access tokens, refresh tokens, passwords, provider secrets, full lyrics, contact plaintext, and sensitive free text.

## Performance / Security Considerations

Performance targets:

- P0 prioritizes stable section-level success over speed.
- P50 10 minutes and P95 30 minutes are observation targets.
- Jobs over 60 minutes become timeout candidates.
- Upload URL TTL starts at 15 minutes.
- Playback URL TTL starts at 5 minutes.
- `P0_MAX_UPLOAD_BYTES` starts at 50 MB.

Security and privacy controls:

- All audio-bearing object storage is private by default.
- Encrypt raw audio, canonical audio, generated preview, reference audio, and contact ciphertext at rest.
- Use short-lived signed access for upload and playback.
- Use least-privilege service access to object storage.
- Deny playback if consent is withdrawn, rights are blocked/expired/under review, deletion is running, audit fails, or artifact is blocked/deleted.
- Keep contact follow-up behind an explicit gate: authenticated encryption, keyed hash, key management service, audit write success, no plaintext UI, no CSV export.
- Use rights state and risk acceptance on every selection, job creation, engine request, and playback issuance path.

## Rollout / Rollback Plan

### Rollout

1. Add database tables and read models behind feature flags.
2. Implement consent records and job consent snapshot.
3. Implement song package and section package read path.
4. Implement recorder/take review UI and upload session path.
5. Implement `completeAudioUpload` and `conversion-job-state`.
6. Integrate `audio-ingest` and StageResult adapter.
7. Add minimal pitch and preview pipeline.
8. Add result playback, rating, and failure tags.
9. Add internal web surfaces for admin, review, governance.
10. Enable contact follow-up only after contact collection gate passes.
11. Run internal allowlist P0 with `Mist intro`.

### Rollback

Rollback must be operationally safe rather than only code rollback.

- Disable song package exposure through rights state or feature flag.
- Disable `completeAudioUpload` for new jobs.
- Keep existing job status readable.
- Block new playback URL issuance when rights/consent/audit/deletion issues occur.
- Retain deletion jobs and audit evidence even if UI is rolled back.
- Disable contact follow-up by turning off the contact collection gate.
- If engine pipeline is unstable, stop outbox publishing for new engine requests and preserve existing jobs as failed/blocked/needs_review with user-safe reasons.

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
- contact preference encryption/deletion validation

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
- contact follow-up disabled without key management gate.
- break-glass raw/canonical audio access disabled without second reviewer/security owner.

## Risks and Tradeoffs

- Keeping engines in their final-target composition preserves long-term architecture but increases perceived P0 scope. Mitigation: constrain P0 implementation depth and ticketing.
- `worker` as canonical job state owner avoids a new orchestrator service but can grow beyond simple queue responsibilities. Mitigation: keep `conversion-job-state` as a bounded module with clear tables and outbox.
- Presigned upload/playback reduces BFF file handling cost but adds object storage validation and signed URL leakage risk. Mitigation: short TTL, object HEAD verification, no signed URL logging.
- App-side audio normalization can reduce server cost but may vary by platform. Mitigation: app may pre-normalize only when output contract is reliable; ingest remains authoritative.
- Contact follow-up is useful for internal learning but introduces sensitive data handling. Mitigation: explicit opt-in, encrypted storage, gate, no plaintext UI, no CSV export.
- `partial_real` can accelerate validation but risks overstating product readiness. Mitigation: require machine-checkable user voice lineage and exclude mock from success metrics.

## Alternatives Considered

- Standalone Conversion Job Orchestrator: better long-term separation, not required for P0 because existing worker can host a bounded module.
- Temporal or Step Functions: strong workflow event history and long-running orchestration, deferred until P0 validates the preview loop and orchestration complexity justifies adoption.
- Kafka for engine pipeline: enterprise-grade stream processing, but heavier than needed for current P0. NATS JetStream remains the preferred P0 engine event stream.
- BFF-owned job state: rejected because BFF should not be source of truth for final processing states.
- Engine-owned final job state: rejected because different engines would compete over app-facing final state.
- Server-only upload path through BFF: rejected for cost and scalability; presigned direct object storage is preferred.
- Contact follow-up outside the product: rejected as the final P0 direction, but contact UI remains disabled if the encryption/key/audit gate is not ready.

## Open Technical Questions

- Which repository will own the database schema for song package, job state, consent, artifact, audit, deletion, and contact tables?
- Will API Gateway expose new internal contracts as gRPC proto first, or will some P0 contracts start as HTTP/internal module calls?
- What exact storage backend will P0 use for local/internal operation: MinIO, S3-compatible storage, or another object store?
- Can the app reliably export recorder takes as `.wav` or `.mp3` on every target platform, or must ingest normalize platform-native formats for all recordings?
- Which minimal `partial_real` or `real_synthesis` engine path will produce the first metric-eligible self-voice preview?
- What is the exact event schema for each engine stage before StageResult normalization?
- Does NATS JetStream already exist in every P0 runtime environment, or must TRD include provisioning work?
- Will internal web surfaces live in `vox2vocal-app` web routes or a separate internal web app?
- What KMS/Vault/OS keychain option will satisfy the contact collection gate for P0?
- Who is the second reviewer or security owner if break-glass or contact plaintext reveal ever becomes necessary?
- Where will rights evidence and risk acceptance be stored as source of truth before the admin web is implemented?
- What is the final MFA or step-up authentication requirement for admin/review/governance surfaces?

## References

- `vox2vocal-docs/pm/vox2vocal-mvp-prd.md`
- `vox2vocal-docs/pm/vox2vocal-mvp-feature-definition.md`
- `vox2vocal-docs/pm/vox2vocal-mvp-page-flow-plan.md`
- `vox2vocal-docs/engine/README.md`
- `vox2vocal-docs/engine/audio-ingest/README.md`
- `vox2vocal-docs/engine/logging/README.md`
- `vox2vocal-bff-server/README.md`
- `vox2vocal-api-gateway/proto/gateway.proto`
- `vox2vocal-worker/README.md`
- `vox2vocal-user-service/README.md`
