# Vox2Vocal Platform Workspace

Expo React Native App/Web과 MSA 서버 프로젝트들을 함께 관리하는 워크스페이스입니다.

이 워크스페이스는 모노레포 패키지 매니저 구성이 아니라, 각 서버가 자체 `package.json`, `Dockerfile`, 설정 파일을 가지는 독립 프로젝트 구조입니다.

## 아키텍처

```txt
RN App / RN Web
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

## 프로젝트

| 프로젝트 | 역할 | 주요 기술 |
| --- | --- | --- |
| `app` | Expo RN App/Web | Expo, React Native Web, Tamagui |
| `bff-server` | GraphQL BFF | NestJS, GraphQL, Apollo, gRPC client |
| `api-gateway` | 내부 API Gateway | NestJS, gRPC server/client |
| `user-service` | 사용자 도메인 서비스 | NestJS, gRPC, Prisma, PostgreSQL |
| `worker` | 비동기 작업 서버 | NestJS, BullMQ, Redis |
| `infra` | minikube/Kubernetes 구성 | Deployment, Service, Ingress, PostgreSQL, Redis |

## 외부 호출

외부 클라이언트가 호출하는 서버는 `bff-server` 하나입니다.

```txt
http://vox2vocal.local/graphql
```

나머지 서버는 Kubernetes 클러스터 내부 통신으로 사용합니다.

## 로컬 실행

각 프로젝트 폴더에서 의존성을 설치하고 실행합니다.

```bash
cd bff-server
npm install
npm run start:dev
```

minikube 배포 방법은 [`infra/README.md`](./infra/README.md)를 참고합니다.

## Root Agent Document

`AGENT.md`는 Vox2Vocal 전체 workspace를 기준으로 작업하는 에이전트 지침 파일이다.

이 파일은 모든 프로젝트의 상위 루트 경로에 있어야 한다.

기준 위치:

```txt
C:\Users\CMS\Desktop\gitbyul\vox2vocal\AGENT.md
```

docs repo에는 루트 배치용 원본을 다음 경로에 보관한다.

```txt
docs/root/AGENT.md
```

workspace 루트의 `AGENT.md`를 갱신하면 `docs/root/AGENT.md`도 함께 갱신한다.
