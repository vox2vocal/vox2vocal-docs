# Rhythm Timing Engine

## 목적

Rhythm Timing Engine은 음성의 음절과 발화 길이를 음악적 박자에 맞게 정렬한다.

보컬처럼 들리기 위해서는 음정뿐 아니라 음절 시작점, 끝점, 쉼표, 긴장감 있는 박자 배치가 필요하다.

## 입력

- 발화 구간 타임라인
- 음절 또는 음소 타임라인
- BPM, beat grid
- 선택 입력: swing, quantize strength, reference rhythm

## 출력

- beat-aligned syllable timeline
- note duration
- rest segment
- time-stretch instruction

## 주요 기능

- 음절 길이 추정
- BPM 기반 beat grid 생성
- 음절 시작점 quantize
- 발음이 깨지지 않는 time-stretch 범위 계산
- 쉼표와 호흡 위치 배치

## MVP 범위

- 고정 BPM 기준 음절 시작점 정렬
- 1/4, 1/8, 1/16 note grid 지원
- 과도한 stretch 방지 규칙
- melody mapping용 duration 생성

## 연결 엔진

- 이전 단계: `engine-voice-analysis`, `engine-phoneme-alignment`
- 다음 단계: `engine-melody-mapping`, `engine-singing-synthesis`
