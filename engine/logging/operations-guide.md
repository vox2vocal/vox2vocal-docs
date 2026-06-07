# Engine Logging Operations Guide

이 문서는 운영 환경에서 `engine-*` 로그를 어떻게 수집, 저장, 검색, 알림, 보관할지 정의한다.

## 운영 목표

운영 로그는 아래 질문에 답해야 한다.

- 지금 어떤 엔진이 실패하고 있는가?
- 특정 `job_id`가 어느 단계에서 멈췄는가?
- 실패가 입력 품질 문제인가, 엔진 오류인가, 외부 인프라 문제인가?
- NATS, storage, model runtime, FFmpeg 같은 dependency 중 어디가 병목인가?
- 최근 배포 이후 실패율이나 처리 시간이 악화되었는가?
- 보이스 권한 거부나 abuse signal이 급증했는가?

## 권장 수집 구조

운영 환경에서 엔진은 파일 로그를 직접 관리하지 않는다.

```txt
engine-* container
  -> JSON stdout/stderr
  -> Kubernetes container log
  -> Fluent Bit or OpenTelemetry Collector
  -> Loki / OpenSearch / Elasticsearch
  -> Grafana / alerting / SIEM export
```

엔진 내부 파일 로그는 아래 문제가 있어 기본 금지한다.

- container 재시작 시 유실 가능성
- volume 관리 비용 증가
- 로그 수집 중복
- 권한/보관 정책 분산
- 장애 시 application I/O block 위험

## 환경별 정책

| 환경 | 로그 형식 | DEBUG 저장 | 보관 |
| --- | --- | --- | --- |
| local | pretty 또는 JSON | 허용 | 개발자 로컬 |
| dev | JSON | 제한 허용 | 3~7일 |
| staging | JSON | 기본 금지 | 7~14일 |
| prod | JSON | 금지 | 정책 기준 |

운영에서는 timestamp를 UTC로 통일한다.

## 로그 저장소 분리

| 저장소 | 대상 | 특징 |
| --- | --- | --- |
| operational log store | INFO/WARN/ERROR/CRITICAL | 검색과 대시보드 중심 |
| security log store | 권한 거부, 비정상 접근, abuse signal | 접근 통제 강화 |
| audit store | 권한/동의/관리자 행위 증적 | append-only, 장기 보관 |
| cold archive | 장기 보관 로그 | 비용 최적화, 조회 느림 |

## 대시보드

최소 대시보드:

### Engine Overview

- 엔진별 요청 수
- 엔진별 성공률/실패율
- p50/p95/p99 처리 시간
- retryable/non-retryable 실패 비율
- CRITICAL 로그 수

### Pipeline Trace

- `trace_id`별 단계 진행 상태
- `job_id`별 마지막 event
- 엔진 간 대기 시간
- NATS publish/consume 지연

### Audio Quality

- silence-only 비율
- clipping 비율
- low-volume 비율
- low confidence pitch frame 비율
- active segment 탐지 실패율

### Dependency Health

- NATS 연결 실패
- storage read/write 실패
- FFmpeg timeout
- model runtime load 실패
- GPU/CPU/memory 사용률

### Safety and Abuse

- policy denied 수
- consent missing 수
- voice model unauthorized access 수
- user별 과도한 실패/요청 패턴
- export/download 급증

## 알림 기준

모든 `ERROR`를 알림으로 보내지 않는다. 운영 영향 기준으로 알림을 건다.

| 조건 | 심각도 | 대응 |
| --- | --- | --- |
| `CRITICAL` 발생 | P1 | 즉시 호출 |
| audit write 실패 | P1 | 권한 관련 요청 중단 검토 |
| 5분 실패율 10% 초과 | P2 | 담당 엔진 확인 |
| NATS 연결 실패 3분 지속 | P1/P2 | 인프라 확인 |
| storage write 실패 3분 지속 | P1/P2 | 데이터 유실 위험 확인 |
| FFmpeg timeout 급증 | P2 | 입력 크기, 리소스, 배포 확인 |
| queue backlog 지속 증가 | P2 | worker scale-out 확인 |
| safety deny 급증 | P2/Security | abuse 또는 정책 변경 확인 |
| 특정 user export 급증 | Security | 계정 탈취 또는 scraping 확인 |

## 보관 기간

초기 권장값:

| 데이터 | 보관 |
| --- | --- |
| DEBUG | 운영 저장 금지 |
| INFO | 14일 |
| WARN | 30일 |
| ERROR/CRITICAL | 90일 |
| security log | 180일 이상 |
| audit event | 1년 이상 또는 정책/계약 기준 |
| integrity digest | audit event보다 길게 보관 |

보이스 권리, 동의, 관리자 행위 관련 audit은 일반 운영 로그보다 길게 보관한다.

## 로그 파이프라인 장애 정책

로그 수집기가 장애여도 일반 엔진 처리는 계속할 수 있다.

예외:

- audit write 실패
- safety decision 기록 실패
- 관리자 행위 기록 실패
- consent 변경 기록 실패

위 이벤트는 보안/권리 증적이므로 fail-closed를 기본으로 한다. 단, 긴급 운영 복구가 필요한 경우 수동 승인 절차와 사후 보강 audit을 남긴다.

## Incident Runbook

운영자가 장애를 조사할 때 순서:

1. dashboard에서 실패율이 증가한 엔진을 찾는다.
2. 대표 `trace_id` 또는 `job_id`를 선택한다.
3. pipeline event를 시간순으로 조회한다.
4. 마지막 성공 event와 첫 실패 event를 찾는다.
5. dependency log를 확인한다.
6. 최근 배포, config 변경, model version 변경 여부를 확인한다.
7. audit/security event와 관련이 있는지 확인한다.
8. 원인, 영향 범위, 복구 조치, 재발 방지 항목을 incident record에 남긴다.

## 운영자가 보면 안 되는 데이터

운영 로그 검색 권한이 있어도 아래 데이터는 조회되지 않아야 한다.

- 원본 음성 파일 내용
- 전체 가사/대본
- 사용자 인증 토큰
- consent 문서 원문
- 모델 weight
- 외부 storage credential

필요한 경우 식별자는 내부 ID 또는 digest로 대체한다.
