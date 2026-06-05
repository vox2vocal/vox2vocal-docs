# Voice Conversion Engine

## 목적

Voice Conversion Engine은 입력 음성의 언어적 내용과 표현을 보존하면서 목표 보이스 톤으로 변환한다.

이 엔진은 사용자 본인 음색, 동의받은 보이스, 라이선스가 확보된 캐릭터 보이스만 대상으로 해야 한다.

## 입력

- source voice feature
- target speaker embedding 또는 target model id
- pitch contour
- phoneme timeline
- safety authorization token

## 출력

- converted acoustic feature
- speaker similarity score
- conversion confidence
- policy decision log id

## 주요 기능

- speaker embedding 추출
- source content feature 추출
- target timbre 적용
- pitch 보존 또는 보정
- 보이스 모델 권한 검증

## MVP 범위

- 사용자 본인 보이스 모델만 허용
- pitch contour 보존
- 짧은 문장 단위 conversion
- safety-rights-engine 권한 체크 필수화

## 연결 엔진

- 이전 단계: `phoneme-alignment-engine`, `voice-pitch-engine`, `safety-rights-engine`
- 다음 단계: `singing-synthesis-engine`, `vocoder-render-engine`
