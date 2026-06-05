# Expression Engine

## 목적

Expression Engine은 보컬이 기계적으로 들리지 않도록 비브라토, 호흡, 강세, 성량 변화, 끝음 처리 같은 표현 요소를 제어한다.

이 엔진은 합성 전 파라미터를 만들 수도 있고, 렌더링 후 오디오 후처리로 일부 표현을 보강할 수도 있다.

## 입력

- note sequence
- pitch curve
- energy curve
- phoneme timeline
- expression preset
- user control values

## 출력

- vibrato instruction
- dynamics curve
- breath placement
- onset/offset expression
- updated synthesis controls

## 주요 기능

- 비브라토 생성 및 제어
- crescendo/decrescendo 곡선 생성
- 발음 시작점 accent 제어
- 끝음 release 제어
- 숨소리 위치 추천

## MVP 범위

- note 길이 기반 기본 비브라토
- 음절 강세 기반 dynamics curve
- phrase 끝 release 처리
- preset 기반 표현 제어

## 연결 엔진

- 이전 단계: `voice-analysis-engine`, `melody-mapping-engine`, `singing-synthesis-engine`
- 다음 단계: `singing-synthesis-engine`, `vocoder-render-engine`, `mix-master-engine`
