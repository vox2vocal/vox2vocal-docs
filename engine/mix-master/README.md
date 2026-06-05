# Mix Master Engine

## 목적

Mix Master Engine은 생성된 보컬을 사용자가 바로 들을 수 있는 형태로 다듬는다.

보컬 합성 모델이 좋은 소리를 만들더라도 EQ, 컴프레션, 디에서, 공간계 처리가 없으면 실제 음악 안에서 완성도가 낮게 느껴질 수 있다.

## 입력

- vocal waveform
- optional backing track
- target loudness
- mix preset

## 출력

- processed vocal track
- preview mix
- mastering report
- loudness metadata

## 주요 기능

- EQ
- compressor
- de-esser
- reverb
- delay
- limiter
- loudness normalization
- 반주와 보컬 레벨 매칭

## MVP 범위

- vocal-only 후처리
- 기본 EQ, compressor, limiter
- wav 출력
- LUFS 또는 peak 기준 정규화

## 연결 엔진

- 이전 단계: `engine-vocoder-render`, `engine-expression`
- 다음 단계: `engine-evaluation`
