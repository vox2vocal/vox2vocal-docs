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

Kubernetes 구성에는 애플리케이션 서비스뿐 아니라 PostgreSQL, Redis, MinIO, NATS JetStream, logging stack, audio ingest engine도 포함된다.

| 리소스 | 이미지 | 역할 |
| --- | --- | --- |
| `postgres` | `postgres:17.10-alpine` | user-service 데이터베이스 |
| `redis` | `redis:7.2.14-alpine3.21` | worker/BullMQ queue |
| `minio` | `quay.io/minio/minio:latest` | local S3-compatible object storage |
| `nats` | `nats:2.11.15-alpine3.22` | audio ingest event stream |
| `engine-audio-ingest` | `vox2vocal/engine-audio-ingest:local` | source audio probe/convert engine |

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

### 3. minikube tunnel 실행

로컬 PC에서 minikube 클러스터의 ingress와 `LoadBalancer` Service를 사용하려면 `minikube tunnel`을 별도 터미널에서 실행해 둔다.

`minikube tunnel`은 실행 중인 터미널을 점유한다. 이 터미널은 로컬 실행과 DB 접속이 끝날 때까지 닫지 않는다.

터미널 1:

```bash
minikube tunnel
```

Windows에서 관리자 권한을 요구할 수 있다. 권한 요청이 나오면 승인하고, 터미널에 tunnel 프로세스가 계속 떠 있는 상태로 둔다.

이후 명령은 새 Git Bash 또는 PowerShell 터미널에서 실행한다.

### 4. 서비스 이미지 빌드

workspace 루트에서 실행한다.

현재 workspace의 service repo 이름은 `vox2vocal-*` 형식이다. app service 이미지는 infra helper를 우선 사용한다.

```bash
./vox2vocal-infra/scripts/build-local-app-images.sh
```

engine image는 FFprobe/FFmpeg 구현 변경 이후 빌드한다.

```bash
docker build --network=host -t vox2vocal/engine-audio-ingest:local ./engine-audio-ingest
minikube image load vox2vocal/engine-audio-ingest:local
```

빌드된 이미지 확인:

```bash
minikube image ls | grep vox2vocal
```

대안으로 `minikube image build`를 직접 사용할 수도 있다.

```bash
minikube image build -t vox2vocal/bff-server:local ./vox2vocal-bff-server
minikube image build -t vox2vocal/api-gateway:local ./vox2vocal-api-gateway
minikube image build -t vox2vocal/user-service:local ./vox2vocal-user-service
minikube image build -t vox2vocal/worker:local ./vox2vocal-worker
minikube image build -t vox2vocal/engine-audio-ingest:local ./engine-audio-ingest
```

Docker default bridge network에서 npm/PyPI registry 접근이 timeout되면 `docker build --network=host` 방식으로 재시도한다.

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

`postgres`와 `redis`도 이 명령으로 함께 생성 또는 갱신된다.

상태 확인:

```bash
kubectl get pods -n vox2vocal
kubectl get svc -n vox2vocal
kubectl get ingress -n vox2vocal
```

### 6. 서비스 이미지 갱신 후 재시작

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

PostgreSQL 또는 Redis 이미지 태그를 변경한 경우에도 상태를 확인한다.

```bash
kubectl get statefulset -n vox2vocal postgres
kubectl get deployment -n vox2vocal redis
kubectl logs -n vox2vocal statefulset/postgres --tail=100
kubectl logs -n vox2vocal deploy/redis --tail=100
```

### 7. 로컬 도메인 연결

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

### 8. 로컬 PostgreSQL 접속

PostgreSQL Service는 `LoadBalancer` 타입이며, service port는 `15432`이다. 로컬 PC에서 접속하려면 `minikube tunnel`이 실행 중이어야 한다.

DB 접속 전제:

```txt
터미널 1: minikube tunnel 실행 중
터미널 2: psql, Prisma, 애플리케이션 등 DB 접속 명령 실행
```

