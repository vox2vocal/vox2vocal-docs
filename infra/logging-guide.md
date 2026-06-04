# Vox2Vocal 로그 확인 가이드

이 문서는 Windows Git Bash에서 minikube/Kubernetes로 실행 중인 Vox2Vocal 서비스 로그를 확인하는 방법을 정리한다.

## 기본 상태 확인

먼저 minikube와 Kubernetes 리소스 상태를 확인한다.

```bash
minikube status
kubectl get pods -n vox2vocal
kubectl get svc -n vox2vocal
kubectl get ingress -n vox2vocal
```

pod가 정상이라면 다음처럼 `READY`가 `1/1`, `STATUS`가 `Running`이어야 한다.

```txt
api-gateway    1/1 Running
bff-server     1/1 Running
postgres       1/1 Running
redis          1/1 Running
user-service   1/1 Running
worker         1/1 Running
```

## 서비스별 로그 확인

Deployment 기준으로 로그를 확인한다.

```bash
kubectl logs -n vox2vocal deploy/bff-server
kubectl logs -n vox2vocal deploy/api-gateway
kubectl logs -n vox2vocal deploy/user-service
kubectl logs -n vox2vocal deploy/worker
```

PostgreSQL과 Redis는 pod 또는 StatefulSet/Deployment 기준으로 확인한다.

```bash
kubectl logs -n vox2vocal statefulset/postgres
kubectl logs -n vox2vocal deploy/redis
```

## 최근 로그만 보기

최근 N줄만 확인하려면 `--tail` 옵션을 사용한다.

```bash
kubectl logs -n vox2vocal deploy/bff-server --tail=100
kubectl logs -n vox2vocal deploy/api-gateway --tail=100
kubectl logs -n vox2vocal deploy/user-service --tail=100
kubectl logs -n vox2vocal deploy/worker --tail=100
```

## 실시간 로그 보기

실시간으로 로그를 따라가려면 `-f` 옵션을 사용한다.

```bash
kubectl logs -n vox2vocal deploy/bff-server -f
```

특정 서비스의 문제를 재현하면서 로그를 볼 때 유용하다.

예를 들어 GraphQL 요청을 확인할 때는 한 터미널에서 로그를 켜고:

```bash
kubectl logs -n vox2vocal deploy/bff-server -f
```

다른 터미널에서 요청을 보낸다.

```bash
curl -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  --data-raw '{"query":"{ __typename }"}'
```

## 이전 컨테이너 로그 보기

`CrashLoopBackOff`, `Error`, 재시작이 발생한 pod는 현재 컨테이너 로그보다 이전 컨테이너 로그가 더 중요할 수 있다.

```bash
kubectl logs -n vox2vocal deploy/bff-server --previous
kubectl logs -n vox2vocal deploy/api-gateway --previous
kubectl logs -n vox2vocal deploy/user-service --previous
kubectl logs -n vox2vocal deploy/worker --previous
```

최근 로그와 함께 볼 수도 있다.

```bash
kubectl logs -n vox2vocal deploy/bff-server --previous --tail=100
```

## pod 이름으로 로그 보기

Deployment 기준 로그가 애매할 때는 pod 이름을 직접 지정한다.

pod 목록 확인:

```bash
kubectl get pods -n vox2vocal
```

예시:

```bash
kubectl logs -n vox2vocal bff-server-574866d98d-bdpgn
kubectl logs -n vox2vocal user-service-5b46c4887b-wc4h4
```

이전 컨테이너 로그:

```bash
kubectl logs -n vox2vocal bff-server-574866d98d-bdpgn --previous
```

## pod 상세 이벤트 확인

로그만으로 원인이 안 보이면 pod 이벤트를 확인한다.

```bash
kubectl describe pod -n vox2vocal <pod-name>
```

예시:

```bash
kubectl describe pod -n vox2vocal bff-server-574866d98d-bdpgn
```

주로 확인할 항목:

- `Events`
- `State`
- `Last State`
- `Restart Count`
- `Readiness probe`
- `Liveness probe`
- `Image`
- `ImagePullBackOff`

## Ingress 로그 확인

`vox2vocal.local` 접속 문제는 ingress-nginx controller 로그를 확인한다.

```bash
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller -f
```

Ingress 리소스 상태:

```bash
kubectl get ingress -n vox2vocal
kubectl describe ingress -n vox2vocal vox2vocal-ingress
```

## GraphQL 요청 로그 확인 흐름

Windows에서 `vox2vocal.local` 접근이 timeout될 수 있으므로, 우선 port-forward로 BFF에 직접 연결해서 확인한다.

터미널 1:

```bash
kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
```

터미널 2:

```bash
curl http://localhost:4000/health
```

GraphQL endpoint는 POST 요청으로 확인한다.

```bash
curl -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  --data-raw '{"query":"{ __typename }"}'
```

BFF 로그:

```bash
kubectl logs -n vox2vocal deploy/bff-server --tail=100
```

내부 gRPC 호출 문제가 의심되면 api-gateway와 user-service 로그도 함께 확인한다.

```bash
kubectl logs -n vox2vocal deploy/api-gateway --tail=100
kubectl logs -n vox2vocal deploy/user-service --tail=100
```

## 자주 보는 오류

### ImagePullBackOff

이미지를 찾지 못하는 상태다.

```bash
kubectl get pods -n vox2vocal
minikube image ls | grep vox2vocal
```

이미지가 없으면 다시 빌드한다.

```bash
minikube image build -t vox2vocal/bff-server:local ./bff-server
minikube image build -t vox2vocal/api-gateway:local ./api-gateway
minikube image build -t vox2vocal/user-service:local ./user-service
minikube image build -t vox2vocal/worker:local ./worker
kubectl rollout restart deployment -n vox2vocal bff-server api-gateway user-service worker
```

### CrashLoopBackOff

컨테이너가 실행 후 반복적으로 종료되는 상태다.

```bash
kubectl logs -n vox2vocal deploy/bff-server --previous --tail=100
kubectl describe pod -n vox2vocal <pod-name>
```

### Cannot find module '/app/dist/main.js'

Dockerfile 실행 경로와 NestJS 빌드 출력 경로가 맞지 않을 때 발생한다.

```dockerfile
CMD ["node", "dist/src/main.js"]
```

### Prisma table does not exist

예시:

```txt
The table `users.users` does not exist in the current database.
```

`user-service`의 Prisma migration이 데이터베이스에 적용되지 않은 상태다. `user-service` 로그와 migration 상태를 확인한다.

```bash
kubectl logs -n vox2vocal deploy/user-service --tail=100
```

## 전체 로그 점검 순서

문제가 발생하면 다음 순서로 확인한다.

```txt
1. kubectl get pods -n vox2vocal
2. 문제가 있는 pod의 STATUS 확인
3. kubectl logs ... --previous
4. kubectl describe pod ...
5. 관련 서비스 로그 함께 확인
6. ingress 문제면 ingress-nginx 로그 확인
7. Windows 접속 문제면 port-forward로 우회 확인
```
