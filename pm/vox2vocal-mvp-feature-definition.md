# Vox2Vocal MVP Feature Definition

문서 버전: v0.1  
작성일: 2026-06-13  
기준 문서: `pm/vox2vocal-mvp-prd.md` v0.10, `pm/vox2vocal-mvp-prd-review.md`  
적용 skill: `feature-definer`

## Context

Vox2Vocal P0 내부 alpha는 음악 학습자가 Ken Kamikita - `Mist`의 `intro` section을 본인 목소리로 녹음해 업로드하고, 앱 안에서 본인 목소리처럼 들리는 보정/생성 preview를 들어보는 경험을 검증한다. P0의 1차 성공은 full-song 생성이나 완성도 높은 상업 품질이 아니라 section-limited self-voice preview가 실제 학습 동기와 품질 판단 근거를 만드는지 확인하는 것이다.

P0는 `Mist` 전체 section map을 song package로 보관하되, 기본 target은 `intro`, timestamp는 등록된 reference audio asset 기준 `0:00-0:28`로 둔다. `intro`는 self-voice preview와 app flow 검증에는 적합하지만 나레이션 중심이므로 singing pitch matching 대표성은 약하다. 따라서 pitch feedback은 P0에서 제공하되, 후렴 수준의 singing validation은 Later 범위로 둔다.

## MVP Features

### 1. 계정 및 역할별 접근 제어 (Account And Role Access)

- User action: 학습자, 교육자/전문가, 관리자, 내부 reviewer가 계정을 만들거나 로그인한다.
- Product behavior: 앱은 인증된 사용자만 곡 선택, 업로드, 결과 조회, review 작업에 접근하도록 제한한다. 사용자 role에 따라 가능한 행동과 조회 가능한 데이터 범위를 다르게 적용한다.
- Business rules:
  - 이메일, 비밀번호, 표시 이름, 약관 동의가 필요하다.
  - 비밀번호는 최소 8자 이상이어야 한다.
  - 인증되지 않은 사용자는 conversion job을 만들 수 없다.
  - role은 최소 `learner`, `educator_or_expert`, `admin`, `engine_developer`, `product_qa`, `security_or_ops`를 구분한다.
- Permissions / roles:
  - `learner`: 본인 업로드, 본인 job, 본인 preview, 본인 pitch report, 삭제 요청 접근
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
  - 로그인 세션이 만료된 상태에서 업로드를 시작한 경우 job 생성 전 재인증한다.
  - role이 없는 내부 사용자는 기본적으로 learner 권한보다 더 넓은 접근을 받지 않는다.
  - 교육자/전문가가 raw audio에 직접 접근하려는 경우 기본 차단한다.
- Dependencies:
  - App auth UI
  - BFF auth GraphQL API
  - API Gateway auth contract
  - User Service account, auth, role/status
  - separated consent records
- Success signal:
  - 학습자 10명 중 5명 이상이 첫 세션에서 practice analysis job을 생성한다.
  - 교육자 2명이 최소 1개 reviewable result에 접근할 수 있다.
  - client/backend 인증 연동 오류가 시도 대비 2% 미만이다.

### 2. 관리자 곡 패키지 및 권리 검증 게이트 (Admin Song Package And Rights Gate)

- User action: 관리자가 내부 alpha용 `Mist` song package를 등록하고 publish 가능 상태로 만든다.
- Product behavior: 시스템은 필수 metadata와 권리 상태가 모두 충족된 song package만 학습자에게 노출한다. 권리 불확실, 만료, complaint, provenance 누락이 있으면 `rights_blocked`로 전환한다.
- Business rules:
  - P0 song package 필수값은 title, artist, language, BPM, key, reference audio, source/provenance, rights clearance status, usage status, retention period, deletion owner, full section map, default target section start/end다.
  - P0 target song은 Ken Kamikita - `Mist`로 제한한다.
  - P0 default target section은 `intro`, timestamp는 reference audio asset 기준 `0:00-0:28`이다.
  - YouTube, Spotify, lyrics provider 등 외부 provider는 P0에서 audio source로 쓰지 않는다.
  - reference audio는 앱 내부 분석/비교에만 사용하고 다운로드, 공유, 학습, remix, third-party voice cloning에 사용하지 않는다.
  - 필수 권리 evidence가 없으면 learners에게 publish할 수 없다.
