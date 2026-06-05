# Engine Audio Ingest Development Preparation

## 1. 문서 목적

이 문서는 `engine-audio-ingest` 개발을 시작하기 전에 확정해야 할 준비 항목을 정리한다.

기술 리서치 문서가 "어떤 기술을 선택할 것인가"를 다룬다면, 이 문서는 "개발을 시작하기 전에 무엇을 정하고 준비해야 하는가"를 다룬다.

`engine-audio-ingest`는 Vox2Vocal 파이프라인의 첫 번째 엔진이다. 이 엔진이 생성하는 표준 오디오와 manifest는 `engine-voice-analysis`, `engine-voice-pitch`, `engine-phoneme-alignment` 등 후속 엔진의 입력이 된다.

따라서 개발 착수 전 다음 항목을 먼저 확정해야 한다.

- MVP 기능 범위
- 입력/출력 계약
- 저장소 구조
- 큐/이벤트 방식
- 메타데이터 모델
- 엔진 프로젝트 구조
- 로컬 실행 환경
- 테스트 및 검증 기준
- 사용자가 결정해야 하는 정책성 항목

## 2. 개발 전 핵심 결론

개발을 바로 시작하기 전에 가장 먼저 필요한 것은 코드가 아니라 **계약과 경계 확정**이다.

특히 아래 세 가지는 먼저 결정되어야 한다.

1. 이 엔진이 어디까지 책임지는가
2. 후속 엔진에 어떤 산출물을 넘기는가
3. 실패/재시도/중복 처리 기준을 어떻게 둘 것인가

권장 개발 시작점은 다음이다.

```txt
1. Python 프로젝트 골격 생성
2. 로컬 파일 기반 ingest CLI 구현
3. FFmpeg/ffprobe wrapper 구현
4. canonical WAV + manifest 생성
5. sample audio 기반 테스트 작성
6. storage abstraction 추가
7. NATS JetStream worker 추가
8. FastAPI health/status API 추가
9. Dockerfile 작성
10. api-gateway 및 후속 엔진 이벤트 계약 연결
```

## 3. MVP 범위 확정

### 3.1 MVP Must

초기 버전에서 반드시 구현할 범위는 다음을 권장한다.

| 항목 | 권장값 |
| --- | --- |
| 입력 포맷 | `wav`, `mp3` |
| 출력 포맷 | 48kHz mono WAV |
| 검증 방식 | `ffprobe` 기반 format/codec/duration 검증 |
| 변환 방식 | `ffmpeg` 기반 decode/resample/mono 변환 |
| 기본 분석 | duration, sample rate, channels, codec, peak, RMS, clipping |
| 구간 분할 | energy threshold 기반 active audio segment 생성 |
| chunk | 30초 단위 WAV chunk |
| 산출물 | `canonical.wav`, `manifest.json`, optional chunks |
| 상태 관리 | `requested`, `processing`, `completed`, `failed` |
| 실패 처리 | retry 가능, error code 기록 |
| 로그 | structured logging |

### 3.2 MVP에서 제외할 항목

다음 항목은 2차 개발로 미루는 것을 권장한다.

- `m4a`, `flac` 입력
- Silero VAD
- loudness normalization
- waveform preview image
- low-resolution preview audio
- pitch/F0 추출
- phoneme alignment
- voice conversion
- singing synthesis
- vocoder rendering
- 권리/동의 최종 판단

## 4. 입력/출력 계약 정의

개발 전 가장 중요한 준비는 API와 이벤트 계약 정의다.

### 4.1 Ingest Job 생성 요청

`api-gateway` 또는 내부 orchestrator가 `engine-audio-ingest`에 job을 생성할 때 필요한 최소 payload는 다음과 같다.

```json
{
  "audio_asset_id": "aud_123",
  "user_id": "user_123",
  "source_object_key": "audio-assets/aud_123/original/input.mp3",
  "original_filename": "input.mp3",
  "content_type": "audio/mpeg",
  "idempotency_key": "upload-session-123"
}
```

### 4.2 Ingest 완료 이벤트

후속 엔진이 받아야 하는 완료 이벤트 예시는 다음과 같다.

```json
{
  "event_type": "audio.ingest.completed",
  "audio_asset_id": "aud_123",
  "manifest_object_key": "audio-assets/aud_123/manifest.json",
  "canonical_object_key": "audio-assets/aud_123/canonical/audio.wav",
  "duration_ms": 182340,
  "sample_rate": 48000,
  "channels": 1,
  "created_at": "2026-06-05T00:00:00Z"
}
```

