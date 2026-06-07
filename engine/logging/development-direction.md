# Engine Logging Development Direction

이 문서는 `engine-*` 로그 기능을 어떤 개발 스택과 아키텍처로 구현할지 정의한다.

목표는 각 엔진이 독립 저장소로 개발되더라도, 운영에서는 하나의 파이프라인처럼 추적되고, 보안감사에서는 하나의 증적 체계처럼 설명될 수 있게 만드는 것이다.

## 개발 방향 요약

`engine-*` 로그 개발은 아래 순서로 진행한다.

```txt
1. 공통 log schema 확정
2. 엔진별 observability module 추가
3. JSON stdout logger 적용
4. trace_id / span_id / job_id 전파
5. NATS message header에 trace context 주입
6. redaction filter 적용
7. 실패 로그 error_code / retryable 표준화
8. audit writer 분리
9. 로그 schema test 추가
10. collector / storage / dashboard 연동
```

초기 MVP에서는 `engine-audio-ingest`에 먼저 적용하고, 검증된 패턴을 `engine-voice-analysis`, `engine-voice-pitch`로 복제한다.

## 저장 위치 책임

엔진 개발자가 가장 먼저 알아야 하는 기준은 로그 저장 위치다.

엔진 애플리케이션은 운영 로그를 직접 파일로 저장하지 않는다.

```txt
engine-* process
  -> JSON log to stdout/stderr
  -> Kubernetes container log
  -> logging collector
  -> operational log store
```

따라서 엔진 코드가 직접 쓰면 안 되는 위치:

- `LOCAL_STORAGE_ROOT`
- audio asset directory
- canonical WAV directory
- manifest directory
- container 내부 임의 log file
- application DB table for 일반 운영 로그

예외는 audit event다. Audit event는 일반 logger가 아니라 `audit_writer`를 통해 별도 audit store에 기록한다.

```txt
rights / consent / admin / export action
  -> audit_writer
  -> append-only audit store
  -> integrity digest / archive
```

초기 저장소 기준:

| 데이터 | 개발자 관점 출력 | 실제 저장 담당 |
| --- | --- | --- |
| operational log | JSON stdout/stderr | logging collector + log store |
| pipeline event log | JSON stdout/stderr | logging collector + log store |
| quality warning log | JSON stdout/stderr | logging collector + log store |
| security signal log | JSON stdout/stderr, 별도 label | security log store 또는 SIEM |
| audit event | `audit_writer.write()` | append-only audit store |

## 권장 개발 스택

Python 엔진 기준 MVP 스택:

| 영역 | 권장 스택 | 이유 |
| --- | --- | --- |
| structured logging | Python `logging` + JSON formatter 또는 `structlog` | 표준 logging과 structured field를 같이 사용 |
| trace/log correlation | OpenTelemetry Python | `trace_id`, `span_id`, metric, trace 연계 |
| context propagation | W3C `traceparent`, NATS headers | 엔진 간 trace 유지 |
| schema validation | Pydantic 또는 JSON Schema | 로그/audit 필수 필드 검증 |
| redaction | 공통 redaction filter | PII, token, full lyrics, signed URL 차단 |
| audit | 별도 `audit_writer` module | 운영 로그와 증적 데이터 분리 |
| tests | `pytest`, `caplog` 또는 stdout capture | 로그 필드와 redaction 검증 |
| collector 연동 | OpenTelemetry Collector 또는 Fluent Bit | stdout/OTLP 수집 |
| 운영 저장소 | Loki 우선, 필요 시 OpenSearch | 로그 검색과 Grafana 연동 |
| audit 저장소 | PostgreSQL append-only 또는 object storage append log | 증적, 무결성, 장기 보관 |

OpenTelemetry는 필수 기능을 모두 처음부터 쓰기보다, 다음 순서로 도입한다.

```txt
phase 1: trace_id / span_id 필드만 log schema에 포함
phase 2: OpenTelemetry tracer와 span 적용
phase 3: OTLP exporter 또는 collector 연동
phase 4: trace, metric, log dashboard 통합
```

## 엔진 내부 아키텍처

각 Python 엔진은 아래 모듈 구조를 권장한다.

```txt
src/<engine_package>/
  observability/
    __init__.py
    context.py
    logger.py
    redaction.py
    schemas.py
    audit_writer.py
    tracing.py
    metrics.py
```

모듈 책임:

| 모듈 | 책임 |
| --- | --- |
| `context.py` | `trace_id`, `span_id`, `job_id`, `audio_asset_id`, `user_id` 보관과 전파 |
| `logger.py` | JSON structured logger, level helper, stdout/stderr 출력 |
| `redaction.py` | 민감정보 제거, masking, allowlist/denylist 관리 |
| `schemas.py` | operational log schema, audit event schema |
| `audit_writer.py` | audit event 생성, append-only 저장 adapter 호출 |
| `tracing.py` | OpenTelemetry tracer 설정, span 생성 |
| `metrics.py` | 처리 시간, 실패율, 품질 경고 count |

비즈니스 코드는 직접 `print()`나 raw logger를 사용하지 않고, observability module을 통해서만 로그를 남긴다.

## Context 객체

엔진 내부에서는 dict를 계속 넘기기보다 context 객체를 사용한다.

권장 필드:

```txt
trace_id
span_id
parent_span_id
job_id
audio_asset_id
user_id
request_id
idempotency_key
engine_name
engine_version
schema_version
```

