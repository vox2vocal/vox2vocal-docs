# Vox2Vocal MVP PRD Draft

문서 버전: v0.2  
작성일: 2026-06-09  
상태: 초안  
작성 기준: `pm-context` + `prd-writer` skill 기준

## Context Brief

### Brief

- Product or feature: Vox2Vocal MVP
- Target users: J-POP, 우타이테, 보컬 곡을 배우는 음악 학습자와 이를 지도하는 보컬/음악 교육자
- Problem: 학습자는 한 곡을 연습하면서 자신의 음정, 박자, 발음, 표현이 원곡 또는 목표 보컬과 얼마나 다른지 객관적으로 파악하기 어렵다. 교육자는 학생의 녹음물을 반복해서 듣고 피드백해야 하며, 구간별 문제를 빠르게 시각화하고 설명할 도구가 부족하다.
- Goal: 내부 alpha에서 사용자가 본인 음성으로 부른 J-POP/우타이테 한 곡 단위 입력을 업로드하고, 시스템이 분석/정렬/피치/타이밍/평가를 수행해 학습 피드백과 보컬 preview 가능성을 검증한다.
- Constraints: 현재 앱은 로그인/회원가입 UI 중심이며 실제 인증 API 연결 전 단계다. 백엔드는 BFF, API Gateway, User Service의 인증 계약이 존재한다. 엔진은 전체 아키텍처와 일부 audio-ingest 구현이 존재하며, 나머지 엔진은 문서 기준 MVP 범위가 정의되어 있다. 한 곡 단위 입력은 처리 시간, 비용, chunking, 품질 일관성에 대한 기술 검토가 필요하다.
- Success criteria: baseline과 target은 아직 실제 사용자/처리 데이터가 없어 가정이 필요하다. 내부 alpha에서는 activation, song-level job completion, processing reliability, learning feedback usefulness, safety block rate를 우선 측정한다.

### Known Facts

- Expo React Native App/Web 앱이 있고 로그인, 회원가입 화면이 구현되어 있다. Source: `vox2vocal-app/README.md`, `vox2vocal-app/src/features/auth/`
- BFF는 앱용 GraphQL endpoint를 제공하고 API Gateway를 gRPC로 호출하는 구조다. Source: `vox2vocal-bff-server/README.md`
- API Gateway는 `SignUp`, `Login`, `GetCurrentUser` gRPC 계약을 가진다. Source: `vox2vocal-api-gateway/proto/gateway.proto`
- User Service는 사용자 도메인, PostgreSQL, Prisma, 비밀번호 인증 정책을 소유한다. Source: `vox2vocal-user-service/README.md`, `src/users/policies/`
- 엔진 아키텍처는 Audio Ingest에서 시작해 Voice Analysis, Voice Pitch, Phoneme Alignment, Rhythm Timing, Melody Mapping, Singing Synthesis, Vocoder Render, Mix Master, Evaluation, Safety Rights로 이어지는 파이프라인을 지향한다. Source: `vox2vocal-docs/engine/README.md`
- Safety Rights는 초기 MVP부터 포함되어야 하며 사용자 본인 업로드 보이스만 허용하는 방향이 문서화되어 있다. Source: `vox2vocal-docs/engine/safety-rights/README.md`

### Assumptions

- 1차 MVP의 핵심 검증은 "상업 출시 품질의 완성 보컬"이 아니라 "학습자와 교육자가 한 곡 연습 결과를 이해하고 개선점을 찾을 수 있는 분석/피드백 경험"이다.
- 제품 목표 입력 길이는 J-POP 또는 우타이테 노래 한 곡 기준이다. 단, 내부 alpha에서는 한 곡 처리의 기술 검토를 포함하며, 필요하면 chunk 단위 처리와 단계별 품질 검증으로 시작한다.
- 보이스 사용 정책은 사용자 본인 음성만 허용한다.
- 외부 배포, 상업 이용, 제3자 음성 복제는 MVP 범위에서 제외하고 Safety Rights에서 차단한다.
- 1차 산출물을 "노래처럼 들리는 preview"로 둘지, "피치/타이밍/발음 학습 리포트"로 둘지는 아직 제품 결정이 필요하다. 내부 alpha에서는 두 산출물을 모두 검증하되 primary success metric은 학습 피드백 유용성으로 둔다.
- 모바일 앱을 primary surface로 두되, Expo Web도 같은 핵심 흐름을 지원하는 방향으로 설계한다.
- 출시 형태는 내부 alpha다.

### Risks

