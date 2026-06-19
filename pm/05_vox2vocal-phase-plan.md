# 페이즈 계획 (Phase Plan)

문서 버전: v0.1
작성일: 2026-06-19
기반 문서:

- `pm/03_vox2vocal-product-vision.md` v0.1
- `pm/04_vox2vocal-target-system-definition.md` v0.1

## 페이즈 전략 요약 (Phase Strategy Summary)

Vox2Vocal의 페이즈 전략은 완성형 개인 시스템을 한 번에 만들지 않고, **단일 개인 사용자가 한 곡의 한 구간에서 자기 목소리 기반 preview를 듣고 평가하는 end-to-end 루프**를 먼저 검증하는 것이다.

현재 제품은 상업 서비스, 강사/교육생 플랫폼, AI 커버 공유 서비스가 아니다. 따라서 초기 페이즈의 판단 기준은 사용자 수, 곡 수, 처리량이 아니라 다음 질문에 답하는 것이다.

- 앱과 관리자 페이지를 분리한 상태에서도 개인 사용자가 흐름을 끝까지 완료할 수 있는가?
- 관리자 페이지에서 곡을 업로드하고 구간을 정의한 뒤, 앱에서 해당 구간을 선택해 본인 음성을 입력할 수 있는가?
- 시스템이 실제 preview를 생성하고, 사용자가 앱 안에서만 재생할 수 있는가?
- preview가 내 목소리처럼 들리는지 4개 세부 문항으로 평가하고, 실패/애매함을 개선 데이터로 남길 수 있는가?
- 원곡과 사용자 음성의 pitch/note 분석, 발성 참고 정보, 보관/삭제/접근 차단이 개인 시스템 수준에서 안전하게 연결되는가?

MVP는 가장 작은 engineering task가 아니라, 제품 비전의 핵심 가치인 **내 목소리 기반 preview 청취와 자기 점검 루프**가 실제로 성립하는 최소 가치 검증 단위다.

## MVP 정의 (MVP Definition)

MVP는 **한 명의 개인 사용자가 한 곡의 한 구간을 기준으로 본인 음성을 입력하고, 앱 내부에서만 Voice to Vocal preview를 들은 뒤, self-voice 평가와 기본 분석 결과를 남기는 개인 학습/실험 루프**다.

MVP의 기본 범위는 다음과 같다.

- 단일 개인 계정 생성과 로그인
- 일반 사용자 계정과 관리자 계정의 권한 분리
- 관리자 페이지에서 곡 1개 이상 업로드
- 곡별 `rights_not_cleared_personal_use_only` 기본 상태와 출처/권리 메모 기록
- 20~30초 안정 검증 구간 1개 이상 정의
- 앱에서 곡/구간 선택
- 사용자 본인 음성 녹음 또는 업로드
- 처리 job 생성, 상태 확인, 실패 사유 기록
- 실제 Voice to Vocal preview 생성
- preview의 앱 내부 재생과 외부 공유/다운로드/public URL 차단
- self-voice 4개 세부 문항 평가
- 1~5점 판정과 failure tag, `review_pending` 기록
- 원곡과 사용자 음성의 pitch/note 분석 결과와 confidence 표시
- 발성 정보는 원곡 기준 정의를 우선 제공하고, 사용자 발성은 confidence 기준을 만족할 때만 참고 표시
- 기본 알림 정책은 push-first 구조로 설계하되, MVP에서는 앱 내부 상태 화면을 정본 fallback으로 둔다
- 삭제 요청, token 폐기, 개인 산출물 삭제, 35일 backup retention 원칙을 문서/데이터 모델에 반영

MVP에서 의도적으로 제외하는 것은 다음과 같다.

- 여러 외부 사용자 지원
- 강사/교육생 분리
- 학생 피드백/리뷰 워크플로우
- 곡 전체 처리
- 외부 공유, 다운로드, 공개 URL, 수익화, 배포
- 타인 음성, 유명인 음성, 제3자 보이스 모델
- 발성 확정 진단 또는 의료/전문 평가
- 정교한 A/B 실험 UI
- 다수 곡/다수 구간의 운영 최적화

MVP 성공은 “모든 분석이 완벽하다”가 아니라, **실제 preview를 듣고 내 목소리 유사도와 실패 원인을 반복 개선 가능한 데이터로 남길 수 있다**는 것으로 판단한다.

## 우선순위 분류 (P0 / P1 / P2)

### P0: 첫 유효 preview 루프 (First Valid Preview Loop)

P0는 MVP를 성립시키는 release-blocking, validation-blocking, risk-control critical 항목이다.

