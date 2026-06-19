# Vox2Vocal Agent Index

이 파일은 Vox2Vocal 작업을 시작하는 에이전트가 프로젝트 구조, 기술 스택, 관련 문서를 어떤 순서로 읽어야 하는지 안내한다.

## 1. 먼저 작업 범위를 정한다

작업 대상 디렉터리를 먼저 확인한다.

| 경로 | 역할 | 주요 스택 |
| --- | --- | --- |
| `app` | Expo React Native App/Web | Expo 56, React 19, React Native 0.85, Expo Router, Tamagui, React Query, Zustand, MMKV |
| `bff-server` | GraphQL BFF | NestJS 11, GraphQL, Apollo Server, class-validator, gRPC client |
| `api-gateway` | 내부 API Gateway | NestJS 11, gRPC server/client, `@nestjs/jwt`, RxJS, ConfigModule |
| `user-service` | 사용자 도메인 서비스 | NestJS 11, CQRS, gRPC, Prisma, PostgreSQL, Argon2 |
| `worker` | 비동기 작업 서버 | NestJS 11, BullMQ, Redis, ConfigModule |
| `infra` | 로컬/클러스터 인프라 | Kubernetes, minikube, PostgreSQL, Redis, Ingress |
| `docs` | 공통 문서 | 테스트, Prisma, 인프라, 워크스페이스, 엔진 아키텍처 가이드 |
| `vox2vocal-agent-skills` | Codex Agent Skill 저장소 | PM/TRD skills, PRD-to-TRD workflow, skill validation |
| `engine-audio-ingest` | 오디오 입력/전처리 엔진 | 오디오 표준화, 무음 탐지, 구간 분할 |
| `engine-voice-analysis` | 음성 분석 엔진 | 에너지, 발화 속도, 억양, 휴지 구간 분석 |
| `engine-voice-pitch` | 피치 분석 엔진 | F0 추정, pitch contour 정제, MIDI note 변환 |

작업이 여러 프로젝트를 건드리면 호출 흐름을 먼저 따라간다.

```txt
app
  -> GraphQL
bff-server
  -> gRPC
api-gateway
  -> gRPC
user-service
  -> PostgreSQL

worker
  <-> Redis Queue
```

보이스투보컬 엔진 계층은 아래 흐름을 기준으로 문서화한다.

```txt
engine-audio-ingest
  -> engine-voice-analysis
  -> engine-voice-pitch
  -> engine-phoneme-alignment
  -> engine-rhythm-timing
  -> engine-melody-mapping
  -> engine-singing-synthesis
  -> engine-vocoder-render
  -> engine-mix-master
```

운영 안전장치와 품질 검증은 `engine-safety-rights`, `engine-evaluation`을 통해 각 단계에서 호출되는 구조로 본다. 표현 확장과 음색 변환은 `engine-expression`, `engine-voice-conversion`에서 단계적으로 고도화한다.

## 2. 문서 인덱스와 읽는 순서

문서 작업 또는 구현 작업을 시작할 때는 먼저 아래 인덱스에서 관련 문서를 찾고, 그 다음 작업 유형별 읽는 순서를 따른다.

### 전체/운영 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [Workspace README](vox2vocal-docs/workspace/README.md) | workspace 구조, 서비스 역할, root AGENT 원본 위치를 확인할 때 |
| [Git Policy](vox2vocal-docs/workspace/git-policy.md) | Git identity, commit convention, hook, CI, ruleset, force push 정책을 확인할 때 |
| [Root Agent Source](vox2vocal-docs/root/AGENT.md) | 루트 `AGENT.md` 원본을 docs repo 기준으로 갱신할 때 |
| [Agent Skills README](vox2vocal-agent-skills/README.md) | PM/TRD Codex skill 목록과 저장소 구조를 확인할 때 |
| [Agent Skills Flow Guide](vox2vocal-agent-skills/prd-trd-skill-flow-guide.md) | PM-to-TRD skill 사용 순서와 prompt 흐름을 확인할 때 |
| [Agent Skills Agent](vox2vocal-agent-skills/AGENT.md) | skill repo 작업 규칙, validation, commit 기준을 확인할 때 |