- 보컬 합성 품질이 낮아도 학습 리포트가 유용하면 alpha 가치는 있을 수 있다. 반대로 리포트가 약하면 preview 품질만으로는 교육 제품의 가치를 설명하기 어렵다.
- 권리/동의 정책이 약하면 제품 리스크가 기술 리스크보다 커질 수 있다.
- 한 곡 단위 입력은 처리 시간, 비용, 메모리, 엔진 간 artifact 크기, 오류 복구 난이도를 크게 키울 수 있다.
- 현재 인증 UI와 백엔드 인증 계약 사이의 실제 연동이 완료되지 않으면 upload/conversion 흐름의 사용자 식별과 audit이 막힌다.

### Recommended Next Skill

- Skill: `prd-writer`
- Reason: 프로젝트 문서와 코드에서 제품 방향, 현재 구현 상태, 엔진 MVP 범위를 확인했으므로 engineering handoff 가능한 PRD 초안 작성으로 진행한다.

## Executive Summary

Vox2Vocal MVP는 음악 학습자와 음악 교육자가 J-POP, 우타이테, 보컬 곡 한 곡 단위의 연습 녹음을 업로드하면, 시스템이 본인 음성을 분석해 피치, 타이밍, 발음/음절 정렬, 표현 단서, 품질 리포트를 제공하고 보컬 preview 가능성을 검증하는 내부 alpha 제품이다.

이 PRD의 목적은 "내 목소리로 부른 한 곡을 학습 가능한 데이터와 들을 수 있는 결과로 바꾸는 경험"이 실제 교육 현장에서 가치가 있는지 확인하는 것이다. 따라서 MVP는 상업 배포, 다중 보이스 모델, 정교한 DAW 편집 기능보다 가입, 업로드, 처리 상태 확인, 구간별 학습 피드백, preview 재생, 기본 품질 리포트, 본인 음성 권한 차단을 우선한다.

## Problem Statement

음악을 배우는 사람은 J-POP, 우타이테, 보컬 곡을 연습할 때 자신이 어느 구간에서 음정이 흔들리는지, 박자가 밀리는지, 발음/음절 길이가 어색한지 스스로 판단하기 어렵다. 원곡을 들으며 반복 연습할 수는 있지만, 자신의 녹음이 목표와 어떻게 다른지 구간별로 확인하기에는 도구가 부족하다.

음악을 가르치는 사람은 학생의 연습 녹음을 듣고 반복적으로 피드백해야 한다. 그러나 수업 시간은 제한적이고, 피치/리듬/발음 문제를 시각적 근거와 함께 설명하는 과정은 수작업에 가깝다. Vox2Vocal은 본인 음성 기반 분석과 preview를 통해 학습자에게는 연습 방향을, 교육자에게는 피드백 근거를 제공해야 한다.

## Target Users

- Primary: J-POP, 우타이테, 애니송, 보컬 곡을 배우는 음악 학습자
- Secondary: 학생의 보컬 연습을 지도하는 음악 교사, 보컬 트레이너, 온라인 강사
- Internal alpha users: 제품팀, 엔진 개발자, QA, 제한된 음악 교육 협력자

## Goals

- 사용자가 계정을 만들고 로그인한 상태에서 오디오 변환 작업을 생성할 수 있다.
- 사용자가 `wav` 또는 `mp3` 형식의 본인 보컬 연습 파일을 업로드할 수 있다.
- 내부 alpha에서 J-POP/우타이테 한 곡 단위 입력 처리의 기술 가능성과 병목을 검토할 수 있다.
- 시스템이 업로드 파일을 표준 오디오 asset으로 변환하고 처리 상태를 추적할 수 있다.
- 시스템이 곡 단위 입력에서 구간별 pitch, timing, melody, alignment, quality 정보를 생성할 수 있다.
- 시스템이 학습자가 이해할 수 있는 피드백 리포트와 본인 음성 기반 preview를 제공할 수 있다.
- 모든 변환 요청은 사용자 권한과 보이스 사용 정책을 통과해야 하며 audit 가능한 기록을 남긴다.
- 사용자는 실패 이유와 재시도 가능 여부를 이해할 수 있다.

## Non-goals

- 제3자 유명인, 아티스트, 캐릭터 보이스 복제 지원
- 사용자가 직접 업로드한 본인 음성이 아닌 보이스 모델 사용
- 상업 배포 라이선스, 정산, 마켓플레이스 기능
- DAW 수준의 waveform 편집, MIDI editor, 멀티트랙 믹싱
- 실시간 변환, 라이브 레슨 스트리밍, 반주 포함 멀티트랙 믹싱
- 모든 장르와 모든 언어의 완전 지원. MVP는 J-POP/우타이테 학습 맥락을 우선한다.
- 고급 expression control, voice conversion, custom voice training

