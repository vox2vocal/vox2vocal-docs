# Vox2Vocal MVP PRD Draft

문서 버전: v0.1  
작성일: 2026-06-09  
상태: 초안  
작성 기준: `pm-context` + `prd-writer` skill 기준

## Context Brief

### Brief

- Product or feature: Vox2Vocal MVP
- Target users: 짧은 음성, 가이드 보컬, 멜로디 아이디어를 보컬 트랙으로 빠르게 실험하려는 크리에이터, 보컬 프로듀서, 작곡가, 데모 제작자
- Problem: 사용자는 말이나 가이드 보이스로 떠오른 멜로디/보컬 아이디어를 실제로 들을 수 있는 보컬 트랙 형태로 빠르게 검증하기 어렵다. 기존 워크플로우는 녹음, 튠, 편집, 합성, 믹싱 단계가 분리되어 있고 기술 장벽이 높다.
- Goal: 가입한 사용자가 짧은 음성/가이드 보컬을 업로드하고, 시스템이 표준화/분석/피치/타이밍/멜로디/합성/렌더/후처리를 거쳐 다운로드 가능한 보컬 preview를 생성하는 1차 end-to-end 흐름을 검증한다.
- Constraints: 현재 앱은 로그인/회원가입 UI 중심이며 실제 인증 API 연결 전 단계다. 백엔드는 BFF, API Gateway, User Service의 인증 계약이 존재한다. 엔진은 전체 아키텍처와 일부 audio-ingest 구현이 존재하며, 나머지 엔진은 문서 기준 MVP 범위가 정의되어 있다.
- Success criteria: baseline과 target은 아직 실제 사용자/처리 데이터가 없어 가정이 필요하다. MVP에서는 activation, job completion, processing reliability, preview quality, safety block rate를 우선 측정한다.

### Known Facts

- Expo React Native App/Web 앱이 있고 로그인, 회원가입 화면이 구현되어 있다. Source: `vox2vocal-app/README.md`, `vox2vocal-app/src/features/auth/`
- BFF는 앱용 GraphQL endpoint를 제공하고 API Gateway를 gRPC로 호출하는 구조다. Source: `vox2vocal-bff-server/README.md`
- API Gateway는 `SignUp`, `Login`, `GetCurrentUser` gRPC 계약을 가진다. Source: `vox2vocal-api-gateway/proto/gateway.proto`
- User Service는 사용자 도메인, PostgreSQL, Prisma, 비밀번호 인증 정책을 소유한다. Source: `vox2vocal-user-service/README.md`, `src/users/policies/`
- 엔진 아키텍처는 Audio Ingest에서 시작해 Voice Analysis, Voice Pitch, Phoneme Alignment, Rhythm Timing, Melody Mapping, Singing Synthesis, Vocoder Render, Mix Master, Evaluation, Safety Rights로 이어지는 파이프라인을 지향한다. Source: `vox2vocal-docs/engine/README.md`
- Safety Rights는 초기 MVP부터 포함되어야 하며 사용자 본인 업로드 보이스만 허용하는 방향이 문서화되어 있다. Source: `vox2vocal-docs/engine/safety-rights/README.md`

### Assumptions

- 1차 MVP의 핵심 검증은 "상업 출시 품질의 완성 보컬"이 아니라 "짧은 입력을 끝까지 처리해 사용자가 preview를 듣고 가능성을 판단하는 경험"이다.
- MVP 입력 길이는 짧은 monophonic voice phrase 중심으로 제한한다.
- 초기 target voice는 사용자 본인 업로드 보이스 또는 시스템이 허용한 단일 기본 보이스 모델로 제한한다.
- 외부 배포, 상업 이용, 제3자 음성 복제는 MVP 범위에서 제외하거나 Safety Rights에서 차단한다.
- 모바일 앱을 primary surface로 두되, Expo Web도 같은 핵심 흐름을 지원하는 방향으로 설계한다.

### Risks

- 보컬 합성 품질이 낮으면 사용자는 end-to-end 흐름이 완성되어도 제품 가치를 느끼지 못할 수 있다.
- 권리/동의 정책이 약하면 제품 리스크가 기술 리스크보다 커질 수 있다.
- 전체 엔진을 한 번에 MVP로 묶으면 scope creep이 발생할 가능성이 높다.
- 현재 인증 UI와 백엔드 인증 계약 사이의 실제 연동이 완료되지 않으면 upload/conversion 흐름의 사용자 식별과 audit이 막힌다.