| 영역 | P0 항목 | 이유 |
|---|---|---|
| 계정/권한 | 단일 개인 계정, 일반/관리자 권한 분리, 관리자 페이지 접근 차단 | 앱과 관리자 경험을 분리하고 데이터 접근을 통제하기 위해 필수 |
| 관리자 곡 관리 | 곡 업로드, 기본 metadata, 출처/권리 메모, 기본 권리 상태 | 사용자가 원하는 곡을 시스템 안에 넣는 첫 관문 |
| 구간 정의 | 20~30초 안정 검증 구간 정의 | 곡 전체 처리 리스크를 낮추고 실험 단위를 고정 |
| 앱 입력 흐름 | 곡/구간 선택, 본인 음성 녹음 또는 업로드 | voice-to-vocal 루프의 사용자 입력 |
| 처리 상태 | job 생성, 대기/처리/성공/실패 상태, 실패 사유 | 실패를 개선 데이터로 남기기 위해 필수 |
| Preview | 실제 preview artifact 생성과 앱 내부 재생 | 제품 핵심 가치 |
| 접근 차단 | 외부 공유, 다운로드, public URL 차단 | 개인 비상업 내부 사용 원칙의 핵심 리스크 제어 |
| Self-voice 평가 | 4개 세부 문항, 1~5점, failure tag, `review_pending` | 시스템 개선을 위한 최소 품질 데이터 |
| 기본 pitch/note | 원곡/사용자 pitch/note confidence 표시 | 자기 점검의 최소 분석 근거 |
| 발성 참고 | 원곡 기준 발성 정의, 사용자 발성은 confidence 기준 충족 시만 표시 | 과도한 판정을 막으면서 학습 가치를 제공 |
| 보관/삭제 | 삭제 요청 상태, token 폐기, 개인 산출물 삭제, backup retention 원칙 | 개인 데이터 신뢰와 안전 기준 |
| 감사 로그 | 로그인, 관리자 변경, playback, 삭제, 차단 이벤트 | 개인 시스템이라도 재현성과 보안 확인이 필요 |

### P1: 반복 실험과 분석 품질 개선 (Repeatable Experimentation and Analysis Quality)

P1은 P0 루프가 실제로 작동한 뒤 품질 개선과 반복 실험을 가능하게 만드는 범위다.

| 영역 | P1 항목 | 이유 |
|---|---|---|
| A/B 비교 | engine setting, take, prompt/config, section 단일 변수 비교 기록 | `review_pending` 결과를 개선 학습으로 연결 |
| 평가 히스토리 | preview별 점수 변화, failure tag 추세, preferred result | 어떤 조건이 self-voice 품질을 높이는지 추적 |
| 다중 구간 | 같은 곡 안에서 여러 구간 관리 | 한 구간 성공이 다른 구간으로 이어지는지 확인 |
| 다중 곡 | 소수 곡 업로드와 구간별 실험 | 특정 곡 의존성을 줄이되 운영 복잡도는 제한 |
| 분석 개선 | pitch/note confidence 95% 기준의 실패 원인 분류 | 분석 품질이 낮을 때 사용자에게 무엇을 숨길지 결정 |
| 발성 용어 사전 | AI-assisted internal review 상태와 용어 카드 관리 | 발성 문구의 신뢰도와 안전성 확보 |
| 알림 | push token 관리, 앱 내부 알림함, 중요 알림 fallback | 처리 완료/실패/삭제 상태를 놓치지 않게 함 |
| 데이터 인벤토리 | 필드별 보관 기간, 삭제 trigger, opt-out 가능 여부 | 보관/삭제 정책을 실제 데이터에 연결 |
| 운영 화면 | job 실패 원인, 재시도, 삭제, audit 확인 | 개인 운영자가 시스템을 개선할 수 있게 함 |

### P2: 개인 시스템 완성도와 확장 준비 (Personal System Maturity and Expansion Readiness)

P2는 개인 시스템으로서 안정성과 완성도를 높이고, 나중에 제품화 여부를 판단할 수 있게 하는 범위다. 단, P2도 자동으로 다중 사용자/상업화로 넘어간다는 뜻은 아니다.