- Permissions / roles:
  - `admin`: song package 등록/수정 요청
  - `policy_or_rights_owner`: rights clearance 승인/차단
  - `platform_storage_owner`: retention/deletion owner 지정
  - `learner`: publish된 song package 조회만 가능
- States:
  - `draft`
  - `metadata_incomplete`
  - `rights_pending`
  - `published`
  - `rights_blocked`
  - `under_review`
  - `retired`
- Edge cases:
  - 등록된 reference audio asset과 section timestamp가 어긋나면 publish를 보류한다.
  - rights complaint가 들어오면 해당 song package와 관련 generated preview playback을 차단한다.
  - BPM/key는 기본값으로 등록하되, P0 correction controls가 열리지 않았으면 학습자가 수정하지 못한다.
  - lyrics는 P0 선택값이며, lyrics sync가 없더라도 publish를 막지 않는다.
- Dependencies:
  - Admin song package API 또는 seed/admin-only 등록 경로
  - object storage for reference audio
  - rights clearance checklist
  - section map validation
  - audit log
  - deletion owner registry
- Success signal:
  - `Mist` package가 필수 metadata와 rights evidence를 갖고 publish된다.
  - learners는 `rights_blocked` package를 선택할 수 없다.
  - 100%의 reference asset publish decision이 provenance, allowed use, retention, deletion owner를 포함한다.

### 3. 학습자 곡 선택, 동의, 보컬 업로드 (Learner Song Selection, Consent, And Vocal Upload)

- User action: 학습자가 관리자 등록 `Mist` package를 선택하고, 본인 보컬 연습 파일을 업로드하며, 본인 음성/preview 생성/전문가 검토/보관 정책에 동의한다.
- Product behavior: 시스템은 동의와 upload validation을 통과한 경우에만 P0 section preview job을 생성한다. 동의가 부족하거나 파일이 부적합하면 engine processing 전에 차단한다.
- Business rules:
  - P0는 학습자의 원곡/reference audio 업로드를 허용하지 않는다.
  - 학습자는 본인 보컬 연습 파일만 업로드할 수 있다.
  - 지원 형식은 `wav`, `mp3`다.
  - upload hard max는 60초다.
  - target section duration은 20-40초 권장이고 P0 `intro`는 28초다.
  - section length tolerance는 target duration 대비 `±5초` 또는 `±20%` 중 더 큰 값이다.
  - own voice consent와 generated preview consent는 필수다.
  - expert review consent는 alpha 참여 조건으로 분리 동의한다.
  - candidate data consent는 opt-in이며 필수 동의와 번들링하지 않는다.
- Permissions / roles:
  - `learner`: 본인 vocal upload와 consent 제출
  - `educator_or_expert`: 동의된 job만 검토
  - `engine_developer`: 필요한 경우 승인된 디버깅 범위에서 artifact 접근
  - `security_or_ops`: audit metadata 접근
- States:
  - `package_selected`
  - `awaiting_consent`
  - `uploading`
  - `upload_validating`
  - `upload_rejected`
  - `ready_to_process`
  - `blocked_by_policy`
- Edge cases:
  - 사용자가 본인 음성 확인을 거부하면 job을 생성하지 않는다.
  - 60초 초과 파일은 자동 full-song 처리하지 않고 trim 또는 expected section 재업로드를 요구한다.
  - 지원하지 않는 파일 형식은 명확한 사유와 함께 거절한다.
  - candidate data opt-in이 없어도 preview generation 자체는 진행할 수 있다.
  - reference audio가 `rights_blocked`로 바뀐 직후 선택 화면에 캐시된 package가 남아 있으면 processing 전에 다시 차단한다.
- Dependencies:
  - song package read model
  - upload initiation API
  - object storage or presigned upload decision
  - consent storage
  - Safety Rights check
  - job creation API
- Success signal:
  - valid upload의 100%가 consent status와 source asset id를 가진다.
  - unauthorized or non-self voice job이 analysis, preview, synthesis에 도달하지 않는다.
  - rejected upload는 사용자에게 format, length, consent, rights 중 어떤 이유인지 보여준다.

### 4. 오디오 수집 및 구간 검증 (Audio Ingest And Section Validation)

- User action: 학습자가 업로드를 완료하고 분석 시작을 요청한다.
- Product behavior: 시스템은 업로드 파일을 canonical audio로 변환하고, duration, loudness, silence, voice segments, section coverage를 계산한다. target section과 맞지 않는 경우 full-song으로 조용히 처리하지 않고 사용자가 이해할 수 있는 상태로 돌려준다.
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
  - 무음 또는 voice segment 부족 파일은 preview job을 진행하지 않고 재업로드를 요청한다.
  - loudness가 너무 낮거나 clipping이 심한 경우 preview 품질 risk를 기록한다.
  - 변환은 성공했지만 section validation이 낮은 confidence인 경우 `needs_review` 또는 `section_mismatch`로 남긴다.
