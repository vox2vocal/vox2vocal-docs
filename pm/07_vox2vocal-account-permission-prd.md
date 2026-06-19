# 개인 계정 및 권한 PRD (Account and Permission PRD)

문서 번호: 07
문서 버전: v0.5
작성일: 2026-06-19
상태: Draft
범위: P0 MVP
기반 문서:

- `pm/03_vox2vocal-product-vision.md` v0.1
- `pm/04_vox2vocal-target-system-definition.md` v0.3
- `pm/05_vox2vocal-phase-plan.md` v0.1
- `pm/06_vox2vocal-account-permission-prd-brief.md` v0.3

## 요약 (Executive Summary)

이 PRD는 Vox2Vocal P0 MVP의 첫 요구사항 문서로, 단일 개인 사용자가 앱과 관리자 페이지를 분리된 경험으로 사용하기 위한 계정, 세션, 권한, 삭제 후 접근 차단, 기본 감사 기준을 정의한다.

Vox2Vocal은 현재 한 명의 개인 사용자가 비상업적 학습/실험 목적으로 사용하는 시스템이다. 같은 개인이 앱 사용자, 곡 관리자, 실험 운영자 역할을 모두 수행하지만, 일반 사용자 계정과 관리자 계정은 별도로 분리한다. 일반 사용자 계정은 앱만 사용할 수 있고 관리자 페이지에는 접근할 수 없다. 관리자 계정은 관리자 페이지와 앱 모두에 접근할 수 있다.

이 PRD는 곡 업로드, 구간 정의, 본인 음성 입력, preview 생성, pitch/note 분석, self-voice 평가는 직접 정의하지 않는다. 대신 해당 기능들이 안전하게 동작하기 위해 필요한 계정 경계와 접근 규칙을 engineering handoff 가능한 수준으로 고정한다.

## 문제 정의 (Problem Statement)

Vox2Vocal은 업로드 곡, 사용자 본인 음성, 변환 preview, pitch/note 분석 결과, 발성 참고 정보, 삭제 요청, 알림 연락처를 다룬다. 계정과 권한 경계가 먼저 명확하지 않으면 다음 문제가 발생한다.

- 일반 사용자 계정이 관리자 페이지에 접근해 곡 metadata, 구간 정의, 권리/출처 메모, 운영 상태를 변경할 수 있다.
- 관리자 계정의 앱 접근 허용이 외부 공유, 다운로드, public URL, 수익화 허용으로 오해될 수 있다.
- 계정 삭제 요청 후에도 기존 세션, playback, job 생성, 관리자 접근이 남아 개인 데이터 통제 원칙을 깨뜨릴 수 있다.
- 권한 차단 상세 사유가 사용자 화면이나 로그에 과도하게 노출되어 개인정보, 권리 상태, 내부 운영 사유가 새어 나갈 수 있다.
- 후속 PRD가 서로 다른 actor, permission, account state를 가정해 곡 업로드, preview, 삭제, audit 기능의 기준이 흔들릴 수 있다.

따라서 P0 MVP가 기능 개발로 들어가기 전에 계정 생성, 로그인, 수동 활성화, 관리자 계정 생성, 권한 차단, 삭제 후 접근 차단, push token 최소 관리, 감사 이벤트 기준을 먼저 잠가야 한다.

## 대상 사용자 (Target Users)

| 사용자 | 설명 | 이 PRD에서 필요한 것 |
|---|---|---|
| 개인 계정 소유자 (Personal Account Owner) | 앱 계정을 만들고 개인 데이터의 소유자가 되는 단일 사용자 | 계정 생성, 로그인, 수동 활성화 상태 이해, 삭제 요청 후 상태 확인 |
| 앱 사용자 (App User) | 앱에서 곡/구간 선택, 본인 음성 입력, preview 재생, 평가를 수행하는 사용자 | 앱 접근 권한, 관리자 접근 차단, 삭제/비활성 상태에서 기능 차단 |
| 관리자 계정 사용자 (Admin Account User) | 같은 개인이지만 별도 관리자 계정으로 관리자 페이지를 사용하는 사용자 | 관리자 페이지 접근, 앱 접근, 곡/구간 관리 권한 |
| 실험 운영자 (Experiment Operator) | 같은 개인이지만 운영 관점에서 job, 실패, 삭제, audit을 확인하는 사용자 | 접근 실패, 권한 차단, 삭제, token 폐기 이벤트 추적 |

현재 대상이 아닌 사용자는 외부 학습자, 강사, 교육기관 운영자, 다중 사용자 팀, 제3자 관리자다.

## 목표 (Goals)

- P0에서 개인 앱 계정을 생성하고 로그인할 수 있게 한다.
- P0 계정 생성은 이메일/비밀번호를 기본 우선순위로 두고, passkey와 소셜 로그인은 후속 확장 또는 인증 provider 제약에 따라 조정한다.
- P0 이메일/비밀번호 계정은 이메일 인증 필수가 아니라 수동 활성화로 시작한다.
- 수동 활성화는 관리자 페이지에서 관리자 계정 사용자가 승인하는 방식으로 처리한다.
- 관리자 승인 기능은 별도 Admin PRD로 미루지 않고, 이 PRD의 P0 최소 admin gate로 포함한다.
- 최소 admin gate는 관리자 계정 로그인 후 접근 가능한 단일 `계정 승인 Gate (Account Approval Gate)` 화면으로 시작한다.
- 일반 사용자 계정과 관리자 계정을 별도 계정으로 분리한다.
- 관리자 계정은 제한된 가입 화면이 아니라 seed/admin script로 생성한다.
- 일반 사용자 계정은 앱 기능에 접근할 수 있지만 관리자 페이지 접근은 차단한다.
- 관리자 계정은 관리자 페이지와 앱 모두에 접근할 수 있게 한다.
- P0에서는 로그인 상태의 비밀번호 변경을 허용한다.
- 계정 삭제 요청 성공 응답 이후 다음 API 요청부터 로그인, playback, job 생성, 관리자 접근을 차단한다.
- 삭제 요청 후에는 삭제 접수/진행 상태를 보여주는 최소 상태 화면만 허용한다.
- P0에서는 동시 로그인을 허용하되, 로그인한 기기와 세션을 추적할 수 있어야 한다.
- 권한 차단 상세 사유는 사용자에게 노출하지 않고 감사 로그에 남긴다.
- push token은 계정 PRD에서 등록/해제와 권한 상태만 정의하고, 상세 알림 정책은 별도 PRD로 분리한다.
- 후속 PRD가 참조할 actor, account state, permission boundary, audit event 기준을 제공한다.

## 비목표 (Non-goals)

