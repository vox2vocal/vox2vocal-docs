# Vox2Vocal MVP Feature Definition

문서 버전: v0.5
작성일: 2026-06-13
기준 문서: `pm/vox2vocal-mvp-prd.md` v0.12, `pm/vox2vocal-mvp-prd-review.md`
적용 skill: `feature-definer`

## Context

Vox2Vocal P0 내부 운영은 음악 학습자가 Ken Kamikita - `Mist`를 선택하고, `intro` section을 선택한 뒤, 앱에서 본인 목소리 take를 녹음해 앱 안에서 본인 목소리처럼 들리는 보정/생성 preview를 들어보는 경험을 검증한다. P0의 1차 성공은 full-song 생성이나 완성도 높은 상업 품질이 아니라 section-limited self-voice preview가 실제 학습 동기와 품질 판단 근거를 만드는지 확인하는 것이다.

P0는 `Mist` 전체 section map을 song package로 보관하되, 기본 target은 `intro`, timestamp는 등록된 reference audio asset 기준 `0:00-0:28`로 둔다. `intro`는 self-voice preview와 app flow 검증에는 적합하지만 나레이션 중심이므로 singing pitch matching 대표성은 약하다. 따라서 pitch feedback은 P0에서 제공하되, 후렴 수준의 singing validation은 Later 범위로 둔다.

리뷰 보강 기준은 다음과 같다.

- P0 confirmed: 기능 정의서에서 기본 구현 방향으로 확정한다.
- P0 provisional default: P0 기본값으로 쓰되, 실제 reference asset 또는 내부 운영 전 검수로 조정할 수 있다.
- User decision required: 제품/정책 owner, 실제 권리 승인, 담당자 지정처럼 Codex가 임의로 확정하면 안 되는 항목이다.

## MVP Features

### 1. 계정 및 역할별 접근 제어 (Account And Role Access)

- User action: 학습자, 교육자/전문가, 관리자, 내부 reviewer가 계정을 만들거나 로그인한다.
- Product behavior: 앱은 인증된 사용자만 곡 선택, section 선택, 녹음/제출, 결과 조회, review 작업에 접근하도록 제한한다. 사용자 role에 따라 가능한 행동과 조회 가능한 데이터 범위를 다르게 적용한다.
- Business rules:
  - 이메일, 비밀번호, 표시 이름, 약관 동의가 필요하다.
  - 회원가입 또는 최초 로그인 시 P0 필수 동의 묶음을 받을 수 있다. 단, 각 동의는 type, version, scope, required/optional 여부가 분리되어야 한다.
  - 최초 동의가 있어도 job 제출 시점에는 현재 정책 version/scope와 일치하는 consent snapshot을 반드시 저장한다.
  - 정책 문서, 동의 scope, required/optional 상태, reference/lyrics display scope가 바뀌면 녹음 제출 전에 재동의를 요구한다.
  - 비밀번호는 최소 8자 이상이어야 한다.
  - 인증되지 않은 사용자는 conversion job을 만들 수 없다.
  - login role은 최소 `learner`, `educator_or_expert`, `admin`, `engine_developer`, `product_qa`, `security_or_ops`를 구분한다.
  - `policy_or_rights_owner`, `platform_storage_owner`는 login role이 아니라 승인/운영 책임 owner metadata로 기록한다.
- Permissions / roles:
  - `learner`: 본인 recorded/fallback input, 본인 job, 본인 preview, 본인 pitch report, 삭제 요청 접근
  - `educator_or_expert`: 사용자가 동의한 job의 preview, pitch report, failure tags 검토
  - `admin`: song package 생성/수정/비활성화
  - `engine_developer`: 디버깅 목적의 제한된 artifact 접근
  - `product_qa`: 비식별 품질 리포트, 실패 태그, metrics 접근
  - `security_or_ops`: audit, retention, deletion, incident metadata 접근
- States:
  - `signed_out`
  - `authenticated`
  - `role_limited`
  - `session_expired`
  - `locked_or_blocked`
- Edge cases:
  - 로그인 세션이 만료된 상태에서 녹음 제출 또는 fallback upload를 시작한 경우 job 생성 전 재인증한다.
  - role이 없는 내부 사용자는 기본적으로 learner 권한보다 더 넓은 접근을 받지 않는다.
  - 교육자/전문가가 raw audio에 직접 접근하려는 경우 기본 차단한다.
- Dependencies:
  - App auth UI
  - BFF auth GraphQL API
  - API Gateway auth contract
  - User Service account, auth, role/status
  - separated consent records and job consent snapshot store
- Success signal:
  - 학습자 10명 중 5명 이상이 첫 세션에서 practice analysis job을 생성한다.
  - 교육자 2명이 최소 1개 reviewable result에 접근할 수 있다.
  - client/backend 인증 연동 오류가 시도 대비 2% 미만이다.

### 2. 관리자 곡 패키지 및 권리 검증 게이트 (Admin Song Package And Rights Gate)

- User action: 관리자가 내부 운영용 `Mist` song package를 등록하고 publish 가능 상태로 만든다.
- Product behavior: 시스템은 필수 metadata와 권리 evidence가 모두 충족된 song package만 일반 학습자에게 노출한다. 권리 evidence가 없는 `Mist`는 `published`로 보지 않고, risk acceptance가 기록된 경우에만 P0 internal allowlist에 제한 노출한다. 권리 불확실, 만료, complaint, provenance 누락이 있으면 `rights_blocked`로 전환한다.
- Business rules:
  - P0 song package 필수값은 title, artist, language, BPM, key, reference audio, source/provenance, rights clearance status, usage status, retention period, deletion owner, full section map, default target section start/end, reference pre-listen flag/scope, lyrics display flag/scope, lyrics sync flag, license expiry/re-review date다.
  - P0 target song은 Ken Kamikita - `Mist`로 제한한다.
  - P0 default target section은 `intro`, timestamp는 reference audio asset 기준 `0:00-0:28`로 확정한다. reference audio asset이 바뀌면 timestamp를 다시 검수한다.
  - YouTube, Spotify, lyrics provider 등 외부 provider는 P0에서 audio source로 쓰지 않는다.
  - reference audio는 기본적으로 앱 내부 분석/비교에만 사용하고 다운로드, 공유, 학습, remix, third-party voice cloning에 사용하지 않는다.
  - learner-facing reference pre-listen은 `reference_prelisten_allowed=true`, scope=`section_prelisten`, 권리 만료 전, audit write success일 때만 허용한다.
  - reference pre-listen은 section-limited, app-only, short-lived signed URL, no-download/share로 제공하며 active microphone recording이 시작되면 즉시 정지한다.
  - lyrics display는 `lyrics_display_allowed=true`이고 display scope가 selected section 또는 line cue를 포함할 때만 노출한다. `lyrics_sync_allowed=false`이면 time-synced lyrics처럼 보이면 안 된다.
  - full lyrics, playable reference URL, lyric free-text는 로그에 남기지 않는다.
  - 필수 권리 evidence가 없으면 정식 `published` 상태로 learners에게 노출할 수 없다.
  - `unlicensed_internal_risk`는 권리 승인 상태가 아니라 권리 evidence 부재를 명시한 risk exception 상태이며, 기본적으로 learner catalog에 노출하지 않는다.
  - P0 내부 운영에서만 `unlicensed_internal_risk_accepted` 상태를 사용할 수 있다. 이 상태는 internal allowlist, feature flag, risk acceptance record가 모두 있을 때만 제한 선택과 section-limited preview processing을 허용한다.
  - `unlicensed_internal_risk_accepted` 예외는 `Mist intro 0:00-0:28`의 section-limited self-voice preview 검증에만 적용하며, full-song 처리나 다른 section 확장에는 적용하지 않는다.
  - rights clearance record는 승인자, 승인 근거, 허용 용도, 금지 용도, 만료/재검토일, retention, deletion owner, complaint owner를 포함해야 한다.
  - risk acceptance record는 rights clearance가 아니며 `risk_acceptance_id`, `song_package_id`, `reference_asset_id`, `checksum`, `allowed_user_ids` 또는 `allowed_group_id`, 최대 참여자 수, 시작/종료일, exact section, 금지 용도, kill-switch owner를 포함해야 한다.
- Permissions / roles:
  - `admin`: song package 등록/수정 요청
  - `policy_or_rights_owner`: rights clearance 승인/차단 책임자 metadata
  - `platform_storage_owner`: retention/deletion owner metadata
  - `learner`: `published` song package 조회 가능. P0에서는 internal allowlist에 포함된 사용자만 `unlicensed_internal_risk_accepted` package를 제한 조회/선택할 수 있다.
- States:
  - `draft`
  - `metadata_incomplete`
  - `rights_pending`
  - `approved_for_internal_analysis`
  - `unlicensed_internal_risk`
  - `unlicensed_internal_risk_accepted`
  - `published`
  - `rights_blocked`
  - `under_review`
  - `retired`
  - `reference_prelisten_allowed`
  - `reference_prelisten_blocked`
  - `lyrics_available`
  - `lyrics_blocked_or_unavailable`
- Edge cases:
  - 등록된 reference audio asset과 section timestamp가 어긋나면 publish를 보류한다.
  - rights complaint가 들어오면 해당 song package와 관련 generated preview playback을 차단한다.
  - rights complaint, provenance dispute, risk acceptance 만료, audit failure, deletion failure가 발생하면 관련 package와 generated preview playback을 즉시 `rights_blocked`로 전환한다.
  - BPM/key는 기본값으로 등록하고 P0 기본 동작은 admin-only 수정이다. 단, P0 내부 운영에서는 allowlist 대상 학습자에게 feature flag로 학습자 BPM/key correction controls를 켠다.
  - lyrics는 P0 선택값이며, lyrics sync가 없더라도 publish를 막지 않는다. 단, 녹음 화면에 lyric cue를 보여주려면 별도 display right와 scope가 필요하다.
- Dependencies:
  - Admin song package API 또는 seed/admin-only 등록 경로
  - object storage for reference audio
  - rights clearance checklist
  - section map validation
  - audit log
  - deletion owner registry
- Success signal:
  - `Mist` package가 필수 metadata와 rights evidence를 갖고 `published`되거나, 권리 evidence가 없을 경우 `unlicensed_internal_risk_accepted`와 internal allowlist exposure rule을 갖는다.
  - learners는 `rights_blocked` package를 선택할 수 없다.
  - 100%의 reference asset exposure decision이 provenance, allowed use, retention, deletion owner를 포함한다.

