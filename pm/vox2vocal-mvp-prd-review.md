# Vox2Vocal MVP PRD 리뷰

리뷰 대상 문서: `pm/vox2vocal-mvp-prd.md` v0.7  
리뷰 기준: `prd-reviewer` skill  
리뷰일: 2026-06-13

## 판정

- 상태: P0 내부 alpha는 조건부 진행 가능, full-song/productized build는 No-go.
- 한 줄 판단: PRD는 J-POP 1곡의 후렴 구간, pitch-first P0, 분리 동의, candidate label data, Conversion Job Orchestrator 방향을 반영했지만, 구체 곡명, deletion owner, label taxonomy, upload/storage 세부 계약이 확정되기 전에는 build 착수가 위험하다.

## 핵심 이슈

- 이슈: P0 alpha가 후렴 section-first로 좁혀졌지만 여전히 여러 엔진 stage의 구현 준비도에 의존한다.
- 중요한 이유: synthesis, pitch extraction, alignment, evaluation을 한 번에 필수로 묶으면, 정작 핵심 가설인 self-voice preview가 학습자에게 가치가 있는지 검증하기 전에 일정과 품질 리스크가 커진다.
- 심각도: full MVP build 기준 Blocker. 단, section-first 처리와 partial output을 허용하면 P0에서는 관리 가능하다.

- 이슈: Song package 필수 필드는 확정됐지만 구체 곡명/아티스트와 운영 owner가 아직 부족하다.
- 중요한 이유: 관리자 등록에는 reference audio, metadata, source/provenance, section, retention 규칙이 필요하다. 필수 필드와 validation 동작이 고정되지 않으면 engineering이 안정적인 관리자 등록 흐름을 만들기 어렵다.
- 심각도: P0 build start 기준 Blocker.

- 이슈: Self-voice preview 성공 기준은 평가 문항과 scoring flow가 고정되어야 한다.
- 중요한 이유: "내 목소리처럼 들림"은 올바른 primary rating이다. 다만 모든 tester가 같은 preview를 들은 직후 같은 질문과 같은 척도로 답해야 alpha 결과를 비교할 수 있다. 특히 1번 문항은 5점 척도에서 최소 4점 이상이어야 성공 응답으로 본다.
- 심각도: High.

- 이슈: Provider ingestion과 lyric sync는 P1로 밀렸지만, 곡 metadata 운영 비용은 남아 있다.
- 중요한 이유: YouTube/Spotify/lyrics ingestion, 라이선스 확인, AI lyric alignment는 P0 필수는 아니지만, 수동 등록 운영 비용과 source/provenance 관리 책임은 여전히 필요하다.
- 심각도: High. Provider 약관과 라이선스가 이미 정리된 것이 아니라면 P1로 유지해야 한다.

- 이슈: Vocal-mode label은 유용하지만 교육적으로도 통계적으로도 취약하다.
- 중요한 이유: 진성/비성/두성 라벨은 교사마다 판단이 달라질 수 있다. label confidence, 교사 간 일치도, taxonomy version을 함께 저장하지 않으면 future model-improvement dataset 품질이 무너질 수 있다.
- 심각도: High.

- 이슈: Retention 정책은 문서화되었지만 deletion ownership이 아직 지정되지 않았다.
- 중요한 이유: "raw audio를 1년 데이터셋에서 제외한다"는 문장은 실제 deletion job, log, review access, owner 책임으로 연결되어야 한다.
- 심각도: High.

## 권장 수정

- 수정: 첫 P0 alpha J-POP 곡명/아티스트와 후렴 section start/end를 구현 전에 확정한다.
- 필요한 owner 또는 결정: Product + 음악 교육 전문가 + engine owner.

- 수정: P0 `SongPackage` 필수 필드 계약을 고정한다. 권장 필드는 title, artist, language, BPM, key, reference audio, source/provenance, usage status, section start/end, optional lyrics다.
- 필요한 owner 또는 결정: Product + backend/API owner.

- 수정: Conversion Job Orchestrator를 source of truth로 두고, P0에서 신규 서비스로 만들지 bounded module로 시작할지 결정한다.
- 필요한 owner 또는 결정: BFF/API Gateway/engine owners.

- 수정: Preview 재생 직후 고정 alpha survey를 사용한다. 1번 문항은 "이 preview가 내 목소리처럼 들린다"이며, 1-5점 중 4점 이상을 성공 응답으로 본다.
- 필요한 owner 또는 결정: Product.

- 수정: "다시 연습하고 싶음", "음정 피드백이 다음 연습에 도움됨"은 2차 평가 문항으로 둔다.
- 필요한 owner 또는 결정: Product.

- 수정: Provider ingestion과 AI lyric sync는 P0 user path dependency가 아니라 P1 technical spike로 둔다.
- 필요한 owner 또는 결정: Product + legal/policy + engine owner.

- 수정: Expert label 수집 전에 vocal-mode label guide를 만든다.
- 필요한 owner 또는 결정: 음악 교육 전문가 + Product.

- 수정: Retention/deletion owner를 지정하고 deletion evidence를 확인할 방법을 정의한다.
- 필요한 owner 또는 결정: Platform/storage owner + policy owner.

## Build 전 질문

- 질문: 첫 alpha J-POP 곡명/아티스트는 무엇인가?
- 왜 막히는가: reference audio, BPM/key, pitch range, 후렴 section, 권리/provenance 검토가 곡 선택에 묶인다.

- 질문: Conversion Job Orchestrator를 신규 서비스로 둘 것인가, P0에서는 bounded module로 시작할 것인가?
- 왜 막히는가: 개발 속도, 유지보수 경계, retry/replay 책임, app-facing status projection 구조가 달라진다.

- 질문: Own voice, generated preview, expert review, candidate data opt-in 동의를 각각 어떤 UI copy와 저장 schema로 받을 것인가?
- 왜 막히는가: Safety Rights와 retention behavior는 동의 범위가 구체화되어야 안전하게 구현할 수 있다.

- 질문: Break-glass raw audio 접근은 누가 승인하고 어떤 incident 조건에서 허용할 것인가?
- 왜 막히는가: Engine debugging과 privacy risk 사이의 운영 경계가 정해져야 한다.

- 질문: Teacher/expert 2명이 vocal-mode label에 동의하지 않으면 어떤 합의 기준을 적용할 것인가?
- 왜 막히는가: Candidate data 품질과 future model-improvement 가능성이 inter-rater agreement에 의존한다.

## 진행/중단 판단

- 권고: PRD의 readiness criteria를 만족한 뒤 P0 내부 alpha만 진행한다.
- 진행 조건: 구체 J-POP 곡명과 후렴 구간 확정, `SongPackage` 계약 확정, Conversion Job Orchestrator 구현 위치 결정, separated consent/audit behavior 정의, preview survey 확정, retention/deletion owner 지정.
- No-go 조건: provider automation이 P0 필수가 되는 경우, section preview 성공 전 full-song processing이 필수가 되는 경우, raw audio retention을 강제할 수 없는 경우, self-voice preview를 일관된 tester flow로 평가할 수 없는 경우.