- 곡 업로드, 곡 metadata, 구간 정의, 권리/출처 등록부를 상세 정의하지 않는다.
- 본인 음성 녹음/업로드, preview 생성, playback 품질, pitch/note 분석, 발성 참고 정보, self-voice 평가를 정의하지 않는다.
- passkey, 소셜 로그인, MFA, 조직 SSO를 P0 필수 구현 범위로 확정하지 않는다.
- 관리자 가입 화면을 만들지 않는다.
- 정교한 사용자 관리 콘솔, 사용자 초대, 권한 위임, 조직 관리 기능을 만들지 않는다.
- 앱/웹의 public self-service 비밀번호 재설정과 계정 복구 기능을 만들지 않는다.
- P0에서는 이메일/연락처 변경 기능을 만들지 않는다.
- 삭제 요청 취소와 삭제 계정 재활성화를 만들지 않는다.
- 사용자가 직접 모든 활성 세션을 조회/종료하는 세션 관리 UI를 만들지 않는다.
- 계정 거절, 계정 재활성화, 관리자 계정 회수, 계정 병합 기능을 만들지 않는다.
- 계정 승인 Gate에서 사용자 검색/필터 고도화, 역할 변경, 관리자 계정 관리, 사용자 상세 프로필 편집을 만들지 않는다.
- 강사/교육생, 조직, 팀, 다중 tenant 권한 모델을 만들지 않는다.
- 알림 템플릿, retry, in-app inbox, fallback 발송 정책을 상세 정의하지 않는다.
- API schema, DB schema, token 저장 방식, auth provider, session 구현 방식을 결정하지 않는다.
- 외부 공유, 다운로드, public URL, 수익화, 배포, 모델 학습 사용을 허용하지 않는다.

## 범위 (Scope)

### 포함 범위 (In Scope)

- 이메일/비밀번호 기반 P0 계정 생성과 로그인 요구사항
- 수동 활성화 상태와 활성화 전 접근 제한
- 관리자 계정 로그인 후 접근 가능한 단일 계정 승인 Gate
- pending 상태 사용자 안내 문구와 허용 버튼
- 로그인 상태 비밀번호 변경
- 관리자용 password reset script와 실행 audit, reset 후 session/token 폐기 기준
- 일반 사용자 계정과 관리자 계정 분리
- 관리자 계정의 seed/admin script 생성 원칙
- 앱 접근, 관리자 접근, 운영 범위의 권한 경계
- 계정 상태별 허용/차단 행동
- 삭제 요청 성공 응답 이후 다음 API 요청 차단과 최소 삭제 상태 화면
- 동시 로그인 허용과 session/device inventory 기록 기준
- 권한 차단 사용자 메시지와 감사 로그 분리
- push token 등록/해제 lifecycle의 최소 요구사항
- 계정/세션/권한/삭제 관련 기본 audit event

### 제외 범위 (Out of Scope)

- 곡, 구간, 음성, preview, 분석 결과의 상세 데이터 모델
- 내부 playback 조건과 signed playback URL 정책
- 삭제 대상 데이터별 retention 수치의 정본 관리
- 데이터 인벤토리, 개인정보 보관 정책, 운영 runbook 상세
- 관리자 곡 업로드 화면과 운영 모니터링 화면 상세
- 이메일/연락처 변경과 계정 프로필 편집
- P1 이후 A/B 비교, 알림함, push 발송 자동화, passkey/social login 확장

## 결정된 기준 및 TRD 입력 가이드 (Resolved Decisions and TRD Guidance)

### 수동 활성화와 계정 승인 Gate (Manual Activation and Account Approval Gate)

P0 수동 활성화는 관리자 페이지에서 관리자 계정 사용자가 승인하는 방식으로 진행한다.

관리자 계정은 seed/admin script로 먼저 생성하고, 일반 사용자 계정은 가입 후 `pending_admin_approval` 상태로 시작한다. 관리자는 관리자 페이지의 최소 승인 화면에서 해당 계정을 승인해 `active` 상태로 전환한다.

P0 승인 화면은 `계정 승인 Gate (Account Approval Gate)`로 정의한다. 이 화면은 관리자 계정 로그인 후 접근 가능한 단일 승인 화면이며, 계정 활성화에 필요한 최소 기능만 가진다.

- `pending_admin_approval` 계정 목록 확인
- 계정 식별에 필요한 최소 정보 표시: account id, masked email, created at, status
- 계정 승인
- 승인 성공/실패 상태 표시
- 승인 actor, target account, result, timestamp, reason code audit 기록

P0 계정 승인 Gate에서 제외하는 기능은 다음이다.

- 사용자 검색/필터 고도화
- 계정 거절
- 계정 재활성화
- 역할 변경
- 관리자 계정 관리
- 사용자 상세 프로필 편집
- 조직/초대/권한 위임

### Pending 안내 기준 (Pending Notice Standard)

`pending_admin_approval` 상태의 사용자에게는 승인 대기 상태만 짧고 일반적으로 안내한다.

기본 문구:

- "계정 승인 대기 중입니다. 관리자가 승인하면 앱을 사용할 수 있습니다."

보조 문구:

- "승인이 완료된 뒤 다시 로그인하거나 화면을 새로고침해 주세요."

허용 버튼:

- 새로고침
- 로그아웃

노출 금지:

- 내부 reason code
- 관리자 이름
- 승인 지연 사유
- audit id
- 보안/권한 상세
- "차단됨", "거절됨" 같은 강한 표현

### 삭제 상태 화면과 Token 폐기 가이드 (Deletion Status and Token Revocation Guidance)

삭제 요청 후에는 일반 로그인 세션, 앱 세션, 관리자 세션, refresh token, push token을 일반 기능 접근 용도로 계속 사용하면 안 된다. 삭제 상태 화면이 필요하더라도 일반 세션을 살리는 방식이 아니라 **삭제 상태 전용 제한 접근**으로 분리한다.

권장 흐름은 다음이다.

1. 사용자가 삭제를 요청한다.
2. 서버가 계정 상태를 `deletion_requested`로 바꾼다.
3. 서버가 기존 app/admin session과 refresh token을 즉시 폐기한다.
4. 서버가 playback, job 생성, admin 접근, 일반 앱 API를 계정 상태 기준으로 차단한다.
5. 삭제 요청 성공 응답 이후 다음 API 요청부터 login, playback, job 생성, admin 접근, 일반 앱 API를 차단한다.
6. 삭제 상태 전용 제한 접근은 삭제 접수/진행 상태 조회만 가능하고 refresh, playback, job 생성, admin API 호출 권한을 갖지 않는다.
7. 삭제 상태 전용 제한 접근이 없거나 만료되면 사용자는 일반 기능으로 돌아가지 못하고, 짧고 일반적인 접근 제한 메시지만 본다.

TRD 입력 가이드는 다음과 같다. 이 항목은 제품 요구사항이 아니라 후속 기술 설계에서 검토할 추천 구현 방향이다. PRD가 확정하는 제품 요구는 **삭제 요청 후 일반 기능 차단과 삭제 상태 조회 권한 격리**다.

- P0 추천 구현안은 server-side session store 기반 status-only session이다.
- 짧은 TTL의 status-only signed token이나 삭제 요청 직후 1회성 status receipt는 P0 기본안으로 쓰지 않는다.
- API gateway 또는 auth layer 기준 차단 반영 목표는 1초 이내로 둔다.
- 분산 구성의 운영 관측 지표는 P99 5초 이내로 둔다.

server-side status-only session은 즉시 폐기, 권한 축소, 감사 추적이 가장 단순하고, stateless token revoke 문제를 줄일 수 있기 때문에 추천한다. 단, 이 PRD의 제품 요구는 구현 방식 자체가 아니라 **삭제 요청 후 일반 기능 차단과 삭제 상태 조회 권한 격리**다.

측정 이벤트 기준은 다음으로 고정한다.

