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

## 2. 문서 읽는 순서

작업 유형별로 아래 순서로 문서를 읽는다.

### 테스트 코드 작성/수정

서비스별 테스트 가이드를 먼저 읽고, 그 다음 해당 서비스의 `AGENT.md`가 있으면 함께 확인한다.

- `user-service`: [User Service Testing Guide](docs/testing/user-service-testing-guide.md), [user-service/AGENT.md](user-service/AGENT.md)
- `api-gateway`: [API Gateway Testing Guide](docs/testing/api-gateway-testing-guide.md), [api-gateway/AGENT.md](api-gateway/AGENT.md)

아직 전용 테스트 가이드가 없는 프로젝트는 해당 `package.json`, `src/`, 기존 `test/`를 먼저 확인하고 기존 패턴보다 스택의 테스트 경계를 우선한다.

### Prisma/DB 작업

`user-service`의 Prisma schema, migration, PostgreSQL 적용 이력을 수정할 때는 아래 문서를 먼저 읽는다.

- [Prisma Migration Guide](docs/prisma/prisma-migration-guide.md)

### 로컬 실행/인프라/로그 작업

서비스 실행, Kubernetes, minikube, PostgreSQL, Redis, logging 관련 작업은 아래 문서를 먼저 읽는다.

- [Workspace README](docs/workspace/README.md)
- [Local Run Guide](docs/infra/local-run-guide.md)
- [Logging Guide](docs/infra/logging-guide.md)
- [Infra README](infra/README.md)

### 앱 작업

`app` 작업은 앱 내부 지침을 먼저 따른다.

- [app/AGENTS.md](app/AGENTS.md)
- [app/README.md](app/README.md)

Expo는 버전 변화가 잦으므로 `app/AGENTS.md`의 Expo 문서 확인 지침을 우선한다.

### 엔진 작업

엔진 작업은 먼저 전체 엔진 문서를 읽고, 그 다음 해당 엔진 디렉터리의 지침을 확인한다.

- 전체 구조: [Engine Architecture](docs/engine/README.md)
- 오디오 입력: [Audio Ingest Engine](docs/engine/audio-ingest/README.md), [engine-audio-ingest/AGENT.md](engine-audio-ingest/AGENT.md)
- 음성 분석: [Voice Analysis Engine](docs/engine/voice-analysis/README.md)
- 피치 분석: [Voice Pitch Engine](docs/engine/voice-pitch/README.md)

아직 구현 저장소가 없는 엔진도 `docs/engine/<engine-name>/README.md`의 입력/출력, MVP 범위, 연결 엔진을 먼저 기준으로 삼는다.
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
- 새 엔진을 추가하거나 엔진 간 입출력 계약을 바꾸면 `docs/engine/README.md`와 해당 엔진 문서를 함께 갱신한다.
- 새 프로젝트별 에이전트 지침이 필요하면 해당 디렉터리에 `AGENT.md`를 둔다.
- 루트 `AGENT.md`는 상세 구현 규칙보다 문서 탐색과 서비스별 기술 스택 인덱싱에 집중한다.

## 6. Commit Convention

Vox2Vocal workspace는 monorepo가 아니라 MSA 구조다. 각 하위 폴더는 독립 Git repository이므로 변경한 repo 안에서 개별 commit/push 한다.

상세 Git 정책은 `vox2vocal-docs/workspace/git-policy.md`를 따른다. 모든 repo에는 동일한 `scripts/validate-git-policy.sh`, `.githooks/commit-msg`, `.githooks/pre-push`, `.github/workflows/git-policy.yml`을 둔다.

커밋 author와 committer는 항상 `gitbyul <gitbyul@gmail.com>`이어야 한다. 다른 개인 계정, bot 계정, 로컬 머신 이메일, GitHub noreply 이메일은 허용하지 않는다.

각 repo에서 다음 명령으로 local Git identity와 hook 경로를 고정한다.

```bash
scripts/install-git-policy-hooks.sh
```

커밋 메시지는 Conventional Commits 형식을 사용한다.

```txt
type(scope): 한글 제목

- 한글 bullet body
```

- `type`은 영어 소문자로 작성한다.
- `scope`는 영어 소문자로 작성하며 변경 대상 repo 또는 기능 영역을 나타낸다.
- 제목은 한글로 작성한다.
- 제목만 작성하지 않고, 커밋 본문에 변경 내용을 한글 bullet로 작성한다.
- 제목과 본문 사이에는 빈 줄을 한 줄만 둔다.
- 본문 bullet 사이에는 빈 줄을 넣지 않는다.
- 본문에도 한글이 포함되어야 한다.
- 커밋 메시지 전체에 불필요한 빈 줄을 여러 번 넣지 않는다.
- 관련 없는 여러 repo의 변경을 하나의 커밋으로 묶지 않는다.
- 변경 단위가 다르면 같은 repo 안에서도 작업 단위별로 커밋을 나눈다.

주요 type 예시:

```txt
feat: 기능 추가
fix: 버그 수정
docs: 문서 변경
chore: 설정, 의존성, 환경 구성 변경
refactor: 동작 변경 없는 구조 개선
test: 테스트 추가 또는 수정
ci: CI/CD 설정 변경
```

scope 예시:

```txt
app
bff
gateway
user
worker
infra
docs
engine
engine-audio-ingest
engine-voice-analysis
engine-voice-pitch
setup
spec
```

커밋 예시:

```txt
chore(infra): 데이터 저장소 이미지 버전 고정

- PostgreSQL 이미지를 postgres:17.10-alpine으로 고정
- Redis 이미지를 redis:7.2.14-alpine3.21로 고정
- infra README에 데이터 저장소 이미지 기준 추가
```

잘못된 예시:

```txt
chore(infra): 데이터 저장소 이미지 버전 고정

- PostgreSQL 이미지를 postgres:17.10-alpine으로 고정

- Redis 이미지를 redis:7.2.14-alpine3.21로 고정

- infra README에 데이터 저장소 이미지 기준 추가
```

push 원칙:

- `app` 변경은 `app` repo에서 commit/push 한다.
- `bff-server` 변경은 `bff-server` repo에서 commit/push 한다.
- `api-gateway` 변경은 `api-gateway` repo에서 commit/push 한다.
- `user-service` 변경은 `user-service` repo에서 commit/push 한다.
- `worker` 변경은 `worker` repo에서 commit/push 한다.
- `infra` 변경은 `infra` repo에서 commit/push 한다.
- `docs` 변경은 `docs` repo에서 commit/push 한다.
- `engine-audio-ingest` 변경은 `engine-audio-ingest` repo에서 commit/push 한다.
- `engine-voice-analysis` 변경은 `engine-voice-analysis` repo에서 commit/push 한다.
- `engine-voice-pitch` 변경은 `engine-voice-pitch` repo에서 commit/push 한다.
- 루트 파일은 현재 독립 Git repo에 속하지 않을 수 있으므로, 커밋 가능 여부를 먼저 확인한다.

강제 장치:

- local `commit-msg` hook은 Git identity와 커밋 메시지를 검사한다.
- local `pre-push` hook은 push 대상 커밋의 author, committer, 메시지를 검사한다.
- GitHub Actions `git-policy` workflow는 PR 신규 커밋과 `workflow_dispatch` 대상 커밋을 검사한다.
- GitHub `main` ruleset에서는 `git-policy` required check, PR 필수, force push 금지, 삭제 금지를 적용한다.
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
