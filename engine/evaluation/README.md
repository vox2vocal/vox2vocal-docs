# Evaluation Engine

## 목적

Evaluation Engine은 생성된 보컬의 품질을 자동으로 점검한다.

초기에는 개발자용 리포트로 시작하고, 이후 사용자에게 음정 정확도, 박자 정확도, 발음 선명도 같은 피드백을 제공하는 기능으로 확장한다.

## 입력

- generated vocal waveform
- target note sequence
- phoneme timeline
- reference audio
- mix metadata

## 출력

- pitch accuracy score
- timing accuracy score
- pronunciation confidence
- artifact report
- overall quality score

## 주요 기능

- 목표 노트 대비 pitch deviation 측정
- beat grid 대비 timing deviation 측정
- clipping, noise, artifact 탐지
- 발음 정렬 confidence 확인
- 회귀 테스트용 품질 지표 저장

## MVP 범위

- pitch deviation 리포트
- timing deviation 리포트
- clipping 탐지
- 엔진별 결과 JSON 비교

## 연결 엔진

- 이전 단계: 모든 audio-producing engine
- 다음 단계: 사용자 피드백, QA, 모델 개선 파이프라인