- 시작: 삭제 요청 API 성공 응답 시점
- 종료: 첫 protected API 차단 관측 시점
- 별도 측정: token revoke completed timestamp

Protected API 예시는 login, 곡 선택, playback, job 생성, 관리자 접근, 일반 앱 API다.

### 비밀번호 변경, 재설정과 계정 복구 (Password Change, Reset, and Account Recovery)

P0에서는 로그인 상태의 비밀번호 변경을 허용한다. 이 기능은 계정 소유자가 이미 로그인되어 있고 계정 상태가 `active`일 때 자신의 비밀번호를 변경하는 흐름이다.

로그인 상태 비밀번호 변경 기준:

- `active` 상태의 로그인 사용자만 자신의 비밀번호를 변경할 수 있다.
- 관리자 계정이 일반 사용자 계정의 비밀번호를 앱/웹 화면에서 직접 변경하는 기능은 만들지 않는다.
- 비밀번호 변경 성공/실패는 감사 대상으로 남긴다.
- 비밀번호 원문, 이전 비밀번호, 새 비밀번호는 audit log에 남기지 않는다.
- 비밀번호 변경 후 현재 session은 유지하고, 동일 계정의 다른 active session과 refresh token은 폐기한다. 이를 보장하는 session/token 저장소, revoke event, audit event는 TRD에서 구체화한다.

P0에서는 앱/웹에서 사용자가 직접 수행하는 public self-service 비밀번호 재설정과 계정 복구를 만들지 않는다.

대신 개인 시스템 운영을 위해 관리자용 password reset script만 허용한다. 이 script는 다음 기준을 만족해야 한다.

- 관리자 계정 또는 로컬 운영 권한으로만 실행 가능
- reset 대상 계정과 실행 actor audit 기록
- reset 후 대상 계정의 기존 session과 refresh token 폐기
- reset 결과를 사용자 화면에 상세 노출하지 않음

계정 복구, 이메일 기반 reset link, 복구 코드, 계정 소유권 재검증 UI는 P0 제외 범위다.

### 이메일과 연락처 변경 (Email and Contact Change)

P0에서는 이메일/연락처 변경을 제공하지 않는다.

- 앱에서 이메일 변경을 제공하지 않는다.
- 앱에서 SNS 계정 또는 알림 연락처 변경을 제공하지 않는다.
- 관리자 페이지에서 일반 사용자 이메일/연락처를 편집하는 기능을 제공하지 않는다.
- 이메일/연락처 변경 정책은 후속 계정 설정 또는 개인정보 PRD에서 별도로 정의한다.

### 삭제 취소와 계정 재활성화 (Deletion Cancellation and Reactivation)

P0에서는 삭제 요청 취소와 삭제 계정 재활성화를 허용하지 않는다.

- `deletion_requested` 이후 사용자는 삭제 취소를 할 수 없다.
- `deletion_in_progress` 이후 취소를 허용하지 않는다.
- `deleted`는 terminal 상태로 본다.
- 실수 삭제 대응은 P0 제품 기능이 아니라 별도 운영 검토 대상으로 둔다.

`disabled` 상태에서 `active`로 되돌리는 재활성화는 삭제 취소와 별개이며, P0에서는 기본 기능으로 만들지 않는다.

### 동시 로그인과 기기 식별 (Concurrent Login and Device Inventory)

P0에서는 동시 로그인을 허용한다. 단, 어떤 기기에서 로그인했는지 프로그램 내부에서 추적할 수 있어야 한다.

최소 session/device inventory는 다음 정보를 가진다.

- `session_id_hash`
- `account_id`
- `device_id`
- `device_name`
- `platform`
- `app_version`
- `browser` 또는 `user_agent`
- `created_at`
- `last_seen_at`
- `last_ip_hash` 또는 coarse location
- `push_token_id`
- `session_status`

P0에서는 사용자가 직접 세션을 조회하거나 종료하는 세션 관리 UI는 만들지 않는다. 다만 후속 기능에서 세션 관리 UI를 만들 수 있도록 데이터는 남긴다.

### 계정 상태명 권장안 (Recommended Account States)

| 상태 | 의미 | 허용 행동 |
|---|---|---|
| `pending_admin_approval` | 일반 사용자 계정 생성 후 관리자 승인 대기 | pending 안내 확인만 허용 |
| `active` | 정상 사용 가능 | 권한 범위 내 앱 접근 허용 |
| `disabled` | 보안, 운영, 수동 차단 상태 | 로그인, playback, job 생성, admin 접근 차단 |
| `deletion_requested` | 삭제 요청 접수 후 일반 기능 차단 상태 | 삭제 상태 전용 최소 화면만 허용 |
| `deletion_in_progress` | 삭제 worker 또는 운영 절차 진행 중 | 삭제 상태 전용 최소 화면만 허용 |
| `deleted` | 활성 개인정보/산출물 삭제 또는 익명화 완료 후 terminal tombstone 상태 | 일반 접근 불가, 최소 audit/tombstone만 유지 |

TRD에서 실제 enum 이름은 바꿀 수 있지만, 위 상태의 제품 의미와 차단 규칙은 유지해야 한다.

### Session/Token 폐기 기준 (Session and Token Revocation Standard)

P0에서는 다음 기준을 TRD의 기본값으로 삼는 것이 좋다.

- 서버가 account status와 session validity의 정본이 된다.
- 권한 변경, 관리자 승인, 비활성화, 삭제 요청 같은 상태 변화 후에는 기존 session을 그대로 신뢰하지 않는다.
- 일반 session과 refresh token은 서버에서 폐기 가능해야 한다.
- access token이 stateless JWT라면 즉시 차단을 위해 짧은 TTL만으로는 부족할 수 있으므로 `token_version`, introspection, denylist, server-side session 중 하나를 사용한다.
- privilege level이 바뀌는 순간에는 session id를 재발급하거나 기존 session을 폐기한다.
- session id/token 원문은 URL, log, localStorage에 남기지 않는다.
- web cookie를 쓴다면 `Secure`, `HttpOnly`, `SameSite=Lax` 또는 `SameSite=Strict`를 기본으로 검토한다.

### Audit Reason Code 권장안 (Recommended Audit Reason Codes)

P0 audit reason code는 너무 촘촘한 진단 체계가 아니라, 원인 추적과 보안 확인에 충분한 최소 enum으로 시작한다.

| 분류 | 권장 reason code |
|---|---|
| 계정 생성/활성화 | `ACCOUNT_CREATED_PENDING`, `ACCOUNT_ACTIVATION_APPROVED`, `ACCOUNT_ACTIVATION_REJECTED`, `ACCOUNT_DISABLED` |
| 로그인/세션 | `AUTH_LOGIN_SUCCESS`, `AUTH_LOGIN_FAILED`, `AUTH_LOGOUT`, `AUTH_SESSION_EXPIRED`, `AUTH_SESSION_REVOKED`, `AUTH_PRIVILEGE_CHANGED` |
| 권한 차단 | `AUTHZ_APP_ACCESS_DENIED`, `AUTHZ_ADMIN_ACCESS_DENIED`, `AUTHZ_PLAYBACK_DENIED`, `AUTHZ_JOB_CREATE_DENIED` |
| 관리자 작업 | `ADMIN_ACCOUNT_SEEDED`, `ADMIN_ACCOUNT_APPROVAL_VIEWED`, `ADMIN_ACCOUNT_APPROVAL_SUBMITTED` |
| 삭제 | `ACCOUNT_DELETION_REQUESTED`, `ACCOUNT_DELETION_STATUS_VIEWED`, `ACCOUNT_DELETION_IN_PROGRESS`, `ACCOUNT_DELETION_COMPLETED` |
| Push token | `PUSH_TOKEN_REGISTERED`, `PUSH_TOKEN_REVOKED`, `PUSH_TOKEN_DISABLED` |

