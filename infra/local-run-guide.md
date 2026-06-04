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

### 3. 서비스 이미지 빌드

workspace 루트에서 실행한다.

Windows Git Bash에서는 Docker Desktop daemon과 minikube 내부 Docker daemon이 서로 다를 수 있다. minikube가 바로 사용할 이미지를 만들기 위해 `minikube image build`를 기본으로 사용한다.

```bash
minikube image build -t vox2vocal/bff-server:local ./bff-server
minikube image build -t vox2vocal/api-gateway:local ./api-gateway
minikube image build -t vox2vocal/user-service:local ./user-service
minikube image build -t vox2vocal/worker:local ./worker
```

빌드된 이미지 확인:

```bash
minikube image ls | grep vox2vocal
```

대안으로 minikube Docker daemon을 현재 Git Bash 세션에 적용한 뒤 `docker build`를 사용할 수도 있다.

```bash
eval $(minikube docker-env)
docker build -t vox2vocal/bff-server:local ./bff-server
docker build -t vox2vocal/api-gateway:local ./api-gateway
docker build -t vox2vocal/user-service:local ./user-service
docker build -t vox2vocal/worker:local ./worker
```

이 방식은 현재 터미널 세션에만 적용된다. 새 Git Bash 창을 열면 다시 `eval $(minikube docker-env)`를 실행해야 한다.

### 4. Kubernetes 리소스 적용

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

### 5. 서비스 이미지 갱신 후 재시작

이미지를 다시 빌드한 경우 기존 Deployment가 새 이미지를 사용하도록 재시작한다.

```bash
kubectl rollout restart deployment -n vox2vocal bff-server api-gateway user-service worker
```

pod 상태를 확인한다.

```bash
kubectl get pods -n vox2vocal -w
```

정상 상태 예시:

```txt
api-gateway    1/1 Running
bff-server     1/1 Running
postgres       1/1 Running
redis          1/1 Running
user-service   1/1 Running
worker         1/1 Running
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

hosts 등록 확인:

```bash
ping vox2vocal.local
```

`vox2vocal.local`이 minikube IP로 해석되면 hosts 등록은 성공이다. Windows 환경에서는 minikube IP가 ICMP ping에 응답하지 않을 수 있으므로, ping timeout만으로 hosts 설정 실패로 판단하지 않는다.

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
-> minikube image build로 서비스 이미지 빌드
-> 기존 abyul namespace가 있으면 삭제
-> kubectl apply -k infra/k8s
-> 이미지 갱신 시 kubectl rollout restart 실행
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

Kubernetes 리소스 상태 확인:

```bash
kubectl get pods -n vox2vocal
kubectl get svc -n vox2vocal
kubectl get ingress -n vox2vocal
```

서비스 로그 확인:

```bash
kubectl logs -n vox2vocal deploy/bff-server
kubectl logs -n vox2vocal deploy/api-gateway
kubectl logs -n vox2vocal deploy/user-service
kubectl logs -n vox2vocal deploy/worker
```

재시작으로 종료된 이전 컨테이너 로그는 `--previous` 옵션으로 확인한다.

```bash
kubectl logs -n vox2vocal deploy/bff-server --previous
```

## 장애 대응

### ImagePullBackOff

`ImagePullBackOff`는 Kubernetes가 지정된 이미지를 찾지 못할 때 주로 발생한다.

확인:

```bash
kubectl get pods -n vox2vocal
minikube image ls | grep vox2vocal
```

해결:

```bash
minikube image build -t vox2vocal/bff-server:local ./bff-server
minikube image build -t vox2vocal/api-gateway:local ./api-gateway
minikube image build -t vox2vocal/user-service:local ./user-service
minikube image build -t vox2vocal/worker:local ./worker
kubectl rollout restart deployment -n vox2vocal bff-server api-gateway user-service worker
```

### Cannot find module '/app/dist/main.js'

NestJS 빌드 결과가 `dist/src/main.js`에 생성되는데 Dockerfile이 `dist/main.js`를 실행하면 발생한다.

각 서버 Dockerfile의 실행 경로는 다음과 같아야 한다.

```dockerfile
CMD ["node", "dist/src/main.js"]
```

또한 Docker build context에 로컬 `dist` 또는 `node_modules`가 섞이지 않도록 각 서버 repo에 `.dockerignore`를 둔다.

```txt
node_modules
dist
coverage
.git
.env
npm-debug.log
```

### bff-server GraphQLModule 패키지 누락

`bff-server` 로그에 다음 오류가 나오면 `@as-integrations/express5` 의존성이 필요하다.

```txt
The "@as-integrations/express5" package is missing.
```

설치:

```bash
cd bff-server
npm.cmd install @as-integrations/express5
```

### user-service Prisma client 초기화 오류

`user-service` 로그에 다음 오류가 나오면 Docker runner 이미지에 Prisma generated client가 포함되지 않은 것이다.

```txt
@prisma/client did not initialize yet. Please run "prisma generate"
```

Dockerfile runner 단계에서 build 단계의 `.prisma` 산출물을 복사해야 한다.

```dockerfile
COPY --from=build /app/node_modules/.prisma ./node_modules/.prisma
```

수정 후 이미지를 다시 빌드하고 Deployment를 재시작한다.

```bash
minikube image build -t vox2vocal/user-service:local ./user-service
kubectl rollout restart deployment -n vox2vocal user-service
```

### vox2vocal.local 접속 timeout

`kubectl get ingress -n vox2vocal`에서 ingress가 보이고 hosts 파일도 등록되어 있는데 Windows에서 `curl http://vox2vocal.local`이 timeout될 수 있다.

먼저 pod와 ingress 상태를 확인한다.

```bash
kubectl get pods -n vox2vocal
kubectl get ingress -n vox2vocal
```

pod가 모두 `1/1 Running`이면 서비스 자체는 정상 기동된 것이다. Windows와 minikube driver 조합에 따라 ingress IP 직접 접근이 막힐 수 있으므로, 이 경우 임시로 port-forward로 확인한다.

```bash
kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
```

별도 Git Bash 창에서 확인:

```bash
curl http://localhost:4000/health
```

## 주의사항

- 이 workspace는 모노레포가 아니다.
- 각 하위 폴더는 독립 Git repository이다.
- 변경 사항은 변경한 repo 안에서 개별 commit/push 한다.
- infra repo에는 Kubernetes manifest와 배포 설정만 관리한다.
- 서비스 코드는 infra repo에 포함하지 않는다.
- Git Bash에서도 npm 실행은 `npm.cmd` 기준으로 사용한다.
- Git `dubious ownership` 오류가 발생하면 해당 repo를 `safe.directory`에 등록한다.