### 4.3 Ingest 실패 이벤트

실패 이벤트에는 재처리와 사용자 안내를 위한 정보가 필요하다.

```json
{
  "event_type": "audio.ingest.failed",
  "audio_asset_id": "aud_123",
  "job_id": "job_123",
  "error_code": "UNSUPPORTED_CODEC",
  "error_message": "Unsupported audio codec",
  "retryable": false,
  "failed_at": "2026-06-05T00:00:00Z"
}
```

### 4.4 Manifest 산출물

후속 엔진은 DB를 직접 조회하기보다 manifest를 기준으로 오디오 자산을 읽는 것이 좋다.

manifest에는 최소한 다음 필드가 포함되어야 한다.

- `audio_asset_id`
- source object key
- canonical object key
- duration
- sample rate
- channels
- codec/format
- peak/RMS/clipping
- active audio segments
- chunks

## 5. 저장소 구조 준비

### 5.1 객체 저장소

원본 오디오와 산출물은 PostgreSQL에 직접 저장하지 않고 객체 저장소에 저장한다.

MVP 권장 저장소 구현체는 MinIO 또는 S3-compatible object storage다. Python client는 `boto3`로 확정한다.

권장 object key 구조:

```txt
audio-assets/{audio_asset_id}/original/{filename}
audio-assets/{audio_asset_id}/canonical/audio.wav
audio-assets/{audio_asset_id}/chunks/0000.wav
audio-assets/{audio_asset_id}/manifest.json
```

### 5.2 로컬 개발 대안

S3-compatible object storage 연결 전에는 local filesystem storage adapter를 먼저 구현할 수 있다.

예시:

```txt
.local/audio-assets/{audio_asset_id}/original/input.mp3
.local/audio-assets/{audio_asset_id}/canonical/audio.wav
.local/audio-assets/{audio_asset_id}/manifest.json
```

이 경우 production storage와 동일한 interface를 유지해야 한다.

## 6. 큐/이벤트 준비

큐/이벤트 시스템은 NATS JetStream으로 확정한다.

선택 이유:

- `engine-audio-ingest` 이후 여러 엔진이 이벤트를 독립적으로 소비해야 한다.
- durable consumer와 replay가 필요하다.
- 작업 큐와 이벤트 스트림을 하나의 메시징 계층으로 다룰 수 있다.
- Kafka보다 가볍고, Redis Streams보다 엔진 간 이벤트 파이프라인에 적합하다.
- Node/NestJS 서비스와 Python 엔진 모두에서 언어 중립적으로 사용할 수 있다.

### 6.1 Stream 설계안

권장 stream:

```txt
VOX2VOCAL_AUDIO
```

권장 subject:

```txt
audio.ingest.requested
audio.ingest.started
audio.ingest.completed
audio.ingest.failed
audio.asset.updated
```

stream subject pattern:

```txt
audio.>
```

### 6.2 Durable Consumer

`engine-audio-ingest` worker는 ingest 요청 이벤트를 소비한다.

```txt
consumer: engine-audio-ingest-worker
subject: audio.ingest.requested
ack policy: explicit ack
deliver policy: new
max deliver: 3
```

후속 엔진은 ingest 완료 이벤트를 각자 독립 consumer로 소비한다.

```txt
consumer: engine-voice-analysis
subject: audio.ingest.completed

consumer: engine-evaluation
subject: audio.ingest.completed

consumer: engine-safety-rights
subject: audio.ingest.completed
```

### 6.3 재처리 기준

개발 전 다음 기준을 정해야 한다.

- `max_deliver` 기본값
- retry backoff 방식
- `ack_wait` 시간
- FFmpeg timeout
- 실패 이벤트 발행 시점
- retryable error와 non-retryable error 구분
- max delivery 초과 시 dead-letter subject로 보낼지 여부

권장 초기값:

```txt
max_deliver: 3
ack_wait: 10 minutes
ffmpeg_timeout: 10 minutes
dead_letter_subject: audio.ingest.dead_letter
```

## 7. 메타데이터 모델 준비

MVP에서 필요한 최소 테이블은 다음 두 개다.

### 7.1 audio_assets

```txt
id
user_id
source_object_key
canonical_object_key
manifest_object_key
input_format
input_codec
duration_ms
sample_rate
channels
status
created_at
updated_at
```

