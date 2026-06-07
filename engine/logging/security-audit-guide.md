# Engine Security Audit Guide

이 문서는 보안감사, 침해 조사, 권한/동의 분쟁 대응을 위해 엔진 로그와 audit 데이터를 어떻게 준비해야 하는지 정의한다.

## 보안감사의 핵심 질문

감사자는 보통 아래를 확인한다.

- 보이스 데이터에 누가 접근할 수 있는가?
- 사용자는 본인 또는 권한 있는 보이스만 사용할 수 있는가?
- 동의가 철회되면 이후 사용이 차단되는가?
- 관리자 행위가 추적되는가?
- 로그와 audit event가 변조되지 않았음을 증명할 수 있는가?
- 사고 발생 시 영향을 받은 사용자, 오디오, 모델, 결과물을 추적할 수 있는가?
- 민감정보가 로그에 노출되지 않는가?

## 감사 증적 패키지

감사 요청이 오면 아래 패키지를 준비할 수 있어야 한다.

| 증적 | 내용 |
| --- | --- |
| log policy | 로그 생성, 수집, 보관, 폐기 정책 |
| audit event catalog | 어떤 행위를 audit하는지 |
| access matrix | 역할별 로그/audit 접근 권한 |
| retention policy | 보관 기간과 삭제 기준 |
| redaction policy | 로그에 남기지 않는 데이터 |
| consent lifecycle | 동의 생성, 변경, 철회 흐름 |
| voice model lineage | 모델 생성, 버전, 배포, 소유권 |
| incident runbook | 탐지, 대응, 복구, 보고 절차 |
| integrity report | audit hash/digest 검증 결과 |
| access review | 정기 권한 검토 기록 |
| deployment record | 엔진 버전, 모델 버전, config 변경 기록 |

## 보안감사 대응 절차

1. 감사 범위와 기간을 확정한다.
2. 관련 `user_id`, `audio_asset_id`, `voice_model_id`, `trace_id`를 식별한다.
3. audit store에서 관련 event를 조회한다.
4. operational log에서 처리 흐름을 재구성한다.
5. policy version과 consent record를 연결한다.
6. 관리자 접근 또는 수동 조치가 있었는지 확인한다.
7. audit integrity digest를 검증한다.
8. 민감정보가 포함되지 않은 export package를 만든다.
9. 조회 및 export 행위 자체를 audit한다.

## Abuse Detection

엔진 로그와 audit event를 이용해 아래 이상 징후를 탐지한다.

| 신호 | 가능성 |
| --- | --- |
| 특정 user의 denied 급증 | 권한 없는 보이스 사용 시도 |
| 특정 voice model 접근 급증 | 모델 탈취 또는 scraping |
| output export 급증 | 계정 탈취 또는 bulk extraction |
| consent revoked 이후 사용 시도 | 정책 우회 또는 stale cache |
| audit write failed | 증적 회피 또는 시스템 장애 |
| 관리자 조회 급증 | 내부자 위협 가능성 |
| 동일 audio hash 반복 업로드 | abuse 또는 재처리 공격 |
| 너무 많은 failed conversion | probing 또는 resource abuse |

## 관리자 행위 통제

관리자 기능은 일반 사용자 기능보다 강한 audit이 필요하다.

필수 기록:

- 관리자 로그인
- 권한 상승
- 사용자 데이터 조회
- audit event 조회
- audit export
- voice model 강제 비활성화
- consent 상태 수동 변경
- policy 변경
- retention 변경

권장 통제:

- 관리자 MFA
- break-glass 계정 분리
- 관리자 세션 사유 입력
- 2인 승인 대상 작업 정의
- 정기 access review

## 민감정보 보호

보안감사 관점에서 가장 위험한 실수는 로그가 새로운 데이터 유출 경로가 되는 것이다.

금지:

- 원본 오디오 payload
- 전체 가사/대본
- 사용자 이메일/전화번호
- access token
- refresh token
- S3 signed URL
- 내부 secret
- consent 문서 원문
- 모델 weight 경로 또는 credential

대체:

- 내부 ID
- digest
- masked value
- normalized object key
- consent record id
- policy decision id

## 침해 조사 타임라인

침해 의심 시 최소 타임라인:

```txt
T0 first suspicious event
T1 first denied/failed security event
T2 first successful unauthorized action if any
T3 affected audio/model/output identified
T4 containment action
T5 user/admin notification decision
T6 remediation completed
T7 postmortem and control update
```

조사에는 반드시 아래 ID를 연결한다.

```txt
trace_id
job_id
user_id
audio_asset_id
voice_model_id
consent_record_id
policy_decision_id
audit_event_id
```

## 보안감사 체크리스트

- 모든 `engine-*` 로그에 `trace_id`가 있는가?
- audit event가 운영 로그와 분리되어 있는가?
- audit event 저장소가 append-only인가?
- audit 조회 행위도 audit되는가?
- consent 철회 후 사용 차단 evidence가 있는가?
- policy 변경 history가 있는가?
- 관리자 접근 기록이 있는가?
- 로그 redaction 테스트가 있는가?
- log injection 테스트가 있는가?
- CRITICAL/security alert가 운영자에게 전달되는가?
- audit integrity 검증 리포트가 있는가?
- 로그 보관 및 삭제 정책이 문서화되어 있는가?

## 놓치기 쉬운 부분

프로젝트에서 특히 놓치기 쉬운 항목:

- 운영 로그와 audit 데이터를 같은 것으로 취급하는 것
- audit 조회 행위 자체를 기록하지 않는 것
- 보이스 모델 lineage를 남기지 않는 것
- consent 철회 후 cache 무효화 증적이 없는 것
- `policy_version` 없이 allowed/denied만 남기는 것
- `config_hash` 없이 모델 결과를 재현하려는 것
- log pipeline 장애 시 fail-open/fail-closed 기준이 없는 것
- 관리자 수동 조치 사유를 남기지 않는 것
- 로그 비용 때문에 WARN/ERROR 보관 기준을 뒤늦게 바꾸는 것
