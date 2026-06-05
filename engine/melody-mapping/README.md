# Melody Mapping Engine

## 목적

Melody Mapping Engine은 음성의 피치 커브 또는 사용자가 제공한 가이드 멜로디를 실제 노래 노트로 변환한다.

이 엔진은 말 억양을 그대로 노래로 만들지, 특정 키와 스케일에 맞게 보정할지, 사용자가 지정한 멜로디에 맞출지를 결정한다.

## 입력

- F0 pitch contour
- 음절 또는 음소 타임라인
- 선택 입력: BPM, key, scale, reference melody, MIDI

## 출력

- note sequence
- note start/end
- target pitch curve
- pitch correction map
- melody confidence

## 주요 기능

- 피치 커브를 노트 단위로 분절
- 키와 스케일 기반 음정 보정
- MIDI 또는 가이드 멜로디 매핑
- 음절별 대표 노트 결정
- 포르타멘토 구간 생성

## MVP 범위

- pitch frame을 MIDI note로 변환
- 음절 단위 대표 음정 산출
- C major 또는 사용자 지정 key 기준 보정
- note sequence JSON 출력

## 연결 엔진

- 이전 단계: `voice-pitch-engine`, `phoneme-alignment-engine`
- 다음 단계: `singing-synthesis-engine`, `expression-engine`
