# Engine Log Storage Policy

이 문서는 `engine-*` 로그가 최종적으로 어디에 저장되는지, 로그 레벨과 종류별로 어떤 저장소를 사용하는지 정의한다.

핵심 원칙은 단순하다.

```txt
일반 운영 로그는 Loki에 저장한다.
보안 신호 로그는 초기에는 Loki에서 분리하고, 확장 시 OpenSearch 또는 SIEM으로 보낸다.
Audit 데이터는 PostgreSQL append-only audit store에 저장한다.
장기 증적은 object storage 또는 WORM archive로 보관한다.
```

## 최종 저장소 결정

MVP 기준 최종 저장소:

| 데이터 | 최종 저장소 | 조회 도구 | 설명 |
| --- | --- | --- | --- |
| operational log | Loki | Grafana | 엔진 상태, dependency, health, 내부 처리 상태 |
| pipeline event log | Loki | Grafana | `job_id`, `trace_id` 기준 파이프라인 진행 추적 |
| quality log | Loki | Grafana | clipping, low volume, low confidence 같은 품질 경고 |
| security signal log | Loki security stream | Grafana, alert | 권한 거부, abuse signal, 비정상 접근 징후 |
| audit event | PostgreSQL append-only table | 제한된 audit query/export | 권한, 동의, 관리자 행위, export/download 증적 |
| audit digest | PostgreSQL 또는 object storage | audit integrity checker | audit event 무결성 검증 |
| cold archive | object storage | 제한된 복원 절차 | 장기 보관용 압축 로그 또는 audit export |

2차 확장 기준:

| 데이터 | 확장 저장소 | 이유 |
| --- | --- | --- |
| security signal log | OpenSearch 또는 SIEM | 보안 검색, 상관분석, 장기 조사 |
| audit event archive | immutable object storage 또는 WORM | 장기 증적, 변조 방지 |
| operational log archive | object storage | 비용 최적화된 장기 보관 |

## 저장 흐름

일반 운영 로그:

```txt
engine-* application
  -> JSON stdout/stderr
  -> Kubernetes container log
  -> Fluent Bit or OpenTelemetry Collector
  -> Loki
  -> Grafana dashboard / alert
```

보안 신호 로그:

```txt
engine-* application
  -> JSON stdout/stderr with log_domain=security
  -> collector routing
  -> Loki security stream
  -> alert
  -> future OpenSearch / SIEM
```

Audit 데이터:

```txt
engine-safety-rights or service requiring audit
  -> audit_writer.write()
  -> PostgreSQL audit_events append-only table
  -> audit_daily_digests
  -> object storage archive
```

중요: `audit_writer`는 일반 logger가 아니다. Audit event는 `stdout`에만 남기고 끝내면 안 된다.

## 로그 레벨별 저장 정책

| Level | 운영 저장소 | 보관 | Alert | 비고 |
| --- | --- | --- | --- | --- |
| `DEBUG` | local only. dev에서는 Loki short retention 가능 | local 또는 3~7일 | no | prod 기본 비활성화 |
| `INFO` | Loki | 14일 | no | 정상 상태 변화, pipeline 진행 |
| `WARN` | Loki | 30일 | 조건부 | 품질 경고, 처리 가능하지만 주의 필요 |
| `ERROR` | Loki | 90일 | 비율 기반 | 작업 실패. `retryable` 필수 |
| `CRITICAL` | Loki + incident record | 90일 이상 | 즉시 | 엔진 지속 불가, 데이터 손상, audit write 실패 |

보안성 `WARN`, `ERROR`, `CRITICAL`은 `log_domain=security`를 붙이고 security stream으로 라우팅한다.

| Security Level | 저장소 | 보관 | Alert |
| --- | --- | --- | --- |
| security `WARN` | Loki security stream | 180일 이상 | 급증 시 |
| security `ERROR` | Loki security stream, future SIEM | 180일 이상 | 조건부 |
| security `CRITICAL` | Loki security stream, future SIEM, incident record | 1년 이상 | 즉시 |

Audit event는 일반 로그 레벨이 아니라 `decision`, `action`, `reason_code` 중심으로 관리한다.

