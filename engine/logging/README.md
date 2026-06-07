# Engine Logging and Audit Management

이 문서는 Vox2Vocal `engine-*` 계열의 로그, 운영 관측성, audit 데이터, 보안감사 대응 방식을 정의한다.

엔진 로그는 단순 디버깅 출력이 아니라, 작업 하나가 전체 보이스투보컬 파이프라인을 어떻게 통과했는지 증명하는 운영 자산이다. 따라서 개발 로그, 운영 로그, audit 데이터를 같은 저장소나 같은 정책으로 다루지 않는다.

## 빠른 결론

로그를 볼 때는 먼저 `level`보다 `log_domain`을 확인한다. `level`은 심각도이고, `log_domain`은 이 로그가 어떤 질문에 답하기 위한 것인지를 의미한다.

| Domain | 쉽게 말하면 | 답해야 하는 질문 |
| --- | --- | --- |
| `operational` | 엔진 상태 로그 | 엔진이 잘 살아있나? |
| `pipeline` | 작업 흐름 로그 | 이 작업이 어느 단계까지 갔나? |
| `quality` | 품질 로그 | 오디오나 모델 결과 품질이 괜찮나? |
| `security` | 보안 신호 로그 | 권한 없는 접근이나 이상행위가 있나? |
| `audit` | 증적 로그 | 나중에 증명해야 하는 행위인가? |
| `model` | 모델 재현성 로그 | 어떤 모델/설정으로 결과가 나왔나? |

저장 위치는 아래 기준으로 고정한다.

```txt
operational / pipeline / quality 로그
  -> engine stdout/stderr
  -> Kubernetes container log
  -> collector
  -> Loki
  -> Grafana

security 로그
  -> engine stdout/stderr with log_domain=security
  -> Kubernetes container log
  -> collector
  -> Loki security stream
  -> 이후 OpenSearch 또는 SIEM 확장

audit 데이터
  -> audit_writer
  -> PostgreSQL append-only audit_events table
  -> audit_daily_digests
  -> 이후 object storage / WORM archive
```

중요한 결정:

- `operational`, `pipeline`, `quality`, 일반 `model` 로그는 Loki에 저장한다.
- `security` 로그는 초기에는 Loki 안에서 security stream으로 분리하고, 이후 OpenSearch 또는 SIEM으로 확장한다.
- `audit` 데이터는 Loki에만 저장하지 않고 PostgreSQL append-only audit store에 저장한다.
- 엔진은 운영 로그를 파일로 직접 저장하지 않는다. 엔진은 JSON을 `stdout`/`stderr`로 출력하고, 저장은 collector와 log store가 담당한다.
- `audit_writer`는 일반 logger와 분리한다.

## 관리 원칙

| 영역 | 목적 | 핵심 기준 |
| --- | --- | --- |
| 개발 로그 | 로컬 디버깅, 테스트 재현 | 상세하지만 짧게 보관하고 민감정보는 금지 |
| 운영 로그 | 장애 탐지, 성능 추적, 품질 모니터링 | JSON stdout, 중앙 수집, 대시보드, 알림 |
| Audit 데이터 | 권한, 동의, 관리자 행위, 보이스 사용 증명 | append-only, 변조 방지, 장기 보관 |
| 보안감사 | 외부/내부 감사 대응, 침해 조사, 컴플라이언스 | 증적 패키지, 접근 통제, 무결성 검증 |

## 에이전트 읽기 순서

엔진 로깅을 구현하거나 수정하는 에이전트는 아래 순서로 문서를 읽는다.

1. [Development Direction](./development-direction.md)
2. [Log Domain Guide](./log-domain-guide.md)
3. [Storage Policy](./storage-policy.md)
4. [Development Guide](./development-guide.md)
5. [Engine Log Event Index](./engine-log-index.md)
6. [Operations Guide](./operations-guide.md)
7. [Audit Data Guide](./audit-data-guide.md)
8. [Security Audit Guide](./security-audit-guide.md)

작업 유형별 우선 문서:

| 작업 | 먼저 읽을 문서 |
| --- | --- |
| 로그 개발 스택/아키텍처 결정 | [Development Direction](./development-direction.md) |
| 로그 도메인 이해 | [Log Domain Guide](./log-domain-guide.md) |
| 로그 저장소와 보관 정책 결정 | [Storage Policy](./storage-policy.md) |
| 엔진 코드에 logger 추가 | [Development Guide](./development-guide.md) |
| 엔진별 필수 event 정의 | [Engine Log Event Index](./engine-log-index.md) |
| Kubernetes/Collector/대시보드 구성 | [Operations Guide](./operations-guide.md) |
| 권한/동의/보이스 사용 이력 저장 | [Audit Data Guide](./audit-data-guide.md) |
| 보안감사, 침해 대응, 증적 준비 | [Security Audit Guide](./security-audit-guide.md) |

## 필수 구현 계약

모든 `engine-*`는 아래 계약을 지킨다.

- 로그는 JSON structured log로 출력한다.
- 운영 환경에서는 파일 로그를 직접 쓰지 않고 `stdout`/`stderr`로 출력한다.
- 모든 pipeline log에는 `trace_id`, `job_id`, `audio_asset_id`, `service`, `event`, `schema_version`을 포함한다.
- 오류 로그에는 `error_code`, `retryable`, `error_type`을 포함한다.
- 모델 또는 알고리즘 결과에는 `model_version`, `config_version` 또는 `config_hash`를 포함한다.
- 원본 음성 내용, 전체 가사, 인증 토큰, 이메일, 전화번호, 비밀키는 로그에 남기지 않는다.
- audit event는 일반 운영 로그와 분리해서 저장한다.
- audit event 저장 실패가 권한/동의 판단에 영향을 주는 경우 fail-closed를 기본값으로 한다.

## 공통 로그 흐름

```txt
engine-* JSON stdout/stderr
  -> Kubernetes container log
  -> logging collector
  -> Loki
  -> dashboard / alert / trace correlation

security or rights decision
  -> audit event writer
  -> PostgreSQL append-only audit store
  -> integrity digest / retention / audit export
```

## 참고 기준

- [NIST SP 800-92: Guide to Computer Security Log Management](https://csrc.nist.gov/pubs/sp/800/92/final)
- [OWASP Logging Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [OpenTelemetry Logs](https://opentelemetry.io/docs/specs/otel/logs/)
- [Kubernetes Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
