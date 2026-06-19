# 목표 시스템 정의 (Target System Definition)

문서 버전: v0.1
작성일: 2026-06-19
기반 문서:

- `pm/03_vox2vocal-product-vision.md` v0.1

## 시스템 요약 (System Summary)

Vox2Vocal의 목표 시스템은 단일 개인 사용자가 앱 계정으로 로그인하고, 분리된 관리자 페이지에서 원하는 곡을 업로드한 뒤, 특정 구간을 선택해 본인 음성을 녹음/업로드하고, 앱 안에서만 Voice to Vocal preview와 pitch/note/발성 참고 정보를 확인하는 비상업적 개인 학습/실험 시스템이다.

시스템은 다중 사용자 교육 플랫폼이 아니라, 한 명의 사용자가 다음 역할을 모두 수행하는 개인 시스템으로 정의한다.

- 앱에서 본인 음성을 입력하고 preview를 듣는 사용자
- 관리자 페이지에서 곡과 구간을 관리하는 곡 관리자
- 처리 상태, 실패, 삭제, 알림, 접근 통제를 확인하는 실험 운영자
- 결과를 평가하고 반복 실험하는 학습자

완성형 시스템의 핵심 경계는 다음과 같다.

- 앱과 관리자 페이지는 분리된 경험으로 유지한다.
- 음원, 원본 음성, 변환 산출물, 분석 결과는 계정 내부에서만 접근한다.
- 산출물은 외부 공유, 다운로드, 공개 URL, 수익화, 배포, 모델 학습에 사용하지 않는다.
- 업로드 곡의 기본 권리 상태는 `rights_not_cleared_personal_use_only`다.
- 원곡 pitch/note와 사용자 목소리 pitch/note는 각각 95% 이상 분석된 경우를 자기 점검 기준으로 삼는다.
- 발성 정보는 확정 진단이 아니라 confidence 기반 참고 정보로 제공한다.

## 사용자 역할 (User Roles)

현재 목표 시스템의 실제 사용자는 한 명이지만, 시스템 경계를 명확히 하기 위해 역할을 분리해 정의한다.

| 역할 | 설명 | 주요 책임 |
|---|---|---|
| 개인 계정 소유자 (Personal Account Owner) | 앱 계정을 생성하고 모든 개인 데이터의 소유자가 되는 단일 사용자 | 로그인, 개인정보/알림 설정, 데이터 삭제 요청, 시스템 접근 |
| 앱 사용자 (App User) | 앱에서 곡/구간을 선택하고 본인 음성을 입력해 preview를 듣는 사용자 | 본인 음성 녹음/업로드, preview 재생, self-voice 평가, 결과 확인 |
| 곡 관리자 (Song Admin) | 관리자 페이지에서 곡과 구간을 관리하는 사용자 | 곡 업로드, 곡별 출처/권리 메모 기록, 구간 정의, 처리 가능 상태 관리 |
| 실험 운영자 (Experiment Operator) | 처리 상태와 실패, 삭제, 알림, 접근 통제를 확인하는 사용자 | job 상태 확인, 실패 원인 확인, 삭제 처리, 알림/접근 로그 확인 |
| 학습자 (Learner Self) | 결과를 바탕으로 자기 목소리와 노래 습관을 이해하는 사용자 | pitch/note/발성 참고 정보 해석, 재녹음/재처리 판단, 실험 반복 |

역할은 같은 개인에게 귀속되지만, 앱/관리자/운영 범위가 섞이지 않도록 권한과 화면 경험은 분리한다.

## 핵심 모듈 (Core Modules)

