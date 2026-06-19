# Vox2Vocal MVP 기획 / API 계약 리뷰

리뷰일: 2026-06-15
적용 skill: `trd-reviewer`
리뷰 대상:

- `pm/vox2vocal-mvp-prd.md` v0.13
- `pm/vox2vocal-mvp-feature-definition.md` v0.5
- `pm/vox2vocal-mvp-page-flow-plan.md` v0.3
- `pm/vox2vocal-mvp-trd.md` v0.3
- `pm/vox2vocal-mvp-api-data-contract-plan.md` v0.2

## 판정

상태: **조건부 진행 가능, 구현 티켓 전 계약 기반 티켓을 먼저 작성해야 함**

UI/page planning은 UI 디자인 에이전트에게 와이어프레임 작업을 넘길 수 있을 만큼 정리되어 있다. API/data 계약은 이전보다 구체화되었고, proto/service 이름, downstream engine event schema, playback telemetry 상수, rights/risk launch checklist, `needs_review` SLA, `PreviewArtifact` 물리 스키마 선택까지 문서에 반영되었다.

다만 backend/API/worker/engine 구현 티켓을 바로 기능 단위로 쪼개기 전에, 먼저 계약 구현 티켓이 필요하다. 즉, `spec-to-tickets` 단계에서는 기능 티켓보다 proto 생성, event schema validation, GraphQL-proto mapping test, playback telemetry validation, rights/risk seed data 작업을 선행 티켓으로 분리해야 한다.

## 요구사항 커버리지

- 학습자 flow는 auth, 곡 선택, section 선택, recorder/take review, upload, processing, preview, playback telemetry, rating, failure tag, deletion까지 커버한다.
- 내부 surface는 제한된 admin, review queue/detail, governance evidence를 커버한다.
- TRD는 `JobProjection`, `PreviewArtifact`, `PlaybackSession`, `PlaybackEvent` 공유 계약을 포함한다.
- API/data contract plan은 endpoint behavior, request/response shape, validation, error cases, permission, compatibility, data model, migration, indexing, events/analytics를 포함한다.
- P0 contact follow-up 범위는 일관되다. P0에서는 disabled capability status만 저장하고 연락처 값, 발송, 복호화, export는 하지 않는다.

## 기존 핵심 이슈와 반영 상태

### 1. Proto/service ownership 부족

- 이전 문제: GraphQL operation은 있었지만 proto package, service, message 이름과 owning repo가 명확하지 않았다.
- 반영 상태: `pm/vox2vocal-mvp-api-data-contract-plan.md` v0.2에 `VoxAuthService`, `VoxConsentService`, `VoxCatalogService`, `VoxUploadService`, `VoxJobService`, `VoxPlaybackService`, `VoxReviewService`, `VoxGovernanceService`를 추가했다.
- 남은 작업: 실제 `.proto` 파일 생성과 GraphQL-proto mapping test가 필요하다.

### 2. Downstream engine event schema 부족

- 이전 문제: `voice_pitch`, `target_pitch_mapping`, `preview_synthesis`, `render`, `preview_evaluation`, `safety_rights` payload가 열려 있었다.
- 반영 상태: 공통 `EngineStageEvent<TPayload>` envelope와 stage별 success/failure payload minimum을 추가했다.
- 남은 작업: 각 engine repo에서 해당 schema version을 구현하고 worker adapter contract test를 만들어야 한다.

### 3. Rights/risk approval owner 부족

- 이전 문제: 실제 곡 `Mist` 사용 전에 누가 승인하고 어떤 evidence를 어디에 남기는지 불명확했다.
- 반영 상태: P0 temporary approver, evidence ref, internal allowlist, allowed section, prohibited uses, re-review deadline, kill-switch owner, complaint owner를 launch checklist로 정의했다.
- 남은 작업: DB source of truth에 실제 `Mist` rights/risk record와 evidence ref를 seed/admin-write 해야 한다.

### 4. Playback telemetry tolerance 부족

- 이전 문제: `preview_played=true` 기준은 있었지만 late flush와 impossible progress 수치가 없었다.
- 반영 상태: `PLAYBACK_LATE_FLUSH_GRACE_MS=15000`, `PLAYBACK_WALL_CLOCK_TOLERANCE_MS=2000`, `PLAYBACK_MAX_EVENT_BATCH_SIZE=50`, stale session rejection rule을 추가했다.
- 남은 작업: server-side coverage 계산과 anti-abuse test가 필요하다.

### 5. `needs_review` SLA 부족

