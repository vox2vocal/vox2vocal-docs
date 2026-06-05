# Phoneme Alignment Engine

## 목적

Phoneme Alignment Engine은 입력 음성과 텍스트, 음절, 음소를 시간축 위에 정렬한다.

한국어 보컬에서는 모음 길이, 받침, 연음, 된소리, 숨표 처리가 자연스러움에 큰 영향을 주므로 별도 엔진으로 분리한다.

## 입력

- 표준화된 오디오
- 가사 또는 대본 텍스트
- 언어 코드
- 선택 입력: 발음 사전, 사용자 수정 타임라인

## 출력

- syllable timeline
- phoneme timeline
- word boundary
- alignment confidence
- pronunciation variant

## 주요 기능

- 텍스트 정규화
- grapheme-to-phoneme 변환
- 강제 정렬
- 음절/음소별 시작 및 종료 시각 추정
- 한국어 발음 변이 처리

## MVP 범위

- 한국어 음절 단위 정렬
- 단어 boundary 추출
- 음소 타임라인은 선택 기능으로 설계
- alignment confidence 제공

## 연결 엔진

- 이전 단계: `engine-audio-ingest`
- 다음 단계: `engine-rhythm-timing`, `engine-melody-mapping`, `engine-singing-synthesis`
