# Engine Audio Ingest Technical Research

## 1. 문서 목적

이 문서는 Vox2Vocal의 `engine-audio-ingest` 구축을 위해 필요한 기술 요소와 권장 기술 스택을 정리한다.

`engine-audio-ingest`는 단순 파일 업로드 서버가 아니라, 사용자가 업로드한 음성/가이드 보컬/반주 파일을 후속 엔진들이 일관되게 사용할 수 있는 표준 오디오 자산으로 변환하는 첫 번째 엔진이다.

따라서 이 엔진의 핵심 역할은 다음과 같다.

- 다양한 오디오 포맷 수신 및 검증
- 원본 오디오 보존
- 표준 PCM 오디오 생성
- 샘플레이트, 채널, 길이, loudness, peak, RMS 등 메타데이터 추출
- 무음 구간 및 발화 구간 분할
- 긴 오디오의 chunk 분리
- 후속 엔진이 참조할 `audio_asset_id`와 manifest 생성
- 후속 엔진으로 ingest 완료 이벤트 발행

## 2. 결론 요약

MVP 기준 권장 스택은 다음과 같다.

| 영역 | MVP 추천 | 확장 시 |
| --- | --- | --- |
| 엔진 런타임 | Python 3.12+ | Python + Go/Rust ingest proxy |
| API | FastAPI + Uvicorn, 내부 gRPC | gRPC streaming, WebSocket, WebRTC |
| 오디오 디코딩/변환 | FFmpeg, ffprobe | GStreamer |
| 오디오 처리 | librosa, soundfile/libsndfile, NumPy | torchaudio, PyAV, ONNX Runtime, Silero VAD |
| 원본/산출물 저장소 | MinIO 또는 S3-compatible object storage, boto3 client | AWS S3, GCS, Azure Blob |
| 메타데이터 저장소 | PostgreSQL | PostgreSQL 유지 |
| 큐/이벤트 | NATS JetStream | Kafka |
| 관측성 | OpenTelemetry, Prometheus | Grafana, Loki, Tempo |
| 배포 | Docker, Kubernetes | KEDA/HPA 기반 autoscaling |

최종 추천은 **Python 기반 오디오 처리 엔진 + FFmpeg + librosa/soundfile/NumPy + boto3 + MinIO/S3 + NATS JetStream + gRPC/HTTP API** 조합이다.

이 조합은 현재 Vox2Vocal의 NestJS/gRPC/Redis/Kubernetes 구조와 잘 붙고, 오디오 처리 및 향후 ML 파이프라인 확장에도 가장 유리하다.

### 2.1 MVP Runtime Dependency Decision

MVP pod 필수 구성은 다음으로 확정한다.

```txt
python:3.12-slim
ffmpeg / ffprobe
libsndfile1
FastAPI / Uvicorn
nats-py
librosa
soundfile
numpy
boto3
python-json-logger
```

MVP 필수에서 제외하는 항목:

```txt
torchaudio
torch
minio Python SDK
celery
rabbitmq client
redis client
```

결정 이유:

- ingest 단계의 핵심 처리는 FFmpeg, ffprobe, librosa, soundfile, NumPy로 충분하다.
- `torchaudio`와 `torch`는 Docker image를 무겁게 만들기 때문에 후속 ML 엔진에서 필요할 때 추가한다.
- object storage는 MinIO/S3-compatible storage를 사용하되, Python client는 AWS S3 이식성이 좋은 `boto3`를 사용한다.
- 큐/이벤트는 NATS JetStream으로 확정했으므로 Celery/RabbitMQ/Redis client는 MVP 필수가 아니다.

## 3. 엔진 책임 범위

### 3.1 In Scope

`engine-audio-ingest`가 직접 책임져야 하는 기능은 다음과 같다.

- `wav`, `mp3`, `m4a`, `flac` 입력 지원
- `ffprobe` 기반 파일 검증
- codec/container 정보 추출
- duration, sample rate, channel count, bit depth, bitrate 추출
- `ffmpeg` 기반 decode/resample/channel normalize
- mono PCM 변환
- 44.1kHz 또는 48kHz 표준 오디오 생성
- peak, RMS, loudness, clipping 여부 분석
- 무음 구간 탐지
- 발화 구간 timestamp 생성
- 긴 오디오 chunk 분리
- 원본 파일과 표준화 파일 저장
- manifest JSON 생성
- 후속 엔진에 ingest 완료 이벤트 발행

### 3.2 Out of Scope

다음 기능은 후속 엔진에서 처리하는 것이 좋다.