Audit event에는 최소한 `event_id`, `actor_account_id`, `actor_account_type`, `target_type`, `target_id`, `action`, `reason_code`, `result`, `occurred_at`, `request_id` 또는 `interaction_id`를 남긴다. 비밀번호, token 원문, 음성 내용, 곡 파일 내용, 민감한 권리/출처 전문은 직접 기록하지 않는다.

### 참고 기준 (Reference Standards)

- OWASP Session Management Cheat Sheet
- OWASP Logging Cheat Sheet
- NIST SP 800-63B Session Management

## 사용자 스토리 (User Stories)

- 개인 계정 소유자로서, 나는 앱 계정을 생성하고 수동 활성화 이후 로그인해 내 개인 학습/실험 데이터를 계정 기준으로 관리하고 싶다.
- 개인 계정 소유자로서, 나는 로그인된 상태에서 내 비밀번호를 변경하고 변경 이력이 안전하게 기록되기를 원한다.
- 개인 계정 소유자로서, 나는 내 계정이 아직 활성화되지 않았을 때 일반 기능에 접근하지 못하고 현재 상태를 이해할 수 있는 최소 안내를 보고 싶다.
- 앱 사용자로서, 나는 일반 사용자 계정으로 앱 기능을 사용할 수 있지만 관리자 페이지에는 접근할 수 없기를 원한다.
- 관리자 계정 사용자로서, 나는 별도 관리자 계정으로 관리자 페이지에 접근하고, 같은 관리자 계정으로 앱도 확인할 수 있기를 원한다.
- 실험 운영자로서, 나는 권한 없는 접근, 관리자 접근 실패, 삭제 요청, token 폐기 같은 이벤트가 나중에 원인을 확인할 수 있도록 감사 로그에 남기를 원한다.
- 개인 계정 소유자로서, 나는 계정 삭제를 요청하면 성공 응답 이후 다음 API 요청부터 로그인, playback, job 생성, 관리자 접근이 막히고, 삭제 접수/진행 상태만 확인할 수 있기를 원한다.
- 개인 계정 소유자로서, 나는 처리 완료/실패/삭제/보안 상태 알림을 나중에 받을 수 있도록 push token 등록과 해제를 계정 설정 범위에서 관리하고 싶다.

## 기능 요구사항 (Functional Requirements)

### 계정 생성 및 활성화 (Account Creation and Activation)

| ID | 요구사항 |
|---|---|
| APR-FR-001 | P0 계정 생성 기본 방식은 이메일/비밀번호로 둔다. |
| APR-FR-002 | passkey는 이메일/비밀번호 이후 확장 후보로 둔다. |
| APR-FR-003 | 소셜 로그인은 passkey 이후 확장 후보로 둔다. 단, 선택한 인증 provider 또는 개발 제약상 소셜 로그인이 선행되어야 하면 별도 결정으로 순서를 조정할 수 있다. |
| APR-FR-004 | P0 이메일/비밀번호 계정은 이메일 인증 완료를 요구하지 않고 관리자 승인 기반 수동 활성화 상태를 거친다. |
| APR-FR-005 | 수동 활성화 전 계정은 일반 앱 기능, playback, job 생성, 관리자 접근을 사용할 수 없다. |
| APR-FR-006 | 수동 활성화 전 사용자는 "계정 승인 대기 중입니다. 관리자가 승인하면 앱을 사용할 수 있습니다." 문구와 "승인이 완료된 뒤 다시 로그인하거나 화면을 새로고침해 주세요." 보조 문구를 볼 수 있어야 한다. 이 화면은 새로고침과 로그아웃만 허용하고, 내부 reason code, 관리자 이름, 승인 지연 사유, audit id, 보안/권한 상세, "차단됨", "거절됨" 문구를 노출하지 않는다. |
| APR-FR-007 | 활성화된 일반 사용자 계정은 앱 기능에 접근할 수 있다. |
| APR-FR-008 | 관리자 계정 사용자는 관리자 페이지의 단일 계정 승인 Gate에서 수동 활성화 대기 계정을 승인할 수 있어야 한다. |
| APR-FR-009 | 계정 승인 이벤트는 승인 actor, target account, result, timestamp, reason code와 함께 감사 대상으로 남긴다. |

### 세션 및 계정 상태 (Session and Account State)

| ID | 요구사항 |
|---|---|
| APR-FR-010 | 시스템은 최소한 `pending_admin_approval`, `active`, `disabled`, `deletion_requested`, `deletion_in_progress`, `deleted`에 해당하는 계정 상태 의미를 가져야 한다. 실제 상태명은 TRD에서 조정할 수 있으나 제품 의미는 유지해야 한다. |
| APR-FR-011 | 로그아웃, 세션 만료, 계정 비활성화, 삭제 요청 이후에는 기존 접근 권한이 남지 않아야 한다. |
| APR-FR-012 | 삭제 요청 또는 비활성화 상태에서는 새 로그인뿐 아니라 기존 세션 기반 앱 접근도 차단해야 한다. |
| APR-FR-013 | 세션 만료나 권한 차단 시 사용자에게는 짧고 일반적인 메시지만 보여준다. 상세 사유는 감사 로그로 남긴다. |
| APR-FR-014 | 계정 상태 변경 후 서버는 계정 상태와 session validity를 기준으로 권한을 다시 판단해야 한다. |
| APR-FR-015 | privilege level이 바뀌는 순간 기존 session을 그대로 승격해 사용하지 않는다. |
| APR-FR-016 | P0에서는 같은 계정의 동시 로그인을 허용한다. |
| APR-FR-017 | 각 로그인 세션은 기기와 세션을 식별할 수 있는 session/device inventory에 기록되어야 한다. |
| APR-FR-018 | P0에서는 사용자가 직접 활성 세션을 조회하거나 종료하는 세션 관리 UI를 제공하지 않는다. |

### 관리자 계정 및 권한 분리 (Admin Account and Permission Separation)

| ID | 요구사항 |
|---|---|
| APR-FR-020 | 일반 사용자 계정과 관리자 계정은 별도 계정으로 분리한다. |
| APR-FR-021 | 관리자 계정은 제한된 관리자 가입 화면이 아니라 seed/admin script로 생성한다. |
| APR-FR-022 | 일반 사용자 계정은 관리자 페이지에 접근할 수 없다. |
| APR-FR-023 | 관리자 계정은 관리자 페이지에 접근할 수 있다. |
| APR-FR-024 | 관리자 계정은 앱 사용자 권한을 포함해 앱에도 접근할 수 있다. |
| APR-FR-025 | 관리자 계정 권한은 외부 공유, 다운로드, public URL, 수익화, 배포, 모델 학습 사용을 허용하지 않는다. |
| APR-FR-026 | 관리자 접근 실패와 권한 차단은 사용자 화면에는 일반 메시지로 표시하고, 상세 reason code는 감사 로그에 남긴다. |
| APR-FR-027 | 일반 사용자 계정이 관리자 페이지에 접근할 때 관리자 화면, pending 계정 목록, 계정별 권한 정보, 승인 버튼은 표시되지 않아야 한다. |
| APR-FR-028 | 계정 승인 Gate는 account id, masked email, created at, status, 승인 버튼, 승인 성공/실패 상태만 P0 필수 표시 대상으로 가진다. |

