# Vox2Vocal MVP API / Data Contract Plan

문서 버전: v0.1
작성일: 2026-06-15
상태: 초안
적용 skill: `api-data-contract-planner`
기준 문서:

- `pm/vox2vocal-mvp-prd.md` v0.12
- `pm/vox2vocal-mvp-trd.md` v0.2
- `pm/vox2vocal-mvp-feature-definition.md` v0.5
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.3

## Behavior Supported

This contract plan supports the P0 internal flow:

```text
signup/login
-> song selection
-> section selection
-> recorder/take review
-> presigned upload
-> completeAudioUpload
-> job projection
-> partial-real preview artifact
-> signed playback
-> playback telemetry
-> preview rating/failure tags
-> deletion/consent settings
-> limited admin/review/governance surfaces
```

P0 contract principles:

- BFF GraphQL is app-facing. API Gateway/proto or internal service contracts are authoritative for service boundaries.
- BFF and API Gateway do not own canonical job state.
- Worker `conversion-job-state` owns P0 job projection, stage ledger, artifact refs, outbox, playback sessions, ratings, and deletion workflow.
- User Service owns identity, account consent, and disabled contact follow-up capability state.
- Client-provided consent, rights, risk, song package revision, and reference asset refs are not authoritative.
- All write APIs must define idempotency, validation, auth/permission, audit behavior, state transition effects, and user-safe error codes.
- Every public response includes `schema_version`.

## Shared Types

### Common Envelope

```ts
type ApiResponse<T> = {
  schema_version: string
  request_id: string
  trace_id: string
  data?: T
  error?: UserSafeError
}
```

```ts
type UserSafeError = {
  code:
    | "auth_required"
    | "invalid_credentials"
    | "email_already_exists"
    | "email_unverified"
    | "account_locked"
    | "role_denied"
    | "not_found"
    | "consent_missing"
    | "consent_outdated"
    | "rights_blocked"
    | "rights_under_review"
    | "deletion_running"
    | "audit_failed"
    | "upload_expired"
    | "upload_invalid"
    | "unsupported_format"
    | "duration_exceeded"
    | "idempotency_conflict"
    | "stage_failed"
    | "preview_unavailable"
    | "playback_blocked"
    | "rating_locked"
    | "unsupported_schema_version"
    | "validation_failed"
    | "internal_error"
  message: string
  retryable: boolean
  field_errors?: Array<{ field: string; code: string; message: string }>
}
```

### JobProjection

```ts
type JobProjection = {
  schema_version: string
  job_id: string
  user_id: string
  song_package_id: string
  section_id: string
  canonical_state:
    | "created"
    | "queued"
    | "processing"
    | "preview_ready"
    | "completed"
    | "failed"
    | "failed_with_partial_artifacts"
    | "blocked"
    | "needs_review"
    | "deleted"
  output_flags: {
    preview_available: boolean
    pitch_report_available: boolean
    quality_report_available: boolean
    section_limited: boolean
    playback_blocked: boolean
    deletion_pending: boolean
  }
  user_safe_reason?: string
  consent_snapshot_id: string
  consent_snapshot_hash: string
  rights_record_revision_id: string
  risk_acceptance_id?: string
  song_package_revision_id: string
  section_revision_id: string
  reference_asset_id: string
  stage_summaries: StageSummary[]
  artifact_summary: ArtifactSummary
  committed_at?: string
  timeout_at?: string
  terminal_at?: string
  blocked_at?: string
  created_at: string
  updated_at: string
}
```

### PreviewArtifact

```ts
type PreviewArtifact = {
  schema_version: string
  artifact_id: string
  job_id: string
  source_audio_asset_id: string
  source_audio_checksum: string
  render_artifact_ref: string
  pipeline_mode: "mock" | "partial_real" | "real_synthesis"
  generation_mode_version: string
  mock_fixture_used: boolean
  section_limited: boolean
  section_id: "intro" | string
  section_start_ms: number
  section_end_ms: number
  preview_duration_ms: number
  section_coverage_ratio: number
  lineage_verification_status: "verified" | "failed" | "unknown"
  lineage_source_event_id: string
  source_user_id: string
  consent_snapshot_id: string
  consent_snapshot_hash: string
  quality_status: "ok" | "warning" | "failed" | "needs_review"
  clipping_detected: boolean
  loudness_summary?: Record<string, number>
  artifact_warning_codes: string[]
  confidence_summary?: Record<string, number>
  playback_blocked: boolean
  rights_snapshot_id: string
  risk_acceptance_id?: string
  deletion_state?: "active" | "deletion_pending" | "deleted" | "deletion_failed"
  retention_deadline: string
}
```

