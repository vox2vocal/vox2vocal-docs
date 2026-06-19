# 개인 계정 및 권한 PRD (Account and Permission PRD)

문서 번호: 07
문서 버전: v0.3
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
- 일반 사용자 계정과 관리자 계정을 별도 계정으로 분리한다.
- 관리자 계정은 제한된 가입 화면이 아니라 seed/admin script로 생성한다.
- 일반 사용자 계정은 앱 기능에 접근할 수 있지만 관리자 페이지 접근은 차단한다.
- 관리자 계정은 관리자 페이지와 앱 모두에 접근할 수 있게 한다.
- 계정 삭제 요청 후 로그인, playback, job 생성, 관리자 접근을 즉시 차단한다.
- 삭제 요청 후에는 삭제 접수/진행 상태를 보여주는 최소 상태 화면만 허용한다.
- 권한 차단 상세 사유는 사용자에게 노출하지 않고 감사 로그에 남긴다.
- push token은 계정 PRD에서 등록/해제와 권한 상태만 정의하고, 상세 알림 정책은 별도 PRD로 분리한다.
- 후속 PRD가 참조할 actor, account state, permission boundary, audit event 기준을 제공한다.

## 비목표 (Non-goals)

- 곡 업로드, 곡 metadata, 구간 정의, 권리/출처 등록부를 상세 정의하지 않는다.
- 본인 음성 녹음/업로드, preview 생성, playback 품질, pitch/note 분석, 발성 참고 정보, self-voice 평가를 정의하지 않는다.
- passkey, 소셜 로그인, MFA, 조직 SSO를 P0 필수 구현 범위로 확정하지 않는다.
- 관리자 가입 화면을 만들지 않는다.
- 정교한 사용자 관리 콘솔, 사용자 초대, 권한 위임, 조직 관리 기능을 만들지 않는다.
- 강사/교육생, 조직, 팀, 다중 tenant 권한 모델을 만들지 않는다.
- 알림 템플릿, retry, in-app inbox, fallback 발송 정책을 상세 정의하지 않는다.
- API schema, DB schema, token 저장 방식, auth provider, session 구현 방식을 결정하지 않는다.
- 외부 공유, 다운로드, public URL, 수익화, 배포, 모델 학습 사용을 허용하지 않는다.

## 범위 (Scope)

### 포함 범위 (In Scope)

- 이메일/비밀번호 기반 P0 계정 생성과 로그인 요구사항
- 수동 활성화 상태와 활성화 전 접근 제한
- 관리자 페이지에서의 최소 계정 승인 요구사항
- 일반 사용자 계정과 관리자 계정 분리
- 관리자 계정의 seed/admin script 생성 원칙
- 앱 접근, 관리자 접근, 운영 범위의 권한 경계
- 계정 상태별 허용/차단 행동
- 삭제 요청 후 즉시 차단과 최소 삭제 상태 화면
- 권한 차단 사용자 메시지와 감사 로그 분리
- push token 등록/해제 lifecycle의 최소 요구사항
- 계정/세션/권한/삭제 관련 기본 audit event

### 제외 범위 (Out of Scope)

- 곡, 구간, 음성, preview, 분석 결과의 상세 데이터 모델
- 내부 playback 조건과 signed playback URL 정책
- 삭제 대상 데이터별 retention 수치의 정본 관리
- 데이터 인벤토리, 개인정보 보관 정책, 운영 runbook 상세
- 관리자 곡 업로드 화면과 운영 모니터링 화면 상세
- P1 이후 A/B 비교, 알림함, push 발송 자동화, passkey/social login 확장

## 결정된 기준 및 TRD 입력 가이드 (Resolved Decisions and TRD Guidance)

### 수동 활성화 방식 (Manual Activation Method)

P0 수동 활성화는 관리자 페이지에서 관리자 계정 사용자가 승인하는 방식으로 진행한다.

관리자 계정은 seed/admin script로 먼저 생성하고, 일반 사용자 계정은 가입 후 `pending_admin_approval` 상태로 시작한다. 관리자는 관리자 페이지의 최소 승인 화면에서 해당 계정을 승인해 `active` 상태로 전환한다.