### 삭제 요청 및 접근 차단 (Deletion Request and Access Blocking)

| ID | 요구사항 |
|---|---|
| APR-FR-030 | 계정 삭제 요청이 성공 처리되면 계정은 삭제 요청 또는 비활성 상태로 전환되어야 한다. |
| APR-FR-031 | 삭제 요청 후 기존 session/token은 폐기되어야 한다. |
| APR-FR-032 | 삭제 요청 성공 응답 이후 다음 API 요청부터 로그인, playback, job 생성, 관리자 접근은 모두 차단되어야 한다. |
| APR-FR-033 | 삭제 요청 직후 일반 기능 접근은 모두 막되, 삭제 접수/진행 상태를 보여주는 최소 상태 화면만 허용한다. |
| APR-FR-034 | 삭제 상태 화면은 삭제 접수 여부, 처리 중 상태, 일반 기능 차단 사실만 보여준다. |
| APR-FR-035 | 삭제 상태 화면은 playback, job 생성, 관리자 페이지, 일반 앱 기능으로 이동할 수 없어야 한다. |
| APR-FR-036 | 삭제 상태 화면은 내부 삭제 사유, 데이터 위치, audit reason code, 권리/출처 세부정보를 노출하지 않는다. |
| APR-FR-037 | 삭제 상태 화면은 일반 app/admin session이 아니라 삭제 상태 전용 제한 접근으로만 표시되어야 한다. |
| APR-FR-038 | 삭제 상태 전용 제한 접근은 삭제 상태 조회 외의 API 권한을 갖지 않는다. |
| APR-FR-039 | `deletion_requested`, `deletion_in_progress`, `deleted` 상태에서는 삭제 취소와 계정 재활성화를 허용하지 않는다. |

### Push Token 최소 관리 (Minimum Push Token Management)

| ID | 요구사항 |
|---|---|
| APR-FR-040 | 계정 PRD는 push token의 등록, 해제, 비활성화, 권한 상태만 다룬다. |
| APR-FR-041 | push token은 계정 유지, 로그아웃, 수신 거부, 앱 삭제 감지, 삭제 요청 상태와 연결되어야 한다. |
| APR-FR-042 | push payload에는 곡명, 음원 세부정보, 권리 상태, 음성 내용, 내부 reason code 같은 민감정보를 포함하지 않는다. |
| APR-FR-043 | 알림 템플릿, 발송 retry, in-app inbox, 이메일/SNS fallback은 별도 알림 PRD에서 정의한다. |

### 감사 및 로그 (Audit and Logging)

| ID | 요구사항 |
|---|---|
| APR-FR-050 | 로그인 성공/실패, 로그아웃, 세션 만료, 수동 활성화 승인, 계정 비활성화, 삭제 요청, token 폐기 이벤트는 감사 대상으로 남긴다. |
| APR-FR-051 | 관리자 페이지 접근 성공/실패, 권한 차단, 곡/구간 관리 권한 차단, 운영 범위 차단 이벤트는 감사 대상으로 남긴다. |
| APR-FR-052 | push token 등록/해제/비활성화 이벤트는 감사 또는 운영 추적 대상으로 남긴다. |
| APR-FR-053 | 감사 로그에는 비밀번호, token 원문, 음성 내용, 곡 파일 내용, 민감한 권리/출처 전문을 직접 남기지 않는다. |
| APR-FR-054 | 사용자에게 보여주는 오류 메시지와 감사 로그 reason code는 분리한다. |

### 비밀번호 변경, 재설정 및 계정 복구 (Password Change, Reset, and Account Recovery)

| ID | 요구사항 |
|---|---|
| APR-FR-070 | P0에서는 `active` 상태로 로그인한 사용자가 자신의 비밀번호를 변경할 수 있어야 한다. |
| APR-FR-071 | 로그인 상태 비밀번호 변경 성공/실패는 감사 대상으로 남긴다. |
| APR-FR-072 | 비밀번호 원문, 이전 비밀번호, 새 비밀번호는 audit log에 남기지 않는다. |
| APR-FR-073 | P0에서는 앱/웹의 public self-service 비밀번호 재설정 기능을 제공하지 않는다. |
| APR-FR-074 | P0에서는 앱/웹의 public self-service 계정 복구 기능을 제공하지 않는다. |
| APR-FR-075 | P0에서는 관리자용 password reset script만 허용한다. |
| APR-FR-076 | 관리자용 password reset script 실행 시 actor, target account, result, timestamp, reason code를 감사 대상으로 남긴다. |
| APR-FR-077 | 관리자용 password reset script 실행 후 대상 계정의 기존 session과 refresh token은 폐기되어야 한다. |
| APR-FR-078 | P0에서는 이메일/연락처 변경 기능을 제공하지 않는다. |
| APR-FR-079 | 관리자 페이지에서 일반 사용자 이메일/연락처를 편집하는 기능을 제공하지 않는다. |

### 후속 PRD 참조 기준 (Downstream PRD Reference)

| ID | 요구사항 |
|---|---|
| APR-FR-060 | 곡 업로드/구간 관리 PRD는 관리자 계정만 곡 metadata, 출처/권리 메모, 구간 정의를 변경할 수 있다는 기준을 참조해야 한다. |
| APR-FR-061 | 앱 입력/preview PRD는 일반 사용자 계정과 관리자 계정 모두 앱 접근이 가능하되, 삭제/비활성 상태에서는 playback과 job 생성이 차단된다는 기준을 참조해야 한다. |
| APR-FR-062 | 삭제/보관/감사 PRD는 이 PRD의 삭제 요청 성공 응답 이후 다음 API 요청 차단 범위와 최소 상태 화면 기준을 참조해야 한다. |
| APR-FR-063 | 알림 PRD는 이 PRD의 push token 최소 lifecycle과 민감정보 payload 금지 기준을 참조해야 한다. |

## 인수 조건 (Acceptance Criteria)

