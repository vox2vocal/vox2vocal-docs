# Vox2Vocal MVP Page / Flow Plan

문서 버전: v0.3
작성일: 2026-06-13
기준 문서: `pm/vox2vocal-mvp-feature-definition.md` v0.5
적용 skill: `page-flow-planner`

## Flow Summary

Vox2Vocal P0의 핵심 흐름은 접근 권한이 있는 학습자가 `Mist` 곡을 선택하고, `intro` section을 선택한 뒤, 앱에서 본인 목소리 take를 녹음하고, 앱 안에서 section-limited self-voice preview를 재생해 "내 목소리처럼 들린다"를 평가하는 것이다.

P0 화면은 네 surface로 나눈다.

- Learner app: access eligibility, 곡 선택, section 선택, consent, recording/take review, fallback upload, processing, preview/result, rating, deletion request
- Limited admin: reference audio upload, song package, section map, rights/risk exposure control
- Educator/expert review: 동의된 job의 preview, pitch report, failure tags, quality summary 검토
- Internal governance: audit/deletion evidence, contact follow-up disabled capability, break-glass disabled 상태 확인

P0는 시각 mockup을 만들지 않는다. 이 문서는 화면 목적, 데이터, 상태, CTA, navigation, feature mapping을 정리해 `spec-to-tickets`로 넘기는 기준 문서다.

## Page Inventory

### 1. 로그인 / 회원가입 (Login / Signup)

- Purpose: 사용자가 본인 계정으로 P0 surface에 접근할 수 있게 한다.
- Primary action: 이메일/비밀번호로 로그인하거나 회원가입 후 이메일 인증을 완료한다.
- Secondary actions: 약관 확인, 세션 만료 후 재로그인, role별 접근 불가 안내 확인.
- Required data:
  - email, password, display name
  - email verification status
  - user id, role, account status
  - terms consent version
  - required P0 consent versions and scope
- Key components:
  - login form
  - signup form
  - email verification status banner
  - session expired prompt
  - role/permission notice
- Empty state: 신규 사용자는 회원가입 CTA를 본다.
- Loading state: 로그인/회원가입 요청 중 form submit button을 disabled 처리하고 진행 상태를 표시한다.
- Error state:
  - invalid credentials
  - email not verified
  - session expired
  - account locked or blocked
- Permission state:
  - `learner`: learner flow 진입
  - `admin`, `educator_or_expert`, `engine_developer`, `product_qa`, `security_or_ops`: 역할별 surface로 이동
  - role이 없는 내부 계정은 learner보다 넓은 접근을 받지 않는다.
- Navigation:
  - From: app entry, protected route redirect
  - To: Access Eligibility / Song Selection, Limited Admin, Review Queue, Governance Evidence
- Success signal:
  - client/backend 인증 연동 오류가 시도 대비 2% 미만
  - 인증된 사용자가 role에 맞는 첫 화면으로 이동

### 2. Access Eligibility / Song Selection

- Purpose: 접근 권한이 있는 학습자가 `Mist` song package를 선택할 수 있는지 확인하고, 권리/노출 상태에 따라 안전하게 진입시킨다.
- Primary action: `Mist` song을 선택한다.
- Secondary actions: song metadata 확인, 이용 제한 조건 확인, 접근 불가 사유 확인, 다른 곡 선택 안내.
- Required data:
  - user id, role, internal eligibility flag
  - song package id, title, artist, language
  - rights state: `published`, `unlicensed_internal_risk_accepted`, `rights_blocked`, `under_review`, `retired`
  - reference pre-listen and lyrics display defaults
  - feature flag for learner BPM/key correction
  - risk acceptance exposure decision
- Key components:
  - access eligibility banner
  - song card for `Mist`
  - default section teaser
  - rights/availability status message
  - disabled state reason panel
- Empty state: 사용 가능한 곡이 없으면 "현재 선택 가능한 곡이 없음"을 보여준다.
- Loading state: song package와 eligibility를 불러오는 동안 skeleton 또는 compact loading state를 표시한다.
- Error state:
  - package metadata load failure
  - stale cached package blocked by latest rights state
  - audit allow decision failure
- Permission state:
  - 일반 learner는 `published` package만 선택 가능
  - 내부 allowlist에 포함된 사용자만 `unlicensed_internal_risk_accepted` package 선택 가능
  - `rights_blocked`, `under_review`, `retired` 상태는 selection, processing, playback 진입 차단