## Scope

### In Scope

- 이메일/비밀번호 기반 회원가입, 로그인, 현재 사용자 조회
- 모바일 앱 및 웹에서 인증 후 MVP 작업 화면 접근
- 오디오 업로드: `wav`, `mp3`, 사용자 본인 보컬 연습 파일
- 입력 길이 목표: J-POP/우타이테 노래 한 곡 기준. 내부 alpha에서는 기술 검토 결과에 따라 chunk 단위 처리, 구간별 처리, 또는 길이 제한을 둘 수 있다.
- 입력 metadata: 곡/작업명, 언어, BPM, key, 가사 또는 대본 텍스트, 원곡/목표곡 정보
- Audio Ingest: 표준 PCM/WAV 변환, mono 변환, 무음 탐지, 발화 구간 timestamp 생성
- Voice Analysis: RMS energy, 발화 속도, 휴지 구간, 강세 후보 timestamp
- Voice Pitch: F0 추출, confidence filtering, MIDI note 변환, JSON 결과
- Phoneme Alignment: 한국어 음절 단위 정렬, word boundary, alignment confidence
- Rhythm Timing: 고정 BPM 기준 1/4, 1/8, 1/16 grid 정렬과 duration 생성
- Melody Mapping: MIDI note sequence, key 기반 보정, note sequence JSON
- Singing Synthesis: 사용자 본인 음성 기반 vocal preview 가능성 검증
- Vocoder Render: offline wav render, clipping report
- Mix Master: vocal-only EQ/compressor/limiter, loudness normalization, wav output
- Evaluation: pitch deviation, timing deviation, clipping 탐지, 구간별 학습 피드백, 엔진별 JSON 비교
- Safety Rights: 사용자 본인 업로드 보이스만 허용, conversion audit log
- 작업 상태 화면: queued, processing, completed, failed
- 결과 화면: 구간별 학습 리포트, preview 재생, 다운로드, 기본 품질 리포트, 실패 사유

### Out Of Scope

- 소셜 로그인 실제 연동
- 비밀번호 재설정 flow
- 결제/구독/크레딧
- 팀/협업 프로젝트
- 공개 공유 링크
- 고급 vocal editing UI
- model training pipeline
- 운영자 관리 콘솔
- 제3자 보이스 모델 선택, 타인 음성 변환, 캐릭터/아티스트 보이스 복제
- 내부 alpha 단계의 공개 출시 또는 외부 배포 기능

## User Stories

- As a music learner, I want to sign up and log in so that my practice recordings and feedback reports are tied to my account.
- As a music learner, I want to upload my own J-POP or utaite practice recording so that I can understand how my singing differs from the target song.
- As a music learner, I want to enter or confirm song metadata such as BPM, key, language, and lyrics so that the analysis can align to the song correctly.
- As a music learner, I want to see pitch, timing, syllable alignment, and quality feedback by section so that I know what to practice next.
- As a music learner, I want to preview the processed result in my own voice so that I can hear the intended correction direction.
- As a music teacher, I want to review a student's report quickly so that I can explain specific practice priorities during a lesson.
- As a music teacher, I want failed or low-confidence sections to be visible so that I know where manual review is needed.
- As a user, I want unauthorized voice usage to be blocked clearly so that I understand that only my own voice is allowed.
- As a product/engineering team member, I want evaluation metrics and engine artifacts so that I can debug quality regressions and full-song processing bottlenecks.

## Functional Requirements

### FR-001 Account Creation

- The app must allow users to create an account with email, password, display name, and terms consent.
- Email must be normalized and validated.
- Password must be at least 8 characters.
- Display name must be required and limited to the backend policy maximum.
- Successful signup must return access token, refresh token, expiry, and user profile.

### FR-002 Login And Current User

- The app must allow users to log in with email and password.
- Failed login attempts must follow backend lockout policy.
- The app must store session state securely enough for MVP and attach access tokens to authenticated requests.
- The app must retrieve the current user through the BFF/Gateway flow.

### FR-003 Create Conversion Project

- The user must be able to create a practice analysis project with title, source file, language, optional BPM, optional key, optional lyrics, and target song label.
- MVP must require users to acknowledge that the uploaded recording contains their own voice.
- The system must assign a project/job id before engine processing starts.