| 모듈 | 목적 | 포함 범위 |
|---|---|---|
| 계정 및 인증 (Account and Authentication) | 단일 개인 사용자의 계정 기반 접근을 보장한다 | 회원가입, 로그인, 세션, 계정 상태, 인증 이벤트 |
| 개인정보 및 알림 설정 (Personal Data and Notification Preferences) | 개인정보 사용 목적과 push-first 알림 설정을 관리한다 | 알림 동의, push token, fallback 상태, 수신 설정, 계정 연락처 |
| 개인 앱 경험 (Personal App Experience) | 곡/구간 선택, 본인 음성 입력, preview 재생, 결과 평가를 제공한다 | 곡/구간 선택, 녹음/업로드, 처리 상태, preview, 분석 결과, 평가 |
| 관리자 곡 관리 (Admin Song Management) | 사용자가 원하는 곡을 업로드하고 실험 가능한 구간을 관리한다 | 곡 업로드, metadata, 출처/권리 메모, 구간 map, 사용 상태 |
| 권리 및 사용 제한 관리 (Rights and Use Restriction Management) | 업로드 곡과 산출물의 개인 비상업 사용 제약을 명시한다 | `rights_not_cleared_personal_use_only`, 외부 공유/수익화/배포/model training 금지 |
| 발성 용어 리서치 및 정의 (Vocal Terminology Research and Definition) | 제품 내부 발성 용어 체계를 조사/정의/보강한다 | 음성 산출 구조, 성구/음향/청각 용어, ASHA/The Voice Foundation/NATS/Estill 기준, AI-assisted internal review |
| 오디오 입력 관리 (Audio Input Management) | 사용자 본인 음성 입력과 업로드 파일을 관리한다 | 본인 음성 녹음, fallback 업로드, 입력 검증, 구간 매칭 |
| 처리 및 작업 상태 (Processing and Job State) | 곡/구간/음성 입력을 처리 상태로 추적한다 | job 생성, 대기/처리/성공/실패 상태, 재시도 가능성, 실패 사유 |
| 분석 및 발성 참고 (Analysis and Vocal Guidance) | pitch/note/발성 참고 정보를 제공한다 | 원곡 pitch/note, 사용자 pitch/note, 발성 taxonomy, confidence, 비교 결과 |
| Preview 산출물 관리 (Preview Artifact Management) | 변환 preview를 내부 접근으로만 제공한다 | preview artifact, 앱 내 재생, playback 상태, 외부 공유 차단 |
| Self-voice 평가와 A/B 비교 (Self-voice Evaluation and A/B Comparison) | 사용자가 preview를 자기 목소리 기준으로 평가하고 시스템 개선 실험을 기록하게 한다 | 1~5점, 4개 세부 평가 문항, 실패 tag, `review_pending`, A/B experiment |
| 알림 및 상태 센터 (Notification and Status Center) | 처리 상태와 중요 이벤트를 push와 앱 내부 상태로 알려준다 | push, in-app fallback, 작업 상태 화면, 최소 payload |
| 삭제 및 보관 관리 (Deletion and Retention Management) | 원본 음성, 산출물, 계정/알림 데이터 삭제와 보관 기준을 관리한다 | 삭제 요청, 수동 삭제, 7일 활성 데이터 삭제, 35일 backup retention, backup 복원 시 tombstone 재적용 |
| 감사 및 접근 로그 (Audit and Access Logging) | 내부 접근과 차단 이벤트를 추적한다 | 로그인, playback, download/share 차단, 삭제, admin 변경 이력 |

## 주요 워크플로우 (Key Workflows)

| 워크플로우 | 역할 | 제품 결과 |
|---|---|---|
| 계정 생성 및 알림 설정 (Account and Notification Setup) | 개인 계정 소유자 | 앱 계정이 생성되고 개인정보/알림 사용 목적이 정리된다 |
| 관리자 곡 업로드 (Admin Song Upload) | 곡 관리자 | 곡 파일, metadata, 출처/권리 메모, 기본 권리 상태가 기록된다 |
| 구간 정의 (Section Definition) | 곡 관리자 | 곡 안에서 실험할 구간과 권장 길이/목표 범위가 정리된다 |
| 곡/구간 선택 (Song and Section Selection) | 앱 사용자 | 앱에서 처리할 곡과 특정 구간이 선택된다 |
| 본인 음성 입력 (Own Voice Input) | 앱 사용자 | 사용자의 본인 음성이 녹음 또는 업로드된다 |
| 처리 요청 및 상태 확인 (Processing Request and Status Tracking) | 앱 사용자, 실험 운영자 | 처리 job이 생성되고 성공/실패/재시도 상태가 추적된다 |
| Preview 재생 (Preview Playback) | 앱 사용자 | 변환 preview가 앱 내부에서만 재생된다 |
| 분석 결과 확인 (Analysis Review) | 학습자 | pitch/note/발성 참고 정보가 자기 점검 가능한 형태로 표시된다 |
| Self-voice 평가 (Self-voice Rating) | 학습자 | 1~5점, 4개 세부 평가, 실패 사유, `review_pending`이 기록된다 |
| A/B 비교 실험 (A/B Comparison Experiment) | 학습자, 실험 운영자 | engine setting, take, prompt/config, section 중 하나의 변수만 바꿔 결과를 비교한다 |
| 반복 실험 (Repeat Experimentation) | 학습자, 실험 운영자 | 다른 녹음/구간/곡으로 재시도하거나 비교한다 |
| Push 및 상태 알림 (Push and Status Notification) | 개인 계정 소유자 | 처리 완료/실패/삭제/보안 상태가 push 또는 앱 내부 fallback으로 전달된다 |
| 삭제 및 접근 차단 (Deletion and Access Control) | 개인 계정 소유자, 실험 운영자 | 원본 음성/preview/분석 결과가 삭제되거나 외부 접근이 차단된다 |