- Navigation:
  - From: Login / Signup, completed/deleted job에서 새 job 생성
  - To: Section Selection / Part Selection
  - Blocked To: 다른 곡 선택 안내 또는 문의
- Success signal:
  - learners는 `rights_blocked` package를 선택할 수 없음
  - 100%의 song exposure decision이 provenance, allowed use, retention, deletion owner를 포함

### 3. Section Selection / Part Selection

- Purpose: 선택한 곡 안에서 녹음하고 처리할 target part를 고른다. P0에서는 `intro` 하나만 선택 가능해도 이 단계를 유지한다.
- Primary action: `intro 0:00-0:28` section을 선택한다.
- Secondary actions: section label, timestamp, expected duration, recording focus, 권리/가이드 가능 여부 확인.
- Required data:
  - selected song package id
  - section id, label, start/end timestamp, expected duration
  - BPM/key default
  - reference pre-listen flag/scope
  - lyrics display flag/scope
  - lyrics sync flag
  - rights state and risk acceptance exposure decision
- Key components:
  - section list or single-section selector
  - section timeline row
  - expected recording length indicator
  - guide availability badges for reference pre-listen and lyrics
  - disabled state reason panel
- Empty state: 선택 가능한 section이 없으면 곡 선택으로 되돌린다.
- Loading state: section map and guide flags loading.
- Error state:
  - section map load failure
  - selected section retired or blocked
  - rights state changed after song selection
- Permission state:
  - `rights_blocked`, `under_review`, `retired` section은 recording 진입 차단
  - reference/lyrics guide가 blocked여도 section recording은 계속 가능
- Navigation:
  - From: Access Eligibility / Song Selection
  - To: Record Take / Take Review
  - Back To: Song Selection
- Success signal:
  - song selected -> section selected funnel event가 분리 추적됨
  - P0 job의 100%가 selected song id와 section id를 가짐

### 4. Record Take / Take Review

- Purpose: 학습자가 선택한 section 기준으로 앱에서 본인 목소리 take를 녹음하고, 제출 전 자기 take를 확인한다. BPM/key와 필수 동의 확인은 별도 사용자-visible page가 아니라 recorder 진입 gate와 제출 전 검증으로 처리한다. Fallback upload는 recorder를 사용할 수 없거나 내부 운영에서 enabled일 때만 제공한다.
- Primary action: 본인 목소리 take를 녹음하고 replay 후 processing 시작을 요청한다.
- Secondary actions: BPM/key snapshot 확인 또는 feature flag 기반 수정, consent 상세 보기와 재동의, optional candidate data consent, reference pre-listen before recording when allowed, lyric cue 확인 when allowed, retake, fallback upload, format/length guide 확인.
- Required data:
  - selected song package and section
  - BPM/key default and learner override value
  - consent policy document version/hash, consent scope, required/optional status
  - consent snapshot eligibility, snapshot hash
  - consent types: `own_voice_processing`, `generated_preview`, `expert_review`, `retention_notice_ack`, `candidate_data_opt_in`
  - no capture/redistribution/public posting agreement for reference audio, lyrics, generated preview, and app output
  - disabled contact follow-up capability status
  - recorder session id
  - mic permission status
  - local take id
  - upload session id
  - presigned upload URL TTL
  - accepted MIME and extension
  - `P0_MAX_UPLOAD_BYTES`
  - selected song package id, target section id
  - server-computed consent snapshot id/hash after submit
  - server-computed rights/risk snapshot id/hash after submit
  - idempotency key
- Key components:
  - section guide header
  - recorder entry gate for current consent, BPM/key, and usage restriction checks
  - required consent checklist or compact current-consent summary
  - optional consent section
  - reference pre-listen player before recording when allowed
  - permitted lyric cue when allowed
  - mic permission prompt
  - count-in
  - recording timer
  - input level meter
  - record/stop controls
  - own-take replay player
  - retake CTA
  - fallback file picker/dropzone when enabled
  - upload progress indicator
  - selected take/file summary
  - take validation result
  - "처리 시작" CTA mapped to `completeAudioUpload`
- Empty state: selected song/section이 없으면 Section Selection으로 되돌린다. Recorder 진입 후에는 아직 녹음된 take 없음.
- Loading state:
  - consent policy and song setup data loading
  - mic permission requesting
  - recorder initializing
  - upload session creating
  - take/file uploading
  - `completeAudioUpload` committing