- Dependencies:
  - Audio Ingest engine
  - object storage
  - canonical job state owner
  - stage result record
  - retention/deletion scheduler
- Success signal:
  - valid P0 upload의 100%가 canonical asset과 ingest metadata를 생성한다.
  - section mismatch가 발생한 job은 full-song으로 조용히 처리되지 않는다.
  - ingest failure stage와 plain-language reason이 job status에 반영된다.

### 5. 목표 음정 매핑 및 신뢰도 처리 (Target Pitch Mapping And Confidence Handling)

- User action: 학습자가 `intro` section에 대한 pitch feedback 결과를 본다.
- Product behavior: 시스템은 reference audio analysis, engine-derived note sequence, manual/admin override 중 출처가 있는 target note sequence를 생성하고, uploaded vocal pitch와 비교한다.
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
  - pitch report만 성공하고 preview가 실패하면 alpha success로 보지 않는다.
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
- Product behavior: 시스템은 사용자의 본인 음성만을 사용해 `intro` section-limited preview를 생성하고, 앱 내 재생만 허용한다. preview가 완전히 실패하면 job은 성공으로 표시하지 않는다.
- Business rules:
  - 생성 가능한 vocal identity는 업로드한 사용자 본인 목소리뿐이다.
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
  - playback URL은 short-lived signed access로만 제공한다.
  - rights complaint 또는 deletion failure가 발생하면 playback을 차단한다.
- Dependencies:
  - minimal self-voice preview engine path
  - canonical job state owner
  - render storage
  - app playback component
  - Safety Rights
  - signed access URL mechanism
- Success signal:
  - valid P0 section jobs의 60% 이상이 `completed`와 `preview_available=true`, `section_limited=true`에 도달한다.
  - 학습자 10명 중 5명 이상이 primary question에서 4점 이상을 준다.
  - 2점 이하 응답이 2명을 초과하지 않는다.

### 7. 표준 작업 상태 및 부분 산출물 처리 (Canonical Job State And Partial Artifact Handling)

- User action: 사용자가 job 생성 후 처리 상태를 확인하거나, 실패/부분 성공 결과를 확인한다.
- Product behavior: 시스템은 하나의 canonical job state owner가 stage result를 idempotent하게 반영하고, app-facing job status를 제공한다. BFF나 개별 worker는 최종 상태를 임의로 invent하지 않는다.
- Business rules:
  - P0는 신규 Conversion Job Orchestrator 서비스를 필수로 만들지 않고 기존 backend/worker bounded module로 시작할 수 있다.
  - canonical state는 `created`, `queued`, `processing`, `preview_ready`, `completed`, `failed`, `failed_with_partial_artifacts`, `blocked`, `needs_review`, `deleted`를 지원한다.
  - output availability는 `preview_available`, `pitch_report_available`, `section_limited`, `rating_required`, `failure_tags_required`로 state와 분리 추적한다.
  - engine worker는 job id, stage, status, artifact pointer, error reason, confidence summary, timing metadata를 emit한다.
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
  - job이 60분을 초과하면 timeout/failure 후보로 전환하고 내부 debugging에 노출한다.
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
- Product behavior: 결과 화면은 preview playback을 가장 먼저 제공하고, 그 다음 pitch feedback, quality report, rating, failure tags를 보여준다. 4점 미만 rating은 failure tag 없이는 제출 완료로 보지 않는다.
- Business rules:
  - primary rating question은 "이 preview가 내 목소리처럼 들린다"다.
  - 4점 이상만 primary self-voice metric success로 계산한다.
  - 4점 미만은 `not_my_voice`, `not_song_like`, `pitch_wrong`, `timing_wrong`, `robotic_or_artifact`, `noise_or_clipping`, `too_short_or_incomplete`, `playback_issue`, `other` 중 최소 1개를 요구한다.
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
  - preview를 재생하지 못한 사용자는 `playback_issue`를 선택할 수 있어야 한다.
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
  - deletion 실패 시 `deletion_failed`로 표시하고 rights-sensitive playback을 차단한다.
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
  - audit logging 실패 시 reference audio analysis 또는 preview generation을 진행하지 않는다.
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