### 제품/기획 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [MVP PRD](vox2vocal-docs/pm/vox2vocal-mvp-prd.md) | 제품 목표, 사용자 흐름, MVP 요구사항, 성공 기준을 확인할 때 |
| [MVP PRD Review](vox2vocal-docs/pm/vox2vocal-mvp-prd-review.md) | PRD의 누락, 모호성, 리스크를 확인할 때 |
| [MVP Feature Definition](vox2vocal-docs/pm/vox2vocal-mvp-feature-definition.md) | MVP 기능 범위, 상태 모델, 권리/동의/검토 요구사항을 구현 단위로 확인할 때 |
| [PMI Functional Spec Draft](vox2vocal-docs/pmi-functional-spec-draft.md) | 초기 기능 명세 초안 또는 제품 방향의 배경을 확인할 때 |

### 프론트엔드 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [Frontend Architecture](vox2vocal-docs/frontend/architecture.md) | app 폴더 구조, route/feature/shared 계층, 상태 관리 기준을 확인할 때 |
| [Design System Guide](vox2vocal-docs/frontend/design-system-guide.md) | 반응형 layout, token, component state, 접근성, QA 기준을 확인할 때 |
| [Frontend Design](vox2vocal-docs/frontend/design.md) | 브랜드 컬러, 인증 화면, 자산 사용 기준을 빠르게 확인할 때 |
| [App Agent](vox2vocal-app/AGENTS.md) | Expo app 작업 전 버전별 Expo 지침과 task flow를 확인할 때 |

### 테스트/품질 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [User Service Testing Guide](vox2vocal-docs/testing/user-service-testing-guide.md) | user-service 테스트 경계, Prisma/repository/handler 테스트를 작성할 때 |
| [API Gateway Testing Guide](vox2vocal-docs/testing/api-gateway-testing-guide.md) | api-gateway controller, token, gRPC client, proto contract 테스트를 작성할 때 |
| [Audio Upload MinIO E2E Test Plan](vox2vocal-docs/testing/audio-upload-minio-e2e-test-plan.md) | 오디오 업로드와 MinIO 연동 E2E 범위를 확인할 때 |

### 인프라/DB/로그 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [Local Run Guide](vox2vocal-docs/infra/local-run-guide.md) | minikube, Kubernetes, 서비스 로컬 실행, port-forward 절차를 확인할 때 |
| [Infra Logging Guide](vox2vocal-docs/infra/logging-guide.md) | Kubernetes 로그 확인, 문제 분석, logging 운영 절차를 확인할 때 |
| [Prisma Migration Guide](vox2vocal-docs/prisma/prisma-migration-guide.md) | user-service Prisma schema와 migration을 변경할 때 |
| [Infra README](vox2vocal-infra/README.md) | infra repo의 Kubernetes manifest와 로컬 클러스터 구성을 확인할 때 |
| [Infra Agent](vox2vocal-infra/AGENT.md) | infra repo 작업 규칙, commit 기준, logging 운영 기준을 확인할 때 |

