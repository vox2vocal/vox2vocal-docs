# Audio Ingest Engine

## 목적

Audio Ingest Engine은 사용자가 업로드한 음성, 노래, 가이드 멜로디, 반주 파일을 Vox2Vocal 내부 표준 포맷으로 변환하는 진입 엔진이다.

이 엔진의 품질이 낮으면 이후 피치 추정, 음소 정렬, 보컬 합성 결과가 모두 흔들리므로, 단순 파일 업로드 처리보다 오디오 품질 정규화 계층으로 설계한다.

## 입력

- 사용자 음성 파일: `wav`, `mp3`, `m4a`, `flac`
- 가이드 멜로디 또는 레퍼런스 보컬
- 선택 입력: BPM, 키, 언어, 가사 텍스트

## 출력

- 표준 PCM 오디오
- 샘플레이트, 채널, 길이, 라우드니스 메타데이터
- 무음 구간, 발화 구간, 노이즈 추정 정보
- 후속 엔진이 참조할 `audio_asset_id`

## 주요 기능

- 샘플레이트 표준화
- 모노/스테레오 채널 정리
- 피크 및 라우드니스 정규화
- 노이즈 프로파일 추정
- 무음 제거 및 발화 구간 분할
- 긴 오디오의 청크 분리

## MVP 범위

- `wav`, `mp3` 입력 지원
- 44.1kHz 또는 48kHz 기준 표준화
- mono PCM 변환
- 무음 구간 탐지
- 발화 구간별 타임스탬프 생성

## 현재 구현 상태 (2026-06-14)

`engine-audio-ingest` repo 기준 현재 완료된 구현은 다음이다.

- FastAPI app factory와 `GET /health`
- Docker image runtime에 `ffmpeg`, `ffprobe`, `libsndfile1` 포함
- NATS JetStream stream 확인/생성 골격
- `audio.ingest.requested` durable consumer 골격
- requested/completed/failed 이벤트 payload 모델
- `audio.ingest.requested` payload validation
- local filesystem storage adapter
- `ffprobe` metadata wrapper
- `ffmpeg` canonical WAV 변환 wrapper

검증된 상태:

- unit/integration test: `51 passed`
- Ruff: `All checks passed`
- Docker image: `vox2vocal/engine-audio-ingest:local`
- Minikube deployment: `engine-audio-ingest` `1/1 Running`
- pod 내부 `convert_to_canonical_wav` import와 `ffmpeg` 실행 확인

아직 남은 구현:

- manifest JSON 생성
- requested event를 ffprobe/ffmpeg/manifest 처리 흐름에 연결
- completed/failed 이벤트 발행
- Kubernetes/NATS 통합 메시지 처리 검증

## 연결 엔진

- 다음 단계: `engine-voice-analysis`
- 보조 단계: `engine-evaluation`, `engine-safety-rights`