- pitch/F0 추정
- 음소 정렬
- 가사/텍스트 alignment
- voice conversion
- singing synthesis
- vocoder rendering
- mix/mastering
- 저작권/동의 정책 판단

다만 `engine-audio-ingest`는 Safety Rights 엔진이 검증할 수 있도록 원본 파일, 업로드 주체, 생성 시간, 처리 이력, 파일 hash 등 감사 가능한 메타데이터를 남겨야 한다.

## 4. 권장 아키텍처

### 4.1 MVP 처리 흐름

```txt
App / BFF
  -> api-gateway
  -> upload session 생성
  -> client uploads original audio to MinIO/S3
  -> engine-audio-ingest job enqueue
  -> Python ingest worker
      -> ffprobe validate
      -> ffmpeg decode/resample/mono
      -> librosa/soundfile/numpy metadata + segmentation
      -> store canonical wav + manifest
  -> publish audio.ingested
  -> engine-voice-analysis
```

### 4.2 왜 직접 API 업로드보다 객체 저장소 업로드가 좋은가

오디오 파일은 크기가 크고 네트워크 실패가 잦을 수 있다. API 서버가 파일 body를 오래 물고 있으면 gateway timeout, memory pressure, retry 중복 처리 문제가 생긴다.

따라서 MVP부터 다음 방식을 권장한다.

1. API가 upload session을 생성한다.
2. API가 presigned upload URL 또는 resumable upload endpoint를 내려준다.
3. 클라이언트가 원본 파일을 객체 저장소에 직접 업로드한다.
4. 업로드 완료 후 ingest job을 큐에 넣는다.
5. ingest worker가 객체 저장소에서 파일을 가져와 처리한다.

대용량 파일이나 모바일 네트워크를 고려하면 S3 multipart upload 또는 tus resumable upload 프로토콜도 검토할 수 있다.

## 5. 기술 스택 상세

### 5.1 Python

Python을 엔진 런타임으로 추천한다.

이유는 다음과 같다.

- 오디오 처리 라이브러리 생태계가 가장 풍부하다.
- `librosa`, `soundfile`, NumPy를 바로 사용할 수 있고, PyTorch 기반 `torchaudio`는 후속 ML 엔진에서 필요할 때 추가할 수 있다.
- 향후 voice analysis, pitch, phoneme alignment, synthesis 계열 ML 모델과 연결하기 쉽다.
- FFmpeg subprocess orchestration이 단순하다.
- FastAPI, gRPC Python, nats-py, boto3 등 서버/인프라 연동 구성도 충분히 성숙해 있다.

NestJS는 기존 BFF/API Gateway/일반 서비스에는 적합하지만, DSP/ML 중심 오디오 엔진에는 Python이 더 적합하다.

### 5.2 FastAPI

FastAPI는 다음 용도로 사용한다.

- health check
- job 생성 API
- ingest 상태 조회 API
- 내부 admin/debug endpoint
- 필요 시 upload callback endpoint

무거운 오디오 처리는 FastAPI background task 안에서 직접 처리하지 않고 큐/워커로 분리하는 것이 좋다. FastAPI 공식 문서에서도 무거운 background computation은 Celery 같은 별도 worker system 사용을 언급한다.

### 5.3 gRPC

기존 Vox2Vocal 구조가 gRPC 기반이므로 내부 서비스 계약은 gRPC가 적합하다.

적합한 사용처는 다음과 같다.

- `api-gateway -> engine-audio-ingest` job 생성
- ingest status 조회
- 후속 engine contract 정의
- 향후 streaming ingest 또는 bidirectional stream 확장

MVP에서는 HTTP + 큐 조합으로 시작하고, 엔진 간 계약이 안정화되면 gRPC proto를 정식화하는 방식도 가능하다.

### 5.4 FFmpeg / ffprobe

FFmpeg는 오디오 ingest의 핵심 의존성이다.

주요 사용처는 다음과 같다.

- 다양한 입력 포맷 디코딩
- sample rate 변환
- mono/stereo channel 변환
- PCM WAV 생성
- clipping 방지용 filter 적용
- loudness normalization 후보 분석
- duration 및 stream metadata 추출

`ffprobe`는 파일 검증과 메타데이터 추출에 사용한다.

주의할 점:

- 파일 확장자를 신뢰하지 않는다.
- `ffprobe` 결과를 기준으로 실제 format/codec을 판단한다.
- FFmpeg 실행에는 timeout을 둔다.
- 처리 가능한 duration/file size 상한을 둔다.
- subprocess stderr/stdout을 structured log로 남긴다.

### 5.5 torchaudio

`torchaudio`는 PyTorch 기반 오디오 처리를 위해 적합하지만 MVP 필수 의존성에서는 제외한다.