## 데이터 객체 (Data Objects)

| 데이터 객체 | 설명 | 주요 속성 예시 |
|---|---|---|
| 계정 (Account) | 단일 개인 사용자의 계정 | account id, login method, status, created at |
| 개인정보 프로필 (Personal Profile) | 인증과 알림에 필요한 최소 개인정보 | email/contact, notification preference, consent status |
| 알림 대상 (Notification Target) | push 및 fallback 알림 대상 | push token, token status, in-app inbox state, critical email opt-in |
| 곡 패키지 (Song Package) | 관리자 페이지에서 업로드한 곡 단위 | title, artist, source note, upload owner, rights status, usage scope |
| 곡 자산 (Song Asset) | 업로드된 음원 또는 관련 파일 | file id, object key, checksum, duration, format, storage state |
| 권리/출처 메모 (Rights and Source Memo) | 개인 비상업 사용 제한과 출처 기록 | `rights_not_cleared_personal_use_only`, source, allowed use, prohibited use |
| 권리/출처 등록부 (Rights and Source Register) | 곡별 출처/권리 메모의 정본 | title, artist, source, checksum, upload owner, allowed/prohibited use, section links, deletion state |
| 구간 (Section) | 곡 안의 실험 단위 | section id, start/end, duration, label, target type |
| 원곡 분석 결과 (Reference Analysis Result) | 원곡 pitch/note/발성 기준 | pitch/note confidence, vocal label, section target, analysis version |
| 발성 용어 사전 (Vocal Terminology Taxonomy) | 내부 발성 용어 정의와 분류 | anatomy/function/acoustic/perceptual/pedagogy/health category, definition, display copy, forbidden copy, measurable signal, source, AI review result, review status |
| 사용자 음성 입력 (User Voice Take) | 본인 음성 녹음/업로드 | take id, section id, duration, format, validation status |
| 처리 작업 (Processing Job) | 변환/분석 처리 단위 | job id, status, mode, retry state, failure reason |
| Preview 산출물 (Preview Artifact) | 앱 내부 재생용 변환 결과 | artifact id, job id, playback scope, storage state |
| 사용자 분석 결과 (User Analysis Result) | 사용자 pitch/note/발성 참고 결과 | pitch/note confidence, vocal confidence, comparison result |
| Self-voice 평가 (Self-voice Rating) | preview 자기 평가 | rating 1~5, weighted criteria, failure tags, review state |
| A/B 실험 (A/B Experiment) | self-voice 개선 조건을 비교하는 실험 단위 | experiment id, baseline artifact, candidate artifact, variable type, variable diff, config hash, prompt version, preferred result, failure tags |
| Playback 세션 (Playback Session) | preview 재생 확인 | session id, coverage, foreground state, blocked state |
| 삭제 요청/삭제 기록 (Deletion Request and Deletion Record) | 데이터 삭제 흐름 | target object, requested at, completed at, backup exception |
| 감사 로그 (Audit Log) | 접근/변경/차단 이력 | actor role, action, object type, timestamp, result |

## 권한 모델 (Permission Model)