`minikube tunnel`을 실행 중인 터미널을 닫으면 `LoadBalancer` 접속 경로가 종료된다. 로컬에서 DB에 접속하는 동안 tunnel 터미널을 유지한다.

PostgreSQL Service의 접속 IP를 확인한다.

```bash
kubectl get svc -n vox2vocal postgres
```

출력 예시:

```txt
NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
postgres   LoadBalancer   10.96.123.45    127.0.0.1       15432:xxxxx/TCP
```

`EXTERNAL-IP`가 `127.0.0.1`이면 `localhost:15432`로 접속한다. 다른 IP가 표시되면 해당 IP와 `15432` 포트를 사용한다.

연결 확인:

```powershell
Test-NetConnection -ComputerName localhost -Port 15432
```

성공 예시:

```txt
TcpTestSucceeded: True
```

로컬 접속 정보:

```txt
host: localhost
port: 15432
database: vox2vocal
user: vox2vocal
password: vox2vocal
schema: users
```

DATABASE_URL:

```env
DATABASE_URL=postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users
```

`psql`이 설치되어 있다면 다음 명령으로 접속할 수 있다.

```bash
psql "postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users"
```

만약 `minikube tunnel` 환경에서 `localhost:15432` 접속이 되지 않으면 `kubectl get svc -n vox2vocal postgres`의 `EXTERNAL-IP`를 host로 사용한다.

대체 방법으로 `kubectl port-forward`를 사용할 수도 있다. 이 경우에만 별도 터미널에서 다음 명령을 실행한다.

```bash
kubectl port-forward -n vox2vocal svc/postgres 15432:15432
```

port-forward를 사용하면 `localhost:15432`로 접속한다. port-forward 터미널을 닫거나 `Ctrl + C`를 누르면 연결이 종료된다.

### 9. App 실행

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

### 10. macOS App 인증 연동 실행

macOS에서 Web, iOS simulator, Android emulator로 로그인/회원가입을 테스트할 때는 BFF를 Mac의 `localhost:4000`으로 먼저 열어야 한다.

별도 터미널:

```bash
kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
```

포트가 열렸는지 확인한다.

```bash
curl -I http://localhost:4000/health
```

정상 예시:

```txt
HTTP/1.1 200 OK
```

App 기본 BFF endpoint는 플랫폼별로 다르다.

| 실행 대상 | 기본 GraphQL endpoint | 비고 |
| --- | --- | --- |
| Web | `http://localhost:4000/graphql` | browser Origin이 BFF allowlist에 있어야 한다. |
| iOS simulator | `http://localhost:4000/graphql` | simulator의 `localhost`는 Mac host를 가리킨다. |
| Android emulator | `http://10.0.2.2:4000/graphql` | emulator에서 Mac host를 가리키는 특별 주소다. |

Android에서 기존 JS bundle이 `localhost:4000`을 사용하고 있거나, 명시적으로 `EXPO_PUBLIC_BFF_GRAPHQL_URL=http://localhost:4000/graphql`을 넣어 실행한 경우에는 `adb reverse`도 함께 설정한다.

```bash
adb devices
adb reverse tcp:4000 tcp:4000
adb reverse --list
```

정상 예시:

```txt
host-12 tcp:8081 tcp:8081
host-12 tcp:4000 tcp:4000
```

Web 실행 URL은 BFF의 `BFF_ALLOWED_ORIGINS`에 포함되어야 한다. 로컬 개발 기본 허용 Origin은 다음 계열이다.

```txt
http://localhost:8081
http://127.0.0.1:8081
http://localhost:8090
http://127.0.0.1:8090
http://localhost:19006
http://127.0.0.1:19006
```

새 포트를 사용하면 `vox2vocal-infra/k8s/configmap.yaml`의 `BFF_ALLOWED_ORIGINS`에 Origin을 추가하고 BFF를 재시작한다.

```bash
kubectl apply -f vox2vocal-infra/k8s/configmap.yaml
kubectl rollout restart deployment/bff-server -n vox2vocal
kubectl rollout status deployment/bff-server -n vox2vocal
```