### 엔진 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [Engine Overview](vox2vocal-docs/engine/README.md) | 전체 엔진 pipeline, 입력/출력 계약, MVP 연결 흐름을 확인할 때 |
| [Audio Ingest README](vox2vocal-docs/engine/audio-ingest/README.md) | 오디오 입력, 표준화, storage, event contract를 확인할 때 |
| [Audio Ingest Development Preparation](vox2vocal-docs/engine/audio-ingest/DEVELOPMENT_PREPARATION.md) | audio ingest 구현 준비, 개발 순서, 환경 전제를 확인할 때 |
| [Audio Ingest Technical Research](vox2vocal-docs/engine/audio-ingest/TECHNICAL_RESEARCH.md) | audio ingest 기술 조사와 라이브러리 판단 근거를 확인할 때 |
| [Voice Analysis](vox2vocal-docs/engine/voice-analysis/README.md) | 음성 분석 엔진의 역할과 출력 지표를 확인할 때 |
| [Voice Pitch](vox2vocal-docs/engine/voice-pitch/README.md) | pitch/F0 분석과 pitch feedback 요구사항을 확인할 때 |
| [Phoneme Alignment](vox2vocal-docs/engine/phoneme-alignment/README.md) | phoneme/syllable alignment 범위와 후속 연결을 확인할 때 |
| [Rhythm Timing](vox2vocal-docs/engine/rhythm-timing/README.md) | rhythm/timing 분석 범위와 입력/출력 기준을 확인할 때 |
| [Melody Mapping](vox2vocal-docs/engine/melody-mapping/README.md) | melody mapping과 target pitch 비교 기준을 확인할 때 |
| [Singing Synthesis](vox2vocal-docs/engine/singing-synthesis/README.md) | singing synthesis 범위와 downstream contract를 확인할 때 |
| [Vocoder Render](vox2vocal-docs/engine/vocoder-render/README.md) | vocoder rendering 범위와 산출물 기준을 확인할 때 |
| [Mix Master](vox2vocal-docs/engine/mix-master/README.md) | mix/master 산출물과 품질 처리 범위를 확인할 때 |
| [Safety Rights](vox2vocal-docs/engine/safety-rights/README.md) | 권리, 동의, 안전 차단 기준을 확인할 때 |
| [Evaluation](vox2vocal-docs/engine/evaluation/README.md) | 엔진 평가, 품질 metric, 비교 기준을 확인할 때 |
| [Expression](vox2vocal-docs/engine/expression/README.md) | 표현 확장과 style/expression 처리 범위를 확인할 때 |
| [Voice Conversion](vox2vocal-docs/engine/voice-conversion/README.md) | voice conversion 범위와 안전 전제를 확인할 때 |
| [Engine Event Contract](engine-audio-ingest/docs/ENGINE_EVENT_CONTRACT.md) | 구현 repo의 audio ingest event payload와 contract를 확인할 때 |
| [Audio Ingest Agent](engine-audio-ingest/AGENT.md) | audio ingest 구현 repo 작업 규칙과 Python convention을 확인할 때 |
| [Audio Ingest Architecture](engine-audio-ingest/ARCHITECTURE.md) | 구현 repo의 Python 구조, module 책임, 코드 convention을 확인할 때 |

### 로깅 도메인 문서

| 문서 | 언제 읽는가 |
| --- | --- |
| [Logging Overview](vox2vocal-docs/engine/logging/README.md) | engine logging 문서 세트의 진입점을 확인할 때 |
| [Logging Development Direction](vox2vocal-docs/engine/logging/development-direction.md) | logging MVP 개발 방향과 단계별 범위를 확인할 때 |
| [Logging Development Guide](vox2vocal-docs/engine/logging/development-guide.md) | logging 구현 규칙과 개발 절차를 확인할 때 |
| [Logging Operations Guide](vox2vocal-docs/engine/logging/operations-guide.md) | 운영 중 로그 확인, 알림, 장애 대응 절차를 확인할 때 |
| [Log Domain Guide](vox2vocal-docs/engine/logging/log-domain-guide.md) | 로그 event domain과 field 기준을 확인할 때 |
| [Engine Log Index](vox2vocal-docs/engine/logging/engine-log-index.md) | 엔진별 log event index를 확인할 때 |
| [Audit Data Guide](vox2vocal-docs/engine/logging/audit-data-guide.md) | audit table, digest, retention 기준을 확인할 때 |
| [Security Audit Guide](vox2vocal-docs/engine/logging/security-audit-guide.md) | 보안 감사 로그와 unauthorized access 추적 기준을 확인할 때 |
| [Storage Policy](vox2vocal-docs/engine/logging/storage-policy.md) | 로그 저장, 보관, 삭제 정책을 확인할 때 |

### 서비스별 로컬 지침

