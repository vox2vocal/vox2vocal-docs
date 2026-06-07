# Engine Audit Data Guide

이 문서는 Vox2Vocal 엔진 계층에서 audit 데이터를 어떻게 생성, 저장, 보호, 조회, 폐기할지 정의한다.

## Audit 데이터의 목적

Audit 데이터는 장애 분석용 로그가 아니라 증적이다.

답해야 하는 질문:

- 누가 어떤 보이스 자산에 접근했는가?
- 어떤 보이스 모델이 어떤 동의 기록에 근거해 사용되었는가?
- 어떤 정책 버전이 허용 또는 거부 판단을 내렸는가?
- 관리자 또는 시스템이 어떤 변경을 수행했는가?
- 결과물이 언제 export/download 되었는가?
- 침해 또는 분쟁 발생 시 변조되지 않은 기록을 제시할 수 있는가?

## Audit과 운영 로그의 차이

| 항목 | 운영 로그 | Audit 데이터 |
| --- | --- | --- |
| 목적 | 관측성, 장애 분석 | 권한/동의/행위 증명 |
| 변경 가능성 | retention 후 삭제 가능 | append-only |
| 조회 대상 | 개발/운영자 | 제한된 보안/감사 담당 |
| 저장 기간 | 짧음 | 장기 |
| 무결성 | 권장 | 필수 |
| 실패 시 정책 | 대개 계속 처리 | 권한 관련 작업은 fail-closed |

## Audit Event Schema

기본 schema:

```json
{
  "audit_event_id": "audit_123",
  "timestamp": "2026-06-06T01:23:45.123Z",
  "schema_version": "1.0",
  "actor_type": "user",
  "actor_id": "user_123",
  "action": "voice_conversion.requested",
  "resource_type": "voice_model",
  "resource_id": "model_456",
  "decision": "allowed",
  "reason_code": "OWNER_CONSENT_VALID",
  "policy_version": "2026-06-01",
  "consent_record_id": "consent_789",
  "trace_id": "trace_123",
  "job_id": "job_123",
  "audio_asset_id": "aud_123",
  "source_service": "engine-safety-rights",
  "prev_hash": "sha256:...",
  "event_hash": "sha256:..."
}
```

필수 필드:

| 필드 | 설명 |
| --- | --- |
| `audit_event_id` | audit event 고유 ID |
| `timestamp` | UTC timestamp |
| `actor_type` | `user`, `admin`, `service`, `system` |
| `actor_id` | 내부 actor ID |
| `action` | 수행 행위 |
| `resource_type` | 대상 리소스 종류 |
| `resource_id` | 대상 리소스 ID |
| `decision` | `allowed`, `denied`, `changed`, `created`, `deleted`, `viewed` |
| `reason_code` | 판단 이유 |
| `policy_version` | 적용 정책 버전 |
| `trace_id` | 운영 로그와 연결할 추적 ID |
| `source_service` | audit event를 생성한 서비스 |
| `event_hash` | 무결성 검증용 hash |

## Audit Event Catalog

반드시 기록:

### Identity and Access

- `auth.login.succeeded`
- `auth.login.failed`
- `auth.logout`
- `role.granted`
- `role.revoked`
- `admin.session.started`
- `admin.session.ended`

### Audio Asset

- `audio.upload.requested`
- `audio.upload.completed`
- `audio.asset.deleted`
- `audio.asset.accessed`
- `audio.asset.exported`

### Consent

- `consent.created`
- `consent.updated`
- `consent.revoked`
- `consent.expired`
- `consent.checked`

### Voice Model

- `voice_model.created`
- `voice_model.trained`
- `voice_model.version.promoted`
- `voice_model.deleted`
- `voice_model.accessed`
- `voice_model.ownership.verified`

### Safety Rights

- `rights.check.requested`
- `rights.decision.allowed`
- `rights.decision.denied`
- `rights.policy.changed`
- `rights.policy.rollback`

### Engine Job

- `engine.job.started`
- `engine.job.completed`
- `engine.job.failed`
- `engine.job.retried`
- `engine.job.cancelled`

### Export and Distribution

- `output.preview.generated`
- `output.downloaded`
- `output.exported`
- `output.shared`
- `distribution.intent.declared`

### Audit System

- `audit.event.written`
- `audit.write.failed`
- `audit.event.viewed`
- `audit.export.generated`
- `audit.retention.changed`
- `audit.integrity.verified`
- `audit.integrity.failed`

## 저장 방식

권장 구조:

```txt
engine or service
  -> audit writer
  -> append-only audit store
  -> integrity digest chain
  -> cold archive
  -> controlled audit export
```

초기 구현:

- PostgreSQL append-only table 또는 object storage append log
- application update/delete 금지
- DB 권한으로 update/delete 제한
- 일 단위 digest 생성
- digest를 별도 위치에 복제

고도화:

- WORM storage
- KMS 기반 암호화
- hash chain
- SIEM 연동
- immutable backup

## 무결성

각 audit event는 canonical JSON으로 직렬화한 뒤 hash를 만든다.

```txt
event_hash = sha256(canonical_audit_event_without_event_hash)
```

연속성 검증이 필요한 경우:

```txt
event_hash = sha256(prev_hash + canonical_audit_event)
```

일 단위로 digest summary를 만든다.

```json
{
  "date": "2026-06-06",
  "event_count": 123456,
  "first_event_id": "audit_001",
  "last_event_id": "audit_999",
  "root_hash": "sha256:..."
}
```

## 접근 통제

Audit 저장소 접근은 최소 권한으로 제한한다.

| 역할 | 권한 |
| --- | --- |
| application service | append only |
| security auditor | read limited |
| security admin | export with approval |
| developer | direct access 없음 |
| operator | dashboard aggregate only |

중요: audit event 조회 자체도 `audit.event.viewed`로 기록한다.

## 보관과 삭제

보이스 시스템은 삭제 요구와 증적 보관 의무가 충돌할 수 있다.

원칙:

- 원본 음성 파일 삭제와 audit event 삭제를 분리한다.
- audit event에는 원본 데이터 대신 내부 ID와 digest를 저장한다.
- 사용자 삭제 요청 시 직접 식별 가능한 필드는 anonymization 또는 tombstone 처리한다.
- 법적/계약상 보관이 필요한 audit event는 retention policy에 따라 보관한다.

## Fail-Closed 기준

아래 audit event를 저장하지 못하면 작업을 중단한다.

- consent 생성/철회
- rights decision allowed/denied
- voice model ownership 변경
- 관리자 권한 변경
- output export/download
- audit retention 변경
- policy 변경

아래 event는 운영 상황에 따라 retry queue로 보낼 수 있다.

- 일반 engine job started/completed
- 품질 분석 결과
- 내부 preview 생성

## Audit 구현 체크리스트

- audit writer는 일반 logger와 분리되어 있는가?
- audit event schema version이 있는가?
- 권한 판단에는 `policy_version`과 `reason_code`가 있는가?
- consent 기반 판단에는 `consent_record_id`가 있는가?
- voice model 사용에는 `voice_model_id`, `model_version`이 있는가?
- audit event 저장 실패가 감지되고 알림되는가?
- audit 조회 행위도 audit되는가?
- 무결성 digest를 정기 생성하는가?
- retention 정책이 문서화되어 있는가?