- Error state:
  - consent policy load failure
  - consent policy document, version, scope, or required/optional status outdated
  - rights state changed before recording or commit
  - contact follow-up capability disabled
  - mic permission denied
  - microphone unavailable
  - recorder failed
  - unsupported format
  - duration exceeded after ingest
  - silence or no voice detected
  - clipping or too-low volume warning
  - file too large
  - upload URL expired
  - missing object after upload
  - audit failure
- Permission state:
  - signed-out/session expired 사용자는 재인증 필요
  - non-self voice suspicion은 P0에서 자동 판정하지 않지만 본인 음성 확인 동의 없이는 차단
  - expert review consent를 거부하면 P0 preview job은 시작하지 않지만 계정 사용은 유지
  - contact follow-up capability는 P0에서 disabled이며 contact opt-in UI는 숨김
  - feature flag가 꺼진 사용자는 BPM/key correction read-only
  - learner reference audio upload는 허용하지 않음
  - reference pre-listen과 lyrics display는 권리 플래그가 허용한 경우만 노출
  - active microphone recording 중 reference audio playback은 중지
- Navigation:
  - From: Section Selection / Part Selection
  - To: Processing Status after `completeAudioUpload`
  - Back To: Section Selection
- Success signal:
  - valid recording/fallback upload의 100%가 consent snapshot과 source asset id를 가짐
  - 필수 동의가 없으면 engine processing에 도달하지 않음
  - `completeAudioUpload`가 audio asset, job id, initial StageResult, outbox event를 idempotent하게 기록
  - unauthorized or non-self voice job이 analysis, preview, synthesis에 도달하지 않음
  - preventable rejection rate for silence/clipping/too-short/too-long can be measured before engine processing

### 5. Processing Status

- Purpose: 장기 처리 중 사용자가 현재 job 상태와 다음 행동을 이해하게 한다.
- Primary action: job 상태를 확인하고, preview가 준비되면 결과로 이동한다.
- Secondary actions: 상태 새로고침, 실패 사유 확인, 재시도/재업로드, 문의, 삭제 요청.
- Required data:
  - job id
  - canonical state: `created`, `queued`, `processing`, `preview_ready`, `completed`, `failed`, `failed_with_partial_artifacts`, `blocked`, `needs_review`, `deleted`
  - output flags: `preview_available`, `pitch_report_available`, `section_limited`, `playback_blocked`, `deletion_pending`
  - stage summaries and user-safe reason
  - timeout deadline
- Key components:
  - state label
  - stage progress summary
  - expected wait or observation timing
  - primary CTA by state
  - blocked/failed reason card
- Empty state: active job이 없으면 Song Selection으로 이동할 CTA를 제공한다.
- Loading state: job projection loading, state polling/subscription connecting.
- Error state:
  - job not found
  - projection unavailable
  - timeout
  - duplicate/late event handled but user projection not available
- Permission state:
  - learner는 본인 job만 조회
  - educator/expert는 동의된 job만 조회
  - deletion 이후 playable output 제공 안 함
- Navigation:
  - From: Record Take / Take Review
  - To: Result / Preview when `preview_ready` or `completed`
  - To: Song Selection for retry/new job
- Success signal:
  - 100%의 P0 jobs가 final state와 output availability flags를 가짐
  - BFF는 canonical projection을 읽고 final state를 독자적으로 결정하지 않음

### 6. Result / Preview / Rating

- Purpose: self-voice section preview를 가장 먼저 들려주고, pitch feedback과 quality summary를 확인한 뒤 primary rating과 failure tags를 수집한다.
- Primary action: preview를 재생한다.
- Secondary actions: rating 제출, 1-3점 failure tag 제출, pitch report 보기, playback problem 신고, 삭제 요청, 새 job 생성.
- Required data:
  - preview artifact id, signed playback URL eligibility
  - `preview_available`, `preview_played`, `playback_blocked`
  - `playback_session_id`, playback progress events, unique timeline coverage ratio
  - `section_id=intro`, section label and timestamp
  - pipeline mode: `partial_real` or `real_synthesis`
  - `mock_fixture_used=false`
  - pitch report, target/current pitch summary, low confidence ranges
  - quality summary: clipping, loudness, artifact candidates
  - rating prompt version and prior rating status
  - failure tag taxonomy version
- Key components:
  - app-only audio player
  - section badge
  - preview state and no-download/share notice
  - rating control after `preview_played=true`
  - failure tag selector for 1-3점
  - pitch feedback panel
  - quality warning panel
  - deletion request entry