| 문서 | 언제 읽는가 |
| --- | --- |
| [API Gateway Agent](vox2vocal-api-gateway/AGENT.md) | api-gateway 테스트와 contract 작업 전 |
| [User Service Agent](vox2vocal-user-service/AGENT.md) | user-service 테스트와 domain 작업 전 |
| [BFF README](vox2vocal-bff-server/README.md) | bff-server 실행, GraphQL, gateway client 구조를 확인할 때 |
| [Worker README](vox2vocal-worker/README.md) | worker 실행과 queue 처리 구조를 확인할 때 |

### 작업 유형별 읽는 순서

작업 유형별로 아래 순서로 문서를 읽는다.

### 테스트 코드 작성/수정

서비스별 테스트 가이드를 먼저 읽고, 그 다음 해당 서비스의 `AGENT.md`가 있으면 함께 확인한다.

- `user-service`: [User Service Testing Guide](vox2vocal-docs/testing/user-service-testing-guide.md), [vox2vocal-user-service/AGENT.md](vox2vocal-user-service/AGENT.md)
- `api-gateway`: [API Gateway Testing Guide](vox2vocal-docs/testing/api-gateway-testing-guide.md), [vox2vocal-api-gateway/AGENT.md](vox2vocal-api-gateway/AGENT.md)

아직 전용 테스트 가이드가 없는 프로젝트는 해당 `package.json`, `src/`, 기존 `test/`를 먼저 확인하고 기존 패턴보다 스택의 테스트 경계를 우선한다.

### Prisma/DB 작업

`user-service`의 Prisma schema, migration, PostgreSQL 적용 이력을 수정할 때는 아래 문서를 먼저 읽는다.

- [Prisma Migration Guide](vox2vocal-docs/prisma/prisma-migration-guide.md)

### 로컬 실행/인프라/로그 작업

서비스 실행, Kubernetes, minikube, PostgreSQL, Redis, logging 관련 작업은 아래 문서를 먼저 읽는다.

- [Workspace README](vox2vocal-docs/workspace/README.md)
- [Local Run Guide](vox2vocal-docs/infra/local-run-guide.md)
- [Logging Guide](vox2vocal-docs/infra/logging-guide.md)
- [Infra README](vox2vocal-infra/README.md)

### 앱 작업

`app` 작업은 앱 내부 지침을 먼저 따른다.

- [vox2vocal-app/AGENTS.md](vox2vocal-app/AGENTS.md)
- [vox2vocal-app/README.md](vox2vocal-app/README.md)

Expo는 버전 변화가 잦으므로 `vox2vocal-app/AGENTS.md`의 Expo 문서 확인 지침을 우선한다.

### Agent Skill 작업

PM/TRD Codex skill을 추가, 수정, 검증할 때는 아래 문서를 먼저 읽는다.

- [vox2vocal-agent-skills/AGENT.md](vox2vocal-agent-skills/AGENT.md)
- [vox2vocal-agent-skills/README.md](vox2vocal-agent-skills/README.md)
- [vox2vocal-agent-skills/prd-trd-skill-flow-guide.md](vox2vocal-agent-skills/prd-trd-skill-flow-guide.md)

### 엔진 작업

엔진 작업은 먼저 전체 엔진 문서를 읽고, 그 다음 해당 엔진 디렉터리의 지침을 확인한다.

- 전체 구조: [Engine Architecture](vox2vocal-docs/engine/README.md)
- 오디오 입력: [Audio Ingest Engine](vox2vocal-docs/engine/audio-ingest/README.md), [engine-audio-ingest/AGENT.md](engine-audio-ingest/AGENT.md)
- 음성 분석: [Voice Analysis Engine](vox2vocal-docs/engine/voice-analysis/README.md)
- 피치 분석: [Voice Pitch Engine](vox2vocal-docs/engine/voice-pitch/README.md)

아직 구현 저장소가 없는 엔진도 `vox2vocal-docs/engine/<engine-name>/README.md`의 입력/출력, MVP 범위, 연결 엔진을 먼저 기준으로 삼는다.
새 엔진 저장소를 만들 때는 디렉터리 이름과 GitHub repository 이름을 모두 `engine-xxx` 형식으로 맞춘다.