### Recommended Next Skill

- Skill: `prd-writer`
- Reason: 프로젝트 문서와 코드에서 제품 방향, 현재 구현 상태, 엔진 MVP 범위를 확인했으므로 engineering handoff 가능한 PRD 초안 작성으로 진행한다.

## Executive Summary

Vox2Vocal MVP는 사용자가 짧은 음성 또는 가이드 보컬을 업로드하면, 시스템이 이를 내부 표준 오디오로 변환하고, 발화/피치/타이밍/멜로디 정보를 분석한 뒤, 단일 보이스 기반의 짧은 보컬 preview를 생성해 사용자가 듣고 다운로드할 수 있게 하는 1차 제품이다.

이 PRD의 목적은 "음성에서 보컬로"라는 핵심 가치가 실제 사용자 흐름에서 검증 가능한지 확인하는 것이다. 따라서 MVP는 고급 음색 변환, 상업 배포, 다중 보이스 모델, 정교한 DAW 편집 기능보다 가입, 업로드, 처리 상태 확인, preview 재생, 기본 품질 리포트, 권한 차단을 우선한다.

## Problem Statement

작곡가, 프로듀서, 크리에이터는 멜로디나 보컬 아이디어를 말하거나 흥얼거리는 방식으로 빠르게 떠올리지만, 이를 실제 보컬 트랙처럼 들어보려면 녹음, 피치 보정, 타이밍 보정, 보컬 합성, 믹싱을 여러 도구에서 처리해야 한다. 이 과정은 느리고 복잡하며, 기술 장벽 때문에 아이디어 검증 전환율이 낮다.

Vox2Vocal은 사용자의 음성 입력을 보컬 제작 파이프라인에 넣어 "아이디어를 들을 수 있는 보컬 preview"로 바꾸는 시간을 줄여야 한다. 다만 음성 권리, 합성 품질, 처리 안정성, 사용자 기대치 관리가 동시에 해결되어야 한다.

## Target Users

- Primary: 보컬 멜로디 아이디어를 빠르게 데모로 듣고 싶은 작곡가, 프로듀서, 싱어송라이터
- Secondary: 보컬 녹음 전 가사/멜로디 스케치를 확인하려는 콘텐츠 크리에이터
- Internal/early users: 엔진 품질을 검증하고 회귀 테스트를 수행하는 개발자, QA, 제품팀

## Goals

- 사용자가 계정을 만들고 로그인한 상태에서 오디오 변환 작업을 생성할 수 있다.
- 사용자가 `wav` 또는 `mp3` 짧은 음성/가이드 보컬 파일을 업로드할 수 있다.
- 시스템이 업로드 파일을 표준 오디오 asset으로 변환하고 처리 상태를 추적할 수 있다.
- 시스템이 짧은 monophonic phrase 기준으로 pitch, timing, melody 정보를 생성할 수 있다.
- 시스템이 단일 보이스 기반 preview vocal track을 생성하고 사용자가 재생/다운로드할 수 있다.
- 모든 변환 요청은 사용자 권한과 보이스 사용 정책을 통과해야 하며 audit 가능한 기록을 남긴다.
- 사용자는 실패 이유와 재시도 가능 여부를 이해할 수 있다.

## Non-goals

- 제3자 유명인, 아티스트, 캐릭터 보이스 복제 지원
- 사용자가 업로드하지 않았거나 동의가 확인되지 않은 target voice model 사용
- 상업 배포 라이선스, 정산, 마켓플레이스 기능
- DAW 수준의 waveform 편집, MIDI editor, 멀티트랙 믹싱
- 긴 곡 전체 변환, polyphonic input 처리, 실시간 변환
- 여러 언어 전체 지원. MVP는 한국어 중심으로 가정한다.
- 고급 expression control, voice conversion, custom voice training

## Scope

### In Scope