### 3. 학습자 곡/파트 선택, 동의, 녹음/업로드 (Learner Song And Section Selection, Consent, Recording, And Upload)

- User action: 학습자가 관리자 등록 `Mist` package를 선택하고, `intro` section을 선택한 뒤, 앱 recorder로 본인 목소리 take를 녹음한다. Recorder를 사용할 수 없거나 내부 운영상 허용된 경우에만 fallback file upload를 사용한다.
- Product behavior: 시스템은 song selection, section selection, current consent snapshot, recording/take validation을 통과한 경우에만 P0 section preview job을 생성한다. 동의가 부족하거나 파일/녹음이 부적합하면 engine processing 전에 차단한다.
- Business rules:
  - P0는 학습자의 원곡/reference audio 업로드를 허용하지 않는다.
  - 학습자는 본인 목소리 take만 녹음 또는 fallback upload로 제출할 수 있다.
  - 곡 선택 후 section 선택이 필요하다. P0에서 `intro` 하나만 노출하더라도 별도 Section Selection step을 유지한다.
  - Recorder는 mic permission, count-in, timer, input meter, stop, replay own take, retake, submit을 지원해야 한다.
  - Fallback upload 지원 형식은 `wav`, `mp3`다.
  - upload hard max는 60초다.
  - target section duration은 20-40초 권장이고 P0 `intro`는 28초다.
  - section length tolerance는 target duration 대비 `±5초` 또는 `±20%` 중 더 큰 값이다.
  - required service consent는 회원가입 또는 최초 로그인 시 1회 받을 수 있다. 단, `completeAudioUpload` 또는 recorder submit 시점에 consent snapshot id/hash를 job에 저장해야 한다.
  - own voice consent와 generated preview consent는 필수다. 현재 consent version/scope가 맞지 않거나 철회되었으면 녹음 제출을 차단한다.
  - expert review consent는 P0 job 제출 조건으로 분리 동의한다. 사용자가 거부하면 계정 사용은 가능하지만 P0 preview job은 시작하지 않는다.
  - candidate data consent는 opt-in이며 필수 동의와 번들링하지 않는다.
  - consent는 type, version, scope, required/optional, timestamp, withdrawal status를 기록한다.
  - BPM/key correction은 job 생성 전 song setup/upload review 단계에서만 수정 가능하다. `completeAudioUpload` 이후에는 해당 job의 BPM/key snapshot을 read-only로 보여주고, 변경하려면 새 job을 만들어야 한다.
  - 녹음 화면의 원곡/reference pre-listen과 lyrics display는 권리 플래그가 허용할 때만 노출한다. 기본값은 숨김이다.
  - 원곡/reference audio는 녹음 전 section pre-listen으로만 제공한다. 활성 녹음 중 동시 재생은 P0에서 허용하지 않는다.
  - "녹음/캡처/재배포/공개 게시 금지" 문구는 reference audio, lyrics, generated preview, app output에 적용되는 제한으로 작성하고, 사용자의 본인 목소리 녹음 자체를 금지하는 문구로 보이면 안 된다.
- Permissions / roles:
  - `learner`: 본인 recorded/fallback voice input과 consent 제출
  - `educator_or_expert`: 동의된 job만 검토
  - `engine_developer`: 필요한 경우 승인된 디버깅 범위에서 artifact 접근
  - `security_or_ops`: audit metadata 접근
- States:
  - `song_selected`
  - `section_selected`
  - `mic_permission_pending`
  - `mic_permission_denied`
  - `recorder_ready`
  - `recording`
  - `take_ready`
  - `take_needs_retake`
  - `take_submitting`
  - `awaiting_consent`
  - `uploading`
  - `upload_validating`
  - `upload_rejected`
  - `ready_to_process`
  - `blocked_by_policy`
  - `blocked_by_consent_version`
- Edge cases:
  - 사용자가 본인 음성 확인을 거부하면 job을 생성하지 않는다.
  - mic permission이 거부되거나 기기가 없으면 재시도 안내를 제공하고, fallback upload가 enabled인 경우에만 대체 경로를 제공한다.
  - reference pre-listen 또는 lyrics display가 unavailable이어도 녹음 자체는 진행 가능해야 한다.
  - reference audio가 재생 중이면 녹음 시작 시 정지한다.
  - 무음, clipping, 너무 낮은 volume, 너무 짧은 take는 processing 전에 retake를 권장한다.
  - 60초 초과 파일/take는 자동 full-song 처리하지 않고 trim, retake, expected section 재제출을 요구한다.
  - 지원하지 않는 파일 형식은 명확한 사유와 함께 거절한다.
  - candidate data opt-in이 없어도 preview generation 자체는 진행할 수 있다.
  - reference audio가 `rights_blocked`로 바뀐 직후 선택 화면에 캐시된 package가 남아 있으면 processing 전에 다시 차단한다.
  - upload session이 만료되면 가능하면 local take를 유지하고 새 upload session을 요청한다.
- Dependencies:
  - song package read model
  - section selection read model
  - app recorder and mic permission handling
  - take review UI
  - upload initiation API
  - presigned object storage upload
  - consent storage
  - Safety Rights check
  - job creation API
- Success signal:
  - song selected -> section selected -> recorder ready -> take submitted -> preview played -> rating submitted funnel event가 분리 추적된다.
  - valid recording/fallback upload의 100%가 consent snapshot, selected song/section, rights flag snapshot, source asset id를 가진다.
  - unauthorized or non-self voice job이 analysis, preview, synthesis에 도달하지 않는다.
  - rejected take/upload는 사용자에게 mic, silence, clipping, format, length, consent, rights 중 어떤 이유인지 보여준다.

### 4. 오디오 수집 및 구간 검증 (Audio Ingest And Section Validation)

- User action: 학습자가 recorded take 또는 fallback upload를 제출하고 분석 시작을 요청한다.
- Product behavior: 시스템은 제출된 voice input을 canonical audio로 변환하고, duration, loudness, silence, voice segments, section coverage를 계산한다. target section과 맞지 않는 경우 full-song으로 조용히 처리하지 않고 사용자가 이해할 수 있는 상태로 돌려준다.
- Business rules:
  - accepted input은 mono WAV/PCM canonical asset으로 변환한다.
  - metadata에는 sample rate, channels, duration, loudness estimate, silence segments, voice segments, `audio_asset_id`를 포함한다.
  - target section mismatch는 warning 또는 trim 요청으로 처리한다.
  - raw audio와 canonical audio는 기본 30일 보관한다.
  - raw audio는 1년 장기 보관 dataset에 포함하지 않는다.
- Permissions / roles:
  - `learner`: 본인 ingest status와 validation result 조회
  - `engine_developer`: stage artifact와 ingest metadata 조회
  - `product_qa`: 비식별 ingest metrics 조회
- States:
  - `ingest_queued`
  - `ingest_processing`
  - `ingest_completed`
  - `section_mismatch`
  - `ingest_failed`
  - `deletion_scheduled`
- Edge cases:
  - 무음 또는 voice segment 부족 take는 preview job을 진행하지 않고 retake 또는 fallback upload를 요청한다.
  - loudness가 너무 낮거나 clipping이 심한 경우 preview 품질 risk를 기록한다.
  - 변환은 성공했지만 section validation이 낮은 confidence인 경우 `needs_review` 또는 `section_mismatch`로 남긴다.
- Dependencies:
  - Audio Ingest engine
  - object storage
  - canonical job state owner
  - stage result record
  - retention/deletion scheduler
- Success signal:
  - valid P0 voice input의 100%가 canonical asset과 ingest metadata를 생성한다.
  - section mismatch가 발생한 job은 full-song으로 조용히 처리되지 않는다.
  - ingest failure stage와 plain-language reason이 job status에 반영된다.

### 5. 목표 음정 매핑 및 신뢰도 처리 (Target Pitch Mapping And Confidence Handling)

- User action: 학습자가 `intro` section에 대한 pitch feedback 결과를 본다.
- Product behavior: 시스템은 reference audio analysis, engine-derived note sequence, manual/admin override 중 출처가 있는 target note sequence를 생성하고, submitted vocal pitch와 비교한다.
- Business rules:
  - P0는 pitch-first로 동작하고 lyric sync는 요구하지 않는다.
  - note sequence JSON은 note start/end, target pitch, duration, confidence, source attribution을 포함한다.
  - pitch frame confidence `>= 0.70`은 trusted로 scoring에 사용한다.
  - confidence `0.45-0.69`는 low confidence로 표시하고 overconfident scoring에서 제외한다.
  - confidence `< 0.45`는 pitch scoring에서 제외한다.
  - reference와 engine note 차이가 `> 50 cents`이고 voiced frames의 20% 이상이면 disputed section으로 표시한다.
  - reference와 engine note 차이가 `> 1 semitone`이고 voiced frames의 10% 이상이면 `needs_review` 후보로 표시한다.
  - `intro`는 나레이션 중심이므로 speech-like pitch range를 singing pitch accuracy처럼 과대 평가하지 않는다.
- Permissions / roles:
  - `learner`: simplified pitch feedback 조회
  - `educator_or_expert`: 동의된 job의 pitch report 검토
  - `engine_developer`: source attribution, confidence, disputed ranges 조회
  - `product_qa`: 비식별 pitch usefulness metric 조회
- States:
  - `target_notes_ready`
  - `pitch_processing`
  - `pitch_report_ready`
  - `low_confidence`
  - `disputed`
  - `needs_review`
  - `pitch_failed`
- Edge cases:
  - target note source 간 충돌이 큰 구간은 강제 판정하지 않는다.
  - pitch report만 성공하고 preview가 실패하면 P0 self-voice success로 보지 않는다.
  - unvoiced/narration-heavy frame은 pitch score에서 제외하거나 낮은 confidence로 처리한다.
- Dependencies:
  - reference audio asset
  - Voice Pitch engine
  - Target Pitch Mapping
  - section map
  - confidence threshold config
  - internal review artifact store
- Success signal:
  - 학습자 10명 중 5명 이상이 current-vs-target pitch feedback이 다음 연습 포인트를 찾는 데 도움이 된다고 응답한다.
  - low-confidence/disputed 구간이 user-facing 또는 internal report에서 숨겨지지 않는다.
  - pitch-only success가 completed job으로 잘못 표시되지 않는다.