- Empty state: preview가 없는 job은 Result page 대신 failed/partial result state를 보여준다.
- Loading state:
  - signed playback URL issuing
  - pitch/quality report loading
  - rating submission
- Error state:
  - playback URL denied
  - playback error
  - report unavailable
  - rating submission failure
  - artifact deleted or blocked
  - preview playback did not reach 80% unique timeline coverage
- Permission state:
  - learner는 본인 preview만 재생
  - withdrawn consent, rights blocked, deletion running/failed, audit failure 상태에서는 새 playback URL 발급 차단
  - preview를 재생하지 못하면 rating 없이 `playback_problem_reported` 제출 가능
- Navigation:
  - From: Processing Status
  - To: Data / Consent / Deletion Settings
  - To: Song Selection for new job
  - To: Processing Status when report still running
- Success signal:
  - distinct participating learner 10명 중 5명 이상이 첫 metric-eligible played preview에서 4점 이상
  - `preview_played=true`는 non-mock `partial_real` 또는 `real_synthesis` preview의 80% unique timeline coverage 기준을 만족해야 함
  - 4점 미만 rating의 100%가 failure tag 또는 `other` 포함
  - pitch-only success가 completed job으로 잘못 표시되지 않음

### 7. Data / Consent / Deletion Settings

- Purpose: 사용자가 job 단위 consent, retention, deletion status를 확인하고 철회/삭제 요청을 할 수 있게 한다.
- Primary action: 본인 데이터 삭제를 요청하거나 consent를 철회한다.
- Secondary actions: retention notice 확인, candidate data opt-in 철회, job 목록 확인. Contact follow-up 동의/철회는 gate가 later enabled일 때만 노출한다.
- Required data:
  - user id
  - job ids and artifact ids
  - consent records and policy versions
  - retention deadlines
  - deletion status and deletion evidence summary
  - disabled contact follow-up capability status
- Key components:
  - consent status list
  - retention summary
  - deletion request CTA
  - withdrawal impact confirmation
  - deletion status tracker
  - contact follow-up disabled status
- Empty state: 처리된 job이 없으면 삭제/보관 대상 없음 상태를 보여준다.
- Loading state: consent and artifact retention loading.
- Error state:
  - deletion request failed
  - deletion evidence unavailable
  - contact follow-up capability status unavailable
- Permission state:
  - learner는 본인 데이터만 조회/철회/삭제 가능
  - `own_voice_processing` 또는 `generated_preview` 철회 시 existing preview playback 즉시 차단
  - expert review 철회 시 review queue에서 숨김
- Navigation:
  - From: Result / Preview / Rating, Account menu
  - To: Result if playback still allowed
  - To: Song Selection after deletion/new job
- Success signal:
  - raw audio artifact의 100%가 30일 기본 retention과 deletion deadline을 가짐
  - deletion deadline 이후 24시간 내 deletion job 실행 여부 확인 가능

### 8. Limited Admin Song Package / Rights Gate

- Purpose: 관리자가 P0 `Mist` package, reference audio, section map, rights/risk exposure state를 제한된 범위에서 관리한다.
- Primary action: `Mist` song package를 등록/수정하고 exposure state를 결정한다.
- Secondary actions: reference audio upload, section map 검수, risk acceptance record 작성, package block/retire, kill switch 실행.
- Required data:
  - song package metadata
  - reference audio asset, checksum, uploader, upload timestamp
  - section map and default target section
  - rights state and risk acceptance state
  - allowed user ids/group id
  - retention/deletion owner
  - complaint owner and kill-switch owner
- Key components:
  - package metadata form
  - reference audio upload control
  - section timeline table
  - rights/risk checklist
  - exposure decision panel
  - internal allowlist control
  - audit status indicator
- Empty state: 등록된 package가 없으면 create package CTA를 보여준다.
- Loading state: package/reference asset loading, upload session creating, exposure decision saving.
- Error state:
  - metadata incomplete
  - reference audio upload failed
  - section timestamp mismatch
  - risk acceptance missing
  - audit write failed
- Permission state:
  - `admin`만 package 등록/수정 가능
  - `policy_or_rights_owner`, `platform_storage_owner`는 login role이 아니라 owner metadata
  - admin publish/block action은 audit write success 없이는 fail closed
- Navigation:
  - From: role-gated internal admin entry. MFA/step-up is Later for high-risk actions unless already implemented.
  - To: Governance Evidence for audit/deletion records
  - To: Review Queue only if user also has review role