| 영역 | P2 항목 | 이유 |
|---|---|---|
| 고급 A/B 비교 | 실험 템플릿, 결과 비교 리포트, config lineage | 엔진 개선 의사결정의 재현성 강화 |
| 발성 가이드 고도화 | 원곡/사용자 발성 비교, confidence 설명, 숨김 기준 개선 | 자기 점검 가치 확대 |
| 구간 확장 | 30~45초 발성 참고 구간, 긴 구간 처리 검증 | 안정 검증 이후 학습 범위 확장 |
| 성능/안정성 | 첫 preview까지 시간, 전체 결과까지 시간, 재시도 정책 | 개인 사용 경험의 답답함 감소 |
| 백업/삭제 자동화 | tombstone 재적용 자동화, 삭제 검증 리포트 | 개인정보/산출물 통제 신뢰도 강화 |
| 권리/출처 등록부 고도화 | checksum, section link, 삭제 상태, 사용 금지 범위 추적 | 개인 비상업 사용의 운영 안정성 강화 |
| 제품화 판단 자료 | 개인 사용 반복성, 성공률, 실패 유형, 기술 비용 정리 | 다중 사용자나 수익화 검토 전 근거 확보 |

## 페이즈별 범위 (Phase Scope)

### Phase 0: 기준 문서와 시스템 골격 확정 (Planning Baseline)

목표는 제품의 중심을 흔들리지 않게 고정하고, PRD로 들어가기 전 범위를 잠그는 것이다.

포함 범위:

- business-context, product-strategy, product-vision, target-system-definition 확정
- MVP/P0/P1/P2 경계 정의
- 앱/관리자 분리 원칙 확정
- 권리/출처, 개인정보/보관, 발성 용어, A/B 비교의 기본 정책 확정
- 첫 PRD 작성 후보와 작성 순서 결정

제외 범위:

- PRD 상세 작성
- 화면/기능 정의
- 기술 설계
- 티켓 분해

### Phase 1: P0 MVP 구현 범위 정의 (P0 MVP Definition)

목표는 첫 유효 preview 루프를 만들기 위한 PRD 묶음을 작성할 수 있을 만큼 제품 범위를 구체화하는 것이다.

포함 범위:

- 개인 계정 및 권한
- 관리자 곡 업로드와 구간 정의
- 앱 곡/구간 선택과 본인 음성 입력
- 처리 job과 preview 재생
- self-voice 평가와 기본 failure tag
- pitch/note confidence 표시
- 접근 차단, 삭제 요청, 기본 audit

제외 범위:

- 정교한 A/B 비교 UI
- 다수 곡/다수 구간 운영 최적화
- 발성 용어 사전 전체 구축
- push 알림 전체 자동화
- 제품화/외부 사용자 준비

### Phase 2: 반복 실험과 분석 신뢰도 강화 (Experimentation and Analysis Confidence)

목표는 P0에서 나온 결과를 바탕으로 시스템 개선 루프를 만든다.

포함 범위:

- A/B 비교 실험 기록
- 평가 히스토리와 failure tag 추세
- pitch/note 분석 실패 원인 분류
- 발성 용어 카드와 AI-assisted internal review 상태
- 앱 내부 알림함과 push token 관리
- 데이터 인벤토리와 삭제 trigger 연결
- 운영 화면에서 job 실패/재시도/삭제 확인

제외 범위:

- 완전한 전문가 발성 검토
- 대량 곡 처리
- 외부 공유/export
- 상업화 정책

### Phase 3: 개인 시스템 완성도와 확장 판단 (Personal System Maturity)

목표는 개인 시스템으로 안정적으로 반복 사용 가능한 상태를 만들고, 이후 제품화 여부를 판단할 근거를 확보한다.

포함 범위:

- 구간 확장과 긴 preview 처리 검증
- 고급 A/B 비교와 config lineage
- 발성 가이드 고도화
- 성능/안정성 지표 관리
- 백업/삭제 자동화 검증
- 권리/출처 등록부 고도화
- 제품화 판단 리포트

제외 범위:

- 외부 사용자 onboarding
- 강사/교육생 모델
- 결제/구독
- 음원 배포/공유
- 공개 AI 커버 서비스

## 의존성 (Dependencies)