- 이메일/비밀번호 기반 회원가입, 로그인, 현재 사용자 조회
- 모바일 앱 및 웹에서 인증 후 MVP 작업 화면 접근
- 오디오 업로드: `wav`, `mp3`, 짧은 monophonic voice phrase
- 입력 metadata: 곡/작업명, 언어, BPM, key, 가사 또는 대본 텍스트
- Audio Ingest: 표준 PCM/WAV 변환, mono 변환, 무음 탐지, 발화 구간 timestamp 생성
- Voice Analysis: RMS energy, 발화 속도, 휴지 구간, 강세 후보 timestamp
- Voice Pitch: F0 추출, confidence filtering, MIDI note 변환, JSON 결과
- Phoneme Alignment: 한국어 음절 단위 정렬, word boundary, alignment confidence
- Rhythm Timing: 고정 BPM 기준 1/4, 1/8, 1/16 grid 정렬과 duration 생성
- Melody Mapping: MIDI note sequence, key 기반 보정, note sequence JSON
- Singing Synthesis: 단일 보이스 모델 기반 짧은 vocal phrase acoustic feature 생성
- Vocoder Render: offline wav render, clipping report
- Mix Master: vocal-only EQ/compressor/limiter, loudness normalization, wav output
- Evaluation: pitch deviation, timing deviation, clipping 탐지, 엔진별 JSON 비교
- Safety Rights: 사용자 본인 업로드 보이스만 허용, target model 소유권 확인, conversion audit log
- 작업 상태 화면: queued, processing, completed, failed
- 결과 화면: preview 재생, 다운로드, 기본 품질 리포트, 실패 사유

### Out Of Scope

- 소셜 로그인 실제 연동
- 비밀번호 재설정 flow
- 결제/구독/크레딧
- 팀/협업 프로젝트
- 공개 공유 링크
- 고급 vocal editing UI
- model training pipeline
- 운영자 관리 콘솔

## User Stories

- As a new creator, I want to sign up and log in so that my uploaded voice assets and conversion jobs are tied to my account.
- As a creator, I want to upload a short voice or guide vocal file so that I can turn an idea into a vocal preview.
- As a creator, I want to enter BPM, key, language, and lyrics so that the generated vocal aligns with my musical intent.
- As a creator, I want to see processing progress and failure reasons so that I know whether to wait, retry, or change the input.
- As a creator, I want to preview and download the generated vocal wav so that I can judge whether the idea is worth developing.
- As a creator, I want unsafe or unauthorized voice usage to be blocked clearly so that I understand what is allowed.
- As a product/engineering team member, I want evaluation metrics and engine artifacts so that I can debug quality regressions.

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

- The user must be able to create a conversion project with title, source file, language, optional BPM, optional key, and optional lyrics.
- MVP must require users to acknowledge that they have rights to the uploaded voice.
- The system must assign a project/job id before engine processing starts.

### FR-004 Upload And Audio Ingest

- The system must accept `wav` and `mp3` inputs for MVP.
- The system must reject unsupported formats with a clear reason.
- The system must convert accepted input to a standard mono WAV/PCM asset.
- The system must produce metadata including sample rate, channels, duration, loudness estimate, silence segments, speech/voice segments, and `audio_asset_id`.

### FR-005 Safety Rights Check

- The system must verify that the user owns or is allowed to use the source audio asset.
- MVP must allow only the user's own uploaded source voice and approved default target voice model.
- The system must deny unauthorized target voice usage before synthesis or export.
- Each conversion request must create an audit log with user id, source asset id, target model id, operation, decision, and policy reason.

### FR-006 Voice Analysis

- The system must produce energy RMS curve, speech rate estimate, pause segments, and stress candidate timestamps from the canonical audio.
- The analysis output must be tied to the job id and audio asset id.

### FR-007 Pitch Extraction

- The system must extract F0 frames for short monophonic input.
- The system must mark voiced/unvoiced frames and confidence score.
- The system must convert confident F0 frames into MIDI note numbers.
- Low-confidence segments must be surfaced in the evaluation report.

### FR-008 Text And Syllable Alignment

- If lyrics/text are provided, the system must produce Korean syllable timeline, word boundaries, and alignment confidence.
- If text is missing, the system must either request text for synthesis or use a documented fallback path.

### FR-009 Rhythm And Melody Mapping

- The system must generate a beat-aligned syllable timeline using BPM and grid options.
- MVP must support 1/4, 1/8, and 1/16 note grids.
- The system must produce note sequence JSON with note start/end, target pitch, duration, and confidence.
- The system must support C major default or user-provided key.

### FR-010 Vocal Synthesis And Render

- The system must generate a short monophonic vocal phrase using a single approved voice model.
- The system must render the result as a wav file through offline vocoder rendering.
- The render output must include clipping and loudness metadata.