- Success signal:
  - `Mist` package가 `published` 또는 `unlicensed_internal_risk_accepted`와 internal allowlist exposure rule을 가짐
  - `rights_blocked` package는 learner selection과 playback URL 발급이 차단됨

### 9. Educator / Expert Review Queue

- Purpose: 교육자/전문가가 동의된 reviewable result를 찾아 검토할 수 있게 한다.
- Primary action: 검토할 job을 선택한다.
- Secondary actions: filter by status, low confidence, failure tag, rating, date; completed review 보기.
- Required data:
  - educator/expert role
  - expert review consent status
  - reviewable job list
  - learner pseudonymous id
  - job state and output flags
  - rating/failure tag summary
- Key components:
  - review queue list
  - filter controls
  - consent/permission badge
  - job state badge
  - empty review queue state
- Empty state: 동의된 reviewable result가 없으면 빈 queue 안내.
- Loading state: review queue loading.
- Error state:
  - queue load failure
  - consent withdrawn between list and open
  - job deleted or playback blocked
- Permission state:
  - 교육자/전문가는 동의된 job만 접근
  - raw audio 직접 접근/다운로드는 기본 제공하지 않음
  - expert review consent 철회 시 queue에서 숨김
- Navigation:
  - From: role-gated reviewer entry. MFA/step-up is Later for high-risk actions unless already implemented.
  - To: Review Detail / Internal Reviewer Mode
  - Back To: reviewer home
- Success signal:
  - 교육자 2명이 최소 1개 reviewable result에 접근 가능
  - withdrawn consent job이 queue에 남지 않음

### 10. Review Detail / Internal Reviewer Mode

- Purpose: 동의된 job의 preview, pitch report, failure tags, quality summary를 검토하고 교육적 코멘트 또는 technical tags를 분리 입력한다.
- Primary action: review comment 또는 technical tag를 제출한다.
- Secondary actions: preview playback, pitch report 확인, low-confidence range 확인, quality summary 확인, needs_review 해소 여부 기록.
- Required data:
  - job id and review permission
  - generated preview playback eligibility
  - pitch report and low confidence ranges
  - user rating and failure tags
  - quality summary
  - internal technical tag taxonomy
  - stage result references
- Key components:
  - preview player with app-only restrictions
  - pitch report panel
  - user perception tag panel
  - educator comment field
  - internal reviewer mode technical tag panel
  - audit/access notice
- Empty state: review detail target이 없거나 접근 가능한 artifact가 없으면 queue로 돌아간다.
- Loading state: review artifacts and signed playback URL loading.
- Error state:
  - playback blocked
  - report missing
  - technical tag submission failed
  - consent withdrawn
- Permission state:
  - educator/expert는 raw audio 기본 접근 없음
  - engine developer는 internal reviewer mode에서 technical tag만 입력 가능
  - user perception tags와 technical tags는 분리 저장
- Navigation:
  - From: Review Queue
  - To: Review Queue after submit
  - To: Governance Evidence only for users with governance permission
- Success signal:
  - user perception tags와 technical tags가 분리 저장됨
  - low-confidence/disputed 구간이 review detail에서 숨겨지지 않음

### 11. Governance Evidence / Audit / Deletion

- Purpose: P0 운영자가 audit, deletion evidence, risk acceptance, contact follow-up disabled capability, break-glass disabled 상태를 확인한다.
- Primary action: evidence record를 확인하고 unresolved risk/deletion failure를 처리한다.
- Secondary actions: deletion job 상태 확인, audit failure 확인, contact follow-up disabled 상태 확인, break-glass request 차단 사유 확인.
- Required data:
  - audit records
  - deletion evidence
  - risk acceptance records
  - retention deadlines
  - rights complaint and block records
  - disabled contact follow-up capability status
  - break-glass approver/second reviewer availability
- Key components:
  - evidence table
  - risk acceptance detail
  - deletion job status
  - audit fail-closed status
  - contact follow-up disabled capability notice
  - break-glass disabled notice
- Empty state: unresolved governance issue가 없으면 clean state를 보여준다.
- Loading state: audit/deletion evidence loading.
- Error state:
  - evidence store unavailable
  - audit write failed
  - deletion job failed
  - key management gate missing
- Permission state:
  - `security_or_ops`, platform/storage owner metadata owner만 접근
  - P0 1인 운영에서 second reviewer가 없으면 raw/canonical audio human access와 contact plaintext reveal disabled
  - contact plaintext UI와 CSV export는 P0에서 금지