## 3. 서비스별 테스트 관점

### user-service

중심 테스트 대상:

- CQRS command/query handler
- policy, mapper, security service
- Prisma repository integration test
- gRPC/protobuf contract

주의:

- handler unit test에서는 repository dependency를 mock한다.
- Prisma repository는 실제 테스트 DB integration test로 검증한다.
- `PORT`, `GRPC_URL` 기본값은 현재 `main.ts` bootstrap 내부에 있으므로 ConfigModule만으로 검증하지 않는다.

### api-gateway

중심 테스트 대상:

- `GatewayController` orchestration
- `TokenService` JWT 발급/검증
- `UserClientService` downstream gRPC client adapter
- inbound `gateway.proto`와 downstream `user.proto` mapping

주의:

- gateway 테스트에서 `user-service` 비즈니스 로직을 재검증하지 않는다.
- `UserClientService` unit test는 `ClientGrpc.getService()`가 반환하는 stub을 mock한다.
- `PORT`, `GRPC_URL` 기본값은 현재 `main.ts` bootstrap 내부에 있으므로 ConfigModule만으로 검증하지 않는다.

### bff-server

중심 테스트 대상:

- GraphQL resolver
- GraphQL schema/DTO validation
- downstream gateway gRPC client adapter
- GraphQL response shape와 error mapping

주의:

- resolver 테스트에서 downstream gateway 실제 network 호출을 피한다.
- GraphQL e2e는 핵심 query/mutation happy path와 대표 error path만 둔다.

### worker

중심 테스트 대상:

- BullMQ processor
- queue job payload validation
- Redis/BullMQ integration boundary

주의:

- processor unit test에서는 Redis를 사용하지 않는다.
- BullMQ/Redis 동작은 별도 integration test에서 테스트 전용 Redis로 검증한다.

### app

중심 테스트 대상:

- store/state logic
- React Query provider 경계
- form validation
- navigation-visible behavior
- React Native component behavior

주의:

- Expo/React Native 버전별 문서를 먼저 확인한다.
- UI 테스트는 구현 세부 DOM보다 사용자가 보는 동작을 우선한다.

### engine 계열

중심 테스트 대상:

- 오디오 fixture 기반 deterministic test
- 입력/출력 JSON schema 또는 DTO contract
- 샘플레이트, 타임스탬프, pitch frame 같은 수치 경계
- 엔진 간 handoff data shape
- 긴 오디오 chunk 처리와 빈/무음 오디오 edge case

주의:

- 엔진별 구현 스택이 확정되기 전에는 문서의 I/O contract를 먼저 고정한다.
- 모델 또는 DSP 알고리즘 품질 테스트와 API contract 테스트를 분리한다.
- 큰 오디오 fixture나 모델 weight는 Git에 바로 추가하지 않고 별도 저장 전략을 먼저 정한다.

## 4. 검증 명령

PowerShell에서는 `npm` 대신 `npm.cmd`를 우선 사용한다.

서비스별 검증:

```bash
cd user-service
npm.cmd run verify
```

```bash
cd api-gateway
npm.cmd run verify
```

빠른 테스트:

```bash
npm.cmd test -- --runInBand
```

전체 프로젝트가 단일 npm workspace로 묶여 있지 않으므로, 루트에서 한 번에 모든 서비스를 검증한다고 가정하지 않는다.
각 프로젝트 디렉터리에서 해당 `package.json`의 scripts를 확인한 뒤 실행한다.

엔진 저장소는 구현 스택이 다를 수 있으므로, 각 `engine-*` 디렉터리에서 먼저 `README.md`, `AGENT.md`, `package.json`, `pyproject.toml`, `Cargo.toml` 등 실제 존재하는 프로젝트 파일을 확인한 뒤 검증 명령을 선택한다.

## 5. 문서 유지 규칙