### FR-004 Upload And Audio Ingest

- The system must accept `wav` and `mp3` inputs for MVP.
- The system must reject unsupported formats with a clear reason.
- The system must convert accepted input to a standard mono WAV/PCM asset.
- The system must produce metadata including sample rate, channels, duration, loudness estimate, silence segments, speech/voice segments, and `audio_asset_id`.
- The system must record original duration and estimated processing cost class so full-song processing feasibility can be reviewed during internal alpha.
- If a full song cannot be processed in one pass, the system must use a documented chunking or section-based fallback instead of silently truncating the input.

### FR-005 Safety Rights Check

- The system must verify that the user owns or is allowed to use the source audio asset.
- MVP must allow only the user's own uploaded voice.
- The system must deny any request that attempts to use another person's voice, a celebrity/artist voice, a character voice, or an unverified target voice model.
- Each analysis, preview, or export request must create an audit log with user id, source asset id, operation, decision, and policy reason.

### FR-006 Voice Analysis

- The system must produce energy RMS curve, speech rate estimate, pause segments, and stress candidate timestamps from the canonical audio.
- The analysis output must be tied to the job id and audio asset id.

### FR-007 Pitch Extraction

- The system must extract F0 frames for vocal practice recordings.
- The system must mark voiced/unvoiced frames and confidence score.
- The system must convert confident F0 frames into MIDI note numbers.
- The system must compare detected pitch against target song notes when target melody or derived note sequence is available.
- Low-confidence pitch segments must be surfaced in the result screen and evaluation report.

### FR-008 Text And Syllable Alignment

- If lyrics/text are provided, the system must produce Korean syllable timeline, word boundaries, and alignment confidence.
- If text is missing, the system must either request text for synthesis or use a documented fallback path.

### FR-009 Rhythm And Melody Mapping

- The system must generate a beat-aligned syllable timeline using BPM and grid options.
- MVP must support 1/4, 1/8, and 1/16 note grids.
- The system must produce note sequence JSON with note start/end, target pitch, duration, and confidence.
- The system must support C major default or user-provided key.

### FR-010 Vocal Synthesis And Render

- The system must generate a vocal preview using the user's own voice only, if preview generation is included in the alpha build.
- The system must render the result as a wav file through offline vocoder rendering.
- The render output must include clipping and loudness metadata.
- If preview generation is not technically ready for full-song input, the system must still produce the learning feedback report and mark preview as unavailable or section-limited.

### FR-011 Basic Mix/Master

- The system must apply vocal-only basic EQ, compressor, limiter, and loudness normalization.
- The output must remain downloadable as wav.
- The system must preserve the unprocessed render artifact for debugging.

### FR-012 Job Status And Result UI

- The app must show job states: `created`, `queued`, `processing`, `completed`, `failed`, `blocked`.
- Completed jobs must show section-level learning feedback, low-confidence sections, optional audio preview playback, download action if allowed, and basic quality report.
- Failed jobs must show stage, reason, and whether retry is allowed.
- Blocked jobs must show policy reason without exposing sensitive policy internals.

### FR-013 Evaluation And Observability

- The system must store pitch deviation, timing deviation, clipping detection, and engine artifact comparison results.
- Each engine stage must emit structured logs tied to job id.
- Quality reports must be available for internal review even if the user-facing version is simplified.
- Full-song jobs must expose duration, chunk count, per-stage processing time, memory/cost class, and failed section details for technical review.

## Acceptance Criteria

- Given a new user enters valid signup details and accepts terms, when they submit, then an account is created and the app receives auth tokens and user profile.
- Given a user enters invalid email or password, when they submit signup/login, then validation errors are shown and no conversion job is created.
- Given an authenticated user uploads a supported `wav` or `mp3` practice recording, when the upload completes, then the system creates a practice analysis job and an `audio_asset_id`.
- Given an uploaded recording is song-length, when processing begins, then the system either processes it in one pass or records a documented section/chunk strategy for the job.
- Given an unsupported file format is uploaded, when validation runs, then the user sees a clear unsupported format message.
- Given the user does not confirm the source audio is their own voice, when they attempt to start analysis or preview generation, then the job is blocked before engine processing.
- Given the user attempts to use another person's voice or an unauthorized target voice model, when Safety Rights runs, then the job is denied and an audit record is created.
- Given a valid practice recording with required metadata, when the pipeline completes, then a section-level learning report is available.
- Given preview generation is enabled for the job, when the pipeline completes, then a self-voice preview is available for playback or the unavailable/section-limited state is clearly shown.
- Given a pipeline stage fails, when the user views the job, then the failed stage, plain-language reason, and retry guidance are shown.
- Given a completed job, when internal reviewers inspect the artifacts, then pitch deviation, timing deviation, clipping status, and stage outputs are available.
- Given a song-length internal alpha job completes or fails, when internal reviewers inspect the artifacts, then duration, chunk count, stage timings, and failed sections are available.
- Given mobile viewport `360 x 640`, when the user signs up, logs in, uploads, and views a result, then primary actions remain reachable without horizontal scroll.