## Later Features

- Full-song one-pass processing 또는 chunked full-song merge
- `chorus_1`, `verse_1_a_humming` 등 singing validation target 추가
- YouTube, Spotify, lyrics provider 기반 metadata/lyrics 자동 수집
- provider-approved 또는 별도 라이선스 기반의 reference asset ingestion 자동화
- AI 또는 alignment engine 기반 lyric timestamp sync
- 일본어 phoneme/syllable alignment와 syllable-level feedback
- 자동 vocal-mode candidate 생성과 teacher/expert-facing 검토 UI
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
- 내부 alpha 결과물 다운로드, export, public share link, public posting, commercial use
- 공개 출시, 결제, 구독, 크레딧, 마켓플레이스, 정산
- DAW 수준 waveform editing, MIDI editor, multitrack mixing, advanced vocal editing
- 실시간 변환, live lesson streaming
- P0 learner-facing 자동 진성/비성/두성 판정
- P0 lyric sync, 일본어 음절 정렬, 가사 기반 feedback
- full-song preview 품질 보장
- model training pipeline
- 운영자 전체 관리 콘솔. 단, P0 song package 등록을 위한 최소 admin-only 등록 경로는 MVP에 포함한다.
- 소셜 로그인, 비밀번호 재설정, 팀/협업 프로젝트

## Scope Risks

- `intro`는 나레이션 중심이라 self-voice preview 검증에는 좋지만 singing pitch accuracy와 후렴 고음 synthesis 대표성이 약하다.
- reference audio rights clearance owner와 evidence 저장 위치가 확정되지 않으면 P0 launch gate를 통과할 수 없다.
- P0 최소 엔진 경로가 mock, partial-real, real synthesis 중 무엇인지 확정되지 않으면 alpha success 판정이 흔들린다.
- canonical job state owner가 불명확하면 retry, partial artifact, failed_with_partial_artifacts, app-facing status가 흔들린다.
- upload/storage 계약이 늦게 정해지면 앱, BFF, worker, storage, audit 구현이 동시에 막힌다.
- retention/deletion owner가 없으면 30일 raw audio 삭제와 1년 non-audio dataset 분리가 운영 통제가 아니라 문서 선언에 그칠 수 있다.
- failure tag가 사용자 증상과 기술 원인을 분리하지 못하면 다음 build decision에 필요한 metric 품질이 낮아진다.
- app-only playback과 signed access를 느슨하게 구현하면 내부 alpha라도 권리/음성 악용 리스크가 커진다.

## Open Questions

- `Mist` section map과 `intro = 0:00-0:28`은 실제 등록 reference audio asset 기준으로 검수 완료할 것인가, 조정 가능성을 남길 것인가?
- reference audio rights clearance 최종 승인자는 누구이며, 승인 evidence는 어느 저장소/문서/record에 남길 것인가?
- P0 canonical job state owner는 worker bounded module, backend service module, API Gateway 중 어디로 확정할 것인가?
- P0 최소 엔진 경로는 partial-real pipeline으로 시작할 것인가, real synthesis까지 요구할 것인가?
- upload는 앱에서 BFF로 직접 전송할 것인가, presigned URL/object storage 직접 업로드로 갈 것인가?
- internal reviewer가 기술 tag를 입력할 최소 surface는 admin-only 화면인가, DB/운영 도구 기반인가?
- 4점 미만 failure tag 목록은 현재 목록으로 확정할 것인가, alpha 전 pilot에서 한 번 줄일 것인가?
- `other` 자유입력에 개인정보/민감정보가 들어올 경우 review와 redaction 기준은 어떻게 둘 것인가?
- deletion owner, policy owner, platform/storage owner를 실제 담당자 또는 팀으로 어떻게 지정할 것인가?
- 교육자/전문가 view는 P0에서 learner result 화면 재사용으로 시작할 것인가, 별도 review 화면을 만들 것인가?
- P0에서 BPM/key correction controls는 관리자만 수정 가능한가, 학습자에게도 열 것인가?

## Recommended Next Skill

`page-flow-planner`

기능 behavior, state, permission, edge case가 page/flow 설계에 들어갈 만큼 분리됐다. 다음 단계에서는 학습자, 관리자, 교육자/전문가, 내부 reviewer의 화면 흐름을 나누고, 특히 upload consent, processing status, preview playback, rating/failure tagging, rights blocked/deletion 상태를 화면 단위로 설계해야 한다.
