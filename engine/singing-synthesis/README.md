# Singing Synthesis Engine

## 목적

Singing Synthesis Engine은 음소, 노트, 타이밍, 피치, 표현 정보를 받아 노래하는 음성의 음향 특징을 생성한다.

Vox2Vocal의 핵심 생성 엔진이며, 최종 보컬 품질을 결정하는 중심 모듈이다.

## 입력

- phoneme timeline
- note sequence
- target pitch curve
- rhythm timeline
- speaker or voice model id
- expression controls

## 출력

- acoustic feature
- mel spectrogram
- refined pitch curve
- synthesis metadata

## 주요 기능

- 음소 기반 보컬 발음 생성
- 노트별 sustain 처리
- pitch transition 생성
- speaker timbre 반영
- expression control 반영

## MVP 범위

- 단일 보이스 모델
- 짧은 monophonic vocal phrase 생성
- pitch와 duration 입력 기반 mel spectrogram 생성
- engine-vocoder-render으로 넘길 표준 feature 출력

## 연결 엔진

- 이전 단계: `engine-melody-mapping`, `engine-rhythm-timing`, `engine-phoneme-alignment`
- 다음 단계: `engine-vocoder-render`, `engine-expression`, `engine-evaluation`