- Navigation:
  - From: Limited Admin, security/ops entry
  - To: Limited Admin for package block/retire
  - To: affected job detail only if permission allows
- Success signal:
  - 100%의 analysis/preview requests가 allow/deny decision과 audit record를 가짐
  - raw audio가 1년 retained dataset에 포함되지 않음

## UI Design Handoff Notes

This page plan is ready for a UI design agent, but it is not a visual mockup.

Design priorities:

- The learner app should feel like a focused practice tool, not a marketing landing page.
- The first screen after login is the actual song selection flow, not an explanatory hero.
- The core learner sequence must remain visually obvious: song -> section -> record -> processing -> preview -> rating.
- The preview player is the primary component on the result page. Pitch and quality details are secondary.
- Do not expose "alpha test" wording in the UI.
- Contact follow-up UI, value collection, send, decrypt, and export are out of P0.
- Reference pre-listen and lyrics cues must visually appear gated and section-limited when enabled; they must not look like full-song playback or full lyrics.
- State labels should separate job status from rating status, especially `completed` versus rating submitted.
- Permission-denied and rights-blocked states should give safe reasons without exposing internal policy details.

Expected design deliverables:

- Mobile-first learner flow wireframes for pages 1-7.
- Web-first internal surface wireframes for pages 8-11.
- State variants for blocked, failed, partial artifact, loading, empty, and permission-denied states.
- Component inventory for recorder controls, preview player, rating control, failure tag selector, job state panel, rights status panel, and governance evidence table.

## Primary User Flow

1. 사용자가 로그인하거나 회원가입 후 이메일 인증을 완료한다.
2. 시스템은 role과 내부 접근 권한을 확인한다.
3. 접근 권한이 있는 학습자는 `Mist` song을 선택한다.
4. 사용자는 `intro 0:00-0:28` section을 선택한다.
5. 사용자는 recorder 화면에 진입한다. 이 화면의 entry gate에서 BPM/key를 확인하거나 feature flag가 켜진 경우 수정하고, 필수 consent가 현재 policy document/version/scope/required-optional 상태와 일치하는지 확인한다.
6. consent가 없거나 outdated이면 recorder 화면 안에서 재동의를 완료해야 녹음/제출을 진행할 수 있다.
7. 사용자는 recorder 화면에서 section guide를 확인한다. 권리 플래그가 허용하면 recording 전 section pre-listen 또는 permitted lyric cue를 볼 수 있다.
8. 사용자는 본인 목소리 take를 녹음하고, replay 후 retake 또는 submit을 선택한다.
9. 앱은 recorder take 또는 fallback upload를 presigned object storage path로 전송한다.
10. 사용자가 처리 시작을 누르면 `completeAudioUpload`가 object HEAD를 검증하고, 서버가 consent snapshot과 rights/risk snapshot을 직접 계산한 뒤 audit allow decision과 함께 job을 생성한다.
11. Processing Status는 canonical job state와 output flags를 표시한다.
12. `preview_ready`가 되면 사용자는 Result page에서 preview를 재생한다.
13. 앱은 foreground, unmuted playback 중 1초마다 progress event를 보내고, 서버가 동일 preview artifact의 80% unique timeline coverage 기준을 만족한다고 계산하면 `preview_played=true`가 되며 rating UI가 열린다.
14. 사용자는 "이 preview가 내 목소리처럼 들린다"를 1-5점으로 평가한다.
15. 1-3점이면 failure tag 또는 `other`를 제출한다.
16. 사용자는 pitch feedback과 quality summary를 확인하고, 필요 시 deletion/consent settings로 이동한다.

## Alternate / Edge Flows