사용처:

- waveform load
- resampling
- tensor 기반 feature 계산
- 후속 ML 모델 입력 형식과 정합성 유지

후속 엔진들이 PyTorch 계열 모델을 사용할 가능성이 높다면 `engine-voice-analysis`, `engine-voice-pitch` 같은 ML 엔진에서 도입하는 것이 더 적합하다. `engine-audio-ingest`는 FFmpeg, librosa, soundfile, NumPy 기반으로 가볍게 유지한다.

### 5.6 librosa

`librosa`는 전통적인 오디오 분석과 빠른 prototyping에 유용하다.

사용처:

- silence trim/split
- onset/energy 기반 segmentation 보조
- RMS, spectral feature 계산
- 분석 notebook/prototype

MVP에서는 `librosa.effects.split` 또는 유사한 energy threshold 기반 split을 사용해 발화 후보 구간을 생성할 수 있다.

### 5.7 soundfile/libsndfile

`soundfile`은 WAV/FLAC 등 PCM 계열 파일 read/write에 적합하다.

사용처:

- canonical wav 저장 검증
- chunk 파일 저장
- PCM sample precision 확인

단, MP3/M4A 등 codec/container 입력은 FFmpeg를 우선 사용한다.

### 5.8 VAD

MVP에서는 energy threshold 기반 무음/발화 분할로 시작할 수 있다.

확장 시 후보:

- WebRTC VAD: 가볍고 빠른 speech activity detection
- Silero VAD: CPU에서도 실용적인 neural VAD
- pyannote.audio: 더 정교한 speech segmentation/diarization 계열

Vox2Vocal의 초기 목표가 singing/voice pipeline이면, 일반 speech VAD만으로는 노래 발성 구간을 놓칠 수 있다. 따라서 MVP에서는 "speech"라는 이름보다 "active audio region" 또는 "voiced/sounded segment"로 정의하는 편이 안전하다.

## 6. 저장소 설계

### 6.1 객체 저장소

원본 오디오와 산출물은 PostgreSQL에 직접 넣지 않고 객체 저장소에 저장한다.

MVP에서는 MinIO 또는 S3-compatible object storage를 추천한다.

이유:

- S3-compatible API를 제공한다.
- 로컬 Kubernetes/minikube 환경에 붙이기 쉽다.
- 운영 환경에서 AWS S3 등으로 이전하기 쉽다.
- multipart upload, presigned URL 패턴으로 확장 가능하다.

권장 object key 예시:

```txt
audio-assets/{audio_asset_id}/original/{filename}
audio-assets/{audio_asset_id}/canonical/audio.wav
audio-assets/{audio_asset_id}/chunks/{chunk_index}.wav
audio-assets/{audio_asset_id}/manifest.json
```

### 6.2 Python Object Storage Client

Python object storage client는 `boto3`로 확정한다.

선택 이유:

- AWS S3 공식 Python SDK다.
- MinIO 같은 S3-compatible storage에도 endpoint URL 설정으로 연결할 수 있다.
- 추후 AWS S3로 이전할 때 코드 변경 폭이 작다.
- `upload_file`, `upload_fileobj` 기반 multipart/parallel upload를 활용할 수 있다.

`minio` Python SDK는 MVP 필수 의존성에서 제외한다.

제외 이유:

- MinIO 전용 개발 편의성은 좋지만 장기 provider 이식성은 `boto3`가 더 유리하다.
- 현재 엔진은 특정 MinIO 기능보다 S3-compatible object read/write가 더 중요하다.

### 6.3 PostgreSQL 메타데이터

PostgreSQL에는 파일 자체가 아니라 색인 가능한 메타데이터를 저장한다.

예시:

```txt
audio_assets
- id
- user_id
- source_object_key
- canonical_object_key
- manifest_object_key
- input_format
- input_codec
- duration_ms
- sample_rate
- channels
- status
- created_at
- updated_at

audio_ingest_jobs
- id
- audio_asset_id
- status
- attempt_count
- error_code
- error_message
- started_at
- finished_at
```

## 7. 큐/이벤트 선택

### 7.1 NATS JetStream

큐/이벤트 시스템은 NATS JetStream으로 확정한다.

선택 이유:

- `engine-audio-ingest` 이후 여러 엔진이 같은 이벤트를 독립적으로 소비해야 한다.
- durable consumer와 replay를 지원한다.
- 작업 큐와 이벤트 스트림을 하나의 메시징 계층으로 다룰 수 있다.
- Kafka보다 가볍고 Redis Streams보다 엔진 간 이벤트 파이프라인에 적합하다.
- Node/NestJS 서비스와 Python 엔진 모두에서 언어 중립적으로 사용할 수 있다.

