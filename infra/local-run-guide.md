# Vox2Vocal 로컬 구동 가이드

이 문서는 Vox2Vocal workspace 전체 프로그램을 로컬에서 구동하는 순서를 정리한다.

Vox2Vocal은 모노레포가 아니라 MSA 방식으로 관리된다. 각 하위 폴더는 독립 Git repository이며, 전체 연결 검증은 `infra`의 Kubernetes manifest를 기준으로 진행한다.

## 전체 아키텍처

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

## 사전 준비

다음 프로그램이 설치되어 있어야 한다.

- Docker Desktop
- minikube
- kubectl
- Node.js / npm

이 문서는 Windows Git Bash에서 실행하는 명령을 기준으로 작성한다. npm 명령은 Windows 실행 정책과 셸 차이를 줄이기 위해 `npm.cmd`를 사용한다.

설치 확인:

```bash
docker version
minikube version
kubectl version --client
node --version
npm.cmd --version
```

Git Bash에서 `minikube: command not found`가 발생하면 `minikube.exe` 설치 경로가 PATH에 등록되어 있는지 확인한다.

일반적인 Windows 설치 경로:

```txt
C:\Program Files\Kubernetes\Minikube
```

## Kubernetes 기반 전체 구동

전체 서비스 연결을 확인할 때는 minikube 기반 구동을 기본으로 한다.

### 1. Docker Desktop 실행

먼저 Docker Desktop을 실행하고 Docker CLI가 동작하는지 확인한다.

```bash
docker version
```

### 2. minikube 시작

```bash
minikube start
minikube addons enable ingress
```

### 3. minikube Docker daemon 적용

Kubernetes에서 로컬 이미지를 사용할 수 있도록 minikube Docker daemon으로 전환한다.

```bash
eval $(minikube docker-env)
```

적용 여부는 Docker daemon 정보로 확인한다.

```bash
docker info | grep -i "Name"
```

### 4. 서비스 이미지 빌드

workspace 루트에서 실행한다.

```bash
docker build -t vox2vocal/bff-server:local ./bff-server
docker build -t vox2vocal/api-gateway:local ./api-gateway
docker build -t vox2vocal/user-service:local ./user-service
docker build -t vox2vocal/worker:local ./worker
```

### 5. Kubernetes 리소스 적용

예전에 `abyul` namespace 기준 manifest를 적용한 적이 있다면, 새 manifest 적용 전에 기존 namespace를 삭제한다.

```bash
kubectl get all -n abyul
kubectl delete namespace abyul
```

삭제 후 `abyul` namespace가 더 이상 조회되지 않는지 확인한다.

```bash
kubectl get namespace
```

그 다음 `vox2vocal` 기준 manifest를 적용한다.

```bash
kubectl apply -k infra/k8s
```

상태 확인:

```bash
kubectl get pods -n vox2vocal
kubectl get svc -n vox2vocal
kubectl get ingress -n vox2vocal
```

### 6. 로컬 도메인 연결

로컬 GraphQL endpoint는 다음 주소를 기준으로 한다.

```txt
http://vox2vocal.local/graphql
```

minikube IP를 확인한다.

```bash
minikube ip
```

Windows hosts 파일에 `vox2vocal.local`을 등록한다.

hosts 파일 위치:

```txt
C:\Windows\System32\drivers\etc\hosts
```

Git Bash에서 관리자 권한으로 hosts 파일을 직접 수정하기 어렵다면, 관리자 권한으로 메모장을 열어 수정한다.

예시:

```txt
192.168.49.2 vox2vocal.local
```

`192.168.49.2` 부분은 `minikube ip` 결과로 교체한다.

### 7. App 실행

별도 터미널에서 `app` 프로젝트를 실행한다.

```bash
cd app
npm.cmd run web
```

Expo 개발 서버를 직접 실행할 수도 있다.

```bash
cd app
npm.cmd start
```

앱의 GraphQL endpoint는 로컬 Kubernetes 구동 기준으로 다음 값을 사용한다.

```txt
http://vox2vocal.local/graphql
```

## 실행 순서 요약

```txt
Docker Desktop 실행
-> minikube start
-> minikube addons enable ingress
-> eval $(minikube docker-env)
-> bff-server / api-gateway / user-service / worker 이미지 빌드
-> 기존 abyul namespace가 있으면 삭제
-> kubectl apply -k infra/k8s
-> pods, services, ingress 상태 확인
-> hosts에 vox2vocal.local 등록
-> app 실행
```

## 서비스 직접 실행 방식

개발 중 특정 서비스만 빠르게 확인할 때는 각 repo에서 직접 실행할 수 있다.

```bash
cd bff-server
npm.cmd run start:dev
```

```bash
cd api-gateway
npm.cmd run start:dev
```

```bash
cd user-service
npm.cmd run start:dev
```

```bash
cd worker
npm.cmd run start:dev
```

단, 직접 실행 방식은 PostgreSQL, Redis, gRPC endpoint 환경변수 등 서비스 간 연결 설정을 별도로 맞춰야 한다. 전체 연결 검증은 Kubernetes 기반 구동을 우선한다.

## 주요 포트

| 서비스         | 역할        | 포트    |
| -------------- | ----------- | ------- |
| `bff-server`   | GraphQL BFF | `4000`  |
| `api-gateway`  | HTTP health | `3001`  |
| `api-gateway`  | gRPC        | `50050` |
| `user-service` | HTTP health | `3002`  |
| `user-service` | gRPC        | `50051` |
| `worker`       | HTTP health | `3003`  |

## 검증 명령

각 서비스 repo에서 가능한 경우 다음 명령으로 검증한다.

```bash
npm.cmd run format:check
npm.cmd run verify
```

`user-service`는 Prisma client 생성이 필요할 수 있다.

```bash
cd user-service
npm.cmd run prisma:generate
```

infra manifest 렌더링 확인:

```bash
kubectl kustomize infra/k8s
```

## 주의사항

- 이 workspace는 모노레포가 아니다.
- 각 하위 폴더는 독립 Git repository이다.
- 변경 사항은 변경한 repo 안에서 개별 commit/push 한다.
- infra repo에는 Kubernetes manifest와 배포 설정만 관리한다.
- 서비스 코드는 infra repo에 포함하지 않는다.
- Git Bash에서도 npm 실행은 `npm.cmd` 기준으로 사용한다.
- Git `dubious ownership` 오류가 발생하면 해당 repo를 `safe.directory`에 등록한다.