Metric-eligible preview requires:

- `pipeline_mode in ["partial_real", "real_synthesis"]`
- `mock_fixture_used=false`
- `lineage_verification_status="verified"`
- `section_limited=true`
- `section_coverage_ratio >= 0.8`
- `playback_blocked=false`

### StageSummary

```ts
type StageSummary = {
  stage:
    | "upload_validation"
    | "audio_ingest"
    | "section_validation"
    | "target_pitch_mapping"
    | "user_pitch_extraction"
    | "preview_synthesis"
    | "render"
    | "preview_evaluation"
    | "safety_rights"
    | "retention_deletion"
  status: "queued" | "running" | "succeeded" | "failed" | "blocked" | "skipped"
  error_code?: string
  retryable?: boolean
  timing_ms?: number
  updated_at: string
}
```

## API Contracts

### signUp

- Method: GraphQL mutation `signUp`; internal User Service auth call.
- Auth / permission: anonymous.
- Request:

```ts
{
  email: string
  password: string
  display_name: string
  terms_version: string
  required_consent_acceptances?: ConsentAcceptance[]
}
```

- Response:

```ts
{
  user: { user_id: string; email: string; display_name: string; email_verified: boolean; roles: string[] }
  access_token?: string
  refresh_token?: string
  expires_at?: string
  next_step: "verify_email" | "enter_app"
}
```

- Validation:
  - normalized email must be unique and valid.
  - password minimum is 8 characters.
  - required consent items must be unbundled and versioned if collected at signup.
- Error cases: `validation_failed`, `email_already_exists`, `consent_missing`, `audit_failed`, `internal_error`.
- Compatibility:
  - Existing auth clients may omit P0 consent. Recorder entry must re-check consent currentness later.

### login

- Method: GraphQL mutation `login`.
- Auth / permission: anonymous.
- Request:

```ts
{ email: string; password: string }
```

- Response:

```ts
{
  user: { user_id: string; email: string; email_verified: boolean; roles: string[] }
  access_token: string
  refresh_token: string
  expires_at: string
}
```

- Validation:
  - email/password required.
  - failed attempts follow User Service lockout policy.
- Error cases: `auth_required`, `invalid_credentials`, `email_unverified`, `account_locked`, `internal_error`.
- Compatibility:
  - Token claims must include user id and role/status readable by BFF/API Gateway.

### me

- Method: GraphQL query `me`.
- Auth / permission: authenticated.
- Request: none.
- Response:

```ts
{
  user_id: string
  email: string
  display_name: string
  roles: Array<"learner" | "educator_or_expert" | "admin" | "engine_developer" | "product_qa" | "security_or_ops">
  email_verified: boolean
  account_status: "active" | "locked" | "blocked"
}
```

- Validation: token must be current and not revoked.
- Error cases: `auth_required`, `role_denied`, `account_locked`.
- Compatibility: add roles/status without breaking existing auth profile fields.

### getConsentStatus

- Method: GraphQL query `getConsentStatus`.
- Auth / permission: authenticated user.
- Request:

```ts
{ scope?: "account" | "p0_job"; song_package_id?: string; section_id?: string }
```

- Response:

```ts
{
  required: ConsentStatus[]
  optional: ConsentStatus[]
  current: boolean
  blocking_reason?: "missing" | "outdated" | "withdrawn"
}
```

- Validation:
  - required P0 consent types: `own_voice_processing`, `generated_preview`, `expert_review`, `retention_notice_ack`.
  - `candidate_data_opt_in` is optional.
  - `contact_for_followup` is unavailable in P0.
- Error cases: `auth_required`, `consent_outdated`, `internal_error`.
- Compatibility: old consent records remain valid only if type, version, scope, policy hash, and required/optional status match.

### grantConsent

- Method: GraphQL mutation `grantConsent`.
- Auth / permission: authenticated user, own account.
- Request:

```ts
{
  consent_type: string
  version: string
  scope: string
  policy_document_hash: string
  granted: boolean
}
```

- Response:

```ts
{ consent_record_id: string; current: boolean; granted_at: string }
```

- Validation:
  - consent must be unbundled.
  - cannot grant disabled `contact_for_followup` in P0.
  - policy hash must match server current policy.
- Error cases: `validation_failed`, `consent_outdated`, `audit_failed`, `internal_error`.
- Compatibility: additive consent types allowed; unknown required type blocks P0 job creation.

### getContactFollowupCapability

- Method: GraphQL query `getContactFollowupCapability`.
- Auth / permission: authenticated user.
- Request: none.
- Response:

```ts
{
  enabled: false
  reason: "disabled_in_p0"
  allowed_channels: []
  allowed_purposes: []
}
```

- Validation: always disabled in P0.
- Error cases: `auth_required`, `internal_error`.
- Compatibility: future versions may add enabled states behind explicit gate; P0 clients must treat missing/disabled as no UI.

### listSongPackages

- Method: GraphQL query `listSongPackages`.
- Auth / permission: authenticated learner/admin/reviewer according to surface.
- Request:

```ts
{ surface: "learner" | "admin" | "review"; include_blocked?: boolean }
```

- Response:

```ts
{
  packages: Array<{
    song_package_id: string
    revision_id: string
    title: string
    artist: string
    language: string
    default_section_id: string
    rights_state: "published" | "unlicensed_internal_risk_accepted" | "rights_pending" | "rights_blocked" | "under_review" | "retired"
    exposure_decision: "selectable" | "blocked" | "internal_allowlist_only"
    user_safe_reason?: string
  }>
}
```

- Validation:
  - learner surface cannot return selectable packages unless rights/risk state allows the user.
  - `Mist` P0 requires reference asset, section map, source/provenance, rights/risk record, retention, and deletion owner.
- Error cases: `auth_required`, `role_denied`, `rights_blocked`, `audit_failed`.
- Compatibility: add fields without changing exposure decision enum semantics.

### getSongPackage

- Method: GraphQL query `getSongPackage`.
- Auth / permission: authenticated user with package visibility.
- Request:

```ts
{ song_package_id: string }
```

- Response:

```ts
{
  song_package_id: string
  revision_id: string
  title: string
  artist: string
  language: string
  bpm: number
  key: string
  rights_state: string
  default_section_id: string
  reference_asset_id: string
  reference_asset_checksum: string
  risk_acceptance_id?: string
}
```

- Validation: package must not be retired or blocked for caller surface.
- Error cases: `rights_blocked`, `role_denied`, `not_found`.
- Compatibility: provider metadata remains optional in P0.

### listSongSections

- Method: GraphQL query `listSongSections`.
- Auth / permission: authenticated user with package visibility.
- Request:

```ts
{ song_package_id: string }
```

- Response:

```ts
{
  sections: Array<{
    section_id: string
    revision_id: string
    label: string
    start_ms: number
    end_ms: number
    duration_ms: number
    selectable: boolean
    user_safe_reason?: string
  }>
}
```

- Validation:
  - P0 default selectable section is `intro` `0:00-0:28`.
  - actual registered reference asset checksum/duration must be recorded before learner exposure.
- Error cases: `not_found`, `rights_blocked`, `validation_failed`.
- Compatibility: more sections can be added without changing the separate section selection flow.

### getSelectedSectionGuide

- Method: GraphQL query `getSelectedSectionGuide`.
- Auth / permission: authenticated learner.
- Request:

```ts
{ song_package_id: string; section_id: string }
```

- Response:

```ts
{
  song_package_id: string
  section_id: string
  section_revision_id: string
  label: string
  start_ms: number
  end_ms: number
  expected_duration_ms: number
  bpm: number
  key: string
  reference_prelisten_allowed: boolean
  lyrics_display_allowed: boolean
  lyrics_sync_allowed: boolean
  restrictions: string[]
  rights_state: string
}
```

- Validation: rights state, section state, and risk acceptance must be current.
- Error cases: `rights_blocked`, `rights_under_review`, `audit_failed`, `not_found`.
- Compatibility: lyrics payload is omitted in P0 unless rights flag allows.