### FR-011 Basic Mix/Master

- The system must apply vocal-only basic EQ, compressor, limiter, and loudness normalization.
- The output must remain downloadable as wav.
- The system must preserve the unprocessed render artifact for debugging.

### FR-012 Job Status And Result UI

- The app must show job states: `created`, `queued`, `processing`, `completed`, `failed`, `blocked`.
- Completed jobs must show audio preview playback, download action, and basic quality report.
- Failed jobs must show stage, reason, and whether retry is allowed.
- Blocked jobs must show policy reason without exposing sensitive policy internals.

### FR-013 Evaluation And Observability

- The system must store pitch deviation, timing deviation, clipping detection, and engine artifact comparison results.
- Each engine stage must emit structured logs tied to job id.
- Quality reports must be available for internal review even if the user-facing version is simplified.

## Acceptance Criteria

- Given a new user enters valid signup details and accepts terms, when they submit, then an account is created and the app receives auth tokens and user profile.
- Given a user enters invalid email or password, when they submit signup/login, then validation errors are shown and no conversion job is created.
- Given an authenticated user uploads a supported short `wav` or `mp3`, when the upload completes, then the system creates a conversion job and an `audio_asset_id`.
- Given an unsupported file format is uploaded, when validation runs, then the user sees a clear unsupported format message.
- Given the user does not confirm rights to the source audio, when they attempt to start conversion, then the job is blocked before engine processing.
- Given the user selects an unauthorized target voice model, when Safety Rights runs, then the job is denied and an audit record is created.
- Given a valid short monophonic input with required metadata, when the pipeline completes, then a wav preview is available for playback and download.
- Given a pipeline stage fails, when the user views the job, then the failed stage, plain-language reason, and retry guidance are shown.
- Given a completed job, when internal reviewers inspect the artifacts, then pitch deviation, timing deviation, clipping status, and stage outputs are available.
- Given mobile viewport `360 x 640`, when the user signs up, logs in, uploads, and views a result, then primary actions remain reachable without horizontal scroll.

## Success Metrics

- Activation
  - Baseline: Unknown. No production usage data available.
  - Target: At least 40% of signed-in MVP testers create one conversion job within their first session.
  - Guardrail: Signup/login failure due to client/backend integration errors below 2% of attempts.
- Job completion
  - Baseline: Unknown. Engine pipeline not yet fully implemented.
  - Target: At least 70% of valid supported input jobs reach `completed` in controlled MVP testing.
  - Guardrail: No unauthorized voice jobs reach synthesis/export.
- Time to preview
  - Baseline: Unknown.
  - Target: P50 under 3 minutes for short phrase inputs in MVP environment.
  - Guardrail: P95 processing time and failure reasons are visible internally for debugging.
- Preview usefulness
  - Baseline: Unknown.
  - Target: At least 50% of testers rate the preview as useful enough to iterate on the idea.
  - Guardrail: Clipping detected in fewer than 5% of completed preview outputs.
- Quality diagnostics
  - Baseline: Unknown.
  - Target: 100% of completed jobs include pitch/timing/clipping reports.
  - Guardrail: Evaluation artifact generation failure does not hide final render failure states.
- Safety and rights
  - Baseline: Unknown.
  - Target: 100% of conversion/export requests produce an allow/deny decision and audit record.
  - Guardrail: If audit logging fails for rights-sensitive operations, the system fails closed.

## Risks

- Quality risk: A technically complete pipeline may still produce vocals that users find unnatural or unusable.
- Scope risk: Attempting full voice conversion, expression, custom voices, and full-track rendering in MVP will likely delay end-to-end validation.
- Safety risk: Voice rights and consent gaps can create product, legal, and trust risk even with a small beta.
- Delivery risk: Current app is auth UI only, while upload/result screens and job orchestration are not yet visible in code.
- Integration risk: BFF currently exposes `me`; signup/login GraphQL mutations and upload/job APIs appear not yet implemented.
- Engine risk: Audio ingest is partially implemented, while many downstream engines are currently documented rather than implemented.
- Metric risk: There is no real baseline for conversion completion, preview quality, or activation.
- UX risk: Users may expect finished vocals, while MVP output may be closer to a technical preview.

## Dependencies

