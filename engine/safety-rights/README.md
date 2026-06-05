# Safety Rights Engine

## 목적

Safety Rights Engine은 보이스 데이터, 보이스 모델, 변환 결과물의 사용 권한과 정책 준수 여부를 검증한다.

보이스 변환 시스템은 기술 품질만큼 권한 관리가 중요하므로, 이 엔진은 초기 MVP부터 반드시 포함한다.

## 입력

- user id
- source audio asset id
- target voice model id
- requested operation
- distribution intent

## 출력

- allow/deny decision
- policy reason
- consent record id
- audit log id
- required user action

## 주요 기능

- 보이스 모델 소유권 확인
- 동의 기록 확인
- 제3자 보이스 사용 차단
- 결과물 배포 가능 여부 확인
- 감사 로그 저장
- 민감한 사용 패턴 탐지

## MVP 범위

- 사용자 본인 업로드 보이스만 허용
- target voice model 소유권 확인
- 모든 conversion 요청 감사 로그 기록
- 권한 없는 모델 사용 차단

## 연결 엔진

- 호출 시점: 업로드, 보이스 모델 선택, voice conversion, export
- 보호 대상: `engine-voice-conversion`, `engine-singing-synthesis`, `engine-mix-master`