### createAudioUploadSession

- Method: GraphQL mutation `createAudioUploadSession`; internal storage session creation.
- Auth / permission: authenticated learner, own job.
- Request:

```ts
{
  original_filename: string
  content_type: "audio/wav" | "audio/x-wav" | "audio/wave" | "audio/vnd.wave" | "audio/mpeg" | "audio/mp3"
  content_length: number
  song_package_id: string
  section_id: string
  source_type: "recorder_take" | "fallback_upload"
}
```

- Response:

```ts
{
  upload_session_id: string
  object_key: string
  presigned_put_url: string
  expires_at: string
  max_bytes: 52428800
  accepted_content_types: string[]
}
```

- Validation:
  - authenticated and email verified.
  - selected song/section visible to user.
  - required consent currentness may be checked here but must be rechecked in `completeAudioUpload`.
  - content length > 0 and <= 50 MB.
  - fallback upload accepts `.wav`/`.mp3` only.
- Error cases: `auth_required`, `email_unverified`, `rights_blocked`, `consent_missing`, `upload_invalid`, `unsupported_format`, `audit_failed`.
- Compatibility: app recorder may submit platform-native format only if issued content type is accepted by current ingest contract.

### completeAudioUpload

- Method: GraphQL mutation `completeAudioUpload`; internal worker job creation command.
- Auth / permission: authenticated learner, owner of upload session.
- Request:

```ts
{
  upload_session_id: string
  idempotency_key: string
  song_package_id: string
  section_id: string
  take_id?: string
}
```

- Response:

```ts
{
  job: JobProjection
  audio_asset_id: string
  outbox_event_id: string
}
```

- Validation:
  - upload session owner matches user.
  - upload session not expired.
  - object HEAD exists, object key matches issued key, object size > 0 and <= 50 MB.
  - issued content type matches normalized content type.
  - server computes consent snapshot, rights/risk snapshot, song package revision, section revision, BPM/key snapshot, reference asset snapshot.
  - audit allow decision must succeed.
  - same idempotency key and same payload returns existing projection.
  - same idempotency key with conflicting payload returns `idempotency_conflict`.
- Error cases: `upload_expired`, `upload_invalid`, `consent_missing`, `consent_outdated`, `rights_blocked`, `rights_under_review`, `audit_failed`, `idempotency_conflict`.
- Compatibility: no client-supplied consent or rights refs are accepted.

### getJobStatus

- Method: GraphQL query `getJobStatus`.
- Auth / permission: learner owns job; reviewer/admin according to role and consent.
- Request:

```ts
{ job_id: string }
```

- Response: `JobProjection`.
- Validation: BFF reads canonical worker projection only.
- Error cases: `not_found`, `role_denied`, `deletion_running`, `internal_error`.
- Compatibility: unknown stage enum maps to user-safe `processing` or `needs_review`.

### subscribeJobStatus

- Method: GraphQL subscription `subscribeJobStatus`.
- Auth / permission: same as `getJobStatus`.
- Request:

```ts
{ job_id: string; since_revision?: number }
```

- Response:

```ts
{ job: JobProjection; projection_revision: number }
```

- Validation: caller permission checked on connect and on each event.
- Error cases: `auth_required`, `role_denied`, `not_found`.
- Compatibility: clients must fall back to polling `getJobStatus`.

### getJobResult

- Method: GraphQL query `getJobResult`.
- Auth / permission: learner owns job; reviewer/admin with consent.
- Request:

```ts
{ job_id: string }
```

- Response:

```ts
{
  job: JobProjection
  preview_artifact?: PreviewArtifact
  pitch_report?: ArtifactSummary
  quality_report?: ArtifactSummary
  rating_status: { rating_required: boolean; rating_submitted: boolean; failure_tags_required: boolean }
}
```

- Validation:
  - deleted or blocked artifacts are not returned as playable.
  - pitch-only success cannot unlock primary self-voice rating.
- Error cases: `preview_unavailable`, `playback_blocked`, `role_denied`, `deletion_running`.
- Compatibility: response can omit optional reports.

### issuePreviewPlaybackUrl

- Method: GraphQL mutation `issuePreviewPlaybackUrl`.
- Auth / permission: learner owns job; reviewer allowed only for consented reviewable job.
- Request:

```ts
{ job_id: string; artifact_id: string }
```

- Response:

```ts
{
  playback_session_id: string
  signed_get_url: string
  expires_at: string
  preview_duration_ms: number
}
```

- Validation:
  - preview artifact exists and is not deleted/blocked.
  - rights, consent, audit, deletion state allow playback.
  - signed URL TTL starts at 5 minutes.
  - audit write success is required before returning URL.
- Error cases: `preview_unavailable`, `playback_blocked`, `rights_blocked`, `rights_under_review`, `consent_outdated`, `deletion_running`, `audit_failed`.
- Compatibility: already issued URLs cannot be reliably revoked; later blocking state denies new URLs and rating unlock.

### recordPreviewPlaybackEvent

- Method: GraphQL mutation `recordPreviewPlaybackEvent`.
- Auth / permission: owner/reviewer of playback session.
- Request:

```ts
{
  playback_session_id: string
  artifact_id: string
  event_id: string
  client_sequence: number
  schema_version: string
  occurred_at: string
  event_type: "playback_started" | "playback_progress" | "playback_paused" | "playback_seeked" | "playback_ended" | "playback_error"
  position_ms: number
  duration_ms: number
  muted: boolean
  app_foreground: boolean
  played_range_start_ms?: number
  played_range_end_ms?: number
  error_code?: string
}
```

- Response:

```ts
{
  playback_session_id: string
  unique_timeline_coverage_ms: number
  unique_timeline_coverage_ratio: number
  preview_played: boolean
  rating_unlocked: boolean
}
```

- Validation:
  - event tuple must match authenticated user, playback session, artifact, and job.
  - `occurred_at` must be inside session validity window or accepted within flush grace.
  - played range must satisfy `0 <= start < end <= preview_duration_ms`.
  - muted or background playback does not increase metric-eligible coverage.
  - duplicated `event_id` is idempotent only if payload is identical.
  - impossible progress is ignored or rejected.
  - latest consent/rights/deletion/audit/artifact state is rechecked before rating unlock.
- Error cases: `auth_required`, `validation_failed`, `playback_blocked`, `unsupported_schema_version`, `rating_locked`.
- Compatibility: clients may send more frequent progress, but server computes coverage from distinct ranges only.

### submitPreviewRating

- Method: GraphQL mutation `submitPreviewRating`.
- Auth / permission: learner owns job. Reviewer cannot submit learner primary rating.
- Request:

```ts
{
  job_id: string
  artifact_id: string
  playback_session_id: string
  rating: 1 | 2 | 3 | 4 | 5
  rating_prompt_version: string
}
```

- Response:

```ts
{
  rating_id: string
  failure_tags_required: boolean
  self_voice_success_candidate: boolean
}
```

- Validation:
  - `preview_played=true` must be computed by server.
  - playback session must belong to same user/job/artifact.
  - latest consent/rights/deletion/audit/artifact state must still allow rating.
  - rating 1-3 requires failure tags before rating flow is complete.
- Error cases: `rating_locked`, `playback_blocked`, `validation_failed`, `audit_failed`.
- Compatibility: prompt version is required so future wording changes do not mix metrics.

### submitFailureTags

- Method: GraphQL mutation `submitFailureTags`.
- Auth / permission: learner owns rating/job.
- Request:

```ts
{
  job_id: string
  artifact_id: string
  rating_id: string
  tags: Array<"not_my_voice" | "not_song_like" | "pitch_wrong" | "timing_wrong" | "robotic_or_artifact" | "noise_or_clipping" | "too_short_or_incomplete" | "other">
  other_text?: string
}
```

- Response:

```ts
{ submitted: true; failure_tag_ids: string[] }
```

- Validation:
  - rating below 4 requires at least one tag or `other`.
  - `other_text` length capped and screened for sensitive free-text handling.
  - no contact values should be requested or encouraged.
- Error cases: `validation_failed`, `role_denied`, `audit_failed`.
- Compatibility: taxonomy version must be stored.

### reportPlaybackProblem

- Method: GraphQL mutation `reportPlaybackProblem`.
- Auth / permission: authenticated user with job/artifact access.
- Request:

```ts
{ job_id: string; artifact_id: string; playback_session_id?: string; reason: string }
```

- Response:

```ts
{ submitted: true; event_id: string }
```

- Validation: not counted as low preview quality rating.
- Error cases: `validation_failed`, `role_denied`.
- Compatibility: playback problem taxonomy can evolve separately from failure tags.

### requestDeletion

- Method: GraphQL mutation `requestDeletion`.
- Auth / permission: learner owns job/artifact; admin/security ops for governance cases.
- Request:

```ts
{ job_id?: string; artifact_id?: string; reason: "user_request" | "consent_withdrawal" | "rights_complaint" | "admin_policy" }
```

- Response:

```ts
{ deletion_request_id: string; status: "scheduled" | "running"; playback_blocked: true }
```

- Validation:
  - at least one target id required.
  - playback blocked immediately for voice-bearing or rights-sensitive artifacts.
  - deletion evidence must not contain raw audio or playable preview.
- Error cases: `not_found`, `role_denied`, `audit_failed`, `deletion_running`.
- Compatibility: deletion can be requested even if job is already terminal.

### getDeletionStatus

- Method: GraphQL query `getDeletionStatus`.
- Auth / permission: requester owner or governance role.
- Request:

```ts
{ deletion_request_id: string }
```

- Response:

```ts
{
  deletion_request_id: string
  status: "scheduled" | "running" | "completed" | "failed"
  evidence_refs: string[]
  user_safe_reason?: string
}
```

- Validation: evidence refs are metadata only.
- Error cases: `not_found`, `role_denied`.
- Compatibility: evidence schema is additive.

### createSongPackage

- Method: GraphQL mutation `createSongPackage`.
- Auth / permission: `admin`.
- Request:

```ts
{
  title: string
  artist: string
  language: string
  bpm: number
  key: string
  default_section_id: string
  section_map: Array<{ section_id: string; label: string; start_ms: number; end_ms: number }>
}
```

- Response:

```ts
{ song_package_id: string; revision_id: string; status: "draft" | "metadata_incomplete" }
```

- Validation:
  - P0 package must be Ken Kamikita - `Mist` unless feature flag permits another package.
  - section map must include `intro` `0:00-0:28` before learner exposure.
  - publish requires reference asset, rights/risk record, retention, deletion owner.
- Error cases: `role_denied`, `validation_failed`, `audit_failed`.
- Compatibility: provider ids/artwork optional in P0.

### uploadReferenceAudioSession

- Method: GraphQL mutation `uploadReferenceAudioSession`.
- Auth / permission: `admin`.
- Request:

```ts
{ song_package_id: string; content_type: string; content_length: number; original_filename: string }
```

- Response:

```ts
{ upload_session_id: string; object_key: string; presigned_put_url: string; expires_at: string }
```

- Validation:
  - admin only.
  - source/provenance must be recorded before package can be exposed.
  - uploaded reference asset checksum and duration must be stored after ingest.
- Error cases: `role_denied`, `upload_invalid`, `audit_failed`.
- Compatibility: same S3-compatible storage contract as user upload.

### updateRightsState

- Method: GraphQL mutation `updateRightsState`.
- Auth / permission: `admin` plus policy/right owner metadata; audit required.
- Request:

```ts
{
  song_package_id: string
  rights_state: "rights_pending" | "unlicensed_internal_risk_accepted" | "rights_blocked" | "under_review" | "retired" | "published"
  evidence_ref: string
  allowed_users_or_groups?: string[]
  allowed_section_ids?: string[]
  prohibited_uses: string[]
  re_review_deadline: string
  kill_switch_owner: string
}
```

- Response:

```ts
{ rights_record_revision_id: string; exposure_decision: string; audit_id: string }
```

- Validation:
  - `published` requires valid rights clearance evidence.
  - `unlicensed_internal_risk_accepted` requires allowlist, section, duration, prohibited uses, kill-switch owner, re-review deadline.
  - audit write failure fails closed.
- Error cases: `role_denied`, `validation_failed`, `audit_failed`, `rights_blocked`.
- Compatibility: rights state changes must immediately block new selection/job/playback where applicable.

### listReviewableJobs

- Method: GraphQL query `listReviewableJobs`.
- Auth / permission: `educator_or_expert`, `product_qa`, `engine_developer` with scoped access.
- Request:

```ts
{ status?: string[]; low_confidence?: boolean; failure_tag?: string; cursor?: string; limit?: number }
```

- Response:

```ts
{ jobs: JobProjection[]; next_cursor?: string }
```

- Validation:
  - only jobs with expert review consent are visible.
  - raw audio access is not included by default.
- Error cases: `role_denied`, `consent_missing`, `internal_error`.
- Compatibility: cursor pagination required before list grows.

### getReviewDetail

- Method: GraphQL query `getReviewDetail`.
- Auth / permission: review role and job consent.
- Request:

```ts
{ job_id: string }
```

- Response:

```ts
{
  job: JobProjection
  preview_artifact?: PreviewArtifact
  pitch_report?: ArtifactSummary
  quality_report?: ArtifactSummary
  user_failure_tags: string[]
  technical_tags: string[]
}
```

- Validation: raw audio direct URL excluded unless a later break-glass path exists.
- Error cases: `role_denied`, `consent_outdated`, `playback_blocked`.
- Compatibility: review detail can omit preview playback URL; reviewers request URL separately.

### submitTechnicalTags

- Method: GraphQL mutation `submitTechnicalTags`.
- Auth / permission: `educator_or_expert`, `engine_developer`, `product_qa`.
- Request:

```ts
{ job_id: string; tags: string[]; note?: string; taxonomy_version: string }
```

- Response:

```ts
{ submitted: true; technical_tag_ids: string[] }
```

- Validation:
  - technical tags are stored separately from user perception tags.
  - sensitive free text must be minimized.
- Error cases: `role_denied`, `validation_failed`, `audit_failed`.
- Compatibility: taxonomy version required.

## Data Model Changes

### User Service Owned

- `users`: add/confirm role/status fields and email verification status.
- `consent_records`:
  - `id`, `user_id`, `consent_type`, `version`, `scope`, `required`, `policy_document_version`, `policy_document_hash`, `granted_at`, `withdrawn_at`, `source_session_id`, `source_job_id`.
  - No P0 contact value columns.
- `contact_followup_capability`:
  - `user_id`, `enabled=false`, `reason=disabled_in_p0`, `updated_at`.
  - P0 stores capability state only, not contact values.

### Worker `conversion-job-state` Owned

- `song_packages`: metadata, status, default section, BPM/key, owner fields.
- `section_maps`: section id, timestamp, duration, label, revision.
- `reference_assets`: object key, checksum, duration, provenance, uploader, retention deadline.
- `rights_records`: rights state, allowed/prohibited uses, evidence ref, approver, expiry/re-review, complaint owner.
- `risk_acceptance_records`: allowlist, sections, duration, kill-switch owner, re-review deadline.
- `upload_sessions`: object key, owner, content type, expiry, selected song/section, source type.
- `audio_assets`: source object, checksum, duration, owner, canonical object ref after ingest.
- `job_consent_snapshots`: immutable consent/policy refs.
- `conversion_jobs`: canonical state, snapshots, source asset, selected section, terminal metadata.
- `stage_results`: normalized stage ledger.
- `artifact_refs`: storage pointer, data class, retention, deletion, playback eligibility.
- `preview_artifacts`: lineage and preview-specific metadata, or equivalent typed artifact row.
- `playback_sessions`: signed URL audit refs and unique timeline coverage.
- `playback_events`: event id, sequence, range, muted/foreground/error.
- `ratings`: primary rating, prompt version, metric eligibility.
- `failure_tags`, `technical_tags`.
- `deletion_requests`, `deletion_evidence`.
- `outbox_events`: engine request publishing.
- `audit_records`: allow/deny and governance actions.

## Migration Plan

1. Add nullable/additive tables for song package, section map, rights/risk, upload sessions, jobs, stage results, artifact refs, playback sessions/events, ratings, tags, deletion evidence, and outbox.
2. Add User Service consent records and disabled contact follow-up capability state.
3. Add DB roles/grants so BFF has read-only projection access, worker writes conversion schema, User Service writes user/consent schema, engines write no job tables.
4. Deploy read paths and disabled feature flags.
5. Backfill no P0 rows except seed/admin creation for `Mist`.
6. Add unique constraints after initial write paths are stable.
7. Tighten not-null constraints only after internal P0 smoke tests.
8. Keep rollback as feature flag/outbox pause/read-only fallback. Do not drop P0 tables or audit/deletion evidence.