P0 승인 화면은 계정 승인에 필요한 최소 기능만 가진다.

- pending 계정 목록 확인
- 계정 승인
- 계정 거절 또는 비활성 처리는 P0에서 필수로 두지 않고 필요 시 후속 범위로 둔다
- 승인 actor, target account, result, timestamp, reason code audit 기록

### 삭제 상태 화면과 Token 폐기 가이드 (Deletion Status and Token Revocation Guidance)

삭제 요청 후에는 일반 로그인 세션, 앱 세션, 관리자 세션, refresh token, push token을 일반 기능 접근 용도로 계속 사용하면 안 된다. 삭제 상태 화면이 필요하더라도 일반 세션을 살리는 방식이 아니라 **server-side status-only session**으로 분리한다.

권장 흐름은 다음이다.

1. 사용자가 삭제를 요청한다.
2. 서버가 계정 상태를 `deletion_requested`로 바꾼다.
3. 서버가 기존 app/admin session과 refresh token을 즉시 폐기한다.
4. 서버가 playback, job 생성, admin 접근, 일반 앱 API를 계정 상태 기준으로 차단한다.
5. 삭제 요청 응답에서 server-side status-only session을 만들어 최소 상태 화면만 허용한다.
6. status-only session은 삭제 접수/진행 상태 조회만 가능하고 refresh, playback, job 생성, admin API 호출 권한을 갖지 않는다.
7. status-only session이 없거나 만료되면 사용자는 일반 기능으로 돌아가지 못하고, 짧고 일반적인 접근 제한 메시지만 본다.

P0에서는 다음 방식을 확정한다.

- server-side session store 기반 status-only session

짧은 TTL의 status-only signed token이나 삭제 요청 직후 1회성 status receipt는 P0 기본안으로 쓰지 않는다. server-side status-only session은 즉시 폐기, 권한 축소, 감사 추적이 가장 단순하고, stateless token revoke 문제를 줄일 수 있기 때문이다.

### 계정 상태명 권장안 (Recommended Account States)

| 상태 | 의미 | 허용 행동 |
|---|---|---|
| `pending_admin_approval` | 일반 사용자 계정 생성 후 관리자 승인 대기 | pending 안내 확인만 허용 |
| `active` | 정상 사용 가능 | 권한 범위 내 앱 접근 허용 |
| `disabled` | 보안, 운영, 수동 차단 상태 | 로그인, playback, job 생성, admin 접근 차단 |
| `deletion_requested` | 삭제 요청 접수 후 즉시 차단 상태 | 삭제 상태 전용 최소 화면만 허용 |
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
- 개인 계정 소유자로서, 나는 내 계정이 아직 활성화되지 않았을 때 일반 기능에 접근하지 못하고 현재 상태를 이해할 수 있는 최소 안내를 보고 싶다.
- 앱 사용자로서, 나는 일반 사용자 계정으로 앱 기능을 사용할 수 있지만 관리자 페이지에는 접근할 수 없기를 원한다.
- 관리자 계정 사용자로서, 나는 별도 관리자 계정으로 관리자 페이지에 접근하고, 같은 관리자 계정으로 앱도 확인할 수 있기를 원한다.
- 실험 운영자로서, 나는 권한 없는 접근, 관리자 접근 실패, 삭제 요청, token 폐기 같은 이벤트가 나중에 원인을 확인할 수 있도록 감사 로그에 남기를 원한다.
- 개인 계정 소유자로서, 나는 계정 삭제를 요청하면 로그인, playback, job 생성, 관리자 접근이 즉시 막히고, 삭제 접수/진행 상태만 확인할 수 있기를 원한다.
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
| APR-FR-006 | 수동 활성화 전 사용자는 짧고 일반적인 pending 안내를 볼 수 있어야 한다. 내부 활성화 사유, 운영 로그, 권한 reason code는 노출하지 않는다. |
| APR-FR-007 | 활성화된 일반 사용자 계정은 앱 기능에 접근할 수 있다. |
| APR-FR-008 | 관리자 계정 사용자는 관리자 페이지에서 수동 활성화 대기 계정을 승인할 수 있어야 한다. |
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