| 의존성 | 필요한 페이즈 | 설명 |
|---|---|---|
| 제품 비전과 목표 시스템 정의 | Phase 0 | MVP가 제품 중심에서 벗어나지 않기 위한 기준 |
| 계정/권한 모델 | Phase 1 | 앱과 관리자 페이지 접근 분리를 위해 필요 |
| 파일/object storage | Phase 1 | 곡, 사용자 음성, preview, 분석 산출물 보관에 필요 |
| 관리자 곡 업로드 | Phase 1 | 앱에서 선택할 곡/구간을 만들기 위한 선행 조건 |
| 구간 정의 | Phase 1 | 처리 범위를 20~30초 안정 검증 단위로 제한하기 위한 선행 조건 |
| 오디오 처리 엔진 | Phase 1 | 실제 preview와 pitch/note 분석을 만들기 위한 핵심 의존성 |
| 내부 playback 권한 | Phase 1 | 앱 내부 재생과 외부 공유 차단을 동시에 보장해야 함 |
| Self-voice 평가 모델 | Phase 1 | 시스템 개선 데이터를 수집하기 위한 필수 구조 |
| 감사 로그 | Phase 1 | 접근, 변경, 삭제, 차단 이벤트를 추적해야 함 |
| A/B 실험 데이터 모델 | Phase 2 | `review_pending` 결과를 개선 실험으로 연결하기 위한 구조 |
| 발성 용어 사전 | Phase 2 | 발성 참고 정보를 안전하게 노출하기 위한 기준 |
| 데이터 인벤토리 | Phase 2 | 보관/삭제 정책을 필드 단위로 실행하기 위한 기준 |
| push 알림 인프라 | Phase 2 | 처리 상태를 앱 밖에서도 알려주기 위한 수단 |
| 백업 tombstone 재적용 | Phase 3 | 삭제 요청 이후 복원 시 개인정보/산출물 재노출을 막기 위한 안전장치 |

## 페이즈 종료 기준 (Phase Exit Criteria)

### Phase 0 종료 기준

- 제품 대상이 단일 개인 사용자로 명확히 고정되어 있다.
- 앱과 관리자 페이지가 분리된 경험으로 정의되어 있다.
- MVP, P0, P1, P2 경계가 문서화되어 있다.
- 권리/출처, 보관/삭제, 발성 용어, A/B 비교의 기본 정책 질문이 닫혀 있다.
- 첫 PRD 후보와 작성 순서가 결정되어 있다.

### Phase 1 종료 기준

- 관리자 페이지에서 곡 1개와 20~30초 구간 1개를 등록할 수 있다.
- 앱에서 해당 곡/구간을 선택하고 본인 음성을 입력할 수 있다.
- 시스템이 mock이 아닌 실제 preview artifact를 생성한다.
- preview는 앱 내부에서만 재생되며 외부 공유/다운로드/public URL이 차단된다.
- 사용자가 preview를 듣고 4개 self-voice 문항과 1~5점 평가를 남길 수 있다.
- 실패 또는 3점 `review_pending` 결과에 failure tag가 기록된다.
- 원곡/사용자 pitch/note confidence가 표시되거나, 기준 미달 시 숨김/실패 사유가 표시된다.
- 삭제 요청과 token 폐기, 기본 audit log가 동작한다.

### Phase 2 종료 기준

- `review_pending` 결과를 A/B 비교 실험으로 연결할 수 있다.
- engine setting, take, prompt/config, section 중 하나의 변수만 바꾸는 비교가 기록된다.
- 평가 히스토리에서 어떤 조건이 self-voice 점수를 개선했는지 추적할 수 있다.
- 발성 용어 카드가 `draft`, `ai_reviewed`, `internal_approved`, `needs_human_expert_review`, `deprecated` 상태를 가진다.
- 데이터 인벤토리에서 주요 개인정보, 음성, preview, 분석, 로그 데이터의 보관/삭제 trigger가 연결된다.
- 앱 내부 알림함 또는 push token 관리가 처리 상태와 연결된다.

### Phase 3 종료 기준

- 여러 구간과 소수 곡에서 반복 실험이 가능하다.
- 고급 A/B 비교 결과와 config lineage가 재현 가능하게 남는다.
- 발성 참고 정보가 confidence, 숨김 기준, 사용자 문구 안전 기준을 만족한다.
- 백업 복원 시 삭제 tombstone 재적용 절차가 검증된다.
- 개인 시스템으로 반복 사용 가능한 안정성, 성능, 실패 대응 기준이 정리된다.
- 이후 다중 사용자/상업화/강사 모델로 확장할지 판단할 근거가 충분히 쌓여 있다.

## 리스크와 트레이드오프 (Risks and Tradeoffs)

