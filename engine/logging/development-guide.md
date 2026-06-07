# Engine Logging Development Guide

이 문서는 각 `engine-*` 저장소에서 로그를 어떻게 개발해야 하는지 정의한다.

개발 스택, 저장 위치 책임, 모듈 아키텍처, 단계별 적용 계획은 [Development Direction](./development-direction.md)을 먼저 따른다. 이 문서는 실제 로그 필드, 레벨, event 이름, 테스트 규칙을 정의한다.

## 목표

개발 단계의 로그는 아래 질문에 답할 수 있어야 한다.

- 이 작업은 어떤 `trace_id`와 `job_id`에 속하는가?
- 어떤 엔진이 어떤 입력을 처리했는가?
- 어떤 모델, 설정, threshold로 처리했는가?
- 처리 결과가 성공, 경고, 실패 중 무엇인가?
- 실패가 재시도 가능한가?
- 사용자의 보이스 권한 또는 동의 판단이 개입되었는가?
- 같은 입력을 다시 처리했을 때 결과를 재현할 수 있는가?

## 로그 종류

| 종류 | 설명 | 저장 위치 |
| --- | --- | --- |
| operational log | 엔진 시작, 연결, health, 내부 처리 상태 | 운영 로그 저장소 |
| pipeline event log | 파이프라인 단계별 시작/완료/실패 | 운영 로그 저장소 |
| quality log | clipping, low confidence, silence, pitch deviation 같은 품질 경고 | 운영 로그 저장소 |
| security log | 권한 거부, 비정상 접근, policy mismatch | 보안 로그 또는 운영 로그 저장소 |
| audit event | 권한/동의/관리자 행위/보이스 사용 증적 | audit 저장소 |

`audit event`는 일반 로그가 아니다. 일반 logger로만 남기지 말고 audit writer를 통해 별도 저장한다.

## 공통 JSON 필드

모든 엔진 로그는 아래 필드를 기준으로 작성한다.

| 필드 | 필수 | 설명 |
| --- | --- | --- |
| `timestamp` | yes | UTC ISO-8601 timestamp |
| `level` | yes | `DEBUG`, `INFO`, `WARN`, `ERROR`, `CRITICAL` |
| `service` | yes | 예: `engine-audio-ingest` |
| `env` | yes | `local`, `dev`, `staging`, `prod` |
| `event` | yes | 표준 event 이름 |
| `schema_version` | yes | 로그 schema version |
| `trace_id` | pipeline yes | 전체 요청 추적 ID |
| `span_id` | 권장 | 엔진 내부 작업 단위 ID |
| `job_id` | pipeline yes | 작업 ID |
| `audio_asset_id` | audio yes | 오디오 자산 ID |
| `user_id` | 권장 | 내부 user ID. 이메일 등 직접 식별자는 금지 |
| `engine_version` | yes | 엔진 코드 또는 image version |
| `duration_ms` | 완료/실패 yes | 처리 시간 |
| `error_code` | 실패 yes | 표준 error code |
| `error_type` | 실패 yes | exception class 또는 오류 분류 |
| `retryable` | 실패 yes | 재시도 가능 여부 |
| `model_version` | 모델 사용 시 yes | 모델 버전 |
| `config_hash` | 모델/알고리즘 사용 시 yes | 설정 재현용 hash |
| `policy_version` | 권한 판단 시 yes | 정책 버전 |

## 로그 레벨

| 레벨 | 사용 기준 |
| --- | --- |
| `DEBUG` | 로컬 재현용 세부값. 운영 기본 저장 금지 |
| `INFO` | 정상 상태 변화. 요청 수신, 시작, 완료 |
| `WARN` | 처리는 성공했지만 품질 또는 운영 주의 필요 |
| `ERROR` | 작업 실패. 재시도 가능 여부를 반드시 포함 |
| `CRITICAL` | 엔진 지속 불가, 데이터 손상 위험, audit 저장 실패 같은 즉시 대응 대상 |

## Event 이름 규칙

형식:

```txt
<domain>.<capability>.<state>
```

예시:

```txt
audio.ingest.started
audio.ingest.completed
audio.ingest.failed
voice.pitch.extraction.started
voice.pitch.extraction.completed
safety.rights.decision.denied
audit.write.failed
```