권한 모델은 단일 개인 사용자 전제를 따른다. 다만 같은 개인이 수행하더라도 앱 사용자 권한과 관리자 권한, 시스템 처리 권한은 분리한다. 일반 사용자 계정은 관리자 페이지에 접근할 수 없고, 관리자 계정은 앱 사용자 권한을 포함해 앱에 접근할 수 있다.

| 권한 범위 | 허용되는 작업 | 차단되는 작업 |
|---|---|---|
| 일반 사용자 계정 (Regular User Account) | 앱 로그인, 곡/구간 선택, 본인 음성 입력, 처리 상태 확인, preview 재생, self-voice 평가, 삭제 요청 | 관리자 페이지 접근, 곡 metadata/권리 상태 수정, 관리자 upload 변경, 시스템 전체 job 조작 |
| 관리자 계정 (Admin Account) | 앱 사용자 권한 전체, 관리자 페이지 접근, 곡 업로드, metadata 관리, 출처/권리 메모 기록, 구간 정의, 처리 가능 상태 관리 | preview 외부 공유 허용, 수익화 상태 변경, 권리 확보로 오인될 상태 변경 |
| 운영 권한 (Operator Scope) | job 상태 확인, 실패 원인 확인, 재시도/차단/삭제 처리, audit 확인 | 데이터 목적 외 열람, 외부 export, 모델 학습 사용 |
| 시스템 처리 권한 (System Processing Scope) | 내부 처리에 필요한 원곡/사용자 음성/preview/분석 객체 접근 | 외부 공개 URL 생성, 임의 다운로드, 목적 외 전송 |
| 알림 권한 (Notification Scope) | 최소 payload push, 앱 내부 알림함 기록, 중요 알림 opt-in fallback | 민감한 곡명/권리 정보/음성 내용 push payload 포함 |

모든 권한의 기본 원칙은 계정 내부 접근, 최소 권한, 외부 공유 차단이다.

공유되는 계정/세션 정책은 인증 provider, account id, 로그인 보안 이벤트, session 만료/폐기, 계정 삭제 상태다. 분리되는 권한/로그는 관리자 페이지 접근 권한, 곡 upload/change log, 권리/출처 변경 log, 운영 job 조작 log, 앱 playback/self-voice 평가 log다.

## 연동 지점 (Integration Points)

다음 연동 지점은 목표 시스템 관점의 필요 영역이며, 구체 기술 선택은 후속 TRD에서 결정한다.

| 연동 지점 | 목적 | 경계 |
|---|---|---|
| 인증/계정 시스템 (Auth / Account) | 앱 계정 생성, 로그인, 세션 관리 | 단일 사용자 계정 기준 |
| 파일 저장소 (File / Object Storage) | 곡 자산, 원본 음성, preview, 분석 산출물 보관 | 외부 공개 접근 차단 |
| 오디오 처리 엔진 (Audio Processing Engines) | pitch/note 분석, 발성 참고, voice-to-vocal preview 생성 | 개인 비상업 실험 목적 |
| 알림 서비스 (Notification Service) | push-first 알림, 앱 내부 fallback | 최소 payload, 민감정보 제외 |
| 앱 내부 상태 센터 (In-app Status Center) | push 실패 시 정본 fallback | 처리 상태/삭제/보안 상태 확인 |
| 관리자 페이지 (Admin Console) | 곡 업로드, 구간 정의, 출처/권리 메모 관리 | 앱 사용자 경험과 분리 |
| 감사/로그 저장소 (Audit / Logging) | 접근/변경/차단/삭제 이력 추적 | 민감정보 직접 기록 최소화 |
| 개인정보/보관 정책 문서 (Privacy and Retention Policy) | 계정/연락처/삭제/보관 기준의 정본 | PRD에서 참조, 중복 작성 방지 |
| 권리/출처 등록부 (Rights and Source Register) | 업로드 곡의 출처/권리 메모 정본 | Admin Song Upload and Rights PRD에서 관리 |
| 발성 용어 리서치 자료 (Vocal Terminology Research Sources) | 발성 용어 체계 보강 | ASHA, The Voice Foundation, NATS/Vocapedia, Estill, 선택적 전문가 보강 |
| AI 내부 검토 도구 (AI-assisted Internal Review Tool) | 발성 용어 정의와 제품 문구를 내부 검토한다 | Codex/GPT 검토 결과는 전문가 확정이 아니라 내부 검토 승인으로만 사용 |