### 6. 본인 음성 구간 프리뷰 생성 및 앱 내 재생 (Self-voice Section Preview Generation And App-only Playback)

- User action: 학습자가 처리 완료 후 self-voice preview를 재생한다.
- Product behavior: 시스템은 사용자의 본인 recorded/fallback voice input만을 사용해 `intro` section-limited preview를 생성하고, 앱 내 재생만 허용한다. preview가 완전히 실패하면 job은 성공으로 표시하지 않는다.
- Business rules:
  - 생성 가능한 vocal identity는 제출한 사용자 본인 목소리뿐이다.
  - third-party voice, artist voice, character voice, celebrity voice imitation은 차단한다.
  - P0 output은 section-limited preview로 명확히 표시한다.
  - generated preview는 다운로드, export, share link, public posting, commercial use를 허용하지 않는다.
  - render output에는 clipping, loudness, artifact 후보 metadata를 포함한다.
  - generated preview는 기본 30일 보관한다.
- Permissions / roles:
  - `learner`: 본인 generated preview app-only playback
  - `educator_or_expert`: 동의된 job의 preview playback/review
  - `engine_developer`: 제한된 debug artifact 접근
  - `product_qa`: 비식별 quality report 접근
- States:
  - `synthesis_queued`
  - `synthesis_processing`
  - `render_processing`
  - `preview_ready`
  - `completed`
  - `failed`
  - `failed_with_partial_artifacts`
  - `blocked`
- Edge cases:
  - preview generation이 실패하고 pitch report만 성공하면 `failed` 또는 `failed_with_partial_artifacts`로 표시한다.
  - preview가 너무 짧거나 section coverage가 부족하면 failure tag 후보 `too_short_or_incomplete`를 노출한다.
  - playback URL은 5분 TTL의 short-lived signed access로만 제공한다.
  - rights complaint 또는 deletion failure가 발생하면 playback을 차단한다.
  - rights/deletion/consent 상태가 차단 상태이면 새 playback URL을 발급하지 않는다. 이미 발급된 URL은 TTL 만료까지의 잔여 위험을 줄이기 위해 P0 기본 TTL을 짧게 둔다.
  - `generated_preview` 또는 `own_voice_processing` consent가 철회되면 기존 preview는 즉시 `playback_blocked=true`가 되고 새 signed playback URL을 발급하지 않는다.
- Dependencies:
  - minimal self-voice preview engine path
  - canonical job state owner
  - render storage
  - app playback component
  - Safety Rights
  - signed access URL mechanism
- Success signal:
  - metric-eligible non-mock P0 section jobs의 60% 이상이 `completed`, `preview_available=true`, `section_limited=true`, `playback_blocked=false`에 도달한다.
  - distinct participating learner 10명 중 5명 이상이 첫 metric-eligible played preview의 primary question에서 4점 이상을 준다.
  - 2점 이하 응답이 2명을 초과하지 않는다.

### 7. 표준 작업 상태 및 부분 산출물 처리 (Canonical Job State And Partial Artifact Handling)

- User action: 사용자가 job 생성 후 처리 상태를 확인하거나, 실패/부분 성공 결과를 확인한다.
- Product behavior: 시스템은 `worker` repo의 `conversion-job-state` bounded module을 P0 canonical job state owner로 두고, stage result를 idempotent하게 반영해 app-facing job status를 제공한다. BFF와 API Gateway는 최종 상태를 임의로 invent하지 않는다.
- Business rules:
  - P0는 신규 Conversion Job Orchestrator 서비스를 필수로 만들지 않고 `worker` bounded module로 시작한다.
  - canonical state는 `created`, `queued`, `processing`, `preview_ready`, `completed`, `failed`, `failed_with_partial_artifacts`, `blocked`, `needs_review`, `deleted`를 지원한다.
  - output availability는 `preview_available`, `pitch_report_available`, `section_limited`, `preview_played`, `rating_required`, `failure_tags_required`로 state와 분리 추적한다.
  - `createAudioUploadSession`은 임시 upload session을 만드는 단계이며 conversion job 생성으로 간주하지 않는다.
  - `completeAudioUpload`는 upload session을 P0 section preview job으로 커밋하는 유일한 boundary이며, 이 시점 전에는 engine processing 요청을 발행하지 않는다.
  - `created` job state는 `completeAudioUpload` 이후에만 사용한다. 업로드 전 진행 상태는 `upload_session_id`로 추적한다.
  - engine worker는 job id, stage, status, artifact refs, error reason, confidence summary, timing metadata를 emit한다.
  - duplicate stage result는 idempotent하게 처리한다.
- Permissions / roles:
  - `learner`: 본인 job status와 result projection 조회
  - `educator_or_expert`: 동의된 job status와 reviewable result 조회
  - `engine_developer`: stage result와 retry/debug metadata 조회
  - `product_qa`: status distribution과 failure distribution 조회
- States:
  - `created`
  - `queued`
  - `processing`
  - `preview_ready`
  - `completed`
  - `failed`
  - `failed_with_partial_artifacts`
  - `blocked`
  - `needs_review`
  - `deleted`
- Edge cases:
  - duplicate worker event가 들어와도 user-facing completion이 중복 생성되지 않는다.
  - job이 `completeAudioUpload.committed_at` 기준 60분을 초과하면 job-state owner가 synthetic timeout StageResult를 기록하고 terminal state로 전환한다.
  - `blocked` 상태는 user-facing 정책 이유를 보여주되 민감한 내부 rule은 노출하지 않는다.
  - deletion 이후에는 playable output을 제공하지 않는다.
- Dependencies:
  - job state table/module
  - stage result records or event stream
  - BFF GraphQL job status API
  - app job status UI
  - retry policy
  - retention deadline tracking
- Success signal:
  - 100%의 P0 jobs가 final state와 output availability flags를 가진다.
  - BFF는 canonical projection을 읽고 final state를 독자적으로 결정하지 않는다.
  - P50 10분, P95 30분, timeout 60분은 observation metric으로 기록된다.

### 8. 결과 리뷰, 평가, 실패 원인 태깅 (Result Review, Rating, And Failure Tagging)

- User action: 학습자가 preview를 들은 직후 "이 preview가 내 목소리처럼 들린다"를 1-5점으로 평가하고, 4점 미만이면 실패 원인을 태그한다. 내부 reviewer는 기술 원인을 보강한다.
- Product behavior: 결과 화면은 preview playback을 가장 먼저 제공한다. 동일 preview artifact를 의미 있게 재생해 `preview_played=true`가 된 뒤 rating을 요청하고, pitch feedback과 quality report는 준비되는 대로 붙인다. 4점 미만 rating은 failure tag 없이는 제출 완료로 보지 않는다.
- Business rules:
  - primary rating question은 "이 preview가 내 목소리처럼 들린다"다.
  - 4점 이상만 primary self-voice metric success로 계산한다.
  - `preview_played`는 동일 `preview_artifact_id`에 대해 인증된 playback session이 오류 없이 preview 고유 timeline의 80% 이상에 도달한 이벤트로 정의한다.
  - P0 `Mist intro`는 28초이므로 `preview_played=true` 기준은 최소 22.4초 이상의 distinct timeline coverage다.
  - 같은 3초 구간을 반복 재생한 시간은 `unique_timeline_coverage`로 중복 계산하지 않는다.
  - 앱은 foreground, unmuted playback 중 `playback_progress`를 1초마다 전송하고 pause, seek, end, background, mute, error 시 progress를 flush한다.
  - 모든 playback event는 `event_id`, `schema_version`, `occurred_at`, session 내 단조 증가 `client_sequence`를 포함한다.
  - 서버는 `playback_session_id`, `artifact_id`, `event_id`, `client_sequence`, played range를 기준으로 duplicate/out-of-order event를 idempotent하게 merge한다.
  - `preview_played=true` requires `preview_artifact_id`, `pipeline_mode` in `partial_real` or `real_synthesis`, `mock_fixture_used=false`, `section_limited=true`, `playback_blocked=false`, `playback_session_id`, signed playback URL audit success, app foreground playback start, `player_muted=false`, no severe playback error, and `unique_timeline_coverage_ratio >= 0.8`.
  - `rating_required=true`는 `preview_available=true`, `playback_blocked=false`, `preview_played=true`, 해당 artifact에 rating이 없을 때만 켠다.
  - `failure_tags_required=true`는 rating 1-3점 제출 직후 켠다. 4-5점은 failure tag를 강제하지 않는다.
  - 4점 미만 failure tag taxonomy v0.1은 `not_my_voice`, `not_song_like`, `pitch_wrong`, `timing_wrong`, `robotic_or_artifact`, `noise_or_clipping`, `too_short_or_incomplete`, `other`로 P0 confirmed한다. 재생 실패는 rating/failure tag가 아니라 `playback_problem_reported` event로 분리한다. 내부 운영 후 taxonomy version을 올려 조정할 수 있다.
  - `other`는 선택형 자유입력이며 개인정보/민감정보 유입 검토가 필요하다.
  - internal reviewer technical tags는 user perception tags와 분리한다.
  - 완료된 job은 preview, section label, pitch report, low-confidence sections, quality report를 보여준다.
- Permissions / roles:
  - `learner`: 본인 rating과 failure tag 제출
  - `educator_or_expert`: 동의된 job에 교육적 검토 코멘트 또는 technical-review 보조 입력
  - `engine_developer`: technical tags와 stage failure analysis 입력
  - `product_qa`: metrics와 failure taxonomy 분석
- States:
  - `result_available`
  - `rating_required`
  - `failure_tags_required`
  - `rating_submitted`
  - `review_pending`
  - `review_completed`
- Edge cases:
  - preview를 재생하지 못한 사용자는 rating 없이 `playback_problem_reported` event를 제출할 수 있어야 한다. 이 event는 primary self-voice rating numerator에 포함하지 않는다.
  - preview가 없으면 primary self-voice rating을 요청하지 않는다.
  - 4점 이상 rating에는 failure tag를 강제하지 않는다.
  - user failure tag는 root cause로 확정하지 않는다.
  - quality report 생성이 실패해도 preview/render 실패 state를 숨기지 않는다.
- Dependencies:
  - app result screen
  - playback event tracking
  - rating/failure tag API
  - internal review surface or admin-only review path
  - observability store