- Not eligible: Song Selection에서 `Mist`를 선택할 수 없고 접근 불가 사유를 본다.
- Rights blocked: selection, `completeAudioUpload`, playback URL issuance에서 모두 차단하고 다른 곡 선택 또는 문의 CTA를 제공한다.
- Consent missing or outdated: Record Take / Take Review의 entry gate에서 차단하고 어떤 consent가 부족하거나 오래되었는지 보여준다. 재동의 trigger는 policy document, version, scope, required/optional status, reference/lyrics display scope 변경을 포함한다.
- Reference pre-listen unavailable: recorder에서는 reference player를 숨기거나 disabled 처리하고 녹음은 계속 가능하게 한다.
- Reference pre-listen active: 녹음 시작 전에 reference playback을 정지한다.
- Lyrics unavailable: full lyrics와 line lyrics를 숨기고 section label, timestamp, expected duration만 제공한다.
- Mic denied or unavailable: mic 재시도와 권한 설정 안내를 제공하고, fallback upload가 enabled인 경우 대체 경로를 제공한다.
- Upload URL expired: 새 upload session 생성 CTA를 제공한다.
- Unsupported fallback format or size: Record Take에서 `.wav`, `.mp3`, 50 MB file-size guard를 안내한다.
- Silence, clipping, too low volume, or no voice: Take Review에서 retake를 권장한다.
- Duration exceeded: ingest 이후 Record Take/Status에서 60초 hard max와 trim/retake/reupload CTA를 보여준다.
- Section mismatch: full-song으로 조용히 처리하지 않고 retake 또는 잘라서 재업로드 CTA를 보여준다.
- Timeout: `completeAudioUpload.committed_at` 기준 60분 초과 시 `failed` 또는 `failed_with_partial_artifacts`로 terminalize한다.
- Preview unavailable but pitch report available: partial result로 음정 리포트 보기와 재시도 CTA를 제공하고 primary self-voice rating은 받지 않는다.
- Preview playable in partial artifact state: preview 재생과 rating은 가능하지만 job completion success로 계산하지 않는다.
- Playback problem: rating 없이 `playback_problem_reported` event를 제출한다.
- Consent withdrawal: generated preview playback, expert review access, new signed playback URL을 즉시 차단하고 deletion job을 예약한다.
- Deletion failed: playback을 먼저 차단하고 Governance Evidence에서 owner review를 생성한다.
- Contact follow-up capability disabled: P0에서는 contact follow-up UI, 값 저장, 발송, 복호화, export를 제공하지 않고 core preview/rating flow는 계속 진행한다.

## Page x Feature Matrix

| Page | Feature | User action | Required data | State coverage |
| --- | --- | --- | --- | --- |
| Login / Signup | Account And Role Access | 로그인, 회원가입, 이메일 인증 | user, role, session, verification | signed_out, authenticated, session_expired, locked_or_blocked |
| Access Eligibility / Song Selection | Admin Song Package And Rights Gate | `Mist` song 선택 | song package, rights state, eligibility flag, risk acceptance | empty, loading, rights_blocked, under_review, permission denied |
| Section Selection / Part Selection | Learner Song And Section Selection, Consent, Recording, And Upload | `intro` section 선택 | section map, timestamp, guide flags, rights state | empty, loading, section blocked, rights changed |
| Record Take / Take Review | Learner Song And Section Selection, Consent, Recording, And Upload; Audio Ingest And Section Validation | BPM/key와 consent gate 확인, 녹음, replay, retake, 제출 | consent policy document/version/scope, consent snapshot eligibility, BPM/key, usage restriction, contact gate, recorder session, mic permission, take id, upload session, rights flag snapshot | awaiting_consent, blocked_by_policy, blocked_by_consent_version, mic_permission_pending, recording, take_ready, take_needs_retake, uploading, commit error |
| Processing Status | Canonical Job State And Partial Artifact Handling | 처리 상태 확인 | job projection, stage summaries, output flags | created, queued, processing, preview_ready, failed, blocked, needs_review |
| Result / Preview / Rating | Self-voice Section Preview Generation And App-only Playback | preview 재생, rating 제출 | preview artifact, playback eligibility, pitch report, rating status | preview_ready, completed, playback_blocked, report loading, rating_required |
| Result / Preview / Rating | Result Review, Rating, And Failure Tagging | failure tag 제출, playback 문제 신고 | rating prompt, failure taxonomy, playback event | failure_tags_required, playback_problem_reported, submission error |
| Data / Consent / Deletion Settings | Safety, Audit, Retention, And Deletion Governance | consent 철회, 삭제 요청 | consent records, retention deadlines, deletion status | retention_active, deletion_scheduled, deletion_running, deletion_failed, deleted |
| Limited Admin Song Package / Rights Gate | Admin Song Package And Rights Gate | reference upload, exposure 결정 | package metadata, section map, rights/risk record, allowlist | draft, metadata_incomplete, rights_pending, risk_accepted, rights_blocked |
| Educator / Expert Review Queue | Result Review, Rating, And Failure Tagging | 동의된 job 선택 | reviewable jobs, consent status, rating/failure summary | empty queue, loading, consent withdrawn, permission denied |
| Review Detail / Internal Reviewer Mode | Target Pitch Mapping And Confidence Handling | pitch/quality 검토 | pitch report, low-confidence ranges, technical tags | review_pending, review_completed, playback_blocked |
| Governance Evidence / Audit / Deletion | Safety, Audit, Retention, And Deletion Governance | evidence 확인, failure 처리 | audit records, deletion evidence, risk acceptance, contact gate | audit_failed, deletion_failed, break-glass disabled, permission denied |