| ID | Given | When | Then |
|---|---|---|---|
| APR-AC-001 | 신규 사용자가 이메일/비밀번호로 계정을 생성했다 | 계정 생성이 완료된다 | 계정은 관리자 승인 대기 상태로 표시되고 일반 앱 기능은 열리지 않는다 |
| APR-AC-002 | 계정이 `pending_admin_approval` 상태다 | 사용자가 곡 선택, playback, job 생성, 관리자 접근 중 하나를 시도한다 | 기능은 차단되고 "계정 승인 대기 중입니다. 관리자가 승인하면 앱을 사용할 수 있습니다." 문구, "승인이 완료된 뒤 다시 로그인하거나 화면을 새로고침해 주세요." 보조 문구, 새로고침/로그아웃 버튼만 표시된다 |
| APR-AC-003 | 계정이 수동 활성화 전 상태다 | 사용자가 관리자 페이지 접근을 시도한다 | 접근은 차단되고 상세 reason code는 사용자에게 노출되지 않는다 |
| APR-AC-004 | 일반 사용자 계정이 활성 상태다 | 사용자가 앱에 로그인한다 | 앱 기본 기능 접근이 허용된다 |
| APR-AC-005 | 일반 사용자 계정이 활성 상태다 | 사용자가 관리자 페이지 URL 또는 관리자 기능에 접근한다 | 접근은 차단되고 관리자 화면, pending 계정 목록, 계정별 권한 정보, 승인 버튼이 표시되지 않는다 |
| APR-AC-006 | 관리자 계정이 존재한다 | 사용자가 관리자 계정으로 관리자 페이지에 로그인한다 | 관리자 페이지 접근이 허용된다 |
| APR-AC-007 | 관리자 계정이 존재한다 | 사용자가 관리자 계정으로 앱에 접근한다 | 앱 접근이 허용된다 |
| APR-AC-008 | 관리자 계정이 앱에 접근했다 | 사용자가 외부 공유, 다운로드, public URL, 수익화, 배포, 모델 학습 사용을 시도한다 | 해당 행동은 권한으로 허용되지 않는다 |
| APR-AC-009 | 계정이 활성 상태다 | 사용자가 삭제를 요청한다 | 삭제 요청 또는 비활성 상태로 전환되고 기존 session/token 폐기가 시작된다 |
| APR-AC-010 | 삭제 요청 성공 응답이 반환되었다 | 사용자가 다음 API 요청으로 로그인, playback, job 생성, 관리자 접근을 시도한다 | 모든 행동이 차단된다 |
| APR-AC-011 | 삭제 요청이 접수된 직후다 | 사용자가 앱 화면을 확인한다 | 삭제 접수/진행 상태와 곡 선택, playback, job 생성, 관리자 접근 차단 사실만 담은 최소 상태 화면이 표시될 수 있다 |
| APR-AC-012 | 삭제 상태 화면이 표시된다 | 사용자가 playback, job 생성, 관리자 페이지, 일반 앱 기능으로 이동하려 한다 | 이동은 허용되지 않는다 |
| APR-AC-013 | 삭제 상태 화면이 표시된다 | 화면 내용을 확인한다 | 내부 삭제 사유, 데이터 위치, audit reason code, 권리/출처 세부정보가 표시되지 않는다 |
| APR-AC-014 | 권한 없는 접근이 발생했다 | 사용자 화면과 감사 로그가 생성된다 | 사용자는 일반 메시지만 보고, 감사 로그에는 추적 가능한 상세 reason code가 남는다 |
| APR-AC-015 | push token이 등록되어 있다 | 사용자가 로그아웃, 수신 거부, 앱 삭제 감지, 삭제 요청 중 하나의 상태가 된다 | push token은 비활성화 또는 해제 대상으로 처리된다 |
| APR-AC-016 | push payload가 생성된다 | payload 내용을 검토한다 | 곡명, 음원 세부정보, 권리 상태, 음성 내용, 내부 reason code가 포함되지 않는다 |
| APR-AC-017 | 로그인, 관리자 접근, 권한 차단, 삭제 요청, token 폐기 이벤트가 발생한다 | audit 기록을 확인한다 | 이벤트 종류, actor 범위, 결과, 시간이 추적 가능하며 비밀번호/token 원문/음성 내용은 남지 않는다 |
| APR-AC-018 | 일반 사용자 계정이 관리자 승인 대기 상태다 | 관리자 계정 사용자가 관리자 페이지에서 승인한다 | 계정은 활성 상태가 되고 승인 이벤트가 감사 로그에 남는다 |
| APR-AC-019 | 삭제 상태 전용 제한 접근이 허용되어 있다 | 사용자가 삭제 상태 조회 외 API를 호출한다 | 요청은 차단되고 일반 app/admin 권한으로 승격되지 않는다 |
| APR-AC-020 | 일반 사용자 계정이 로그인되어 있다 | 같은 계정으로 다른 기기에서 로그인한다 | 두 세션은 모두 허용되고 각각 session/device inventory에 기록된다 |
| APR-AC-021 | session/device inventory가 기록된다 | 세션 정보를 확인한다 | session id hash, account id, device id/name, platform, app/browser 정보, created/last seen time, session status를 추적할 수 있다 |
| APR-AC-022 | `active` 상태의 로그인 사용자가 자기 계정의 비밀번호 변경을 요청한다 | 변경이 성공한다 | 비밀번호 변경 성공 audit이 남고 비밀번호 원문, 이전 비밀번호, 새 비밀번호는 audit log에 남지 않는다 |
| APR-AC-023 | 사용자가 앱/웹에서 비밀번호 재설정 또는 계정 복구를 찾는다 | P0 화면을 확인한다 | public self-service 비밀번호 재설정/계정 복구 기능은 제공되지 않는다 |
| APR-AC-024 | 계정이 `deletion_requested`, `deletion_in_progress`, `deleted` 중 하나다 | 사용자가 삭제 취소 또는 재활성화를 시도한다 | 취소/재활성화는 허용되지 않는다 |
| APR-AC-025 | 삭제 요청 API 성공 응답 시점이 기록되어 있다 | auth/gateway 차단 지표를 측정한다 | 시작은 삭제 요청 API 성공 응답 시점, 종료는 첫 protected API 차단 관측 시점, 별도 측정값은 token revoke completed timestamp로 기록된다 |
| APR-AC-026 | 관리자용 password reset script가 실행된다 | reset이 성공한다 | 실행 actor, target account, result, timestamp, reason code가 감사 로그에 남고 기존 session/refresh token이 폐기된다 |
| APR-AC-027 | 사용자가 앱/웹에서 이메일 또는 연락처 변경을 찾는다 | P0 화면을 확인한다 | 이메일/연락처 변경 기능은 제공되지 않는다 |

## 성공 지표 (Success Metrics)

### 기준선 (Baseline)

- 신규 제품이므로 실제 운영 기준선은 없다.
- 현재 기준선은 문서상 결정 사항과 수동 QA 결과로 시작한다.
- 계정 생성 전환율, 로그인 성공률, 권한 차단율, 삭제 요청 처리율, push token 등록률의 실측 baseline은 P0 구현 후 수집해야 한다.
- 삭제 요청 후 차단 반영 시간, session/token 폐기 시간, audit 누락률, session/device inventory 기록률은 P0 구현 후 측정 baseline을 수집해야 한다.
- 삭제 차단 시간의 측정 시작점은 삭제 요청 API 성공 응답 시점, 종료점은 첫 protected API 차단 관측 시점으로 고정한다.

### 목표치 (Target)