macOS에서 port-forward를 계속 유지해야 하면 `screen`을 사용할 수 있다.

```bash
screen -dmS vox2vocal-bff-port-forward kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
screen -ls
```

종료:

```bash
screen -S vox2vocal-bff-port-forward -X quit
```

## 실행 순서 요약

```txt
Docker Desktop 실행
-> minikube start
-> minikube addons enable ingress
-> 별도 터미널에서 minikube tunnel 실행 후 유지
-> minikube image build로 서비스 이미지 빌드
-> 기존 abyul namespace가 있으면 삭제
-> kubectl apply -k infra/k8s
-> 이미지 갱신 시 kubectl rollout restart 실행
-> pods, services, ingress 상태 확인
-> hosts에 vox2vocal.local 등록
-> 로컬 DB 접속이 필요하면 kubectl get svc -n vox2vocal postgres로 EXTERNAL-IP 확인
-> macOS App 인증 테스트는 kubectl port-forward -n vox2vocal svc/bff-server 4000:4000 유지
-> Android emulator는 필요 시 adb reverse tcp:4000 tcp:4000 적용
-> app 실행
```

## 서비스 직접 실행 방식

개발 중 특정 서비스만 빠르게 확인할 때는 각 repo에서 직접 실행할 수 있다.

```bash
cd vox2vocal-bff-server
npm.cmd run start:dev
```

```bash
cd vox2vocal-api-gateway
npm.cmd run start:dev
```

```bash
cd vox2vocal-user-service
npm.cmd run start:dev
```

```bash
cd vox2vocal-worker
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
| `engine-audio-ingest` | HTTP health | `8080` |
| `minio` | S3 API | `9000` |
| `minio` | Console | `9001` |
| `nats` | Client | `4222` |

## 검증 명령

각 서비스 repo에서 가능한 경우 다음 명령으로 검증한다.

```bash
npm.cmd run format:check
npm.cmd run verify
```

`user-service`는 Prisma client 생성이 필요할 수 있다.

```bash
cd vox2vocal-user-service
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
kubectl get statefulset -n vox2vocal
kubectl get deployment -n vox2vocal
```

서비스 로그 확인:

```bash
kubectl logs -n vox2vocal deploy/bff-server
kubectl logs -n vox2vocal deploy/api-gateway
kubectl logs -n vox2vocal deploy/user-service
kubectl logs -n vox2vocal deploy/worker
kubectl logs -n vox2vocal deploy/engine-audio-ingest
kubectl logs -n vox2vocal statefulset/postgres
kubectl logs -n vox2vocal deploy/redis
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
./vox2vocal-infra/scripts/build-local-app-images.sh
docker build --network=host -t vox2vocal/engine-audio-ingest:local ./engine-audio-ingest
minikube image load vox2vocal/engine-audio-ingest:local
kubectl rollout restart deployment -n vox2vocal bff-server api-gateway user-service worker engine-audio-ingest
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
cd vox2vocal-bff-server
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
minikube image build -t vox2vocal/user-service:local ./vox2vocal-user-service
kubectl rollout restart deployment -n vox2vocal user-service
```

### Prisma table does not exist

GraphQL `me` 쿼리 또는 user-service 로그에 다음 오류가 나오면 PostgreSQL은 실행 중이지만 Prisma schema가 데이터베이스에 적용되지 않은 상태다.

```txt
The table `users.users` does not exist in the current database.
```

이 경우 PostgreSQL pod 상태를 먼저 확인한다.

```bash
kubectl get pods -n vox2vocal
kubectl logs -n vox2vocal statefulset/postgres --tail=100
kubectl logs -n vox2vocal deploy/user-service --tail=100
```

그 다음 user-service의 Prisma migration 적용 절차를 진행한다.

### vox2vocal.local 접속 timeout

`kubectl get ingress -n vox2vocal`에서 ingress가 보이고 hosts 파일도 등록되어 있는데 Windows에서 `curl http://vox2vocal.local`이 timeout될 수 있다.