- 새 테스트 패턴을 도입하면 해당 서비스 테스트 가이드를 함께 갱신한다.
- 새 엔진을 추가하거나 엔진 간 입출력 계약을 바꾸면 `vox2vocal-docs/engine/README.md`와 해당 엔진 문서를 함께 갱신한다.
- 새 프로젝트별 에이전트 지침이 필요하면 해당 디렉터리에 `AGENT.md`를 둔다.
- 문서 버전업을 진행하기 전에는 현재까지의 문서 수정 내용을 반드시 먼저 별도 커밋한다.
- 문서 버전업 커밋에는 버전 번호, 날짜, changelog, 해당 버전의 최소 필요 수정만 포함하고 이전 작업 수정과 섞지 않는다.
- 루트 `AGENT.md`는 상세 구현 규칙보다 문서 탐색과 서비스별 기술 스택 인덱싱에 집중한다.

## 6. Commit Convention

Vox2Vocal workspace는 monorepo가 아니라 MSA 구조다. 각 하위 폴더는 독립 Git repository이므로 변경한 repo 안에서 개별 commit/push 한다.

상세 Git 정책은 `vox2vocal-docs/workspace/git-policy.md`를 따른다. 각 repo에는 `scripts/validate-git-policy.sh`, `.githooks/commit-msg`, `.githooks/pre-push`, `.github/workflows/git-policy.yml`을 둔다.

커밋 author와 committer는 항상 `gitbyul <gitbyul@gmail.com>`이어야 한다. 다른 개인 계정, bot 계정, 로컬 머신 이메일, GitHub noreply 이메일은 허용하지 않는다.

각 repo에서 다음 명령으로 local Git identity와 hook 경로를 고정한다.

```bash
scripts/install-git-policy-hooks.sh
```

다른 에이전트가 로컬에서 작업하더라도 commit author와 committer는 `gitbyul <gitbyul@gmail.com>`로 고정한다. 에이전트 고유 이름, bot 계정, GitHub noreply, 로컬 머신 이메일을 commit identity로 남기지 않는다.

에이전트는 commit 전후로 아래를 확인한다.

```bash
scripts/validate-git-policy.sh --check-config
scripts/validate-git-policy.sh --commit HEAD
```

GitHub에서 자기 PR에 `Approve`를 누르지 않는다. `Can not approve your own pull request`는 GitHub 기본 동작이며, Required approvals를 `0`으로 운영하면 자기 approve 없이 PR을 병합할 수 있다.

일반 코드 repo의 커밋 메시지는 ticket을 필수로 포함한다.

```txt
type(scope): [TICKET] 한글 제목

- 한글 bullet body
```

문서 repo는 ticket을 선택으로 두고, 문서번호와 버전을 `_`로 연결해 bracket에 표시한다.

```txt
type(scope): [문서번호_버전] 한글 제목

- 한글 bullet body
```

예시:

```txt
docs(policy): [GIT-POLICY_v0.1] Git 정책 운영 방식 갱신
docs(prd): [MVP-PRD_v0.14] MVP PRD 동의 정책 갱신
```