### 삭제 요청 및 접근 차단 (Deletion Request and Access Blocking)

| ID | 요구사항 |
|---|---|
| APR-FR-030 | 계정 삭제 요청이 접수되면 계정은 즉시 삭제 요청 또는 비활성 상태로 전환되어야 한다. |
| APR-FR-031 | 삭제 요청 후 기존 session/token은 폐기되어야 한다. |
| APR-FR-032 | 삭제 요청 후 로그인, playback, job 생성, 관리자 접근은 모두 즉시 차단되어야 한다. |
| APR-FR-033 | 삭제 요청 직후 일반 기능 접근은 모두 막되, 삭제 접수/진행 상태를 보여주는 최소 상태 화면만 허용한다. |
| APR-FR-034 | 삭제 상태 화면은 삭제 접수 여부, 처리 중 상태, 일반 기능 차단 사실만 보여준다. |
| APR-FR-035 | 삭제 상태 화면은 playback, job 생성, 관리자 페이지, 일반 앱 기능으로 이동할 수 없어야 한다. |
| APR-FR-036 | 삭제 상태 화면은 내부 삭제 사유, 데이터 위치, audit reason code, 권리/출처 세부정보를 노출하지 않는다. |
| APR-FR-037 | 삭제 상태 화면은 일반 app/admin session이 아니라 server-side status-only session에서만 표시되어야 한다. |
| APR-FR-038 | server-side status-only session은 삭제 상태 조회 외의 API 권한을 갖지 않는다. |

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

### 후속 PRD 참조 기준 (Downstream PRD Reference)

| ID | 요구사항 |
|---|---|
| APR-FR-060 | 곡 업로드/구간 관리 PRD는 관리자 계정만 곡 metadata, 출처/권리 메모, 구간 정의를 변경할 수 있다는 기준을 참조해야 한다. |
| APR-FR-061 | 앱 입력/preview PRD는 일반 사용자 계정과 관리자 계정 모두 앱 접근이 가능하되, 삭제/비활성 상태에서는 playback과 job 생성이 차단된다는 기준을 참조해야 한다. |
| APR-FR-062 | 삭제/보관/감사 PRD는 이 PRD의 삭제 요청 후 즉시 차단 범위와 최소 상태 화면 기준을 참조해야 한다. |
| APR-FR-063 | 알림 PRD는 이 PRD의 push token 최소 lifecycle과 민감정보 payload 금지 기준을 참조해야 한다. |

## 인수 조건 (Acceptance Criteria)

