# Engine Log Domain Guide

이 문서는 로그 도메인이 무엇인지 쉽게 설명한다.

로그 도메인은 "이 로그가 어떤 질문에 답하기 위한 것인가"를 나타내는 분류다. 엔진 이름이나 로그 레벨과는 다르다.

예를 들어 같은 `WARN`이라도 의미가 다를 수 있다.

```txt
engine-audio-ingest에서 clipping이 발견됨
  -> quality domain

engine-safety-rights에서 권한 없는 보이스 모델 접근 시도
  -> security domain

audit 저장 실패
  -> audit domain 또는 security domain
```

## 로그 도메인이 필요한 이유

Vox2Vocal은 여러 엔진이 이어지는 파이프라인이다.

```txt
engine-audio-ingest
  -> engine-voice-analysis
  -> engine-voice-pitch
  -> engine-melody-mapping
  -> engine-singing-synthesis
```

운영자가 보고 싶은 질문은 엔진 이름만으로 해결되지 않는다.

- 엔진이 살아있는가?
- 특정 작업이 어디까지 진행됐는가?
- 입력 오디오 품질이 나쁜가?
- 권한 없는 사용 시도가 있었는가?
- 나중에 법적/보안 감사에서 증명해야 하는 행위인가?

이 질문별로 로그를 묶은 것이 로그 도메인이다.

## Vox2Vocal 로그 도메인

| Domain | 쉬운 설명 | 대표 질문 | 최종 저장소 |
| --- | --- | --- | --- |
| `operational` | 엔진이 잘 돌아가는지 보는 로그 | 서버가 살아있나? dependency가 연결됐나? | Loki |
| `pipeline` | 작업 흐름을 따라가는 로그 | 이 `job_id`가 어느 엔진까지 갔나? | Loki |
| `quality` | 오디오/모델 결과 품질을 보는 로그 | clipping, low volume, low confidence가 있나? | Loki |
| `security` | 비정상 접근과 권한 위험을 보는 로그 | 누가 권한 없는 보이스를 쓰려 했나? | Loki security stream, future SIEM |
| `audit` | 나중에 증명해야 하는 행위 기록 | 누가 어떤 동의로 어떤 보이스를 사용했나? | PostgreSQL append-only audit store |
| `model` | 모델과 설정 재현성을 위한 로그 | 어떤 모델/설정으로 결과가 나왔나? | Loki, 중요 변경은 audit |

## Domain과 Level의 차이

`level`은 심각도다.

```txt
INFO
WARN
ERROR
CRITICAL
```

`domain`은 목적이다.

```txt
operational
pipeline
quality
security
audit
model
```

둘은 같이 사용한다.

예시:

```json
{
  "level": "WARN",
  "log_domain": "quality",
  "event": "audio.ingest.quality.warned",
  "quality_code": "CLIPPING_DETECTED"
}
```

```json
{
  "level": "WARN",
  "log_domain": "security",
  "event": "rights.decision.denied",
  "reason_code": "CONSENT_REVOKED"
}
```

두 로그 모두 `WARN`이지만 저장 정책과 알림 기준은 다르다.

## Domain별 예시

### operational

엔진 자체의 상태를 본다.

```json
{
  "level": "INFO",
  "log_domain": "operational",
  "service": "engine-audio-ingest",
  "event": "engine.startup.completed"
}
```

사용하는 곳:

- health dashboard
- dependency 장애 조사
- 배포 후 정상 기동 확인

### pipeline

작업 하나가 파이프라인을 어디까지 통과했는지 본다.

```json
{
  "level": "INFO",
  "log_domain": "pipeline",
  "service": "engine-audio-ingest",
  "event": "audio.ingest.completed",
  "trace_id": "trace_123",
  "job_id": "job_123",
  "audio_asset_id": "aud_123"
}
```

사용하는 곳:

- 특정 사용자 작업 추적
- 엔진 간 병목 확인
- 실패 지점 찾기

### quality

오디오나 모델 결과의 품질 문제를 본다.