Backward compatibility:

- Existing auth API continues to work without P0 consent at login.
- Existing app clients that do not understand P0 job APIs cannot create jobs.
- New enum values must fall back to safe user-facing states.
- Additive schema changes only during P0.

## Indexing / Performance Notes

- `users(email_normalized)` unique.
- `consent_records(user_id, consent_type, version, scope, withdrawn_at)`.
- `song_packages(status, default_section_id)`.
- `section_maps(song_package_id, section_id, revision_id)` unique.
- `reference_assets(song_package_id, checksum)`.
- `rights_records(song_package_id, rights_state, re_review_deadline)`.
- `upload_sessions(upload_session_id, owner_user_id, expires_at)` and `(idempotency_key)`.
- `audio_assets(owner_user_id, checksum, created_at)`.
- `conversion_jobs(user_id, canonical_state, created_at)` and `(upload_session_id)` unique.
- `stage_results(job_id, stage, attempt)` unique.
- `outbox_events(status, next_attempt_at, event_type)`.
- `artifact_refs(job_id, data_class, playback_allowed, retention_deadline)`.
- `preview_artifacts(job_id, pipeline_mode, mock_fixture_used, section_id)`.
- `playback_sessions(playback_session_id, artifact_id, user_id, expires_at)`.
- `playback_events(playback_session_id, event_id)` unique and `(playback_session_id, client_sequence)`.
- `ratings(job_id, artifact_id, user_id, rating_prompt_version)`.
- `deletion_requests(status, requested_at)`.
- `audit_records(trace_id, created_at)` and `(actor_user_id, action, created_at)`.

Performance defaults:

- P0 global conversion concurrency: 2 active jobs.
- Heavy preview/render concurrency: 1 active job.
- Upload TTL: 15 minutes.
- Playback URL TTL: 5 minutes.
- Job timeout: 60 minutes from `completeAudioUpload.committed_at`.

## Events / Analytics

### Domain Events

- `audio.ingest.requested`
- `audio.ingest.started`
- `audio.ingest.completed`
- `audio.ingest.failed`
- `audio.ingest.dead_letter`
- `job.created`
- `job.state_changed`
- `stage_result.upserted`
- `preview_artifact.created`
- `playback_url.issued`
- `playback_url.denied`
- `preview_played.reached`
- `rating.submitted`
- `failure_tags.submitted`
- `deletion.requested`
- `deletion.completed`
- `rights_state.changed`

### Analytics Events

- `auth_signup_completed`
- `auth_login_completed`
- `song_selected`
- `section_selected`
- `recorder_ready`
- `take_recorded`
- `take_submitted`
- `upload_completed`
- `complete_audio_upload_committed`
- `job_processing_started`
- `preview_ready`
- `preview_playback_started`
- `preview_played_threshold_reached`
- `preview_rating_submitted`
- `failure_tags_submitted`
- `playback_problem_reported`
- `consent_withdrawn`
- `deletion_requested`
- `rights_blocked`

Analytics payload rules:

- Include `trace_id`, `job_id`, `artifact_id`, `song_package_id`, `section_id`, `pipeline_mode`, `schema_version`, and pseudonymous user id where appropriate.
- Do not include raw audio, signed URLs, full lyrics, access tokens, contact values, or sensitive free-text.
- Rating analytics must include `rating_prompt_version` and `preview_played=true`.
- Failure tag analytics must include taxonomy version.

## Open Contract Questions

- What exact proto package and service names will API Gateway use for P0 job, song package, playback, review, and governance contracts?
- What is the exact downstream engine event schema for `voice-pitch`, `target_pitch_mapping`, `preview_synthesis`, `render`, `preview_evaluation`, and `safety_rights` before StageResult normalization?
- What is the flush grace window for late playback progress events?
- What tolerance should server use for impossible playback progress versus wall-clock elapsed time?
- Who is the temporary P0 rights/risk approver for `unlicensed_internal_risk_accepted`, and where is the evidence ref stored operationally?
- What SLA should apply to `needs_review` jobs before they become failed, blocked, or manually released?
- Should `PreviewArtifact` be a separate physical table or a typed row in `artifact_refs` with JSON metadata for P0?