| ID | Given | When | Then |
|---|---|---|---|
| APR-AC-001 | 신규 사용자가 이메일/비밀번호로 계정을 생성했다 | 계정 생성이 완료된다 | 계정은 관리자 승인 대기 상태로 표시되고 일반 앱 기능은 열리지 않는다 |
| APR-AC-002 | 계정이 수동 활성화 전 상태다 | 사용자가 앱 기능, playback, job 생성을 시도한다 | 기능은 차단되고 짧고 일반적인 pending 안내가 표시된다 |
| APR-AC-003 | 계정이 수동 활성화 전 상태다 | 사용자가 관리자 페이지 접근을 시도한다 | 접근은 차단되고 상세 reason code는 사용자에게 노출되지 않는다 |
| APR-AC-004 | 일반 사용자 계정이 활성 상태다 | 사용자가 앱에 로그인한다 | 앱 기본 기능 접근이 허용된다 |
| APR-AC-005 | 일반 사용자 계정이 활성 상태다 | 사용자가 관리자 페이지 URL 또는 관리자 기능에 접근한다 | 접근은 차단되고 관리자 데이터는 표시되지 않는다 |
| APR-AC-006 | 관리자 계정이 존재한다 | 사용자가 관리자 계정으로 관리자 페이지에 로그인한다 | 관리자 페이지 접근이 허용된다 |
| APR-AC-007 | 관리자 계정이 존재한다 | 사용자가 관리자 계정으로 앱에 접근한다 | 앱 접근이 허용된다 |
| APR-AC-008 | 관리자 계정이 앱에 접근했다 | 사용자가 외부 공유, 다운로드, public URL, 수익화, 배포, 모델 학습 사용을 시도한다 | 해당 행동은 권한으로 허용되지 않는다 |
| APR-AC-009 | 계정이 활성 상태다 | 사용자가 삭제를 요청한다 | 삭제 요청 또는 비활성 상태로 즉시 전환되고 기존 session/token 폐기가 시작된다 |
| APR-AC-010 | 삭제 요청이 접수된 계정이다 | 사용자가 로그인, playback, job 생성, 관리자 접근을 시도한다 | 모든 행동이 즉시 차단된다 |
| APR-AC-011 | 삭제 요청이 접수된 직후다 | 사용자가 앱 화면을 확인한다 | 삭제 접수/진행 상태와 일반 기능 차단 사실만 담은 최소 상태 화면이 표시될 수 있다 |
| APR-AC-012 | 삭제 상태 화면이 표시된다 | 사용자가 playback, job 생성, 관리자 페이지, 일반 앱 기능으로 이동하려 한다 | 이동은 허용되지 않는다 |
| APR-AC-013 | 삭제 상태 화면이 표시된다 | 화면 내용을 확인한다 | 내부 삭제 사유, 데이터 위치, audit reason code, 권리/출처 세부정보가 표시되지 않는다 |
| APR-AC-014 | 권한 없는 접근이 발생했다 | 사용자 화면과 감사 로그가 생성된다 | 사용자는 일반 메시지만 보고, 감사 로그에는 추적 가능한 상세 reason code가 남는다 |
| APR-AC-015 | push token이 등록되어 있다 | 사용자가 로그아웃, 수신 거부, 앱 삭제 감지, 삭제 요청 중 하나의 상태가 된다 | push token은 비활성화 또는 해제 대상으로 처리된다 |
| APR-AC-016 | push payload가 생성된다 | payload 내용을 검토한다 | 곡명, 음원 세부정보, 권리 상태, 음성 내용, 내부 reason code가 포함되지 않는다 |
| APR-AC-017 | 로그인, 관리자 접근, 권한 차단, 삭제 요청, token 폐기 이벤트가 발생한다 | audit 기록을 확인한다 | 이벤트 종류, actor 범위, 결과, 시간이 추적 가능하며 비밀번호/token 원문/음성 내용은 남지 않는다 |
| APR-AC-018 | 일반 사용자 계정이 관리자 승인 대기 상태다 | 관리자 계정 사용자가 관리자 페이지에서 승인한다 | 계정은 활성 상태가 되고 승인 이벤트가 감사 로그에 남는다 |
| APR-AC-019 | server-side status-only session이 발급되어 있다 | 사용자가 삭제 상태 조회 외 API를 호출한다 | 요청은 차단되고 일반 app/admin 권한으로 승격되지 않는다 |

## 성공 지표 (Success Metrics)

### 기준선 (Baseline)

- 신규 제품이므로 실제 운영 기준선은 없다.
- 현재 기준선은 문서상 결정 사항과 수동 QA 결과로 시작한다.
- 계정 생성 전환율, 로그인 성공률, 권한 차단율, 삭제 요청 처리율, push token 등록률의 실측 baseline은 P0 구현 후 수집해야 한다.

### 목표치 (Target)

| 지표 | P0 목표 |
|---|---|
| 관리자 승인 후 일반 사용자 계정 활성화 성공률 | QA 시나리오 기준 100% |
| 일반 사용자 계정의 관리자 접근 차단율 | QA 시나리오 기준 100% |
| 관리자 계정의 관리자 페이지 접근 성공률 | 정상 계정/정상 세션 QA 시나리오 기준 100% |
| 관리자 계정의 앱 접근 성공률 | 정상 계정/정상 세션 QA 시나리오 기준 100% |
| 삭제 요청 후 login/playback/job/admin 접근 차단율 | QA 시나리오 기준 100% |
| server-side status-only session의 권한 격리 | QA 시나리오 기준 100% |
| 권한 차단 상세 사유의 사용자 화면 비노출 | QA 시나리오 기준 100% |
| 계정/세션/권한/삭제 핵심 이벤트 audit 기록률 | QA 시나리오 기준 100% |
| push payload 민감정보 미포함 | QA 시나리오 기준 100% |

