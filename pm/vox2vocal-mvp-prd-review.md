# Vox2Vocal MVP PRD 리뷰

리뷰 대상 문서: `pm/vox2vocal-mvp-prd.md` v0.8  
리뷰 기준: `prd-reviewer` skill  
리뷰일: 2026-06-13

## 판정

- 상태: P0 내부 alpha는 조건부 Go, 전체 곡/productized build는 No-go.
- 한 줄 판단: PRD가 `Ken Kamikita - Mist` 후렴 self-voice preview 검증으로 잘 축소되었지만, 후렴 timestamp, failure reason tag UI, P0 job state owner, upload/storage 계약, deletion owner가 확정되기 전에는 build start가 위험하다.

## 핵심 이슈

- 이슈: 후렴 구간이 제품적으로는 정의되었지만 아직 timestamp가 없다.
- 중요한 이유: P0는 후렴 section start/end를 기준으로 upload validation, target pitch extraction, preview length, success criteria가 결정된다.
- 심각도: Blocker.

- 이슈: Failure reason tagging은 추가됐지만 UI timing과 tag taxonomy가 아직 최종 확정되지 않았다.
- 중요한 이유: 4점 미만 평가가 왜 실패했는지 구분하지 못하면 voice similarity, pitch, timing, artifact, playback 문제를 분리해 다음 build 결정을 내리기 어렵다.
- 심각도: High.

- 이슈: Job state ownership은 기술 구현 강제에서 capability로 낮아졌지만 owner가 아직 비어 있다.
- 중요한 이유: BFF, worker, API Gateway 중 어디가 P0 canonical state를 소유하는지 정하지 않으면 retry, partial artifact, failed_with_partial_artifacts, app-facing status가 흔들린다.
- 심각도: High.

- 이슈: P0에서 vocal-mode와 lyric sync는 잘 제외됐지만, P1 candidate data 문구가 남아 있어 운영자가 수집 범위를 넓힐 수 있다.
- 중요한 이유: P0 alpha의 목적은 preview loop 검증이다. label data 수집은 P1 opt-in으로 남겨야 하며 P0 success metric에 섞이면 안 된다.
- 심각도: Medium.

- 이슈: Retention/deletion owner가 아직 없다.
- 중요한 이유: raw audio 30일 보관과 1년 데이터셋 제외는 실제 deletion job, audit evidence, 승인자 없이 문장으로만 남을 수 있다.
- 심각도: High.

## 권장 수정

- 수정: `Mist` 후렴 start/end timestamp를 확정하고, 권장 길이 20-40초 안에서 P0 기본 길이를 선택한다.
- 필요한 owner 또는 결정: Product + 음악 교육 전문가 + engine owner.

- 수정: 4점 미만 failure tag UI를 preview 재생 직후 강제하고, `other` 입력은 선택형으로 둔다.
- 필요한 owner 또는 결정: Product + Design.

- 수정: P0 canonical job state owner를 하나로 지정한다. 객관적 추천은 신규 서비스가 아니라 기존 worker 또는 backend bounded module에서 시작하는 것이다.
- 필요한 owner 또는 결정: Engineering.

- 수정: P0 success dashboard에서 vocal-mode, lyric sync, full-song metric을 제거한다.
- 필요한 owner 또는 결정: Product.

- 수정: retention/deletion owner와 deletion evidence format을 정한다.
- 필요한 owner 또는 결정: Platform/storage owner + policy owner.

## Build 전 질문

- 질문: `Mist`의 P0 후렴 start/end timestamp는 몇 초인가?
- 왜 막히는가: section validation, target extraction, preview duration, test repeatability가 여기에 묶인다.

- 질문: P0 job state owner는 worker, API Gateway, BFF 중 어디인가?
- 왜 막히는가: final status와 partial artifact 처리의 source of truth가 필요하다.

- 질문: 4점 미만 failure tag의 최종 목록을 그대로 확정할 것인가?
- 왜 막히는가: metric taxonomy가 바뀌면 alpha 결과 비교가 어려워진다.

- 질문: raw audio 삭제 evidence는 어떤 로그/리포트로 남길 것인가?
- 왜 막히는가: retention policy가 실제 운영 통제인지 확인할 수 있어야 한다.

## 진행/중단 판단

- 권고: P0 alpha build는 조건부 Go.
- 진행 조건: 후렴 timestamp 확정, P0 job state owner 확정, failure tag UI 확정, upload/storage 계약 확정, retention/deletion owner 확정.
- No-go 조건: full-song, lyric sync, vocal-mode 자동 분석, provider ingestion, 신규 orchestrator 서비스가 P0 필수 범위로 다시 들어오는 경우.
