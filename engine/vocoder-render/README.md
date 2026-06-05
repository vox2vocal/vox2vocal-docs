# Vocoder Render Engine

## 목적

Vocoder Render Engine은 Singing Synthesis Engine 또는 Voice Conversion Engine이 만든 음향 특징을 실제 오디오 waveform으로 렌더링한다.

이 엔진은 지연 시간, 음질, 잡음, 고음역 안정성, breath 표현에 직접적인 영향을 준다.

## 입력

- mel spectrogram 또는 acoustic feature
- pitch curve
- aperiodicity 또는 noise feature
- render quality preset

## 출력

- vocal waveform
- render metadata
- loudness estimate
- clipping report

## 주요 기능

- neural vocoder 렌더링
- WORLD/DDSP 계열 렌더링 실험
- 고품질 오프라인 렌더
- 저지연 프리뷰 렌더
- clipping 및 artifact 탐지

## MVP 범위

- offline render 우선
- 단일 샘플레이트 출력
- wav 파일 생성
- clipping 여부 리포트

## 연결 엔진

- 이전 단계: `engine-singing-synthesis`, `engine-voice-conversion`
- 다음 단계: `engine-expression`, `engine-mix-master`, `engine-evaluation`