## 시스템 경계 (System Boundaries)

### 포함 범위 (In Scope)

- 단일 개인 사용자 계정
- 앱과 관리자 페이지의 분리된 경험
- 관리자 페이지 곡 업로드
- 곡별 출처/권리 메모와 `rights_not_cleared_personal_use_only` 기본 상태
- 권리/출처 등록부를 통한 곡별 source, checksum, allowed/prohibited use, section link 관리
- 제품 내부 발성 용어 체계와 리서치 기반 보강
- Codex/GPT 기반 `AI-assisted Internal Review Approval`을 통한 발성 용어 내부 검토
- 구간 정의와 `20~30초` 기본 안정 검증, `30~45초` 발성 참고 확장
- 본인 음성 녹음/업로드
- 특정 구간 기준 Voice to Vocal preview 생성
- 원곡과 사용자 목소리 pitch/note 95% 이상 분석 기준
- confidence 기반 발성 참고 정보
- 1~5점 self-voice 평가와 weighted criteria
- `review_pending`, 실패 사유, 반복 실험 기록
- engine setting, take, prompt/config, section 단위의 A/B 비교와 단일 변수 비교 원칙
- 앱 내부 preview 재생과 외부 공유/다운로드/공개 URL 차단
- push-first 알림과 앱 내부 fallback
- 삭제/보관/감사 로그
- 계정/연락처/알림/로그/평가 데이터의 기본 보관 수치와 35일 backup rolling retention

### 제외 범위 (Out of Scope)

- 상업 서비스
- 강사/교육생 플랫폼
- 교육기관 SaaS
- 외부 사용자 모집
- 다중 사용자 협업
- 음원 또는 preview 외부 공유
- 다운로드/export/public URL 제공
- 수익화, 판매, 배포
- 권리 미확인 곡의 외부 공개 또는 모델 학습 사용
- 타인 음성, 유명인 음성, 제3자 보이스 모델 처리
- 발성 확정 진단 또는 의료/전문 평가
- 곡 전체 대량 처리 중심의 음악 제작 플랫폼

### 경계상 주의 영역 (Boundary Watchlist)

- 개인 비상업 사용은 권리 면책을 의미하지 않는다.
- 관리자 페이지 업로드는 허용하지만 권리 확보 상태로 오인되면 안 된다.
- pitch/note 95% 기준은 자기 점검 기준이며, 발성 label의 확정성을 보장하지 않는다.
- push 알림은 민감정보 전달 채널이 아니라 상태 안내 채널이다.
- 앱과 관리자 페이지는 같은 개인이 쓰더라도 목적과 권한이 다르다.
- 일반 사용자 계정은 관리자 페이지에 접근할 수 없고, 관리자 계정은 앱에 접근할 수 있다.
- 개인정보/보관/삭제 기준은 PRD에 흩어 쓰지 않고 정책 정본에서 관리해야 한다.
- 권리/출처 메모는 곡 metadata의 부속 필드가 아니라 별도 등록부의 정본으로 관리해야 한다.
- A/B 비교는 시스템 개선을 위해 넓게 허용하되, 하나의 비교에서 여러 변수를 동시에 바꾸면 원인 해석이 어렵다.
- Codex/GPT 검토는 내부 검토 승인으로만 취급하고, 의료/음성장애/건강 판단이 포함된 발성 용어는 `needs_human_expert_review`로 남겨야 한다.
- 백업 데이터는 직접 부분 삭제보다 짧은 rolling retention과 복원 시 tombstone 재적용으로 관리해야 한다.

## PRD 후보 영역 (PRD Candidate Areas)