### 7.2 audio_ingest_jobs

```txt
id
audio_asset_id
status
attempt_count
error_code
error_message
started_at
finished_at
created_at
updated_at
```

### 7.3 상태값

권장 상태값:

```txt
requested
processing
completed
failed
canceled
```

## 8. 프로젝트 골격 준비

`engine-audio-ingest`는 Python 프로젝트로 시작하는 것을 권장한다.

권장 디렉터리 구조:

```txt
engine-audio-ingest/
  src/
    audio_ingest/
      api/
      audio/
      config/
      events/
      jobs/
      models/
      storage/
      workers/
  tests/
    fixtures/
  Dockerfile
  README.md
  pyproject.toml
  .env.example
```

### 8.1 핵심 모듈

초기 구현에 필요한 모듈은 다음이다.

| 모듈 | 역할 |
| --- | --- |
| FFmpeg wrapper | `ffmpeg`, `ffprobe` 실행과 timeout 처리 |
| Metadata parser | ffprobe 결과를 내부 모델로 변환 |
| Audio normalizer | canonical WAV 생성 |
| Segment detector | active audio segment 생성 |
| Chunk writer | 긴 오디오 chunk 분리 |
| Manifest builder | 후속 엔진용 manifest 생성 |
| Storage client | local/S3-compatible 저장소 abstraction, boto3 기반 구현 |
| Event publisher | NATS JetStream 이벤트 발행 |
| Worker | ingest job 처리 |
| API | health/status/debug endpoint |

## 9. 로컬 실행 환경 준비

개발 시작 전에 로컬에서 다음 도구가 필요하다.

- Python 3.12+
- FFmpeg
- NATS JetStream access
- PostgreSQL
- MinIO/S3-compatible object storage 또는 local storage
- Docker

처음부터 Kubernetes를 붙이기보다, 먼저 다음이 되는지 확인하는 것을 권장한다.

```txt
input.mp3
  -> canonical/audio.wav
  -> manifest.json
```

## 10. Engine Pod 필수 구성

NATS JetStream은 이미 Kubernetes 인프라에 구축되어 있다고 가정한다. 따라서 `engine-audio-ingest` pod에는 NATS 서버가 아니라 NATS에 접속할 client와 오디오 ingest 런타임만 포함한다.

### 10.1 필수 OS packages

Debian slim 기준 필수 OS package:

```txt
ffmpeg
libsndfile1
ca-certificates
curl
```

역할:

| 항목 | 역할 |
| --- | --- |
| `ffmpeg` | 오디오 디코딩, 변환, resampling, mono 변환 |
| `ffprobe` | format, codec, duration, sample rate, channels 검증 |
| `libsndfile1` | `soundfile` 기반 WAV/PCM read/write 지원 |
| `ca-certificates` | S3-compatible storage HTTPS 통신 |
| `curl` | 개발/디버깅용 health 확인 |

`ffprobe`는 일반적으로 `ffmpeg` package에 함께 포함된다.

### 10.2 필수 Python dependencies

MVP 필수 Python dependency:

```txt
fastapi
uvicorn
pydantic
pydantic-settings
nats-py
numpy
soundfile
librosa
boto3
python-json-logger
```

역할:

| 항목 | 역할 |
| --- | --- |
| `fastapi` | health/status endpoint |
| `uvicorn` | FastAPI ASGI server |
| `pydantic` | payload validation |
| `pydantic-settings` | env/config validation |
| `nats-py` | NATS JetStream client |
| `numpy` | 오디오 수치 처리 |
| `soundfile` | WAV/PCM 파일 read/write |
| `librosa` | active audio segment, RMS 등 분석 |
| `boto3` | MinIO/S3-compatible object storage client |
| `python-json-logger` | structured JSON logging |

### 10.3 MVP 필수 제외 항목

다음 항목은 MVP pod 필수 구성에서 제외한다.

```txt
torchaudio
torch
minio
celery
rabbitmq client
redis client
```

제외 이유:

- `torchaudio`와 `torch`는 Docker image를 무겁게 만들고, ingest MVP에는 필수 기능이 아니다.
- PyTorch/Tensor 기반 처리는 후속 `engine-voice-analysis`, `engine-voice-pitch` 등 ML 엔진에서 도입한다.
- object storage는 MinIO/S3-compatible storage를 사용하되, Python client는 AWS S3 이식성이 좋은 `boto3`를 사용한다.
- 큐/이벤트는 NATS JetStream으로 확정했으므로 Celery/RabbitMQ/Redis client는 필요하지 않다.