- Success signal:
  - 4점 미만 rating의 100%가 failure tag 또는 `other`를 포함한다.
  - completed job의 100%가 preview status, pitch/confidence report, clipping/artifact report를 가진다.
  - user perception tags와 technical tags가 분리 저장된다.
  - primary learner success는 valid P0 section job을 제출한 distinct participating learner 중 첫 metric-eligible played preview rating이 4점 이상인 사용자 수로 계산한다.

### 9. 안전성, 감사, 보관, 삭제 거버넌스 (Safety, Audit, Retention, And Deletion Governance)

- User action: 사용자는 동의 상태를 확인하고, 본인 데이터 삭제를 요청할 수 있다. 운영/보안 담당자는 retention, deletion, audit evidence를 검토한다.
- Product behavior: 시스템은 voice rights, reference rights, app-only playback, data retention, deletion evidence를 job/artifact 단위로 기록한다. audit이 실패하면 rights-sensitive operation은 fail closed한다.
- Business rules:
  - NIST CSF risk management 관점으로 data class별 risk와 owner를 둔다.
  - deletion은 NIST SP 800-88 sanitization 개념을 기준으로 recovery infeasible 상태를 목표로 한다.
  - OWASP Logging Cheat Sheet 기준으로 raw audio, playable preview URL, token, password, provider secret, full lyrics, 민감 free-text를 로그에 남기지 않는다.
  - user raw voice audio, canonical audio, generated preview는 기본 30일 보관한다.
  - raw audio 없는 analysis/audit/metrics/failure data는 최대 1년 보관 후보로 둔다.
  - deletion evidence는 artifact id, data class, request source, retention deadline, deletion job id, status, timestamp, actor/service id를 포함하고 raw audio나 playable preview는 포함하지 않는다.
  - deletion evidence는 `deletion_scope`, `storage_system`, `object_version_status`, `replica_backup_scope`, `deletion_method`, `verification_result`, `blocked_access_at`, `completed_at`을 포함한다.
  - deletion 실패 시 `deletion_failed`로 표시하고 rights-sensitive playback을 차단한다.
  - audit write가 실패하면 rights clearance/exposure/block, consent create/revoke, analysis/preview request, signed playback URL 발급, deletion request/job, rights complaint block, break-glass grant는 fail closed한다.
  - raw/canonical audio에 대한 human break-glass 접근은 incident/ticket id, purpose, approver, second reviewer 또는 security owner, TTL, least-privilege grant, immutable audit, post-access review를 요구한다.
- Permissions / roles:
  - `learner`: 본인 데이터 retention notice 확인과 삭제 요청
  - `platform_storage_owner`: deletion job과 storage policy 운영
  - `policy_or_security_owner`: rights/audit/incident review
  - `engine_developer`: 승인된 debug 목적의 제한 접근
  - `product_qa`: 비식별 metrics 접근
- States:
  - `retention_active`
  - `deletion_scheduled`
  - `deletion_running`
  - `deleted`
  - `deletion_failed`
  - `playback_blocked`
  - `audit_failed`
- Edge cases:
  - retention deadline 이후 24시간 내 deletion job이 실패하면 owner review를 생성한다.
  - rights complaint가 발생하면 삭제 완료 전이라도 playback을 먼저 차단한다.
  - candidate data consent가 없으면 해당 데이터를 future model-improvement 후보 dataset에 넣지 않는다.
  - audit logging 실패 시 rights-sensitive operation을 진행하지 않는다.
- Dependencies:
  - consent store
  - audit log store
  - object storage lifecycle/deletion jobs
  - access-control policy
  - structured logging
  - deletion evidence table/report
- Success signal:
  - 100%의 analysis/preview requests가 allow/deny decision과 audit record를 가진다.
  - raw audio artifact의 100%가 30일 기본 retention과 deletion deadline을 가진다.
  - deletion deadline 이후 24시간 내 deletion job 실행 여부가 확인 가능하다.
  - raw audio가 1년 retained dataset에 포함되지 않는다.

## P0 Review Remediation Decisions

| Review Issue | Decision | Remaining User Decision |
| --- | --- | --- |
| P0 화면 범위와 내부 운영 범위가 섞임 | P0 화면은 learner core flow, 제한된 admin 화면, 별도 educator/expert review 화면으로 둔다. full admin console은 Out of Scope다. | 없음 |
| `Mist intro` timestamp가 확정처럼 보임 | `0:00-0:28`은 사용자가 확인한 P0 target timestamp로 확정한다. 실제 등록 reference audio asset이 바뀌면 재검수한다. | 없음 |
| Failure tag 목록이 질문과 충돌 | failure tag taxonomy v0.1은 P0 confirmed로 둔다. 내부 운영 후 versioning으로 조정한다. | 없음 |
| BPM/key correction 범위가 모호함 | P0 기본값은 admin-only 수정으로 두되, P0 내부 운영에서는 allowlist 대상 학습자에게 feature flag로 learner correction controls를 켠다. 수정은 `completeAudioUpload` 전까지만 가능하고 job에는 snapshot을 저장한다. | 없음 |
| Job state owner 미확정 | P0 canonical owner는 `worker` repo의 `conversion-job-state` bounded module로 둔다. | 실제 구현 owner 팀/담당자 |
| Recorder/upload storage 계약 미확정 | P0는 app recorder take와 fallback file을 모두 presigned URL/object storage 경로로 전송한다. `completeAudioUpload`가 job 생성 커밋 boundary다. 서버가 consent/rights snapshot을 직접 계산한다. Upload TTL은 15분, playback TTL은 5분으로 시작한다. | 없음 |
| 녹음 화면 권리 플래그 미정 | Reference pre-listen, lyrics display, lyrics sync는 기본 차단하고, 명시적 권리 플래그와 scope가 있을 때만 section-limited로 노출한다. 활성 녹음 중 reference 동시 재생은 P0에서 제외한다. | 실제 rights/risk record 작성 |
| Mock/partial-real/real synthesis 기준 미확정 | mock은 UI/flow 검증 전용이며 P0 self-voice success로 계산하지 않는다. `partial_real`은 real submitted voice input lineage와 app-playable section-limited preview가 machine-checkable할 때만 P0 success 후보로 인정한다. | 없음 |
| Admin/reviewer path 미확정 | 제한된 admin 화면과 별도 educator/expert review 화면을 P0에 포함한다. | 화면 상세 flow는 page-flow-planner에서 확정 |
| 연락 목적 개인정보 처리 | P0는 disabled contact follow-up capability status만 정의한다. `other` 자유입력은 feedback 텍스트로만 쓰며, SNS/메일 발송용 연락처 UI, 값 저장, 발송, 복호화, export는 P0 범위에서 제외한다. | 수익화/외부 beta 전 연락처 수집 재검토 |
| owner 팀 부재 | P0에서는 개발자인 사용자가 deletion owner, policy owner, platform/storage owner를 겸임한다. 단, 자기 승인 break-glass와 contact plaintext reveal은 허용하지 않는다. | second reviewer 확보 여부 |
| 권리 미확보 `Mist` learner 노출 | `unlicensed_internal_risk`는 learner 노출 금지 상태로 두고, risk acceptance가 기록된 `unlicensed_internal_risk_accepted`에서만 internal allowlist에 제한 노출한다. | 실제 risk acceptance record 작성 |
| Rating timing 모호함 | `preview_played=true` 이후에만 `rating_required=true`가 된다. `completed`는 report 준비 상태이며 rating 완료 상태가 아니다. | 없음 |

## P0 Surface Boundary

| Actor | P0 Surface | P0 Internal Path | Later | Out of Scope |
| --- | --- | --- | --- | --- |
| Learner | signup/login, access eligibility check, `Mist` song selection, `intro` section selection, consent snapshot, in-app recording, take review, fallback upload when enabled, BPM/key correction before job behind feature flag, processing status, preview playback, pitch feedback, rating/failure tags, deletion request | 없음 | progress history, multi-song practice, full correction controls, guided reference playback during active recording after rights/acoustic validation | reference audio upload, export/share |
| Admin | 제한된 admin 화면에서 reference audio 직접 업로드, song package 등록, section map 확인, rights/risk status 표시, publish/block 처리 | seed/script fallback은 개발 초기 보조 수단으로만 사용 | full song catalog console, provider automation, user song upload moderation | general-purpose operator console |
| Educator/Expert | 별도 review 화면에서 동의된 job의 preview, pitch report, failure tags, quality summary 검토 | raw audio 직접 접근 없이 reviewable result만 확인 | review queue 고도화, student progress dashboard | raw audio 기본 접근 |
| Engine Developer | 없음 | stage result, artifact refs, review 화면의 internal reviewer mode technical tags, debug-only limited access | richer observability dashboard | unrestricted raw audio access |
| Product/QA | 없음 | pseudonymous metrics, rating/failure taxonomy, P0 report | analytics dashboard | user-identifiable raw audio review |
| Security/Ops | 없음 | audit, deletion evidence, rights complaint block, break-glass approval/review | incident console | unaudited data access |

## Learner Core Flow And Metric Contract

P0 learner core flow:

```text
signup/login -> access eligibility check -> Mist song selection -> intro section selection -> BPM/key 확인 또는 수정 -> current consent snapshot 확인 -> recorder ready -> record take -> take review -> completeAudioUpload -> processing -> preview_ready -> playback -> rating -> failure tags if rating < 4
```

Job result flow는 evaluation flow와 분리한다. `completed`는 preview와 required report가 준비된 job state이고, rating 제출 여부와 무관하게 result/deletion controls를 제공한다.

Metric eligibility:

- Job completion denominator: required consent, recording/upload validation, section validation, rights gate를 통과해 processing에 들어간 non-mock P0 `Mist intro` section jobs.
- Job completion numerator: final state `completed`, `preview_available=true`, `section_limited=true`, `playback_blocked=false`인 jobs.
- Job completion exclusions: mock jobs, admin/internal QA jobs, duplicate worker attempts, valid job 생성 전 rejected takes/uploads, non-P0 sections.
- Job completion non-exclusions: engine failure, timeout, post-start blocked state, playback failure는 실패 또는 non-success로 계산하며 조용히 제외하지 않는다.
- Primary learner success denominator: valid P0 section job을 제출한 distinct participating learners. 10명 미만이면 "5 of 10" 성공 선언을 하지 않는다.
- Primary learner success numerator: 첫 metric-eligible played preview에서 primary rating 4점 이상을 준 distinct learners.
- Metric-eligible played preview requires `preview_artifact_id`, `section_id=intro`, `section_limited=true`, `preview_played=true`, `playback_blocked=false` at playback time, `pipeline_mode` in `partial_real` or `real_synthesis`, `mock_fixture_used=false`, `playback_session_id`, prompt version, and 1-5 rating.
- Failed jobs and unplayed previews are non-success, not silent exclusions.