| 리스크 | 영향 | 대응 |
|---|---|---|
| P0 범위가 커질 위험 | 첫 preview 검증이 늦어진다 | 한 곡, 한 구간, 한 명의 사용자 루프로 제한한다 |
| 실제 preview 생성 난이도 | MVP 핵심 가치가 검증되지 않을 수 있다 | mock 성공은 MVP 성공으로 보지 않고, 실패도 개선 데이터로 기록한다 |
| pitch/note 95% 기준 부담 | P0에서 분석 기능이 병목이 될 수 있다 | 기준 미달 시 숨김/실패 사유를 표시하고, 분석 품질 개선은 P1로 확장한다 |
| 발성 정보 과신 | 사용자가 확정 진단처럼 받아들일 수 있다 | confidence 기준, 참고형 문구, `needs_human_expert_review` 상태를 둔다 |
| A/B 비교 복잡도 | 실험 원인을 해석하기 어려워진다 | 한 번에 하나의 변수만 바꾸는 원칙을 유지한다 |
| 권리/출처 리스크 | 개인 비상업이어도 면책이 되지 않는다 | 기본 상태를 `rights_not_cleared_personal_use_only`로 두고 외부 사용을 차단한다 |
| 삭제/백업 구현 복잡도 | 개인정보 삭제 신뢰가 낮아질 수 있다 | P0에서는 정책/상태/audit을 만들고, tombstone 자동화는 P3에서 고도화한다 |
| 앱/관리자 분리 비용 | 단일 사용자 제품치고 구현 면이 넓어진다 | 완성형의 시스템 경계를 지키기 위해 P0부터 최소 분리를 유지한다 |

가장 큰 tradeoff는 **작게 만들기 위해 분석/운영 기능을 뒤로 미루는 것**과 **개인 데이터/권리/접근 통제는 P0부터 최소 기준을 지키는 것** 사이에 있다. 이 제품은 음원, 음성, preview를 다루므로 접근 차단, 삭제, 감사 로그는 작게라도 P0에 포함하는 편이 맞다.

## PRD 작성 후보 (PRD Candidates)

첫 PRD는 P0 MVP 루프를 너무 잘게 쪼개기보다, end-to-end 검증이 가능한 순서로 작성한다.

| 순서 | PRD 후보 | 포함 페이즈 | 이유 |
|---:|---|---|---|
| 1 | 개인 계정 및 권한 PRD (Account and Permission PRD) | P0 | 앱/관리자 접근 분리와 계정 기반 데이터 소유권이 모든 흐름의 전제 |
| 2 | 관리자 곡 업로드 및 구간 관리 PRD (Admin Song Upload and Section Management PRD) | P0 | 앱에서 선택할 곡/구간과 권리/출처 상태를 만든다 |
| 3 | 앱 곡/구간 선택 및 본인 음성 입력 PRD (App Song Selection and Own Voice Input PRD) | P0 | 사용자 입력 루프의 시작점 |
| 4 | Voice to Vocal 처리 및 Preview PRD (Processing and Preview PRD) | P0 | 제품 핵심 가치인 실제 preview 생성과 앱 내부 재생을 정의 |
| 5 | Self-voice 평가 PRD (Self-voice Rating PRD) | P0 | 시스템 개선을 위한 4개 문항, 점수, failure tag, `review_pending`을 정의 |
| 6 | Pitch/Note/발성 참고 PRD (Pitch, Note, and Vocal Guidance PRD) | P0/P1 | 자기 점검 분석과 confidence 기반 노출 기준을 정의 |
| 7 | 접근 차단, 삭제, 감사 PRD (Access Control, Deletion, and Audit PRD) | P0/P1 | 외부 공유 차단, 삭제 요청, audit log의 최소 기준을 정의 |
| 8 | A/B 비교 및 실험 히스토리 PRD (A/B Comparison and Experiment History PRD) | P1 | `review_pending`을 시스템 개선 루프로 연결 |
| 9 | 발성 용어 체계 PRD (Vocal Terminology Taxonomy PRD) | P1 | AI-assisted internal review와 용어 카드 구조를 정의 |
| 10 | 개인정보 및 데이터 보관 정책 문서 (Privacy and Data Retention Policy) | P0/P1 | 보관/삭제/수신 거부/backup retention의 정본 |
| 11 | 데이터 인벤토리 문서 (Data Inventory / Data Map) | P1 | 필드별 보관 기간과 삭제 trigger 연결 |
| 12 | 운영 runbook (Admin Operations Runbook) | P1/P2 | 삭제, 비활성화, token 폐기, backup 복원 대응 절차 |

가장 먼저 작성할 PRD 후보는 **개인 계정 및 권한 PRD (Account and Permission PRD)** 다. 그 다음 곡 업로드/구간 관리, 앱 입력, 처리/preview, self-voice 평가 순서로 이어가는 것이 좋다.

## 다음 추천 스킬 (Recommended Next Skill)

다음은 `pm-context`가 적합하다.

이유는 MVP/P0와 페이즈 종료 기준이 정리되었기 때문이다. 다음 단계에서는 첫 PRD 후보인 개인 계정 및 권한 PRD를 바로 쓰기보다, `pm-context`로 해당 PRD의 제품/기능, 대상 사용자, 문제, 목표, 제약, 성공 기준, evidence를 먼저 정리해야 한다.