먼저 `minikube tunnel`이 별도 터미널에서 계속 실행 중인지 확인한 뒤, pod와 ingress 상태를 확인한다.

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

### Web, iOS, Android 로그인 전체 실패

`user@example.com / password123` 계정으로 Web, iOS, Android가 모두 로그인되지 않으면 서버 계정 문제보다 로컬 연결 경로 문제일 가능성이 높다.

먼저 서버와 DB 상태를 확인한다.

```bash
kubectl get pods -n vox2vocal
kubectl get svc,endpoints -n vox2vocal
kubectl logs -n vox2vocal deploy/bff-server --tail=100
```

DB 계정은 `users` schema에 있다. `public.users`가 아니라 `users.users`를 확인한다.

```bash
kubectl exec -n vox2vocal postgres-0 -- sh -lc 'psql -U "$POSTGRES_USER" -d "$POSTGRES_DB" -Atc "select u.email, u.\"displayName\", u.role, u.status, count(pc.id) as credential_count from users.users u left join users.user_password_credentials pc on pc.\"userId\" = u.id where u.email = '\''user@example.com'\'' group by u.email, u.\"displayName\", u.role, u.status;"'
```

정상 예시:

```txt
user@example.com|Vox Demo User|USER|ACTIVE|1
```

그 다음 BFF port-forward를 확인한다.

```bash
lsof -nP -iTCP:4000 -sTCP:LISTEN
curl -I http://localhost:4000/health
```

`localhost:4000`이 열려 있지 않으면 다음 명령을 별도 터미널에서 실행한다.

```bash
kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
```

Web에서만 실패하고 BFF 로그에 다음 오류가 나오면 Origin allowlist 문제다.

```txt
Error: Origin not allowed
```

이 경우 현재 Web URL의 origin을 확인한다.

```txt
http://localhost:8081
http://127.0.0.1:8081
```

브라우저 주소가 `localhost`인지 `127.0.0.1`인지, 포트가 무엇인지까지 모두 Origin에 포함된다. `localhost:8081`과 `127.0.0.1:8081`은 서로 다른 Origin이다.

Android에서만 실패하면 emulator가 Mac host의 BFF를 볼 수 있는지 확인한다.

```bash
adb devices
adb reverse --list
adb shell 'toybox nc -z 10.0.2.2 4000 >/dev/null 2>&1; echo android_to_host_4000=$?'
```

정상 예시:

```txt
android_to_host_4000=0
```

Native 경로 smoke test는 토큰 원문을 출력하지 않고 성공 여부만 확인한다.

```bash
node <<'NODE'
const query = `
  mutation Login($input: LoginInput!) {
    login(input: $input) {
      accessToken
      expiresIn
      refreshToken
      user {
        email
        displayName
        role
      }
    }
  }
`

fetch('http://localhost:4000/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Vox2Vocal-Client': 'native',
  },
  body: JSON.stringify({
    query,
    variables: {
      input: {
        email: 'user@example.com',
        password: 'password123',
      },
    },
  }),
})
  .then(async (response) => {
    const payload = await response.json().catch(() => ({}))
    const login = payload.data?.login

    console.log(
      JSON.stringify(
        {
          status: response.status,
          ok: response.ok,
          hasErrors: !!payload.errors,
          hasAccessToken: !!login?.accessToken,
          bodyHasRefreshToken: !!login?.refreshToken,
          userEmail: login?.user?.email,
        },
        null,
        2,
      ),
    )
  })
  .catch((error) => {
    console.error(error)
    process.exit(1)
  })
NODE
```

## 주의사항

- 이 workspace는 모노레포가 아니다.
- 각 하위 폴더는 독립 Git repository이다.
- 변경 사항은 변경한 repo 안에서 개별 commit/push 한다.
- infra repo에는 Kubernetes manifest와 배포 설정만 관리한다.
- 서비스 코드는 infra repo에 포함하지 않는다.
- Git Bash에서도 npm 실행은 `npm.cmd` 기준으로 사용한다.
- Git `dubious ownership` 오류가 발생하면 해당 repo를 `safe.directory`에 등록한다.