### 10.4 Dockerfile 기본 방향

권장 Dockerfile 방향:

```dockerfile
FROM python:3.12-slim

RUN apt-get update \
  && apt-get install -y --no-install-recommends \
    ffmpeg \
    libsndfile1 \
    ca-certificates \
    curl \
  && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY pyproject.toml ./
COPY src ./src

RUN pip install --no-cache-dir -e .

ENV PORT=3004

CMD ["uvicorn", "audio_ingest.api.main:app", "--host", "0.0.0.0", "--port", "3004"]
```

개발 중 Python wheel build 문제가 생길 때만 임시로 `build-essential`, `gcc`를 추가한다. 운영 이미지에서는 가능하면 제외한다.

### 10.5 필수 환경변수

```env
PORT=3004

NATS_URL=nats://nats:4222
NATS_STREAM=VOX2VOCAL_AUDIO
NATS_SUBJECT_PREFIX=audio

OBJECT_STORAGE_ENDPOINT_URL=http://minio:9000
OBJECT_STORAGE_BUCKET=vox2vocal-audio
OBJECT_STORAGE_ACCESS_KEY=
OBJECT_STORAGE_SECRET_KEY=
OBJECT_STORAGE_REGION=us-east-1
OBJECT_STORAGE_FORCE_PATH_STYLE=true

AUDIO_CANONICAL_SAMPLE_RATE=48000
AUDIO_CANONICAL_CHANNELS=1
AUDIO_MAX_DURATION_SECONDS=300
AUDIO_MAX_FILE_SIZE_MB=100
FFMPEG_TIMEOUT_SECONDS=600
```

### 10.6 Health Check

`GET /health`는 최소 다음 항목을 확인해야 한다.

```json
{
  "status": "ok",
  "ffmpeg_available": true,
  "ffprobe_available": true,
  "nats_configured": true,
  "object_storage_configured": true
}
```

pod 내부 검증 명령:

```bash
python --version
ffmpeg -version
ffprobe -version
python -c "import librosa, soundfile, boto3, nats; print('ok')"
```

## 11. 테스트 및 검증 기준

### 11.1 필수 테스트

MVP에서 필요한 최소 테스트는 다음이다.

- 정상 `wav` 파일 ingest 성공
- 정상 `mp3` 파일 ingest 성공
- 깨진 파일 실패 처리
- 지원하지 않는 포맷 실패 처리
- 너무 긴 파일 실패 처리
- canonical WAV가 48kHz mono인지 확인
- manifest에 duration/sample_rate/channels/segments가 포함되는지 확인
- 동일 job 재시도 시 중복 산출물이 깨지지 않는지 확인
- FFmpeg timeout 시 실패 상태가 기록되는지 확인

### 11.2 샘플 오디오

테스트 fixture에는 다음 파일이 있으면 좋다.

```txt
short_voice.wav
short_voice.mp3
silence.wav
stereo_input.wav
unsupported.txt
corrupted.mp3
```

저작권 문제가 없는 직접 생성 오디오 또는 synthetic audio를 사용해야 한다.

## 12. 보안 및 운영 준비

개발 초기부터 다음 기준을 반영해야 한다.

- 확장자만으로 포맷을 판단하지 않는다.
- MIME type과 ffprobe 결과를 함께 확인한다.
- 파일 크기 상한을 둔다.
- duration 상한을 둔다.
- FFmpeg 실행 timeout을 둔다.
- 임시 파일 경로를 job별로 격리한다.
- object key에 사용자 입력 filename을 그대로 쓰지 않는다.
- 원본 파일 hash를 기록한다.
- job id, audio asset id, user id, trace id를 로그에 남긴다.

## 13. 개발 순서 제안

### Phase 1: Local Prototype

- Python 프로젝트 scaffold
- `ingest-local` CLI 작성
- ffprobe metadata 추출
- ffmpeg canonical WAV 생성
- manifest JSON 생성
- 로컬 fixture 기반 테스트 작성

### Phase 2: Storage and Worker

- storage interface 작성
- local storage adapter 작성
- boto3 기반 S3-compatible storage adapter 작성
- NATS JetStream consumer 작성
- job retry/error handling 작성

### Phase 3: API and Integration

- FastAPI health endpoint
- job status endpoint
- api-gateway 연동 계약 정의
- ingest completed/failed 이벤트 발행
- 후속 `engine-voice-analysis` 입력 연결

