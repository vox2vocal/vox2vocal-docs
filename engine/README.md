# Vox2Vocal Engine Architecture

Vox2Vocal의 최종 목표는 일반 음성 또는 가이드 보이스를 음악적으로 자연스러운 보컬 트랙으로 변환하는 것이다.

이를 위해 시스템을 하나의 거대한 모델로 만들기보다, 분석, 정렬, 변환, 합성, 렌더링, 평가, 권한 검증 엔진으로 나누어 구축한다. 각 엔진은 독립적으로 개선할 수 있어야 하며, 파이프라인 전체에서는 공통 오디오 메타데이터와 타임라인 데이터를 공유한다.

## 권장 파이프라인

```txt
Audio Ingest
  -> Voice Analysis
  -> Voice Pitch
  -> Rhythm Timing
  -> Phoneme Alignment
  -> Melody Mapping
  -> Voice Conversion
  -> Singing Synthesis
  -> Vocoder Render
  -> Expression
  -> Mix Master
  -> Evaluation
  -> Safety Rights
```

`Safety Rights`는 마지막 단계만 담당하는 엔진이 아니라, 입력 업로드, 모델 선택, 보이스 변환, 결과 배포 시점마다 호출되는 정책 엔진으로 설계한다.

## 엔진 목록

| 엔진 | 역할 |
| --- | --- |
| [Audio Ingest](./audio-ingest/README.md) | 입력 오디오 표준화, 노이즈 처리, 구간 분할 |
| [Voice Analysis](./voice-analysis/README.md) | 말투, 에너지, 억양, 발화 구조 분석 |
| [Voice Pitch](./voice-pitch/README.md) | F0 추정, 피치 커브 정제, 음정 후보 생성 |
| [Melody Mapping](./melody-mapping/README.md) | 피치 커브를 노트, 스케일, 멜로디로 변환 |
| [Rhythm Timing](./rhythm-timing/README.md) | 음절 길이, 박자, 템포, 타이밍 보정 |
| [Phoneme Alignment](./phoneme-alignment/README.md) | 텍스트, 음절, 음소, 오디오 타임라인 정렬 |
| [Voice Conversion](./voice-conversion/README.md) | 목표 보이스 톤 또는 사용자 음색 변환 |
| [Singing Synthesis](./singing-synthesis/README.md) | 노래하는 음성의 음향 특징 생성 |
| [Vocoder Render](./vocoder-render/README.md) | 음향 특징을 실제 waveform으로 렌더링 |
| [Expression](./expression/README.md) | 비브라토, 호흡, 강세, 감정 표현 제어 |
| [Mix Master](./mix-master/README.md) | EQ, 컴프레션, 공간계, 라우드니스 후처리 |
| [Evaluation](./evaluation/README.md) | 음정, 박자, 발음, 자연스러움 품질 평가 |
| [Safety Rights](./safety-rights/README.md) | 보이스 권한, 동의, 정책, 감사 로그 관리 |

## MVP 구축 순서

엔진 목록과 구축 순서는 같은 엔진 집합을 기준으로 한다. 다만 모든 엔진을 같은 깊이로 구현하지 않고, 초기에는 최소 기능으로 연결한 뒤 단계적으로 고도화한다.

| 순서 | 엔진 | MVP 단계 | 초기 구현 범위 |
| --- | --- | --- | --- |
| 1 | `engine-audio-ingest` | 1차 MVP | 입력 오디오 표준화, 무음 구간 탐지 |
| 2 | `engine-voice-analysis` | 1차 MVP | 에너지, 발화 속도, 휴지 구간 분석 |
| 3 | `engine-voice-pitch` | 1차 MVP | F0 추출, confidence 필터링, MIDI 변환 |
| 4 | `engine-phoneme-alignment` | 1차 MVP | 한국어 음절 단위 정렬 |
| 5 | `engine-rhythm-timing` | 1차 MVP | BPM grid 기반 음절 타이밍 보정 |
| 6 | `engine-melody-mapping` | 1차 MVP | pitch contour를 note sequence로 변환 |
| 7 | `engine-singing-synthesis` | 1차 MVP | 단일 보이스 기반 짧은 보컬 phrase 생성 |
| 8 | `engine-vocoder-render` | 1차 MVP | mel/acoustic feature를 wav로 렌더링 |
| 9 | `engine-mix-master` | 1차 MVP | 기본 EQ, compressor, limiter 후처리 |
| 10 | `engine-evaluation` | 운영 필수 | pitch, timing, clipping 리포트 |
| 11 | `engine-safety-rights` | 운영 필수 | 보이스 권한 확인, 감사 로그 기록 |
| 12 | `engine-expression` | 2차 확장 | 비브라토, 호흡, 강세, release 제어 |
| 13 | `engine-voice-conversion` | 2차 확장 | 권한 있는 target voice로 timbre 변환 |

초기 MVP는 13개 엔진을 모두 동일한 수준으로 완성하는 것이 아니라, 1차 MVP 엔진을 먼저 end-to-end로 연결하고 `engine-evaluation`과 `engine-safety-rights`는 운영 안전장치로 최소 기능부터 포함한다. `engine-expression`과 `engine-voice-conversion`은 품질과 제품 리스크가 큰 영역이므로 기본 보컬 생성 흐름이 안정화된 뒤 확장한다.