| 지표 | P0 목표 |
|---|---|
| 관리자 승인 후 일반 사용자 계정 활성화 성공률 | QA 시나리오 기준 100% |
| 일반 사용자 계정의 관리자 접근 차단율 | QA 시나리오 기준 100% |
| 관리자 계정의 관리자 페이지 접근 성공률 | 정상 계정/정상 세션 QA 시나리오 기준 100% |
| 관리자 계정의 앱 접근 성공률 | 정상 계정/정상 세션 QA 시나리오 기준 100% |
| 삭제 요청 후 login/playback/job/admin 접근 차단율 | QA 시나리오 기준 100% |
| 삭제 상태 전용 제한 접근의 권한 격리 | QA 시나리오 기준 100% |
| 삭제 요청 후 다음 API 요청 차단 | QA 시나리오 기준 100% |
| auth/gateway 차단 반영 시간 | 목표 1초 이내, 운영 관측 P99 5초 이내 |
| 삭제 차단 측정 이벤트 기록률 | 삭제 요청 API 성공 응답 시점, 첫 protected API 차단 관측 시점, token revoke completed timestamp 기준 QA 시나리오 100% |
| session/device inventory 기록률 | 로그인 QA 시나리오 기준 100% |
| 로그인 상태 비밀번호 변경 audit 기록률 | QA 시나리오 기준 100% |
| 관리자용 password reset script audit 기록률 | QA 시나리오 기준 100% |
| 권한 차단 상세 사유의 사용자 화면 비노출 | QA 시나리오 기준 100% |
| 계정/세션/권한/삭제 핵심 이벤트 audit 기록률 | QA 시나리오 기준 100% |
| push payload 민감정보 미포함 | QA 시나리오 기준 100% |

### 보호 지표 (Guardrail)

- 일반 사용자 계정으로 관리자 데이터가 노출되는 사례 0건
- 삭제 요청 후 preview playback 또는 job 생성 성공 사례 0건
- 삭제 요청 후 일반 app/admin session으로 삭제 상태 화면이 표시되는 사례 0건
- 외부 공유, 다운로드, public URL 생성 허용 사례 0건
- P0에서 public self-service 비밀번호 재설정/계정 복구가 노출되는 사례 0건
- P0에서 이메일/연락처 변경 기능이 노출되는 사례 0건
- 삭제 요청 취소 또는 삭제 계정 재활성화가 허용되는 사례 0건
- audit log에 비밀번호, token 원문, 음성 내용, 곡 파일 내용이 남는 사례 0건
- push payload에 곡명, 음원 세부정보, 권리 상태, 음성 내용, 내부 reason code가 포함되는 사례 0건

## 리스크 (Risks)

| 리스크 | 영향 | 대응 |
|---|---|---|
| 별도 관리자 계정 운영 부담 | 단일 사용자 시스템치고 로그인/운영이 번거로울 수 있다 | P0에서는 seed/admin script로 관리자 계정을 만들고, 관리자 가입 UI는 만들지 않는다 |
| 수동 활성화 UX 불명확 | 신규 앱 계정 생성 후 사용자가 왜 막혔는지 혼란스러울 수 있다 | 활성화 전 상태 화면은 짧고 일반적인 pending 안내를 제공한다 |
| 최소 관리자 승인 gate 범위 확장 | 사용자 관리 콘솔로 커져 P0 범위가 늘어날 수 있다 | pending 목록, 승인, audit만 포함하고 거절/검색/역할 변경/재활성화는 제외한다 |
| 삭제 후 최소 상태 화면과 token 폐기 충돌 | token을 즉시 폐기하면서 상태 화면을 어떻게 보여줄지 구현 판단이 필요하다 | 제품 요구사항은 "일반 기능 차단, 최소 상태만 허용"으로 고정하고 TRD에서 세션 처리 방식을 결정한다 |
| 권한 차단 메시지 과다 노출 | 내부 reason code, 권리 상태, 운영 사유가 사용자 화면에 드러날 수 있다 | 사용자 메시지와 감사 로그 reason code를 분리한다 |
| push token 범위 확장 | 알림 상세 기능이 계정 PRD로 흘러들어 범위가 커질 수 있다 | 계정 PRD는 token lifecycle만 다루고 상세 알림은 별도 PRD로 분리한다 |
| 비밀번호 재설정 제외로 인한 운영 불편 | 비밀번호 분실 시 사용자가 직접 복구할 수 없다 | P0에서는 관리자용 reset script와 audit, session/token 폐기로 대응한다 |
| 비밀번호 변경과 재설정의 혼동 | 로그인 상태 변경과 public reset이 같은 기능으로 구현될 수 있다 | P0에서는 로그인 상태 비밀번호 변경만 앱 기능으로 허용하고 public reset은 제외한다 |
| 이메일/연락처 변경 제외로 인한 운영 불편 | 계정 연락처가 바뀌면 P0에서 직접 수정할 수 없다 | 후속 계정 설정 또는 개인정보 PRD에서 별도로 다룬다 |
| 동시 로그인 허용에 따른 추적 부담 | 여러 기기 세션이 남아 이상 접근 원인 추적이 어려울 수 있다 | session/device inventory를 P0부터 남긴다 |
| 단일 사용자 전제 고착 | 나중에 다중 사용자 또는 제품화 시 권한 모델 재설계가 필요할 수 있다 | P0에서는 단일 사용자 단순성을 우선하고, 확장 질문은 열린 질문으로 남긴다 |

## 의존성 (Dependencies)

- 인증/계정 기반 시스템: 이메일/비밀번호 가입, 로그인, 세션, 계정 상태를 처리해야 한다.
- 앱 라우팅/접근 제어: 계정 상태와 권한에 따라 앱 기능, 관리자 페이지, 삭제 상태 화면 접근을 분리해야 한다.
- 관리자 계정 생성 운영 절차: P0 관리자 계정은 seed/admin script로 만들 수 있어야 한다.
- 최소 관리자 승인 gate: 관리자 페이지에서 pending 계정 확인과 승인, 승인 audit을 처리해야 한다.
- Playback/job 생성 권한 체크: 삭제/비활성/수동 활성화 전 상태에서 playback과 job 생성이 차단되어야 한다.
- Session/device inventory: 동시 로그인 허용 상태에서 세션과 기기 정보를 기록해야 한다.
- 로그인 상태 비밀번호 변경: `active` 상태 사용자만 자기 비밀번호를 변경하고 audit을 남길 수 있어야 한다.
- 관리자용 password reset script: P0 public reset을 제외하는 대신 운영자가 reset을 수행하고 audit/session 폐기를 남길 수 있어야 한다.
- 계정 설정 경계: 이메일/연락처 변경은 P0에서 제공하지 않도록 UI와 권한 경계가 분리되어야 한다.
- 감사 로그 저장소: 계정, 세션, 권한, 삭제, token 이벤트를 추적해야 한다.
- Push token 저장/상태 관리: token 등록, 해제, 비활성화 상태를 계정과 연결해야 한다.
- `Privacy & Data Retention Policy`: 계정 정보, 알림 연락처, 삭제, backup retention의 정본 정책이 필요하다.
- 후속 PRD: 곡 업로드, 앱 입력, 처리/preview, 삭제/감사, 알림 PRD가 이 문서의 권한 경계를 참조해야 한다.

## 가정 (Assumptions)