- App: upload screen, job status screen, result playback/download UI, token/session integration
- BFF: GraphQL mutations/queries for signup, login, create project/job, upload initiation, job status, result retrieval
- API Gateway: orchestration APIs for auth, project/job, asset, conversion, and user context
- User Service: account, auth, user identity, role/status
- Storage: source audio, canonical wav, manifests, render outputs, downloadable artifacts
- Queue/Eventing: NATS JetStream for audio/engine pipeline events, Redis/BullMQ if used for app-facing async jobs
- Engines: audio ingest, voice analysis, voice pitch, phoneme alignment, rhythm timing, melody mapping, singing synthesis, vocoder render, mix master, evaluation, safety rights
- Infra: PostgreSQL, Redis, NATS, local or object storage, Kubernetes deployments, structured logs
- Policy: terms, privacy policy, voice rights consent, audit retention, allowed/disallowed target voice model policy

## Open Questions

### Product And User

- MVP의 첫 타깃은 작곡가/프로듀서인가, 일반 크리에이터인가, 아니면 내부 기술 검증 사용자/베타 테스터인가?
- 사용자가 기대하는 1차 산출물은 "노래처럼 들리는 preview"인가, "피치/멜로디 변환 가능성 리포트"인가?
- 한국어만 MVP로 고정할 것인가, 영어/일본어 등도 early scope에 넣을 것인가?
- 사용자가 직접 부른/말한 본인 음성만 입력하게 할 것인가, 가이드 멜로디/반주/레퍼런스 보컬도 같은 flow에서 받을 것인가?
- 사용자가 입력해야 하는 최소 metadata는 무엇인가: BPM, key, lyrics, language 중 무엇을 필수로 둘 것인가?

### Safety And Policy

- 기본 보이스 모델을 제공할 것인가, 아니면 MVP는 사용자 본인 보이스만 허용할 것인가?
- 보이스 권리 확인은 단순 checkbox로 시작할 것인가, 녹음 동의/보이스 소유 인증 flow가 필요한가?
- 결과물 다운로드와 외부 공유는 MVP에서 허용할 것인가?
- audit log 보관 기간과 접근 권한은 누가 결정하는가?

### Quality And Metrics

- "사용 가능한 preview"의 품질 기준은 무엇인가: 사용자 평점, pitch deviation, timing deviation, 재생 완료율, 다운로드율 중 무엇을 primary metric으로 볼 것인가?
- MVP에서 허용 가능한 처리 시간은 몇 분인가?
- 짧은 phrase의 최대 길이는 10초, 30초, 60초 중 어디까지인가?
- 결과가 나쁘더라도 job은 `completed`로 볼 것인가, quality threshold 미달이면 `failed` 또는 `needs_review`로 볼 것인가?

### Technical Scope

- upload는 앱에서 BFF로 직접 보낼 것인가, presigned URL/object storage를 사용할 것인가?
- conversion job의 source of truth는 어떤 서비스가 소유할 것인가?
- NATS 기반 엔진 이벤트와 앱-facing job status를 어떤 저장소/서비스에서 동기화할 것인가?
- 현재 `worker`의 Redis/BullMQ 역할과 NATS 기반 engine pipeline 역할을 어떻게 나눌 것인가?
- downstream 엔진이 아직 구현되지 않은 단계에서는 mock engine, stub output, or partial pipeline 중 어떤 방식으로 UX를 검증할 것인가?

### Launch

- MVP는 내부 alpha, closed beta, public beta 중 무엇인가?
- 법무/보안 검토 없이 허용 가능한 사용자 수와 데이터 보관 범위는 어디까지인가?
- 실패한 작업의 원본 오디오는 보관할 것인가, 즉시 삭제할 것인가?

## Assumptions

- 이 PRD는 코드와 문서 기반의 제품 초안이며, 실제 고객 인터뷰나 analytics baseline은 아직 반영되지 않았다.
- MVP는 짧은 phrase 중심으로 제한해 품질/속도/권리 리스크를 낮춘다.
- Safety Rights는 feature가 아니라 launch gate로 취급한다.
- 앱은 모바일 first이지만, Expo Web에서도 핵심 flow를 유지한다.
- 현재 구현 상태를 기준으로 auth 연동, upload/job API, result UI, engine orchestration이 주요 신규 개발 범위다.
- PRD 승인 전에는 target user, 입력 길이, 기본 보이스 정책, success metric target을 반드시 확정해야 한다.