## Rights Evidence Checklist

원칙적으로 P0에서 reference audio 또는 generated preview를 learner에게 노출하려면 다음 evidence가 모두 있어야 한다.

사용자 결정: 현재 P0에서는 별도 음원 사용 허가나 권리 evidence가 없다. 또한 현재 목적은 수익 창출이 아니라 내부 검증이며, 수익화 시점에 정식 음원 사용 권리를 확보할 계획이다. 제품 정책상 이것은 rights clearance가 아니라 `unlicensed_internal_risk` 상태로 본다. 이 상태 자체는 learner-facing 선택/preview를 허용하지 않는다. 앱에서 internal allowlist 대상에게 제한 노출하려면 `unlicensed_internal_risk_accepted`로 전환하고, 개발자/정책 owner가 risk acceptance record를 남겨야 한다.

`unlicensed_internal_risk_accepted` 상태에서도 public launch, paid use, marketing, export/share, model training, provider audio ingestion, 외부 배포를 허용하지 않는다. 내부 운영이라도 법적 위험이 사라지는 것은 아니며, allowlist, app-only playback, no export는 노출면을 줄일 뿐 권리 미확보 자체를 해결하지 않는다.

Policy note: non-commercial or research-oriented use does not automatically make copyrighted music use safe. Fair use is fact-specific, and provider terms can independently restrict analysis, caching, remixing, synchronization, or machine-learning use. P0 therefore treats unlicensed reference audio as a risk exception, not a cleared asset. Sources: [U.S. Copyright Office Fair Use Index](https://www.copyright.gov/fair-use/), [YouTube API Services Terms](https://developers.google.com/youtube/terms/api-services-terms-of-service), [Spotify Developer Policy](https://developer.spotify.com/policy).

| Evidence | Required | Notes |
| --- | --- | --- |
| source/provenance | Yes | reference audio asset의 출처와 등록 경로 |
| license/provider terms reference | Yes | 내부 분석과 앱 내 playback이 허용되는 근거 |
| allowed uses | Yes | pitch extraction, quality comparison, internal review 등 |
| prohibited uses | Yes | download, share, training, remix, third-party voice cloning 등 |
| reference pre-listen flag/scope | Yes | default false. 허용 시 section-limited, signed, audited, app-only |
| lyrics display/sync flag/scope | Yes | default false. 허용 시 section/line scope와 sync confidence 필요 |
| rights approver | Yes | 실제 담당자 또는 승인 팀 |
| approval timestamp | Yes | 승인 시점 |
| expiry/re-review date | Yes | 권리 만료 또는 재검토 기준 |
| retention period | Yes | reference audio 보관 기준 |
| deletion owner | Yes | 삭제 실행 책임 |
| complaint owner | Yes | 권리 이슈 접수 후 차단/검토 책임 |
| evidence storage location | Yes | 문서, DB record, ticket 등 실제 저장 위치 |

Rights state transition:

```text
draft -> metadata_incomplete -> rights_pending -> approved_for_internal_analysis -> published
draft -> unlicensed_internal_risk
unlicensed_internal_risk -> unlicensed_internal_risk_accepted
unlicensed_internal_risk -> rights_pending
unlicensed_internal_risk -> rights_blocked
unlicensed_internal_risk_accepted -> rights_pending
unlicensed_internal_risk_accepted -> rights_blocked
draft -> rights_blocked
rights_pending -> rights_blocked
published -> under_review -> rights_blocked
published -> retired
```

`published` 전에는 learner selection과 preview processing을 허용하지 않는 것이 원칙이다. 단, `unlicensed_internal_risk_accepted`는 개발자 owner가 명시적으로 risk acceptance를 기록한 제한된 내부 운영에서만 사용할 수 있다. 이 예외는 internal allowlist, feature flag, audit write success가 모두 충족될 때만 적용되며 수익화, 공개 beta, 외부 고객 제공, 음원/preview 다운로드, 공유, 학습 데이터 사용에는 적용되지 않는다.

P0 risk acceptance record는 제한된 admin 화면의 rights/risk record에 저장한다. 해당 화면이 구현되기 전에는 PM 문서 또는 ticket에 임시로 남기고, admin 화면 구현 후 migration한다. record에는 `risk_acceptance_id`, `song_package_id`, `reference_asset_id`, checksum, 승인자, 범위, 기간, 최대 참여자 수, `allowed_user_ids` 또는 `allowed_group_id`, exact section, 금지 용도, retention, deletion owner, complaint owner, kill-switch owner, 재검토일을 포함한다.

Exposure rule:

- `draft`, `metadata_incomplete`, `rights_pending`: learner 선택 불가.
- `approved_for_internal_analysis`: 내부 분석/검수 가능, 앱 learner preview 불가.
- `unlicensed_internal_risk`: 권리 미확보 risk 상태. learner 노출 없음.
- `unlicensed_internal_risk_accepted`: risk acceptance가 기록된 경우에만 P0 internal allowlist에 제한 노출.
- `published`: 권리 evidence 완비 후 일반 learner 선택 가능.
- `under_review`, `rights_blocked`: 선택, processing, playback URL 발급, 기존 preview playback 모두 차단.
- `retired`: 신규 job 차단. `unlicensed_internal_risk_accepted`에서 retired된 경우 기존 playback도 차단한다. `published`에서 retired된 경우에만 권리 조건이 허용할 때 기존 playback 유지 여부를 별도 판단한다.

Selection, job creation, engine processing, signed playback URL 발급은 매번 rights state, internal allowlist 여부, consent snapshot, audit write 성공 여부를 재검증한다.
Internal allowlist에는 reference audio, lyrics, generated preview, app output의 외부 녹음/캡처, 화면 녹화, 재배포, 공개 게시, 상업적 사용 금지에 동의한 사용자만 포함한다. 이 제한은 사용자의 본인 목소리 녹음 자체를 금지한다는 뜻이 아니다.

Recording guide exposure rule:

- `reference_prelisten_allowed=false`: 녹음 화면에서 원곡/reference audio 재생 UI를 숨기거나 disabled 처리한다.
- `reference_prelisten_allowed=true`: selected section만 pre-listen으로 재생할 수 있으며, signed URL 발급과 playback event는 audit 대상이다.
- Active microphone recording starts: reference pre-listen playback must stop before recording begins.
- `lyrics_display_allowed=false`: full lyrics와 line lyrics를 표시하지 않는다. section label, timestamp, expected duration은 표시할 수 있다.
- `lyrics_display_allowed=true`: selected section 또는 허용된 line cue만 표시한다.
- `lyrics_sync_allowed=false`: time-synced lyric UI처럼 보이는 진행형 highlight를 제공하지 않는다.

P0 reference audio file handling:

- Reference audio는 사용자가 제한된 admin 화면에서 직접 업로드한다.
- 학습자나 일반 사용자의 reference audio upload는 P0에서 허용하지 않는다.
- 업로드 시 file hash, original filename, uploader id, upload timestamp, source note, risk state, reference pre-listen flag/scope, lyrics display/sync flag/scope, retention deadline을 기록한다.
- 수익화 또는 외부 beta 전 정식 음원 권리 확보 방식은 deferred productization task로 남긴다.

## Signup And Recording Verification Direction

회원가입 인증은 "계정 소유자 확인"이고, recording/upload 권리 확인은 "제출 파일과 목소리를 사용할 권리 확인"이다. 따라서 강한 회원가입 인증을 도입해도 reference audio 권리나 본인 음성 사용 권한이 자동으로 해결되지는 않는다.

P0 internal operation:

- 일반 학습자는 email/password + email verification으로 시작한다.
- 관리자, 교육자/전문가, 내부 reviewer는 email verification 이후 passkey 또는 OTP 기반 MFA를 요구한다.
- Reference audio는 제한된 admin 화면에서 개발자인 사용자가 직접 업로드한다.
- 본인 음성 확인, generated preview 동의, expert review 동의, retention notice는 회원가입 또는 최초 로그인 시 1회 받을 수 있다.
- 단, P0 job 생성 시에는 항상 `consent_snapshot_id`, `consent_snapshot_hash`, `policy_document_version`, `policy_document_hash`, `scope`, `granted_at`을 job에 저장한다.
- 현재 consent version/scope가 맞지 않거나 withdrawal 상태이면 recorder submit과 `completeAudioUpload`를 차단하고 재동의를 요구한다.

Future beta/product:

- passkey/WebAuthn을 우선 로그인 옵션으로 추가한다.
- 민감 데이터 접근, admin publish, decrypt contact, break-glass 같은 고위험 action에는 step-up authentication을 요구한다.
- 필요 시 active voice verification을 recording/job 단위 보조 검증으로 추가한다. 음성 자체를 단독 로그인 수단으로 쓰지는 않는다.

Sources: [NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b.html), [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html), [W3C WebAuthn](https://www.w3.org/TR/webauthn-3/).

## Consent Lifecycle

| Consent Type | Required For P0 Job | Default | Collection Timing | Withdrawal Behavior |
| --- | --- | --- | --- | --- |
| `own_voice_processing` | Yes | unchecked | signup/first login allowed, job snapshot required | 철회 즉시 새 analysis/preview job 생성 차단, 기존 voice-bearing artifact playback 차단, deletion job 예약 |
| `generated_preview` | Yes | unchecked | signup/first login allowed, job snapshot required | 철회 즉시 새 preview 생성 차단, 기존 preview playback 차단, 새 signed playback URL 발급 차단, deletion job 예약 |
| `expert_review` | Yes for P0 job | unchecked | signup/first login allowed, job snapshot required | 철회 즉시 educator/expert 접근 차단, review queue에서 숨김 |
| `candidate_data_opt_in` | No | unchecked | optional, can be changed later | candidate dataset에서 제외하고 기존 candidate labels는 `withdrawn`으로 표시 |
| `retention_notice_ack` | Yes | unchecked | signup/first login allowed, job snapshot required | 미동의 시 P0 job 생성 차단 |
| `contact_for_followup` | No | unavailable | not collected in P0; optional and separated only when enabled later | P0에서는 연락처 값이 없으므로 철회/삭제 대상 없음 |

Consent record는 최소 `consent_type`, `version`, `scope`, `required`, `granted_at`, `withdrawn_at`, `source_job_id`, `source_session_id`, `policy_document_version`, `policy_document_hash`를 가진다. Job consent snapshot은 `snapshot_id`, `snapshot_hash`, selected song/section, reference/lyrics display flags, policy version을 함께 저장한다. Candidate data opt-in 철회는 operational audit과 deletion evidence 보관 의무까지 자동 삭제한다는 뜻이 아니다. 다만 future model-improvement dataset에는 포함하지 않는다.

Consent withdrawal effects:

| Withdrawal | Immediate Block | Deletion Target |
| --- | --- | --- |
| `own_voice_processing` | 새 analysis, 새 preview, 기존 generated preview playback, educator/expert access | user raw audio, canonical audio, generated preview |
| `generated_preview` | 새 preview, 기존 generated preview playback, 새 signed playback URL, educator/expert preview access | generated preview |
| `expert_review` | educator/expert review surface access | review visibility only. audit/evidence는 보관 |
| `candidate_data_opt_in` | future model-improvement dataset inclusion | candidate dataset copy/label withdrawn marker |
| `contact_for_followup` | Later only: follow-up/error/interview 발송 | Later only: contact ciphertext, contact hash, contact lookup index |

Voice-bearing artifact deletion은 철회 후 24시간 내 deletion job을 예약하고, 7일 내 완료를 목표로 한다. audit/deletion evidence와 irreversible aggregate metric은 raw audio나 playable preview 없이 최대 1년 보관할 수 있다.

SNS, 메일 등 추후 연락을 위한 개인정보는 failure tag의 `other` 자유입력에 받지 않는다. 나중에 contact collection gate를 켤 경우에만 연락처를 별도 필드와 별도 동의로 수집한다. 표시 화면에서는 마스킹하고, 발송 등 필요한 목적에서는 권한 있는 backend/service만 복호화할 수 있도록 암호화 저장한다. 복호화 접근은 audit 대상이다.

Future contact collection gate:

- Runtime status: hidden/disabled by default. No contact UI, contact value storage, or sending is allowed in P0 unless this gate is explicitly enabled in a later decision.
- Contact channels when later enabled: email, SNS account
- Allowed purposes: follow-up, 오류 안내, 인터뷰 요청
- Not allowed without separate consent: marketing, 광고성 메시지, 제3자 제공
- Retention: 내부 운영 기간 동안 보관하고 내부 운영 종료 후 90일 이내 삭제한다.
- Withdrawal when later enabled: 사용자가 `contact_for_followup` 동의를 철회하면 발송을 즉시 중지하고, 연락처 암호문과 검색용 hash는 30일 이내 삭제한다.
- Evidence: 동의, 철회, 삭제 evidence는 raw contact value 없이 최대 1년 보관할 수 있다.
- Signup/login email은 follow-up contact로 자동 재사용하지 않는다. 같은 이메일을 쓰더라도 별도 unchecked opt-in과 목적 고지가 필요하고, SNS 또는 다른 이메일은 별도 confirm이 필요하다.
- Contact collection은 P0에서 disabled capability status만 정의하고 기본 disabled로 둔다. 연락처 UI, 연락처 값 저장, 발송, 복호화, export는 P0에서 제공하지 않으며, P0 preview/rating flow는 연락처 수집 없이 진행한다.

## Contact Data Encryption And Key Management Guide

연락처는 마스킹만으로는 충분하지 않다. 실제 발송이 필요하므로 복호화 가능한 암호화 저장을 사용하되, 평문 노출 경로를 최소화한다. OWASP Cryptographic Storage Cheat Sheet는 민감정보 저장 최소화, 인증된 암호화 모드, key와 data 분리, key rotation을 권장한다. OWASP Secrets Management Cheat Sheet와 NIST SP 800-57은 key lifecycle과 운영 절차를 별도 관리 대상으로 본다. Sources: [OWASP Cryptographic Storage](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html), [OWASP Secrets Management](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html), [NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final).

Gate-ready storage model:

- `contact_type`: `email` or `sns`
- `contact_label`: `email`, `x`, `instagram`, `discord`, etc.
- `contact_masked`: UI 표시용. 예: `ab***@example.com`, `@ab***`
- `contact_hash`: lowercased/normalized contact value의 keyed hash. 중복 확인과 검색용이며 원문 복구용이 아니다.
- `contact_ciphertext`: AES-256-GCM 등 authenticated encryption으로 암호화한 값
- `key_id`: 어떤 key로 암호화했는지
- `consent_type`: `contact_for_followup`
- `allowed_purposes`: `followup`, `error_notice`, `interview_request`
- `consent_version`, `granted_at`, `withdrawn_at`
- `retention_deadline`: 내부 운영 종료 후 90일 또는 동의 철회 후 30일 중 더 이른 시점

P0 key management:

- P0에서는 개발자인 사용자가 key owner 운영 책임을 가진다. 단, 자기 승인 방식의 contact plaintext reveal은 허용하지 않는다.
- key는 Git, 문서, 로그, DB dump에 저장하지 않는다.
- contact collection을 켜려면 cloud KMS, Vault, OS keychain 또는 동등한 key management service가 필요하다.
- `.env`만으로 보관한 임시 secret은 contact collection gate를 통과하지 못한다.
- 복호화는 backend/service 경로에서만 수행하고, admin UI에는 기본적으로 `contact_masked`만 표시한다.
- 실제 발송이 필요한 순간에도 admin UI에 평문을 노출하지 않고, backend가 purpose-bound template으로 단일 수신자 발송을 수행한다.
- backend/service 복호화 시 `who`, `when`, `purpose`, `contact_id`, `request_id`를 audit log에 남긴다.
- 대량 export나 CSV 다운로드는 P0에서 허용하지 않는다.
- key rotation은 최소 내부 운영 종료 시점과 key compromise 의심 시 수행한다.
- key compromise가 의심되면 새 key를 만들고 기존 contact data를 재암호화하거나 old key를 retired 상태로 관리한다.

Contact collection gate:

- P0 default is `contact_collection_enabled=false`.
- `contact_collection_enabled=true`는 authenticated encryption, keyed hash, KMS/Vault/OS keychain 또는 동등 secret store, audit write success, no plaintext UI, no CSV export가 모두 준비된 경우에만 Later decision으로 허용한다.
- second reviewer 또는 별도 decrypt approver가 없으면 contact plaintext reveal은 disabled다.
- single-recipient service-mediated send는 allowed purpose와 consent가 일치하고 audit write가 성공할 때만 허용한다.
- contact deletion 대상은 ciphertext, keyed hash, masked value, lookup/search index, send queue payload, pending campaign target, UI cache, vendor/send logs의 직접 식별값을 포함한다.

Productization separation:

- Key custodian: KMS/Vault key 생성, rotation, disable 권한 담당
- Decrypt approver: 대량 발송 또는 민감 접근 승인 담당
- App operator: 발송 캠페인 실행 담당, key 원문 접근 불가
- Auditor/security reviewer: decrypt audit과 목적 외 사용 감시 담당

제품화 전에는 최소한 key custodian과 app operator를 분리해야 한다.

## P0 Job State Contract

- Owner: `worker` repo의 `conversion-job-state` bounded module
- Read path: BFF는 API Gateway 또는 internal read model을 통해 app-facing projection만 읽는다.
- Write path: recorder/fallback upload job 생성과 stage result events가 canonical job state module에 반영된다.
- Idempotency key: upload session id 또는 job id + stage + attempt를 기준으로 중복 stage result를 무해하게 처리한다.
- Timeout: `completeAudioUpload.committed_at` 기준 60분 초과 시 job-state owner가 synthetic timeout StageResult를 기록하고, preview 존재 여부에 따라 `failed` 또는 `failed_with_partial_artifacts`로 terminalize한다.

Canonical states:

| State | Meaning | User-facing CTA |
| --- | --- | --- |
| `created` | `completeAudioUpload`가 job record를 커밋했고 engine request 발행 전이거나 발행 대기 중 | 상태 보기 |
| `queued` | engine 대기 중 | 상태 보기 |
| `processing` | 하나 이상의 stage 처리 중 | 상태 보기 |
| `preview_ready` | preview playback 가능, 일부 report는 진행 중일 수 있음 | 프리뷰 재생 |
| `completed` | preview와 required report 준비됨. rating 제출 여부와는 분리됨 | 프리뷰 재생, 평가 제출 |
| `failed` | preview 생성에 실패했고 P0 success가 될 수 없음 | 재시도 또는 재업로드 |
| `failed_with_partial_artifacts` | 일부 artifact 존재. preview 가능 여부에 따라 CTA 분기 | 가능한 결과 보기, 재시도 |
| `blocked` | consent, rights, policy, audit, deletion 문제로 차단됨 | 차단 이유 확인 |
| `needs_review` | 내부 검토 전에는 결과 공개를 보류해야 함 | 대기 또는 문의 |
| `deleted` | playable artifact 삭제됨 | 새 job 생성 |

Output flags:

- `preview_available`
- `pitch_report_available`
- `section_limited`
- `preview_played`
- `rating_required`
- `failure_tags_required`
- `playback_blocked`
- `deletion_pending`

Learner-facing labels:

| Canonical State | Learner Label | Rating Rule |
| --- | --- | --- |
| `preview_ready` | 프리뷰 준비됨 | `preview_played=true` 이후 rating 요청 |
| `completed` | 결과 준비됨 | 미평가면 rating CTA 유지 |
| `failed` | 프리뷰 생성 실패 | rating 요청 안 함 |
| `failed_with_partial_artifacts` | 일부 결과만 준비됨 | preview 가능할 때만 rating 가능 |
| `blocked` | 처리 또는 재생 중지됨 | rating 요청 안 함 |
| `needs_review` | 검토 중 | release 전 rating 요청 안 함 |

Blocked and failed CTA matrix:

| Reason | User-facing CTA | Notes |
| --- | --- | --- |
| `consent_missing` | 동의 확인 | 사용자가 해결 가능 |
| `rights_blocked` | 다른 곡 선택 또는 문의 | package/job/playback 모두 차단 |
| `audit_failed` | 나중에 다시 시도 또는 문의 | rights-sensitive operation은 fail closed |
| `playback_blocked` | 동의/권리 상태 확인 | 기존 preview도 새 URL 발급 금지 |
| `deletion_failed` | 삭제 상태 보기 또는 문의 | playback 먼저 차단 |
| `unsupported_format` | 파일 다시 선택 | `.wav`, `.mp3` 안내 |
| `duration_exceeded` | 파일 다시 선택 또는 trim | 60초 hard max |
| `section_mismatch` | 다시 업로드 또는 잘라서 업로드 | full-song 조용한 처리 금지 |
| `synthesis_failed` | 재시도 또는 문의 | preview 없으면 rating 없음 |
| `render_failed` | 재시도 또는 문의 | preview 없으면 rating 없음 |
| `timeout` | 재시도 또는 문의 | timeout terminal rule 적용 |

Partial artifact CTA matrix:

| Condition | Primary CTA | Rating Rule |
| --- | --- | --- |
| `failed_with_partial_artifacts`, `preview_available=true`, `playback_blocked=false` | 프리뷰 재생 | `preview_played=true` 이후 rating 가능. job completion success로는 계산하지 않음 |
| `failed_with_partial_artifacts`, `preview_available=false`, `pitch_report_available=true` | 음정 리포트 보기, 재시도 | primary self-voice rating 없음 |
| `failed_with_partial_artifacts`, `playback_blocked=true` | 차단 이유 확인 | primary self-voice rating 없음 |
| no preview and no meaningful report | 재시도 또는 재업로드 | `failed`로 내려도 됨 |

Allowed transition summary:

```text
created -> queued -> processing
processing -> preview_ready -> completed
processing -> failed
processing -> failed_with_partial_artifacts
processing -> needs_review
created|queued|processing|preview_ready|completed -> blocked
preview_ready|completed|failed_with_partial_artifacts -> deleted
blocked -> under_review is represented by rights/job review metadata, not learner-facing state
```

Runtime policy change handling:

- Consent withdrawn during `created`, `queued`, `processing`, `preview_ready`, or `completed`: block new engine requests and playback URL issuance immediately. If no deletion is requested, move active jobs to `blocked` with `consent_withdrawn`.
- Rights state becomes `blocked`, `under_review`, or `expired`: block song selection, job creation, engine request, reference pre-listen, and generated preview playback. Preserve artifacts for policy review unless deletion is requested.
- Deletion requested: mark affected artifacts `deletion_pending`, block new playback URLs, enqueue deletion, and write deletion evidence. If deletion fails, mark `deletion_failed` and require platform/storage owner review.
- Audit write failure: fail closed for rights-sensitive operations and do not create jobs, publish engine events, issue playback URLs, or change rights state.

## StageResult Schema

StageResult는 engine event의 대체 이벤트 포맷이 아니라 canonical job state owner가 typed engine event 또는 직접 stage completion fact를 정규화해 저장하는 internal stage ledger다.

```text
typed engine event -> validated consumer/adapter -> StageResult upsert -> canonical job projection
```

Engine event는 각 엔진의 공개 transport contract로 유지하며, event type, 필수 필드, UTC timestamp, enum error code, extra field 거부 정책을 따른다. BFF/API Gateway는 engine event나 StageResult를 근거 없이 invent하지 않는다.

| Field | Required | Notes |
| --- | --- | --- |
| `stage_result_id` | Yes | stable id |
| `schema_version` | Yes | StageResult schema version |
| `job_id` | Yes | canonical job id |
| `stage` | Yes | `upload_validation`, `audio_ingest`, `section_validation`, `target_pitch_mapping`, `user_pitch_extraction`, `preview_synthesis`, `render`, `preview_evaluation`, `safety_rights`, `retention_deletion` |
| `attempt` | Yes | integer, starts at 1 |
| `status` | Yes | `queued`, `running`, `succeeded`, `failed`, `blocked`, `skipped` |
| `source_event_type` | Conditional | engine event type or direct stage completion type |
| `source_event_id` | Conditional | source event id when available |
| `idempotency_key` | Yes | stage-level duplicate handling key |
| `artifact_refs` | Yes | zero or more artifact refs |
| `error_code` | Conditional | app/job-state normalized code, required for failed/blocked |
| `source_error_namespace` | Conditional | engine/source namespace such as `audio_ingest` |
| `source_error_code` | Conditional | original engine/source error code, such as `UNSUPPORTED_FORMAT` or `FFMPEG_TIMEOUT` |
| `retryable` | Conditional | required for failed/timeout |
| `user_safe_reason` | Conditional | user-facing safe reason |
| `confidence_summary` | No | pitch/confidence summary |
| `metrics` | No | stage-specific numeric metrics |
| `timing_ms` | Conditional | required after stage completion/failure when known |
| `engine_name` | Conditional | engine or worker name |
| `engine_version` | Conditional | engine or worker version |
| `started_at` | Conditional | stage start time |
| `completed_at` | Conditional | stage completion time |
| `failed_at` | Conditional | stage failure time |
| `created_at` | Yes | event time |
| `received_at` | Yes | adapter receive time |

P0 error taxonomy starts with:

- `unsupported_format`
- `duration_exceeded`
- `section_mismatch`
- `no_voice_detected`
- `low_confidence_pitch`
- `target_note_conflict`
- `synthesis_failed`
- `render_failed`
- `rights_blocked`
- `consent_missing`
- `audit_failed`
- `playback_blocked`
- `deletion_failed`
- `timeout`

## ArtifactRef And Storage Contract

| Field | Required | Notes |
| --- | --- | --- |
| `artifact_id` | Yes | stable id |
| `job_id` | Yes | source job |
| `audio_asset_id` | Conditional | source audio asset when applicable |
| `data_class` | Yes | `user_raw_audio`, `canonical_audio`, `reference_audio`, `generated_preview`, `pitch_report`, `quality_report`, `audit_log`, `deletion_evidence` |
| `owner_user_id` | Conditional | user-owned artifacts only |
| `storage_backend` | Conditional | object storage provider/backend |
| `bucket` | Conditional | object storage bucket |
| `storage_key` | Conditional | object storage artifact |
| `mime_type` | Conditional | audio/json/report artifacts |
| `content_length_bytes` | Conditional | object size |
| `duration_ms` | Conditional | audio artifacts |
| `checksum_algorithm` | Conditional | checksum algorithm |
| `checksum_sha256` | Conditional | uploaded/generated files |
| `source_stage` | Conditional | stage that produced the artifact |
| `source_event_id` | Conditional | event that produced the artifact |
| `artifact_status` | Yes | `active`, `blocked`, `deletion_scheduled`, `deleted`, `deletion_failed` |
| `retention_deadline` | Yes | data-class based |
| `rights_state` | Yes | `allowed`, `blocked`, `under_review`, `expired`, `not_applicable` |
| `playback_allowed` | Yes | false for raw audio and blocked/deleted previews. Reference audio defaults false and may only be true for a separate section-limited pre-listen artifact when rights flags permit |
| `kms_key_id` | Conditional | encryption key marker when applicable |
| `created_at` | Yes | artifact creation time |
| `deleted_at` | Conditional | deletion completion time |
| `deletion_status` | Conditional | deletion workflow status |

Storage defaults:

- User voice input path: `audio-assets/{audio_asset_id}/original/{filename}`
- Canonical audio path: `audio-assets/{audio_asset_id}/canonical/{artifact_id}.wav`
- Generated preview path: `previews/{job_id}/{artifact_id}.wav`
- JSON reports path: `jobs/{job_id}/reports/{artifact_id}.json`

## Recording, Upload, And Playback Contract

- Recording mode: P0 primary input is in-app recorder. The app records a local take, lets the learner review/retake it, then submits it through the same presigned object-storage path used by fallback upload.
- Upload mode: P0 confirmed presigned direct upload for recorder take submission and fallback upload.
- Flow: App recorder/take review -> BFF GraphQL -> API Gateway -> presigned PUT URL -> App uploads to object storage.
- P0 object storage backend: MinIO with an S3-compatible contract.
- Upload TTL: 15 minutes by default. Expired URL requires a new upload session.
- Accepted extensions: `.wav`, `.mp3`.
- Accepted declared MIME: `audio/wav`, `audio/x-wav`, `audio/wave`, `audio/vnd.wave`, `audio/mpeg`, `audio/mp3`. `audio/mp3`는 허용하되 `audio/mpeg`로 normalize한다.
- Fallback upload is restricted to `.wav` and `.mp3`.
- If mobile recording produces a platform-native format, `audio-ingest` is the authoritative normalization boundary. The app may pre-normalize only when output is reliable, but `audio_ingest=succeeded` requires server/engine validation and canonical normalization.
- Canonical downstream audio is normalized mono WAV/PCM.
- `P0_MAX_UPLOAD_BYTES`: 50 MB default for P0. This is a file-size guard, not the authoritative duration check.
- Declared MIME alone is not trusted. `completeAudioUpload` verifies object HEAD and engine ingest validates real content with ffprobe/ffmpeg.
- `audio/*` broad matching is not the P0 product contract and must be tightened before relying on the allowlist.
- BFF and API Gateway must not log upload body, raw audio, signed URL secret, or playable preview URL.
- Upload completion creates or links `audio_asset_id`, `source_object_key`, checksum when available, content type, owner user id, selected song/section, rights flag snapshot, and consent snapshot. `duration_candidate` is optional client/recorder/upload metadata; authoritative duration comes from the `audio_ingest` StageResult.
- Playback mode: App -> BFF/API Gateway authorized playback request -> short-lived signed GET URL.
- Playback TTL: 5 minutes by default.
- Playback URL issuance requires authenticated user, job access, consent status, rights state, deletion state, and audit write success.
- No new playback URL is issued if `playback_blocked=true`, `rights_state` is blocked/under_review/expired, deletion is running, or consent is withdrawn.
- P0 does not require one-time playback URLs. Authoritative audit is `playback_url.issued` and `playback_url.denied`; player-side `playback.started`, `playback.ended`, `playback.error` are telemetry events and do not replace URL issuance audit.
- Reference pre-listen URL issuance is separate from generated preview playback. It requires `reference_prelisten_allowed=true`, selected section scope, no active recording, no blocked rights state, and audit write success.

Voice input completion contract:

- `createAudioUploadSession` creates an upload session and presigned PUT URL only. It does not create a conversion job.
- `completeAudioUpload` is the only boundary that commits a recorder take or fallback upload session into a P0 section preview job.
- Input: `audio_asset_id` or `upload_session_id`, `idempotency_key`, `song_package_id`, `target_section_id`, optional `take_id`.
- Server-trusted values: `source_object_key`, issued content type, original filename, owner user id, session expiry come from the upload session record rather than client-provided completion fields.
- Preconditions: authenticated user, upload session owner match, session not expired, object HEAD exists, object key matches the issued upload path, HEAD `Content-Type` equals issued or normalized content type, object size is greater than 0 and `<= P0_MAX_UPLOAD_BYTES`, declared MIME allowlist passes, server-computed required consent snapshot exists, rights/risk/audit allow decision succeeds.
- Commit effect: create or link `audio_asset_id`, create P0 section preview `job_id`, set canonical state to `created` or `queued`, write initial `upload_validation` StageResult, persist transactional outbox row for `audio.ingest.requested`.
- Publish rule: engine request is published from transactional outbox after DB commit, not before.
- Idempotency: same key and same payload returns the same job projection. Same key with conflicting payload returns conflict. Missing or invalid object does not create a job and does not publish ingest.

## Review Surface Contract

- Educator/expert review는 별도 review 화면으로 제공한다.
- Review 화면은 동의된 job의 generated preview, section label, pitch report, low-confidence ranges, failure tags, quality summary를 보여준다.
- Review 화면은 raw audio 직접 playback/download를 기본 제공하지 않는다.
- Internal reviewer technical tags는 같은 review 화면의 internal reviewer mode에서만 입력한다. 제한된 admin 화면은 song package, rights/risk, publish/block 책임만 가진다.
- Engine developer가 추가하는 technical tags는 user perception tags와 분리 저장한다.

## P0 Engine Mode Matrix

| Mode | Description | Allowed Use | Counts For P0 Self-voice Success |
| --- | --- | --- | --- |
| `mock` | static or fake preview, not generated from the submitted user voice | UI flow, loading/error/result layout test only | No |
| `partial_real` | real recorder/fallback voice input, ingest, section validation, pitch extraction, and a generated preview derived from the submitted user voice, even if some downstream quality/evaluation stages are constrained | Internal validation candidate if preview is app-playable and clearly labeled section-limited | Yes, if machine-checkable criteria below pass |
| `real_synthesis` | full P0 section preview pipeline using real synthesis/render/evaluation stages | Preferred P0 validation path | Yes |

P0 decision: `partial_real` preview도 P0 self-voice success 후보로 인정한다. 단, preview가 제출된 사용자 음성에서 파생되어야 하고, 앱에서 재생 가능해야 하며, section-limited output으로 명확히 표시되어야 한다. `real_synthesis`는 선호 경로다. Pitch-only success never counts as P0 self-voice success. If preview generation completely fails, the job is `failed` or `failed_with_partial_artifacts`.

`partial_real` P0-eligible minimum:

- `pipeline_mode=partial_real`
- `mock_fixture_used=false`
- real committed user voice input exists with `source_audio_asset_id`
- `audio_ingest=succeeded`
- `section_validation=succeeded`
- `user_pitch_extraction=succeeded`
- generated preview artifact has `job_id`, checksum, duration, `source_audio_asset_id`, `source_canonical_artifact_id` or `parent_artifact_refs`, `section_limited=true`, `preview_available=true`, `playback_allowed=true`
- `preview_artifact.source_audio_asset_id == job.source_audio_asset_id`
- `section_coverage_ratio >= 0.8` or duration is within section length tolerance
- user metric success still requires `preview_played=true` and primary rating `>= 4`

## Break-glass And Audit Fail-closed Contract

Break-glass raw/canonical audio access is not a normal role permission. It requires:

- incident or ticket id
- purpose
- approver
- second reviewer or security owner
- TTL
- least-privilege artifact scope
- immutable audit record
- post-access review

P0 1인 운영에서는 자기 승인 break-glass를 허용하지 않는다. second reviewer 또는 별도 security owner가 없으면 raw/canonical audio human access와 contact plaintext reveal은 disabled다.

Audit write failure must fail closed for:

- rights clearance/exposure/block
- risk acceptance create/update/expiry
- consent create/revoke
- analysis/preview request
- signed upload or playback URL issuance
- reference pre-listen URL issuance
- admin reference audio upload
- deletion request/job
- rights complaint block
- break-glass grant
- contact decrypt or service-mediated contact send
- contact consent lifecycle actions
- key rotation/retire/disable

P0 owner assignment: 별도 팀이 없으므로 개발자인 사용자가 deletion owner, policy owner, platform/storage owner를 겸임한다. 단, 모든 owner decision은 문서/DB/ticket 중 하나에 evidence로 남긴다. 이 owner 겸임은 P0 internal risk acceptance로 기록해야 하며, 자기 승인 break-glass나 contact plaintext reveal 권한을 의미하지 않는다. 제품화 또는 외부 beta 전에는 policy/legal, platform/storage, security/ops 책임을 분리해야 한다.

## Later Features

- Full-song one-pass processing 또는 chunked full-song merge
- `chorus_1`, `verse_1_a_humming` 등 singing validation target 추가
- 활성 microphone recording 중 reference guide playback. P0에서는 reference leakage와 권리 플래그 리스크 때문에 pre-listen만 허용한다.
- YouTube, Spotify, lyrics provider 기반 metadata/lyrics 자동 수집
- provider-approved 또는 별도 라이선스 기반의 reference asset ingestion 자동화
- AI 또는 alignment engine 기반 lyric timestamp sync
- 일본어 phoneme/syllable alignment와 syllable-level feedback
- 자동 vocal-mode candidate 생성과 teacher/expert-facing candidate label review UI
- 교사/전문가 라벨 기반 candidate dataset 및 future model-improvement workflow
- dedicated Conversion Job Orchestrator 서비스 분리
- active voice verification, voice CAPTCHA, stronger identity/voice ownership verification
- 교육자 전용 dashboard, 학생별 progress tracking, review queue
- 사용자 또는 다른 프로젝트의 non-admin song upload flow
- beta 이후 처리속도 최적화: time to first preview와 time to full result 분리 개선
- BPM/key correction controls의 일반 사용자 오픈

## Out of Scope

- 제3자 유명인, 아티스트, 캐릭터, 타인 음성 복제 또는 imitation
- 사용자 본인 음성이 아닌 voice model 선택
- YouTube, Spotify, lyrics provider 등에서 audio를 download, rip, cache, analyze, train, remix하는 기능
- 내부 운영 결과물 다운로드, export, public share link, public posting, commercial use
- 권리 확보 전 reference audio 또는 generated preview의 수익화, 공개 beta, 외부 고객 제공
- 공개 출시, 결제, 구독, 크레딧, 마켓플레이스, 정산
- DAW 수준 waveform editing, MIDI editor, multitrack mixing, advanced vocal editing
- 실시간 변환, live lesson streaming
- P0 learner-facing 자동 진성/비성/두성 판정
- P0 lyric sync, 일본어 음절 정렬, 가사 기반 feedback
- 활성 녹음 중 원곡/reference audio 동시 재생
- full-song preview 품질 보장
- 모든 장르와 모든 언어의 완전 지원
- production/commercial quality 보장
- 고급 expression control
- custom voice training
- model training pipeline
- 운영자 전체 관리 콘솔. 단, P0 song package 등록을 위한 최소 admin-only 등록 경로는 MVP에 포함한다.
- 소셜 로그인, 비밀번호 재설정, 팀/협업 프로젝트

## Scope Risks

- `intro`는 나레이션 중심이라 self-voice preview 검증에는 좋지만 singing pitch accuracy와 후렴 고음 synthesis 대표성이 약하다.
- reference audio rights clearance owner와 evidence 저장 위치가 확정되지 않으면 P0 launch gate를 통과할 수 없다.
- P0 최소 엔진 경로가 mock-only로 후퇴하면 P0 self-voice success 판정이 흔들린다.
- `worker` owned job state module 구현이 늦어지면 retry, partial artifact, failed_with_partial_artifacts, app-facing status가 흔들린다.
- recorder/take review와 presigned upload/playback 계약 구현이 늦어지면 앱, BFF, API Gateway, storage, audit 구현이 동시에 막힌다.
- P0에서 개발자가 deletion, policy, platform/storage owner를 겸임하므로 단일 owner 병목과 검토 부재 리스크가 있다.
- failure tag가 사용자 증상과 기술 원인을 분리하지 못하면 다음 build decision에 필요한 metric 품질이 낮아진다.
- app-only playback과 signed access를 느슨하게 구현하면 내부 운영이라도 권리/음성 악용 리스크가 커진다.

## Deferred Productization Tasks

- 수익화 또는 외부 beta 전 reference audio rights clearance 방식을 확정한다.
- 수익화 또는 외부 beta 전 policy/legal, platform/storage, security/ops owner를 개발자 1인 책임에서 분리한다.
- 제품화 전 contact encryption key custodian, decrypt approver, app operator, auditor/security reviewer 역할을 분리한다.
- 권리 evidence 없는 곡의 learner-facing 제한 노출을 유지할지, 대체 licensed/original reference로 전환할지 제품화 전 재검토한다.

## Open Questions

- `unlicensed_internal_risk_accepted`를 실제로 켜기 위한 risk acceptance record를 누가 작성/승인하고 어디에 저장할 것인가?
- P0 내부 운영 기간 동안 second reviewer를 둘 수 있는가? 둘 수 없다면 break-glass raw/canonical audio access와 contact plaintext reveal은 disabled로 유지한다.
- contact follow-up은 P0에서 disabled capability status만 남긴다. 실제 연락처 UI, 값 저장, 발송, 복호화, export를 언제 어떤 owner/reviewer 체계로 켤 것인가는 수익화 또는 외부 beta 전 재결정한다.
- 권리 evidence 없는 `Mist` 제한 노출을 내부 운영 종료 전 어느 시점에 `rights_pending`, `published`, 또는 `rights_blocked`로 재판정할 것인가?

## Recommended Next Skill

`page-flow-planner`

기능 behavior, state, permission, edge case가 page/flow 설계에 들어갈 만큼 분리됐다. 다음 단계에서는 학습자, 관리자, 교육자/전문가, 내부 reviewer의 화면 흐름을 나누고, 특히 song selection, section selection, current consent snapshot, recorder/take review, processing status, preview playback, rating/failure tagging, rights blocked/deletion 상태를 화면 단위로 설계해야 한다.