- 실제 사용자는 한 명이며 일반 사용자 계정과 관리자 계정을 모두 소유한다.
- P0에서는 일반 사용자 계정과 관리자 계정을 별도 계정으로 유지하는 편이 감사와 접근 경계를 더 선명하게 만든다.
- P0 이메일/비밀번호 계정은 이메일 인증 없이 관리자 승인 기반 수동 활성화로 시작해도 개인 실험 시스템 검증에는 충분하다.
- 관리자 계정은 seed/admin script로 생성하는 방식이 P0 범위에 적합하다.
- 수동 활성화를 관리자 페이지에서 처리하더라도 P0에서는 최소 승인 화면만 있으면 충분하다.
- 삭제 요청 후 최소 상태 화면은 사용자가 삭제 접수/진행 상태만 확인하기 위한 화면이며, 일반 기능 접근을 되살리는 예외가 아니다.
- 삭제 요청 성공 응답 이후 다음 API 요청부터 차단하면 P0의 개인 시스템 안전 기준에는 충분하다.
- 삭제 차단 시간 측정 시작점은 삭제 요청 API 성공 응답 시점으로 고정해도 P0 관측 기준에는 충분하다.
- P0에서는 로그인 상태 비밀번호 변경을 허용하되, public reset과 계정 복구는 제외하는 편이 범위 관리에 적합하다.
- P0에서는 public self-service 비밀번호 재설정/계정 복구보다 관리자용 reset script가 범위 관리에 적합하다.
- P0에서는 이메일/연락처 변경을 제외하고 후속 계정 설정 또는 개인정보 PRD에서 별도 정의해도 된다.
- P0에서는 삭제 취소/재활성화를 허용하지 않는 편이 삭제 상태 전이와 데이터 보관 정책을 단순하게 만든다.
- 동시 로그인은 허용하지만 session/device inventory를 남기면 후속 보안/운영 화면으로 확장할 수 있다.
- push token lifecycle만 계정 PRD에 포함하고 알림 상세는 별도 PRD로 분리하면 P0 계정 범위를 관리할 수 있다.

## 열린 질문 (Open Questions)

제품 정책 질문은 대부분 결정되었다.

결정 사항:

- 일반 사용자 계정의 수동 활성화는 관리자 페이지에서 관리자 계정 사용자가 승인하는 방식으로 처리한다.
- 관리자 승인 기능은 별도 Admin PRD 선행 의존성이 아니라 이 PRD의 P0 최소 admin gate로 포함한다.
- 최소 admin gate는 관리자 계정 로그인 후 접근 가능한 단일 `계정 승인 Gate (Account Approval Gate)` 화면으로 시작한다.
- P0 이후 인증 확장은 passkey를 소셜 로그인보다 먼저 검토한다.
- 삭제 요청 성공 응답 이후 다음 API 요청부터 login/playback/job/admin 접근을 차단한다.
- 삭제 차단 시간 측정 시작점은 삭제 요청 API 성공 응답 시점으로 고정한다.
- 삭제 요청 후 auth/gateway 차단 반영 목표는 1초 이내, 운영 관측 P99는 5초 이내로 둔다.
- 삭제 요청 후에는 일반 session/token을 폐기하고 삭제 상태 전용 제한 접근만 허용한다.
- P0에서는 로그인 상태 비밀번호 변경을 허용한다.
- 비밀번호 변경 성공 후 현재 session은 유지하고, 동일 계정의 다른 active session과 refresh token은 폐기한다.
- P0 public self-service 비밀번호 재설정/계정 복구는 제외하고 관리자용 password reset script만 허용한다.
- P0에서는 이메일/연락처 변경을 제외한다.
- 삭제 요청 취소와 삭제 계정 재활성화는 P0에서 제외한다.
- 동시 로그인은 허용하되 session/device inventory를 필수로 남긴다.
- 삭제 차단 timestamp의 역할별 정본은 API Gateway, User Service, Auth Service로 분리해 TRD에서 구체화한다.
- device id는 앱 생성 `client_device_id`와 서버 등록 `device_record_id` 조합으로 관리하는 방향을 TRD 입력값으로 사용한다.
- 계정 상태명, session/token 폐기 방식, audit reason code는 이 문서의 권장안을 TRD 입력값으로 사용한다.

TRD에서 남은 구체화 질문:

- access token이 stateless JWT일 경우 즉시 차단을 `token_version`, introspection, denylist, server-side session 중 무엇으로 보장할 것인가?
- 비밀번호 변경 후 현재 session은 유지하고 동일 계정의 다른 active session과 refresh token은 폐기한다. TRD에서는 어떤 session/token 저장소, revoke event, audit event로 이를 보장할 것인가?
- 삭제 차단 시간 측정 시작 정본은 API Gateway의 `deletion_response_sent_at`으로 둔다. TRD에서는 `deletion_requested_at`은 User Service, `token_revoked_at`은 Auth Service, `first_protected_api_blocked_at`은 Gateway 또는 Auth Layer 중 어디에 기록할지 어떻게 확정할 것인가?
- 삭제 차단 SLA 계산은 `first_protected_api_blocked_at - deletion_response_sent_at`으로 둔다. TRD에서는 각 service timestamp의 clock skew, request id, correlation id를 어떻게 맞출 것인가?
- device id는 앱 생성 `client_device_id`와 서버 등록 `device_record_id` 조합으로 둔다. TRD에서는 client id 생성/회전, server record 발급, 동일 기기 재등록, 앱 재설치 시 처리 기준을 어떻게 정의할 것인가?
- `session_id_hash`, `last_ip_hash`, `device_name`은 개인정보와 추적 정보가 될 수 있다. TRD에서는 hash 방식, 보관 기간, 마스킹, audit 노출 범위를 어떻게 정의할 것인가?
- audit reason code의 최종 enum 이름과 event payload field를 어떤 코드 표준으로 고정할 것인가?

## 다음 추천 스킬 (Recommended Next Skill)

다음은 `prd-reviewer`가 적합하다.

이 PRD는 계정/권한/삭제 차단/감사 기준을 정의했지만, solution smuggling, metric gap, scope creep, missing non-goals, untestable acceptance criteria가 남아 있는지 강하게 검토해야 한다. 리뷰를 통과하거나 수정한 뒤 `feature-definer`로 기능 정의서를 작성하는 흐름이 적합하다.

## 변경 이력 (Change Log)

| 버전 | 날짜 | 변경 내용 |
|---|---|---|
| v0.1 | 2026-06-19 | `06_vox2vocal-account-permission-prd-brief.md` v0.3을 기반으로 개인 계정 및 권한 PRD 초안 작성 |
| v0.2 | 2026-06-19 | 관리자 승인 기반 수동 활성화, 삭제 상태 전용 제한 context, 계정 상태명, session/token 폐기, audit reason code 가이드 반영 |
| v0.3 | 2026-06-19 | 계정 승인 대기 상태명을 `pending_admin_approval`로 변경하고 삭제 후 상태 화면을 server-side status-only session 방식으로 확정 |
| v0.4 | 2026-06-19 | PRD/ TRD 경계를 재정리하고 최소 admin gate, 삭제 후 차단 SLA, 비밀번호 reset 범위, 삭제 취소 제외, 동시 로그인 기기 추적 기준 반영 |
| v0.5 | 2026-06-19 | 로그인 상태 비밀번호 변경, 이메일/연락처 변경 제외, 계정 승인 Gate 단일 화면, pending 안내 문구, 삭제 차단 측정 이벤트와 TRD 구체화 질문 반영 |