### Phase 4: Deployment

- Dockerfile 작성
- Kubernetes manifest 작성
- NATS/PostgreSQL/object storage env 연결
- OpenTelemetry/Prometheus metric 추가

## 14. 사용자가 결정해야 하는 질문

아래 항목은 구현자가 임의로 정할 수도 있지만, 제품 정책과 운영 기준에 영향을 주므로 개발 착수 전에 사용자의 결정을 받는 것이 좋다.

### 14.1 MVP 입력 포맷

질문:

MVP에서 입력 포맷을 `wav`, `mp3`만 지원할까요, 아니면 처음부터 `m4a`, `flac`까지 포함할까요?

추천:

초기에는 `wav`, `mp3`만 지원하고, `m4a`, `flac`는 2차로 확장한다.

### 14.2 표준 샘플레이트

질문:

canonical audio의 표준 샘플레이트를 48kHz로 할까요, 44.1kHz로 할까요?

추천:

48kHz mono WAV를 기본으로 한다. 다만 후속 모델이 44.1kHz 또는 16kHz를 요구하면 파생 파일을 추가 생성한다.

### 14.3 최대 파일 크기와 길이

질문:

MVP에서 한 파일의 최대 크기와 최대 길이를 얼마로 제한할까요?

추천:

초기 제한은 100MB, 5분으로 시작한다.

### 14.4 저장소 방식

질문:

개발 초기부터 S3-compatible object storage를 붙일까요, 아니면 local filesystem adapter로 먼저 시작한 뒤 boto3 기반 object storage adapter를 붙일까요?

추천:

local filesystem adapter로 먼저 ingest 파이프라인을 검증하고, interface를 유지한 채 boto3 기반 S3-compatible adapter를 추가한다.

### 14.5 큐 방식

결정:

MVP 큐/이벤트 시스템은 NATS JetStream으로 확정한다.

반영 기준:

- ingest 요청은 `audio.ingest.requested` subject로 발행한다.
- ingest worker는 durable consumer로 요청을 처리한다.
- 처리 성공 시 `audio.ingest.completed`를 발행한다.
- 처리 실패 시 `audio.ingest.failed`를 발행한다.
- 후속 엔진은 각자 durable consumer로 완료 이벤트를 소비한다.

### 14.6 API 계약

질문:

`engine-audio-ingest`는 처음부터 gRPC API를 제공할까요, 아니면 FastAPI HTTP endpoint와 NATS JetStream worker부터 시작할까요?

추천:

MVP는 FastAPI health/status + NATS JetStream worker로 시작하고, 내부 계약이 안정화되면 gRPC proto를 추가한다.

### 14.7 audio_asset_id 생성 주체

질문:

`audio_asset_id`는 `api-gateway`에서 생성할까요, `engine-audio-ingest`에서 생성할까요?

추천:

업로드 세션을 만드는 상위 계층, 즉 `api-gateway` 또는 BFF 쪽에서 생성하고 ingest engine은 전달받은 ID를 기준으로 처리한다.

### 14.8 원본 파일 보존 정책

질문:

원본 업로드 파일을 계속 보존할까요, canonical 생성 후 일정 기간 뒤 삭제할까요?

추천:

MVP와 디버깅 단계에서는 원본을 보존한다. 운영 정책이 정해지면 보존 기간을 설정한다.

### 14.9 active audio segment 정의

질문:

구간 분할 결과를 "speech segment"로 부를까요, "active audio segment"로 부를까요?

추천:

노래/허밍/가이드 보컬을 고려해 "active audio segment"로 정의한다.

### 14.10 실패 알림 기준

질문:

ingest 실패 시 사용자에게 즉시 노출할 error message와 내부 debug message를 분리할까요?

추천:

분리한다. 사용자에게는 안전하고 간단한 메시지를, 내부에는 상세 error code/log를 남긴다.

## 15. 결정 후 바로 진행할 수 있는 작업

위 질문에 대한 결정이 끝나면 다음 작업을 바로 진행할 수 있다.

1. `engine-audio-ingest` Python 프로젝트 scaffold
2. `.env.example` 작성
3. `pyproject.toml` 작성
4. FFmpeg/ffprobe wrapper 구현
5. local ingest CLI 구현
6. manifest schema 초안 작성
7. fixture 기반 테스트 추가
8. README에 로컬 실행 절차 작성