### 보호 지표 (Guardrail)

- 일반 사용자 계정으로 관리자 데이터가 노출되는 사례 0건
- 삭제 요청 후 preview playback 또는 job 생성 성공 사례 0건
- 외부 공유, 다운로드, public URL 생성 허용 사례 0건
- audit log에 비밀번호, token 원문, 음성 내용, 곡 파일 내용이 남는 사례 0건
- push payload에 곡명, 음원 세부정보, 권리 상태, 음성 내용, 내부 reason code가 포함되는 사례 0건

## 리스크 (Risks)

| 리스크 | 영향 | 대응 |
|---|---|---|
| 별도 관리자 계정 운영 부담 | 단일 사용자 시스템치고 로그인/운영이 번거로울 수 있다 | P0에서는 seed/admin script로 관리자 계정을 만들고, 관리자 가입 UI는 만들지 않는다 |
| 수동 활성화 UX 불명확 | 신규 앱 계정 생성 후 사용자가 왜 막혔는지 혼란스러울 수 있다 | 활성화 전 상태 화면은 짧고 일반적인 pending 안내를 제공한다 |
| 삭제 후 최소 상태 화면과 token 폐기 충돌 | token을 즉시 폐기하면서 상태 화면을 어떻게 보여줄지 구현 판단이 필요하다 | 제품 요구사항은 "일반 기능 차단, 최소 상태만 허용"으로 고정하고 TRD에서 세션 처리 방식을 결정한다 |
| 권한 차단 메시지 과다 노출 | 내부 reason code, 권리 상태, 운영 사유가 사용자 화면에 드러날 수 있다 | 사용자 메시지와 감사 로그 reason code를 분리한다 |
| push token 범위 확장 | 알림 상세 기능이 계정 PRD로 흘러들어 범위가 커질 수 있다 | 계정 PRD는 token lifecycle만 다루고 상세 알림은 별도 PRD로 분리한다 |
| 단일 사용자 전제 고착 | 나중에 다중 사용자 또는 제품화 시 권한 모델 재설계가 필요할 수 있다 | P0에서는 단일 사용자 단순성을 우선하고, 확장 질문은 열린 질문으로 남긴다 |

## 의존성 (Dependencies)

- 인증/계정 기반 시스템: 이메일/비밀번호 가입, 로그인, 세션, 계정 상태를 처리해야 한다.
- 앱 라우팅/접근 제어: 계정 상태와 권한에 따라 앱 기능, 관리자 페이지, 삭제 상태 화면 접근을 분리해야 한다.
- 관리자 계정 생성 운영 절차: P0 관리자 계정은 seed/admin script로 만들 수 있어야 한다.
- Playback/job 생성 권한 체크: 삭제/비활성/수동 활성화 전 상태에서 playback과 job 생성이 차단되어야 한다.
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
- push token lifecycle만 계정 PRD에 포함하고 알림 상세는 별도 PRD로 분리하면 P0 계정 범위를 관리할 수 있다.

## 열린 질문 (Open Questions)

제품 정책 질문은 대부분 결정되었다.

결정 사항:

- 일반 사용자 계정의 수동 활성화는 관리자 페이지에서 관리자 계정 사용자가 승인하는 방식으로 처리한다.
- P0 이후 인증 확장은 passkey를 소셜 로그인보다 먼저 검토한다.
- 삭제 요청 후에는 일반 session/token을 폐기하고 server-side status-only session만 허용하는 방향을 TRD 기본 가이드로 둔다.
- 계정 상태명, session/token 폐기 방식, audit reason code는 이 문서의 권장안을 TRD 입력값으로 사용한다.

TRD에서 남은 구체화 질문:

- access token이 stateless JWT일 경우 즉시 차단을 `token_version`, introspection, denylist, server-side session 중 무엇으로 보장할 것인가?
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
