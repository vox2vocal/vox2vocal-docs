# Voice Pitch Engine

## 목적

Voice Pitch Engine은 입력 음성 또는 보컬에서 기본 주파수인 F0를 추정하고, 노래 변환에 사용할 수 있는 피치 커브로 정제한다.

현재 생성한 `engine-voice-pitch` 저장소가 이 엔진의 구현 위치가 된다.

## 입력

- 표준화된 오디오
- 발화 또는 보컬 구간 타임스탬프
- 선택 입력: 예상 성역, 키, 스케일, 보컬 타입

## 출력

- 프레임 단위 F0 값
- voiced/unvoiced 구간
- confidence score
- smoothing된 pitch contour
- note candidate

## 주요 기능

- F0 추정
- 무성음 구간 제거
- 튀는 피치 값 보정
- 피치 커브 smoothing
- 피치에서 음정 후보 생성
- 가이드 멜로디와 pitch contour 비교

## MVP 범위

- 짧은 monophonic voice 입력 기준 F0 추출
- confidence 기반 필터링
- MIDI note number 변환
- JSON 결과 출력

## 연결 엔진

- 이전 단계: `engine-voice-analysis`
- 다음 단계: `engine-melody-mapping`, `engine-rhythm-timing`, `engine-singing-synthesis`

## 데이터 예시

```json
{
  "frames": [
    {
      "timeSec": 0.12,
      "f0Hz": 220.0,
      "midi": 57,
      "confidence": 0.91,
      "voiced": true
    }
  ]
}
```