| Audit Decision | 저장소 | 보관 | Alert |
| --- | --- | --- | --- |
| `allowed` | PostgreSQL `audit_events` | 1년 이상 또는 정책 기준 | no |
| `denied` | PostgreSQL `audit_events` | 1년 이상 또는 정책 기준 | 급증 시 |
| `changed` | PostgreSQL `audit_events` | 1년 이상 또는 정책 기준 | 권한/정책 변경은 조건부 |
| `deleted` | PostgreSQL `audit_events` | 1년 이상 또는 정책 기준 | 중요 리소스면 조건부 |
| `viewed` | PostgreSQL `audit_events` | 1년 이상 또는 정책 기준 | audit 조회 급증 시 |

## 로그 종류별 저장 정책

| 로그 종류 | 예시 | 필수 필드 | 최종 저장소 |
| --- | --- | --- | --- |
| development log | 로컬 디버깅 상세값 | 제한 없음. 민감정보 금지 | 중앙 저장 금지 |
| operational log | startup, health, dependency connected | `service`, `event`, `level`, `env` | Loki |
| pipeline event log | `audio.ingest.completed` | `trace_id`, `job_id`, `audio_asset_id` | Loki |
| quality log | `audio.ingest.quality.warned` | `quality_code`, 측정값 | Loki |
| security signal log | `rights.decision.denied`, abuse signal | `actor_id`, `resource_id`, `reason_code` | Loki security stream, future SIEM |
| audit event | `consent.revoked`, `output.downloaded` | `audit_event_id`, `actor_id`, `action`, `decision`, `event_hash` | PostgreSQL append-only |
| audit digest | daily root hash | `date`, `event_count`, `root_hash` | PostgreSQL or object storage |
| metrics | request count, p95 latency | metric name, labels | Prometheus. 로그 저장소 아님 |

## Loki 구성 기준

Loki에는 너무 많은 label을 붙이면 안 된다. 고정 cardinality가 낮은 값만 label로 둔다.

권장 label:

```txt
service
env
level
log_domain
event_family
```

label로 쓰지 않는 값:

```txt
trace_id
job_id
audio_asset_id
user_id
error_message
full object key
```

위 값들은 JSON body에 넣고 query 시 필터링한다.

예시 log stream:

```txt
{service="engine-audio-ingest", env="prod", level="INFO", log_domain="pipeline"}
{service="engine-safety-rights", env="prod", level="WARN", log_domain="security"}
```

## PostgreSQL Audit Store 구성 기준

MVP audit table:

```txt
audit_events
  audit_event_id
  occurred_at
  schema_version
  actor_type
  actor_id
  action
  resource_type
  resource_id
  decision
  reason_code
  policy_version
  consent_record_id
  trace_id
  job_id
  audio_asset_id
  source_service
  event_hash
  prev_hash
  payload_json
```

무결성 digest table:

```txt
audit_daily_digests
  date
  event_count
  first_event_id
  last_event_id
  root_hash
  created_at
```

권한:

| 주체 | 권한 |
| --- | --- |
| application service | insert only |
| developer | direct access 없음 |
| operator | aggregate dashboard only |
| security auditor | read limited |
| security admin | export with approval |

`UPDATE`, `DELETE`는 application 계정에 허용하지 않는다.

## Object Storage Archive

Object storage는 MVP 필수는 아니지만 audit 장기 보관에서 필요해진다.

권장 bucket:

```txt
vox2vocal-audit-archive
vox2vocal-log-archive
```

저장 예시:

```txt
audit/year=2026/month=06/day=06/audit-events.jsonl.gz
audit-digest/year=2026/month=06/day=06/digest.json
logs/year=2026/month=06/day=06/service=engine-audio-ingest/logs.jsonl.gz
```

보관 정책:

- audit archive는 object lock 또는 WORM 구성을 검토한다.
- operational log archive는 비용 최적화 storage class를 사용한다.
- archive export와 restore 행위도 audit한다.

## 저장 금지 위치

운영 로그를 아래 위치에 저장하지 않는다.

- `LOCAL_STORAGE_ROOT`
- audio asset directory
- canonical WAV directory
- manifest directory
- container 내부 임의 파일
- Git repository
- 모델 weight directory

이 경로들은 오디오 자산과 모델 산출물을 위한 위치이지 로그 저장소가 아니다.

## 구현 우선순위

1차 MVP:

```txt
JSON stdout/stderr
Loki
Grafana
PostgreSQL audit_events append-only
audit_daily_digests
```

2차:

```txt
Loki retention 분리
security stream 강화
OpenSearch or SIEM 연동
object storage archive
audit integrity checker
```

3차:

```txt
WORM archive
automated audit export
incident evidence package generator
long-term compliance reporting
```