## Success Metrics

- Activation
  - Baseline: Unknown. No production usage data available.
  - Target: At least 40% of signed-in internal alpha testers create one practice analysis job within their first session.
  - Guardrail: Signup/login failure due to client/backend integration errors below 2% of attempts.
- Job completion
  - Baseline: Unknown. Engine pipeline not yet fully implemented.
  - Target: At least 60% of valid supported song-length internal alpha jobs reach `completed` or `completed_with_preview_limited`.
  - Guardrail: No unauthorized or non-self voice jobs reach analysis, preview, synthesis, or export.
- Time to preview
  - Baseline: Unknown.
  - Target: Needs technical validation for J-POP/utaite one-song inputs. Internal alpha must measure P50/P95 by duration and chunk count before setting a release target.
  - Guardrail: P95 processing time and failure reasons are visible internally for debugging.
- Learning feedback usefulness
  - Baseline: Unknown.
  - Target: At least 50% of learner/teacher alpha testers rate the section-level feedback as useful enough to guide the next practice session.
  - Guardrail: Clipping detected in fewer than 5% of completed preview outputs.
- Quality diagnostics
  - Baseline: Unknown.
  - Target: 100% of completed jobs include pitch/timing/confidence reports. Jobs with preview output also include clipping reports.
  - Guardrail: Evaluation artifact generation failure does not hide final render failure states.
- Safety and rights
  - Baseline: Unknown.
  - Target: 100% of conversion/export requests produce an allow/deny decision and audit record.
  - Guardrail: If audit logging fails for rights-sensitive operations, the system fails closed.
- Full-song technical feasibility
  - Baseline: Unknown.
  - Target: Internal alpha produces enough processing data to choose one of three paths: full-song one-pass, chunked full-song, or section-limited alpha.
  - Guardrail: The product must not imply full-song support if only section-limited processing is technically reliable.

## Risks

- Quality risk: A technically complete pipeline may still produce vocals that users find unnatural or unusable.
- Pedagogy risk: Pitch/timing numbers may be technically correct but not actionable for music learners or teachers.
- Scope risk: Attempting full voice conversion, expression, custom voices, and full-track rendering in MVP will likely delay alpha validation.
- Safety risk: Voice rights and consent gaps can create product, legal, and trust risk even with internal alpha testers.
- Delivery risk: Current app is auth UI only, while upload/result screens and job orchestration are not yet visible in code.
- Integration risk: BFF currently exposes `me`; signup/login GraphQL mutations and upload/job APIs appear not yet implemented.
- Engine risk: Audio ingest is partially implemented, while many downstream engines are currently documented rather than implemented.
- Full-song risk: J-POP/utaite one-song inputs may exceed early engine assumptions around duration, chunking, memory, storage, and latency.
- Metric risk: There is no real baseline for song-level completion, feedback usefulness, preview quality, or activation.
- UX risk: Users may expect polished singing output, while alpha output may be a learning report plus limited preview.

## Dependencies

- App: upload screen, job status screen, learning report screen, result playback/download UI, token/session integration
- BFF: GraphQL mutations/queries for signup, login, create project/job, upload initiation, job status, result retrieval
- API Gateway: orchestration APIs for auth, project/job, asset, conversion, and user context
- User Service: account, auth, user identity, role/status
- Storage: source audio, canonical wav, manifests, render outputs, downloadable artifacts
- Queue/Eventing: NATS JetStream for audio/engine pipeline events, Redis/BullMQ if used for app-facing async jobs
- Engines: audio ingest, voice analysis, voice pitch, phoneme alignment, rhythm timing, melody mapping, singing synthesis, vocoder render, mix master, evaluation, safety rights
- Infra: PostgreSQL, Redis, NATS, local or object storage, Kubernetes deployments, structured logs
- Policy: terms, privacy policy, own-voice consent, audit retention, disallowed voice-use policy
- Music domain inputs: target song metadata, lyrics handling, BPM/key source, reference/target alignment policy