- `type`은 `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `ci` 중 하나로 작성한다.
- `scope`는 영어 소문자 kebab-case로 작성하며 변경 대상 repo 또는 기능 영역을 나타낸다.
- 제목과 본문에는 한글이 포함되어야 한다.
- 제목만 작성하지 않고, 커밋 본문에 변경 내용을 한글 bullet로 작성한다.
- 제목과 본문 사이에는 빈 줄을 한 줄만 둔다.
- 본문 bullet 사이에는 빈 줄을 넣지 않는다.
- 관련 없는 여러 repo의 변경을 하나의 커밋으로 묶지 않는다.
- 변경 단위가 다르면 같은 repo 안에서도 작업 단위별로 커밋을 나눈다.

일반 코드 repo branch 형식:

```txt
type/TICKET-short-summary
```

예시:

```txt
feat/V2V-123-auth-refresh
fix/V2V-204-token-case-mapping
chore/V2V-000-git-policy-pr-flow
```

일반 코드 repo는 `main`에서 직접 작업하지 않는다. ticket branch에서 commit하고 PR을 생성한 뒤 `git-policy` check와 리뷰를 통과시킨다. GitHub UI merge 버튼은 사용하지 않으며, `gitbyul` 로컬 환경에서 fast-forward로 main에 반영한다.

승인된 PR을 main에 반영할 때만 다음 예외를 사용한다.

```bash
GIT_POLICY_ALLOW_MAIN_MERGE_PUSH=1 GIT_POLICY_PR_NUMBER=<number> git push origin main
```

`gh pr merge` 또는 GitHub UI merge는 사용하지 않는다. GitHub가 생성하는 merge/squash/rebase commit은 committer 고정 정책과 충돌할 수 있다.

문서 repo는 기존 운영처럼 직접 문서 커밋을 허용할 수 있다. 단, commit message와 body 규칙, 문서 버전업 전 선행 커밋 규칙은 반드시 지킨다.

push 원칙:

- `app` 변경은 `vox2vocal-app` repo에서 commit/push 한다.
- `bff-server` 변경은 `vox2vocal-bff-server` repo에서 commit/push 한다.
- `api-gateway` 변경은 `vox2vocal-api-gateway` repo에서 commit/push 한다.
- `user-service` 변경은 `vox2vocal-user-service` repo에서 commit/push 한다.
- `worker` 변경은 `vox2vocal-worker` repo에서 commit/push 한다.
- `infra` 변경은 `vox2vocal-infra` repo에서 commit/push 한다.
- `docs` 변경은 `vox2vocal-docs` repo에서 commit/push 한다.
- `design-kit` 변경은 `vox2vocal-design-kit` repo에서 commit/push 한다.
- `agent-skills` 변경은 `vox2vocal-agent-skills` repo에서 commit/push 한다.
- `engine-audio-ingest` 변경은 `engine-audio-ingest` repo에서 commit/push 한다.
- `engine-voice-analysis` 변경은 `engine-voice-analysis` repo에서 commit/push 한다.
- `engine-voice-pitch` 변경은 `engine-voice-pitch` repo에서 commit/push 한다.
- 루트 파일은 현재 독립 Git repo에 속하지 않을 수 있으므로, 커밋 가능 여부를 먼저 확인한다.

강제 장치:

- local `commit-msg` hook은 Git identity, branch, 커밋 메시지를 검사한다.
- local `pre-push` hook은 branch, main push 예외 조건, push 대상 커밋의 author, committer, 메시지를 검사한다.
- GitHub Actions `git-policy` workflow는 PR branch와 PR 신규 커밋, `workflow_dispatch` 대상 커밋을 검사한다.
- GitHub `main` ruleset에서는 `git-policy` required check, force push 금지, 삭제 금지, update 제한을 적용한다.
- 현재 rewritten history는 unsigned 상태이므로 signed commit required ruleset은 signing key 구성 이후 새 커밋부터 적용한다.
## Git Safe Directory Rule

Windows 환경에서 Codex 또는 다른 계정이 생성한 repo를 현재 사용자 `CMS`가 다룰 때 `dubious ownership` 오류가 발생할 수 있다.

Git remote 추가, commit, push, log 확인 등 Git 작업을 진행하기 전에 해당 repo에서 다음 오류가 나는지 먼저 확인한다.

```txt
fatal: detected dubious ownership in repository
```

오류가 발생하면 Git 작업을 계속하기 전에 해당 repo를 `safe.directory`에 등록한다.

```bash
git config --global --add safe.directory C:/Users/CMS/Desktop/gitbyul/vox2vocal/<repo-name>
```

예시:

```bash
git config --global --add safe.directory C:/Users/CMS/Desktop/gitbyul/vox2vocal/engine-audio-ingest
git config --global --add safe.directory C:/Users/CMS/Desktop/gitbyul/vox2vocal/engine-voice-analysis
git config --global --add safe.directory C:/Users/CMS/Desktop/gitbyul/vox2vocal/engine-voice-pitch
```

등록 후 다음 명령으로 Git 접근이 정상인지 확인한다.

```bash
git -C <repo-name> status
git -C <repo-name> remote -v
```

remote 추가는 safe.directory 문제가 해결된 뒤 진행한다.