```json
{
  "level": "WARN",
  "log_domain": "quality",
  "service": "engine-audio-ingest",
  "event": "audio.ingest.quality.warned",
  "quality_code": "LOW_VOLUME",
  "rms": 0.003
}
```

사용하는 곳:

- 입력 품질 분석
- 모델 성능 개선
- 사용자에게 품질 피드백 제공

### security

보안 위험이나 abuse signal을 본다.

```json
{
  "level": "WARN",
  "log_domain": "security",
  "service": "engine-safety-rights",
  "event": "rights.decision.denied",
  "actor_id": "user_123",
  "resource_type": "voice_model",
  "resource_id": "model_456",
  "reason_code": "OWNER_CONSENT_MISSING"
}
```

사용하는 곳:

- 권한 없는 보이스 사용 시도 탐지
- 계정 탈취 의심 탐지
- 관리자 이상행위 탐지

### audit

나중에 증명해야 하는 행위를 저장한다.

```json
{
  "audit_event_id": "audit_123",
  "log_domain": "audit",
  "actor_type": "user",
  "actor_id": "user_123",
  "action": "rights.decision.allowed",
  "resource_type": "voice_model",
  "resource_id": "model_456",
  "decision": "allowed",
  "policy_version": "2026-06-01",
  "consent_record_id": "consent_789",
  "event_hash": "sha256:..."
}
```

사용하는 곳:

- 보안감사
- 권리/동의 분쟁 대응
- 관리자 행위 검증

중요: audit domain은 Loki에만 저장하지 않는다. 반드시 audit store에 append-only로 저장한다.

### model

모델이나 설정 버전 때문에 결과가 달라질 수 있는 지점을 기록한다.

```json
{
  "level": "INFO",
  "log_domain": "model",
  "service": "engine-voice-pitch",
  "event": "voice.pitch.extraction.completed",
  "model_version": "pitch-2026.06.06",
  "config_hash": "sha256:abc123"
}
```

사용하는 곳:

- 결과 재현
- 모델 배포 전후 비교
- 품질 회귀 분석

## Domain 선택 규칙

에이전트는 로그를 만들 때 아래 순서로 domain을 선택한다.

1. 나중에 법적/보안 증명이 필요한가?
   - yes: `audit`
2. 권한, 동의, 비정상 접근, abuse와 관련 있는가?
   - yes: `security`
3. 작업 흐름을 추적하기 위한 로그인가?
   - yes: `pipeline`
4. 오디오 또는 모델 결과 품질과 관련 있는가?
   - yes: `quality`
5. 모델 버전이나 설정 재현성과 관련 있는가?
   - yes: `model`
6. 엔진 상태나 dependency 상태인가?
   - yes: `operational`

애매하면 `pipeline`과 `operational` 중 하나를 고르되, 권한/동의가 조금이라도 있으면 `security` 또는 `audit`을 우선 검토한다.

## Domain별 저장 요약

| Domain | 저장 위치 |
| --- | --- |
| `operational` | Loki |
| `pipeline` | Loki |
| `quality` | Loki |
| `security` | Loki security stream, future OpenSearch/SIEM |
| `audit` | PostgreSQL append-only audit store, archive |
| `model` | Loki. 모델 승격/정책 변경은 audit store |

## 흔한 오해

오해:

```txt
ERROR면 전부 보안 로그다.
```

정정:

```txt
ERROR는 심각도일 뿐이다. FFmpeg timeout은 operational/pipeline error이고, 권한 없는 보이스 사용 시도는 security warning일 수 있다.
```

오해:

```txt
audit도 JSON 로그니까 Loki에 넣으면 된다.
```

정정:

```txt
Audit은 증적이다. Loki 조회용 로그와 별도로 append-only audit store에 저장해야 한다.
```

오해:

```txt
trace_id가 있으면 audit은 필요 없다.
```

정정:

```txt
trace_id는 작업 추적용이고, audit은 권한/동의/행위 증명용이다. 둘은 서로 보완 관계다.
```