| 후보 영역 | 설명 | 주요 산출 질문 |
|---|---|---|
| 개인 계정 및 개인정보/알림 PRD (Account, Privacy, and Notification PRD) | 계정 생성, 로그인, 개인정보 사용 목적, push-first 알림, fallback을 정의한다 | 정의된 보관/삭제 기준을 회원가입, 설정, 삭제 요청 화면에 어떻게 반영할 것인가? |
| 관리자 곡 업로드 및 구간 관리 PRD (Admin Song Upload and Section Management PRD) | 곡 업로드, metadata, 출처/권리 메모, 구간 정의를 정의한다 | 어떤 곡 정보를 필수로 받을 것인가? |
| 권리/출처 등록부 PRD (Rights and Source Register PRD) | 곡별 출처, 권리 메모, 사용 가능/금지 범위, checksum, section link, 삭제 상태의 정본을 정의한다 | 기존 권리/출처 기록 수준을 새 개인 시스템에서 어떻게 유지할 것인가? |
| 앱 곡/구간 선택 및 본인 음성 입력 PRD (App Song Selection and Voice Input PRD) | 앱에서 곡/구간을 선택하고 본인 음성을 녹음/업로드하는 흐름을 정의한다 | 입력 검증과 구간 매칭 기준은 무엇인가? |
| Voice to Vocal 처리 및 Preview PRD (Processing and Preview PRD) | 처리 job, preview 생성, 상태 추적, 실패 처리를 정의한다 | 어떤 상태와 실패 사유를 노출할 것인가? |
| Pitch/Note/발성 분석 PRD (Pitch, Note, and Vocal Guidance PRD) | 95% pitch/note 기준, 발성 confidence, 원곡/사용자 비교를 정의한다 | 어떤 분석 결과를 자기 점검용으로 보여줄 것인가? |
| 발성 용어 체계 PRD (Vocal Terminology Taxonomy PRD) | 내부 발성 용어 체계, 분류 원칙, 기준 자료, AI-assisted internal review 상태를 정의한다 | 어떤 용어를 어떤 근거와 상태로 표시할 것인가? |
| Self-voice 평가 및 반복 실험 PRD (Self-voice Rating and Experiment History PRD) | 4개 세부 문항, 1~5점 평가, weighted criteria, `review_pending`, A/B 비교, 반복 실험 기록을 정의한다 | 단일 변수 A/B 비교를 어떤 UI와 데이터 구조로 기록할 것인가? |
| 내부 재생 및 접근 차단 PRD (Internal Playback and Access Control PRD) | 앱 내부 preview 재생, 외부 공유/다운로드/public URL 차단을 정의한다 | 어떤 조건에서 playback을 허용/차단할 것인가? |
| 삭제/보관/감사 PRD (Deletion, Retention, and Audit PRD) | 원본 음성, preview, 분석 결과, 계정/알림 데이터 삭제와 감사 로그를 정의한다 | 7일 활성 데이터 삭제, 35일 backup retention, 복원 시 tombstone 재적용을 어떻게 자동화할 것인가? |
| 개인정보 및 데이터 보관 정책 문서 (Privacy and Data Retention Policy) | PRD보다 상위 정책으로 계정/연락처/삭제/보관 기준의 정본을 둔다 | 데이터별 보관 기간과 수신 거부/삭제 정책을 어떻게 버전 관리할 것인가? |
| 데이터 인벤토리 문서 (Data Inventory / Data Map) | 필드별 보관 기간, 삭제 trigger, opt-out 가능 여부를 정의한다 | 어떤 데이터가 어느 정책의 적용을 받는가? |
| 운영 runbook (Admin Operations Runbook) | 삭제/비활성화/token 폐기/backup 만료 처리 절차를 정의한다 | 정책 기준을 실제 운영에서 어떻게 실행할 것인가? |

## 열린 질문 (Open Questions)

현재 목표 시스템 정의 단계의 핵심 정책 질문은 결정되었다.

후속 PRD/TRD에서 구체화할 구현 질문은 다음이다.

- A/B 비교 UI에서 baseline/candidate와 variable diff를 어떻게 보여줄 것인가?
- 발성 용어 카드의 review status와 AI review result를 어떤 데이터 구조로 저장할 것인가?
- 삭제 요청 tombstone을 백업 복원, 객체 저장소, 이벤트 로그에 어떻게 재적용할 것인가?

## 다음 추천 스킬 (Recommended Next Skill)

다음은 `phase-planner`가 적합하다.

이유는 목표 시스템의 역할, 모듈, 워크플로우, 데이터 객체, 권한, 연동 지점, 시스템 경계가 정리되었기 때문이다. 다음 단계에서는 이 목표 시스템을 개인 학습/실험의 단계별 구현 범위로 나누어야 한다.