NATS subject와 같은 pipeline boundary event는 가능하면 event 이름도 동일하게 사용한다.

## Trace 전파 규칙

엔진 간 이벤트 payload에는 아래 값을 포함한다.

```json
{
  "trace_id": "trace_01H...",
  "parent_span_id": "span_01H...",
  "job_id": "job_123",
  "audio_asset_id": "aud_123",
  "user_id": "user_123"
}
```

수신 엔진은 새 `span_id`를 만들고, 기존 `trace_id`를 유지한다.

```txt
engine-audio-ingest span
  -> engine-voice-analysis span
  -> engine-voice-pitch span
```

## Python 구현 기준

Python 엔진은 공통 logging helper를 둔다.

권장 인터페이스:

```python
logger.info_event(
    event="audio.ingest.completed",
    trace_id=trace_id,
    job_id=job_id,
    audio_asset_id=audio_asset_id,
    user_id=user_id,
    duration_ms=duration_ms,
    sample_rate=48000,
    channels=1,
)
```

오류 로그:

```python
logger.error_event(
    event="audio.ingest.failed",
    trace_id=trace_id,
    job_id=job_id,
    audio_asset_id=audio_asset_id,
    user_id=user_id,
    error_code="FFMPEG_TIMEOUT",
    error_type="TimeoutError",
    retryable=True,
    duration_ms=600000,
)
```

출력 예시:

```json
{
  "timestamp": "2026-06-06T01:23:45.123Z",
  "level": "INFO",
  "service": "engine-audio-ingest",
  "env": "prod",
  "event": "audio.ingest.completed",
  "schema_version": "1.0",
  "trace_id": "trace_123",
  "span_id": "span_456",
  "job_id": "job_123",
  "audio_asset_id": "aud_123",
  "user_id": "user_123",
  "engine_version": "2026.06.06",
  "duration_ms": 1832,
  "sample_rate": 48000,
  "channels": 1
}
```

## Redaction 규칙

로그 출력 직전에 redaction filter를 적용한다.

로그 금지:

- access token, refresh token, API key
- 이메일, 전화번호, 실명
- 원본 음성의 로컬 절대 경로
- 원본 가사 또는 대본 전문
- 사용자 프롬프트 전문
- consent 문서 원문
- 모델 weight 경로 중 민감한 bucket/prefix

허용:

- `user_id`
- `audio_asset_id`
- `voice_model_id`
- `consent_record_id`
- `policy_decision_id`
- normalized object key
- hash 또는 digest

## 개발 테스트 체크리스트

각 엔진은 로그 관련 테스트를 가진다.

- 성공 로그에 필수 필드가 모두 있는지 검증한다.
- 실패 로그에 `error_code`, `retryable`, `duration_ms`가 있는지 검증한다.
- `trace_id`가 입력 이벤트에서 출력 이벤트로 유지되는지 검증한다.
- 민감정보 redaction 테스트를 둔다.
- log injection 방지를 위해 newline, tab, control character가 안전하게 직렬화되는지 검증한다.
- audit event가 필요한 작업은 일반 log만 남기고 끝나지 않는지 검증한다.

## 금지 패턴

나쁜 예:

```txt
print("failed")
print(user_email)
print(full_lyrics)
print("token=" + token)
logger.error(exception)
```

좋은 예:

```json
{
  "level": "ERROR",
  "event": "audio.ingest.failed",
  "trace_id": "trace_123",
  "job_id": "job_123",
  "audio_asset_id": "aud_123",
  "error_code": "SOURCE_NOT_FOUND",
  "retryable": false
}
```

## 새 엔진을 만들 때 해야 할 일

1. `README.md`에 로그 계약을 추가한다.
2. `AGENT.md`에 필수 로그 이벤트와 금지 필드를 추가한다.
3. [Engine Log Event Index](./engine-log-index.md)에 엔진별 event를 추가한다.
4. 공통 logger helper를 적용한다.
5. 로그 schema 테스트를 추가한다.
6. audit event가 필요한 기능이면 [Audit Data Guide](./audit-data-guide.md)에 event를 추가한다.