## Open Questions

### Product And User

- 내부 alpha의 사용자 구성은 음악 학습자와 음악 교육자 각각 몇 명으로 둘 것인가?
- 사용자가 기대하는 1차 산출물은 "노래처럼 들리는 self-voice preview"인가, "피치/타이밍/발음 학습 리포트"인가, 아니면 둘 다인가?
- 학습자용 화면과 교육자용 화면을 같은 결과 화면으로 시작할 것인가, 역할별로 다르게 보여줄 것인가?
- 한국어만 MVP로 고정할 것인가, 영어/일본어 등도 early scope에 넣을 것인가?
- J-POP/우타이테 학습 맥락에서 일본어 가사 정렬은 내부 alpha에 포함할 것인가, 아니면 한국어/로마자/가사 미입력 fallback부터 시작할 것인가?
- 원곡/목표곡 reference audio를 받을 것인가, 아니면 사용자가 부른 본인 음성만 먼저 받을 것인가?
- 사용자가 입력해야 하는 최소 metadata는 무엇인가: BPM, key, lyrics, language 중 무엇을 필수로 둘 것인가?

### Safety And Policy

- 본인 음성 확인은 내부 alpha에서 단순 checkbox로 충분한가, 아니면 녹음 동의/보이스 소유 인증 flow가 필요한가?
- 결과물 다운로드는 내부 alpha에서 허용할 것인가, 아니면 앱 내 재생만 허용할 것인가?
- 외부 공유는 내부 alpha에서 차단할 것인가?
- audit log 보관 기간과 접근 권한은 누가 결정하는가?

### Quality And Metrics

- "학습에 유용한 결과"의 기준은 무엇인가: teacher rating, learner rating, pitch deviation 이해도, practice 재시도율, report 재방문율 중 무엇을 primary metric으로 볼 것인가?
- J-POP/우타이테 한 곡 입력에서 허용 가능한 처리 시간은 몇 분인가?
- 한 곡 기준은 평균 3-5분 곡 전체인가, 1절/후렴 같은 구간 단위도 alpha 성공으로 볼 것인가?
- 결과가 나쁘더라도 job은 `completed`로 볼 것인가, quality threshold 미달이면 `failed` 또는 `needs_review`로 볼 것인가?
- preview 품질이 낮아도 학습 리포트가 좋으면 alpha success로 볼 것인가?

### Technical Scope

- upload는 앱에서 BFF로 직접 보낼 것인가, presigned URL/object storage를 사용할 것인가?
- conversion job의 source of truth는 어떤 서비스가 소유할 것인가?
- NATS 기반 엔진 이벤트와 앱-facing job status를 어떤 저장소/서비스에서 동기화할 것인가?
- 현재 `worker`의 Redis/BullMQ 역할과 NATS 기반 engine pipeline 역할을 어떻게 나눌 것인가?
- downstream 엔진이 아직 구현되지 않은 단계에서는 mock engine, stub output, or partial pipeline 중 어떤 방식으로 UX를 검증할 것인가?
- 한 곡 입력을 one-pass로 처리할 것인가, chunked pipeline으로 처리할 것인가?
- chunking을 한다면 사용자에게 곡 전체 결과로 보여줄 병합 기준은 무엇인가?

### Launch

- 내부 alpha에서 허용 가능한 사용자 수와 데이터 보관 범위는 어디까지인가?
- 실패한 작업의 원본 오디오는 보관할 것인가, 즉시 삭제할 것인가?

## Assumptions

- 이 PRD는 코드와 문서 기반의 제품 초안이며, 실제 고객 인터뷰나 analytics baseline은 아직 반영되지 않았다.
- MVP의 제품 목표는 J-POP/우타이테 노래 한 곡 기준이지만, 내부 alpha에서는 기술 검토 결과에 따라 chunked full-song 또는 section-limited alpha로 축소될 수 있다.
- 주요 사용자는 음악 학습자와 음악 교육자다.
- 사용자 본인 음성만 허용한다.
- 출시 형태는 내부 alpha다.
- Safety Rights는 feature가 아니라 launch gate로 취급한다.
- 앱은 모바일 first이지만, Expo Web에서도 핵심 flow를 유지한다.
- 현재 구현 상태를 기준으로 auth 연동, upload/job API, result UI, engine orchestration이 주요 신규 개발 범위다.
- PRD 승인 전에는 primary output definition, full-song 처리 방식, success metric target을 반드시 확정해야 한다.