권장 subject:

```txt
audio.ingest.requested
audio.ingest.started
audio.ingest.completed
audio.ingest.failed
audio.ingest.dead_letter
```

권장 stream:

```txt
VOX2VOCAL_AUDIO
subject pattern: audio.>
```

권장 consumer:

```txt
engine-audio-ingest-worker -> audio.ingest.requested
engine-voice-analysis -> audio.ingest.completed
engine-evaluation -> audio.ingest.completed
engine-safety-rights -> audio.ingest.completed
```

### 7.2 Celery + RabbitMQ

Python worker 중심의 장시간 작업 큐만 고려하면 Celery + RabbitMQ도 좋은 후보였다.

장점:

- Python task queue로 성숙함
- retry/backoff/time limit 지원
- worker concurrency 관리가 쉬움
- RabbitMQ durable queue, manual ack, dead-letter exchange를 활용할 수 있음

단점:

- 이벤트 스트림보다는 task queue에 가깝다.
- Node/NestJS 서비스와의 언어 중립 이벤트 계약은 NATS보다 약하다.
- 후속 엔진 fan-out/replay 모델은 NATS JetStream보다 덜 자연스럽다.

### 7.3 Redis Streams

Redis Streams는 기존 Redis 인프라를 재사용할 수 있는 후보였다.

장점:

- 기존 Redis 인프라 재사용
- consumer group 지원
- pending message 재처리 가능
- Python/Node 양쪽에서 접근 가능

단점:

- retry/DLQ/replay 운영 패턴을 직접 설계해야 한다.
- 장기적으로 여러 엔진이 독립 consumer로 이벤트를 소비하는 구조는 NATS JetStream이 더 적합하다.

### 7.4 Kafka

Kafka는 초기 MVP에는 과하다.

다음 조건이 생기면 검토한다.

- ingest/event traffic이 매우 커진다.
- 장기 이벤트 보관과 replay가 핵심 요구사항이 된다.
- 여러 consumer group이 동일 이벤트를 독립적으로 대량 처리한다.
- 데이터 플랫폼/분석 파이프라인과 직접 연결해야 한다.

## 8. Manifest 설계

후속 엔진은 raw DB보다 manifest를 기준으로 오디오 자산을 읽는 것이 좋다.

예시:

```json
{
  "audio_asset_id": "aud_123",
  "source": {
    "object_key": "audio-assets/aud_123/original/input.mp3",
    "format": "mp3",
    "codec": "mp3",
    "duration_ms": 182340
  },
  "canonical": {
    "object_key": "audio-assets/aud_123/canonical/audio.wav",
    "format": "wav",
    "sample_rate": 48000,
    "channels": 1,
    "sample_format": "pcm_s16le",
    "duration_ms": 182340
  },
  "analysis": {
    "peak_dbfs": -1.2,
    "rms_dbfs": -18.4,
    "clipping_detected": false,
    "silence_ratio": 0.18
  },
  "segments": [
    {
      "index": 0,
      "start_ms": 320,
      "end_ms": 5210,
      "type": "active_audio"
    }
  ],
  "chunks": [
    {
      "index": 0,
      "object_key": "audio-assets/aud_123/chunks/0000.wav",
      "start_ms": 0,
      "end_ms": 30000
    }
  ]
}
```

## 9. MVP 기능 범위

### 9.1 Must

- `wav`, `mp3` 입력 지원
- `ffprobe` 기반 파일 검증
- 48kHz mono WAV 생성
- duration/sample rate/channel/codec 메타데이터 생성
- peak/RMS/clipping 기본 분석
- energy threshold 기반 무음/active audio 구간 생성
- 30초 단위 chunk 생성
- manifest JSON 생성
- 객체 저장소 저장
- ingest status 관리
- 실패 시 retry 가능
- structured logging

### 9.2 Should

- `m4a`, `flac` 입력 지원
- S3/MinIO presigned upload
- NATS JetStream durable consumer
- OpenTelemetry trace id propagation
- Prometheus metrics
- upload file hash 기록
- idempotency key 지원

### 9.3 Could

- tus resumable upload
- Silero VAD
- loudness normalization
- stereo 보존 옵션
- waveform preview image 생성
- low-resolution preview audio 생성

### 9.4 Won't

- pitch extraction
- phoneme alignment
- voice conversion
- singing synthesis
- vocoder rendering
- 권리/동의 최종 판단

## 10. 비기능 요구사항

### 10.1 성능

MVP 기준 권장 목표:

- 5분 이하 오디오 처리 성공
- 5분 오디오를 30~60초 내 처리
- real-time factor 0.2 이하 목표
- 100MB 이하 파일 안정 처리

### 10.2 안정성

- job retry/backoff
- 동일 파일 중복 처리 방지
- worker crash 시 pending job 재처리
- FFmpeg timeout
- 파일 size/duration 제한
- 객체 저장소 업로드 완료 여부 확인

### 10.3 보안

- 파일 확장자만 신뢰하지 않음
- MIME type과 ffprobe 결과를 함께 확인
- 파일명 sanitize
- 임시 파일 directory 격리
- FFmpeg command argument escaping
- 사용자별 object key 접근 제어
- 원본 hash 저장
- 감사 로그 저장

### 10.4 관측성

필수 metric:

- ingest requested count
- ingest success/failure count
- ingest duration
- queue lag
- FFmpeg failure count
- file size distribution
- audio duration distribution
- real-time factor
- worker memory/CPU

필수 log field:

- `trace_id`
- `audio_asset_id`
- `job_id`
- `user_id`
- `object_key`
- `stage`
- `duration_ms`
- `error_code`

## 11. 구현 단계 제안

### Phase 1: Local Prototype

- Python 프로젝트 생성
- FFmpeg/ffprobe wrapper 작성
- 로컬 파일 입력 -> canonical WAV 출력
- manifest JSON 생성
- unit test용 sample audio 추가

### Phase 2: Engine Service MVP

- FastAPI health/status endpoint
- NATS JetStream worker 연결
- MinIO/S3 object storage 연결
- Dockerfile 작성
- Kubernetes deployment 초안 작성

### Phase 3: Platform Integration

- api-gateway와 ingest job 생성 계약 정의
- `audio_asset_id` 발급 흐름 확정
- upload session API 설계
- 후속 `engine-voice-analysis`로 `audio.ingested` 이벤트 발행

### Phase 4: Production Hardening

- retry/backoff/idempotency
- OpenTelemetry/Prometheus
- FFmpeg timeout/resource limit
- integration test
- load test
- security checklist

## 12. 기술 선택 최종안

MVP 최종 추천안:

```txt
Language: Python 3.12+
API: FastAPI
Internal Contract: HTTP first, gRPC proto later or parallel
Audio Decode: FFmpeg / ffprobe
Audio Processing: librosa + soundfile + NumPy
Object Storage: MinIO/S3-compatible layout
Object Storage Client: boto3
Queue/Event: NATS JetStream
MVP Excluded: torchaudio, torch, minio Python SDK, celery, rabbitmq client, redis client
Database: PostgreSQL
Runtime: Docker
Orchestration: Kubernetes
Observability: OpenTelemetry + Prometheus
```

이 구성이 가장 좋은 이유:

- 현재 확정된 NATS JetStream/PostgreSQL/Kubernetes 기반 구조와 잘 맞는다.
- Python 오디오 분석 생태계를 바로 활용할 수 있고, PyTorch/torchaudio는 후속 ML 엔진에서 별도로 도입할 수 있다.
- FFmpeg로 입력 포맷 문제를 안정적으로 흡수할 수 있다.
- 객체 저장소 기반 구조라 대용량 파일 처리와 재처리에 유리하다.
- 후속 엔진들이 동일한 manifest와 canonical audio를 기준으로 동작할 수 있다.

## 13. 참고 자료

- FFmpeg Documentation: https://www.ffmpeg.org/documentation.html
- FFmpeg Filters Documentation: https://www.ffmpeg.org/ffmpeg-filters.html
- GStreamer Application Development Manual: https://gstreamer.freedesktop.org/documentation/application-development/introduction/gstreamer.html
- gRPC Core Concepts: https://grpc.io/docs/what-is-grpc/core-concepts/
- FastAPI Background Tasks: https://fastapi.tiangolo.com/tutorial/background-tasks/
- AWS S3 Multipart Upload: https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- MinIO S3 API Compatibility: https://docs.min.io/enterprise/aistor-object-store/developers/s3-api-compatibility/
- NATS JetStream Concepts: https://docs.nats.io/nats-concepts/jetstream
- Redis Streams Documentation: https://redis.io/docs/latest/develop/data-types/streams/
- Apache Kafka Documentation: https://kafka.apache.org/documentation/
- torchaudio Resampling Tutorial: https://docs.pytorch.org/audio/stable/tutorials/audio_resampling_tutorial.html
- librosa Effects API: https://librosa.org/doc/main/effects.html
- OpenTelemetry Documentation: https://opentelemetry.io/docs/
