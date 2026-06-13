# Vox2Vocal MVP PRD 리뷰

리뷰 대상 문서: `pm/vox2vocal-mvp-prd.md` v0.9  
리뷰 기준: `prd-reviewer` skill  
리뷰일: 2026-06-13

## 판정

- 상태: P0 내부 alpha는 조건부 Go, 전체 곡/productized build는 No-go.
- 한 줄 판단: PRD가 `Mist`를 section map 기반으로 재정의하고 `chorus_1`을 `1:44-2:16` P0 target으로 고정한 것은 통과다. 다만 reference audio 권리 승인, P0 job state owner, upload/storage 계약, deletion owner, 최소 엔진 경로가 확정되기 전에는 build start가 여전히 위험하다.

## 핵심 이슈

- 이슈: `Mist` section timestamp는 제안되었지만, 실제 등록할 reference audio asset 기준으로 검수되어야 한다.
- 중요한 이유: 동일 곡이라도 플랫폼/앨범/업로드 버전에 따라 intro 길이가 달라질 수 있다. timestamp가 reference asset과 어긋나면 pitch target, upload validation, preview QA가 모두 흔들린다.
- 심각도: High.

- 이슈: reference audio rights clearance owner와 승인 기준이 아직 비어 있다.
- 중요한 이유: 실제 노래를 reference로 쓰는 제품은 저작권, provider 약관, storage, deletion 리스크가 핵심 blocker다. `rights_blocked` 정책은 생겼지만 누가 어떤 evidence로 승인할지 정해야 한다.
- 심각도: Blocker.

- 이슈: Failure reason tagging은 추가됐지만 UI timing과 tag taxonomy가 아직 최종 확정되지 않았다.
- 중요한 이유: 4점 미만 평가가 왜 실패했는지 구분하지 못하면 voice similarity, pitch, timing, artifact, playback 문제를 분리해 다음 build 결정을 내리기 어렵다.
- 심각도: High.

- 이슈: Job state ownership은 기술 구현 강제에서 capability로 낮아졌지만 owner가 아직 비어 있다.
- 중요한 이유: BFF, worker, API Gateway 중 어디가 P0 canonical state를 소유하는지 정하지 않으면 retry, partial artifact, failed_with_partial_artifacts, app-facing status가 흔들린다.
- 심각도: High.

- 이슈: 최소 엔진 경로가 아직 확정되지 않았다.
- 중요한 이유: PRD는 self-voice section preview를 P0 핵심 가치로 둔다. mock-only, partial-real pipeline, real synthesis 중 무엇으로 alpha를 판단할지 정하지 않으면 구현 ticket과 alpha 성공 기준이 갈라진다.
- 심각도: High.

- 이슈: Retention/deletion owner가 아직 없다.
- 중요한 이유: raw audio 30일 보관, deletion evidence, NIST SP 800-88 기반 sanitization 정책이 추가됐지만 owner와 실행 책임이 지정되어야 운영 통제가 된다.
- 심각도: High.

## 권장 수정

- 수정: `Mist` section map을 실제 reference audio asset 기준으로 검수하고 `chorus_1 = 1:44-2:16`을 확정 또는 조정한다.
- 필요한 owner 또는 결정: Product + 음악 교육 전문가 + engine owner.

- 수정: reference audio source/provenance, rights clearance status, allowed use, retention period, deletion owner의 승인 체크리스트를 만든다.
- 필요한 owner 또는 결정: Product + policy/legal + platform/storage owner.

- 수정: 4점 미만 failure tag UI를 preview 재생 직후 강제하고, `other` 입력은 선택형으로 둔다.
- 필요한 owner 또는 결정: Product + Design.

- 수정: P0 canonical job state owner를 하나로 지정한다. 객관적 추천은 신규 서비스가 아니라 기존 worker 또는 backend bounded module에서 시작하는 것이다.
- 필요한 owner 또는 결정: Engineering.

- 수정: P0 최소 엔진 경로를 partial-real pipeline으로 둘지, real synthesis까지 요구할지 결정한다.
- 필요한 owner 또는 결정: Product + engine owner.

- 수정: retention/deletion owner와 deletion evidence format을 정한다.
- 필요한 owner 또는 결정: Platform/storage owner + policy owner.

## Build 전 질문

- 질문: 현재 `Mist` section map은 실제 등록 reference audio asset 기준으로 검수됐는가?
- 왜 막히는가: timestamp drift가 있으면 P0 결과가 재현되지 않는다.

- 질문: reference audio rights clearance 최종 승인자는 누구이며, 승인 evidence는 어디에 저장되는가?
- 왜 막히는가: 실제 노래를 쓰는 제품에서 권리 검증은 launch gate다.

- 질문: P0 job state owner는 worker, API Gateway, BFF 중 어디인가?
- 왜 막히는가: final status와 partial artifact 처리의 source of truth가 필요하다.

- 질문: P0 최소 엔진 경로는 mock, partial-real pipeline, real synthesis 중 무엇인가?
- 왜 막히는가: self-voice preview가 핵심 가치라 mock-only로는 alpha 성공을 판정하기 어렵다.

- 질문: 4점 미만 failure tag의 최종 목록을 그대로 확정할 것인가?
- 왜 막히는가: metric taxonomy가 바뀌면 alpha 결과 비교가 어려워진다.

- 질문: raw audio 삭제 evidence는 어떤 로그/리포트로 남길 것인가?
- 왜 막히는가: retention policy가 실제 운영 통제인지 확인할 수 있어야 한다.

## 진행/중단 판단

- 권고: P0 alpha build는 조건부 Go.
- 진행 조건: reference asset 기준 section map 검수, rights clearance owner/evidence 확정, P0 job state owner 확정, 최소 엔진 경로 확정, failure tag UI 확정, upload/storage 계약 확정, retention/deletion owner 확정.
- No-go 조건: 권리 검증 없는 reference audio 사용, full-song, lyric sync, vocal-mode 자동 분석, provider audio ingestion, 신규 orchestrator 서비스가 P0 필수 범위로 다시 들어오는 경우.