NATS message를 수신하면:

```txt
1. payload에서 job_id, audio_asset_id, user_id를 읽는다.
2. headers에서 traceparent를 읽는다.
3. traceparent가 없으면 새 trace_id를 만든다.
4. 현재 엔진용 span_id를 만든다.
5. 이후 모든 log, metric, output event에 같은 context를 붙인다.
```

NATS message를 발행하면:

```txt
1. payload에 trace_id, parent_span_id, job_id, audio_asset_id, user_id를 포함한다.
2. header에 traceparent를 주입한다.
3. event 발행 성공/실패를 로그로 남긴다.
```

## Logger 인터페이스

각 엔진은 아래 수준의 공통 인터페이스를 제공한다.

```python
log.info_event(
    event="audio.ingest.completed",
    context=context,
    duration_ms=duration_ms,
    sample_rate=48000,
    channels=1,
)
```

```python
log.warn_event(
    event="audio.ingest.quality.warned",
    context=context,
    warning_code="LOW_VOLUME",
    peak=0.02,
    rms=0.003,
)
```

```python
log.error_event(
    event="audio.ingest.failed",
    context=context,
    error_code="FFMPEG_TIMEOUT",
    error_type="TimeoutError",
    retryable=True,
    duration_ms=600000,
)
```

이 인터페이스 내부에서 처리해야 하는 것:

- 공통 필드 자동 주입
- UTC timestamp 생성
- `schema_version` 자동 주입
- redaction filter 적용
- JSON 직렬화
- stdout/stderr routing

## Audit Writer 인터페이스

Audit은 logger helper와 분리한다.

```python
audit_writer.write(
    action="rights.decision.allowed",
    actor_type="user",
    actor_id=context.user_id,
    resource_type="voice_model",
    resource_id=voice_model_id,
    decision="allowed",
    reason_code="OWNER_CONSENT_VALID",
    policy_version=policy_version,
    consent_record_id=consent_record_id,
    trace_id=context.trace_id,
    job_id=context.job_id,
    audio_asset_id=context.audio_asset_id,
)
```

Audit writer 내부 책임:

- audit schema validation
- `audit_event_id` 생성
- `event_hash` 생성
- append-only 저장 adapter 호출
- 저장 실패 시 fail-closed 여부 판단
- audit write 실패 로그와 알림 event 생성

## 개발 단계별 적용 계획

### Phase 1: engine-audio-ingest

목표:

- JSON stdout logger 적용
- `trace_id`, `job_id`, `audio_asset_id` 필수화
- ffprobe/ffmpeg 단계별 event 추가
- quality warning event 추가
- redaction 테스트 추가

완료 기준:

- `samples/audio` fixture 처리 시 로그가 JSON으로 출력된다.
- 성공/실패 로그 schema test가 통과한다.
- source path, token, email이 로그에 남지 않는다.

### Phase 2: engine-voice-analysis / engine-voice-pitch

목표:

- `engine-audio-ingest`의 observability module 패턴 복제
- NATS trace context 수신/발행 적용
- 모델/알고리즘 버전 필드 추가
- low confidence, silence, low energy warning 표준화

완료 기준:

- 같은 `trace_id`로 ingest, analysis, pitch 로그를 연결할 수 있다.
- pitch confidence warning이 표준 event로 남는다.

### Phase 3: safety / audit

목표:

- `engine-safety-rights`에 audit writer 적용
- rights decision event를 audit store에 저장
- consent, policy version, reason code 필수화
- audit write 실패 시 fail-closed 구현

완료 기준:

- 허용/거부 판단을 audit event로 재구성할 수 있다.
- audit event 조회 행위도 audit된다.

### Phase 4: 운영 통합

목표:

- collector 연동
- dashboard 구성
- alert rule 구성
- retention policy 적용

완료 기준:

- `trace_id`로 엔진 파이프라인을 검색할 수 있다.
- 엔진별 실패율과 p95 처리 시간이 dashboard에 보인다.
- audit write 실패가 P1 alert로 연결된다.

## 에이전트 구현 체크리스트

각 엔진 담당 에이전트는 로그 개발 시 아래 항목을 확인한다.

- [ ] `observability/` module이 있는가?
- [ ] raw `print()`가 없는가?
- [ ] 모든 pipeline log에 `trace_id`, `job_id`, `audio_asset_id`가 있는가?
- [ ] 오류 로그에 `error_code`, `error_type`, `retryable`이 있는가?
- [ ] NATS 수신/발행 시 trace context가 유지되는가?
- [ ] redaction filter가 logger 경계에 있는가?
- [ ] log schema test가 있는가?
- [ ] audit 대상 event가 일반 log로만 끝나지 않는가?
- [ ] audit write 실패 정책이 명확한가?
- [ ] `README.md`와 `AGENT.md`에 엔진별 log event가 반영되었는가?

## 금지 결정

아래 방식은 채택하지 않는다.

- 엔진별로 서로 다른 log field 이름 사용
- 운영 로그를 audio asset storage에 저장
- container 내부 파일 로그를 운영 기본값으로 사용
- audit event를 일반 logger로만 저장
- token, signed URL, full lyrics를 redaction 없이 출력
- `trace_id` 없이 NATS event 발행
- `ERROR`에 `retryable` 없는 로그 출력
- 모델 결과를 `model_version` 또는 `config_hash` 없이 기록