## UX Risks

- `Mist intro`는 나레이션 중심이라 pitch feedback 만족도가 낮을 수 있다. Result page에서 pitch score를 과대 강조하지 않아야 한다.
- `unlicensed_internal_risk_accepted`는 권리 확보가 아니다. Song Selection과 Admin에서 일반 publish처럼 보이면 안 된다.
- Song Selection과 Section Selection을 합치면 향후 verse/chorus 확장 때 정보 구조가 깨진다. P0에서 선택지가 하나여도 page/route/state를 분리한다.
- Reference pre-listen이 active recording 중 섞이면 원곡이 사용자 take에 유입되어 품질과 권리 리스크가 커진다. 녹음 시작 전에 반드시 정지한다.
- Lyrics display가 권리 scope 없이 전체 가사처럼 보이면 저작권 리스크가 커진다. 권리 플래그가 없으면 section label/timestamp만 보여준다.
- "녹음 금지" 문구가 본인 목소리 녹음 금지로 오해되면 core flow를 망친다. reference audio, lyrics, preview, app output의 외부 캡처 금지로 표현한다.
- First-login consent를 너무 넓게 해석하면 later policy change가 누락된다. recorder entry gate와 job snapshot 검증에서 re-consent trigger를 명확히 보여줘야 한다.
- `completed`가 rating 완료로 오해될 수 있다. job state와 evaluation state를 화면에서 분리해야 한다.
- 긴 처리 시간 동안 빈 spinner만 보여주면 이탈이 커질 수 있다. Processing Status는 stage와 plain-language reason을 보여줘야 한다.
- 실패/부분 성공 상태에서 preview 가능 여부가 섞이면 metric이 흐려진다. `preview_available`, `playback_blocked`, `pipeline_mode`를 화면과 analytics에서 분리해야 한다.
- `playback_problem_reported`를 낮은 rating과 섞으면 self-voice 품질 metric이 오염된다.
- consent 철회와 deletion request의 차이가 불분명하면 신뢰가 무너질 수 있다. 철회 영향과 삭제 일정은 명확해야 한다.
- contact follow-up은 P0에서 disabled capability status만 남기고 기본 hidden/disabled다. core preview flow, rating, failure tag 제출의 필수 단계처럼 보여서는 안 된다.
- educator/expert review 화면에서 raw audio 접근이 가능해 보이면 privacy expectation이 깨진다.
- admin 화면이 full operations console처럼 커지면 P0 scope creep이 발생한다.

## Open Questions

- `unlicensed_internal_risk_accepted`를 실제로 켜기 위한 risk acceptance record의 source of truth는 PM 문서, DB, ticket 중 어디인가?
- P0 내부 운영 기간 동안 second reviewer를 둘 수 있는가? 없으면 Governance Evidence는 break-glass disabled 상태를 어떻게 보여줄 것인가?
- contact follow-up은 P0에서 disabled capability status만 정의한다. 실제 UI, backend 저장, 발송, 복호화, export는 수익화 또는 외부 beta 전 권한/암호화/감사 owner를 갖춘 뒤 재결정한다.
- 권리 evidence 없는 `Mist` 제한 노출은 내부 운영 종료 전 어느 시점에 `rights_pending`, `published`, 또는 `rights_blocked`로 재판정할 것인가?
- mobile app과 web에서 admin/review/governance surface를 모두 제공할 것인가, 아니면 learner app은 mobile-first, internal surface는 web-first로 제한할 것인가?
- reference pre-listen과 lyrics display flags의 source of truth는 admin rights/risk record, DB package field, ticket 중 어디인가?

## Recommended Next Skill

`spec-to-tickets`

Page ownership, primary CTA, page states, permission states, navigation, feature mapping이 개발 티켓으로 분해 가능한 수준으로 정리됐다. 남은 Open Questions는 내부 운영/권리 gate와 reference/lyrics flag source of truth에 관한 launch decision이며, learner core flow와 internal surface의 ticket breakdown은 진행 가능하다.