- 이전 문제: low-confidence/disputed 결과가 쌓였을 때 누가 언제 처리하는지 없었다.
- 반영 상태: review reason별 owner, 1영업일 확인, 3영업일 해소, 7일 초과 fallback state를 추가했다.
- 남은 작업: review queue UI/API와 fallback state transition test가 필요하다.

### 6. `PreviewArtifact` 물리 스키마 미결정

- 이전 문제: 별도 테이블인지 `artifact_refs` JSON metadata인지 열려 있었다.
- 반영 상태: P0에서는 별도 `preview_artifacts` physical table로 결정했다.
- 이유: metric eligibility, playback gating, review queue, lineage verification이 typed indexed field를 필요로 한다.

## 남은 중요 리스크

- 계약 문서는 구체화되었지만 아직 실제 proto 파일, generated client, contract test가 없다.
- Engine event schema는 문서상 닫혔지만 각 엔진 repo의 소유자와 versioning 방식은 구현 단계에서 확정해야 한다.
- `Mist` learner exposure는 DB에 rights/risk evidence가 실제로 들어가기 전까지 여전히 No-go다.
- Playback telemetry는 수치가 생겼지만, 모바일 background/foreground 이벤트 손실을 실제 기기에서 검증해야 한다.
- UI 디자인은 가능하지만 recorder, player, rating, failure tag, evidence table의 component-level decision은 아직 필요하다.

## 권장 수정 / 다음 작업

- `spec-to-tickets` 전에 contract foundation epic을 먼저 만든다.
- 첫 티켓 묶음은 proto 파일 생성, GraphQL-proto mapping, shared enum/error code, generated client publish로 잡는다.
- 두 번째 티켓 묶음은 downstream engine event schema validation과 worker StageResult adapter test로 잡는다.
- 세 번째 티켓 묶음은 `Mist` rights/risk seed/admin-write path와 kill switch를 잡는다.
- 네 번째 티켓 묶음은 playback telemetry server calculation, anti-abuse validation, rating unlock test를 잡는다.
- UI 디자인은 `pm/vox2vocal-mvp-page-flow-plan.md` 기준으로 병렬 진행 가능하다.

## 필요한 테스트 / 관측성

- GraphQL schema to proto mapping test
- proto to worker `JobProjection` mapping test
- unknown enum fallback과 unknown major schema version rejection test
- `EngineStageEvent<TPayload>` stage별 validation test
- duplicate/replay event dedupe test
- playback stale event, changed payload replay, impossible timeline range, muted/background exclusion test
- consent/rights/deletion changes between playback URL issuance and rating submission test
- rights/risk exposure decision, risk acceptance id, kill-switch owner, latest rights state audit event

## Rollout / Rollback 우려

- rollout은 contract freeze를 1단계로 두었기 때문에 이전보다 안전하다.
- rollback은 feature flag, outbox pause, playback URL issuance block으로 처리하는 방향이 적절하다.
- 이미 발급된 signed URL은 완전 revocation이 어렵기 때문에 짧은 TTL과 artifact blocked state가 P0 mitigation이다.
- contact follow-up은 P0에서 값이 존재하지 않아야 하므로 rollback risk가 낮다.

## Build 전 확인 질문

- 각 engine stage payload version의 최종 소유 repo는 어디인가?
- P0 이후 break-glass raw/canonical audio access 또는 contact plaintext reveal을 켤 경우, 두 번째 reviewer/security owner는 누가 되는가?
- playback telemetry는 P0 이후에도 single-event mutation으로 유지할 것인가, 아니면 batch endpoint를 추가할 것인가?
- `Mist` rights/risk evidence ref는 DB에서 어떤 private evidence location을 가리킬 것인가?

## Go / Revise / No-go

추천:

- **Go**: UI 디자인 와이어프레임과 page-level UX 설계
- **Go**: contract foundation 티켓 작성
- **Revise**: backend/API/worker/engine feature ticketing은 contract foundation 티켓을 먼저 분리한 뒤 진행
- **No-go**: `Mist` rights/risk evidence가 DB source of truth에 저장되기 전 learner exposure

`spec-to-tickets`로 넘어가기 위한 조건:

- proto service/message 파일 생성 티켓을 먼저 분리한다.
- downstream engine event schema validation 티켓을 먼저 분리한다.
- rights/risk launch checklist seed/admin-write 티켓을 먼저 분리한다.
- playback telemetry coverage calculation과 rating unlock test 티켓을 먼저 분리한다.
- 그 다음 learner flow, recorder, job status, result/rating, review/admin 화면 티켓으로 확장한다.
