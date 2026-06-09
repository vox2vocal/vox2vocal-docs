# Audio Upload MinIO E2E Test Plan

이 문서는 MinIO 기반 오디오 업로드 E2E 테스트 범위와 실행 방법을 정의한다.

## 1. 목적

E2E 테스트의 1차 목적은 클라이언트가 BFF를 통해 presigned upload URL을 발급받고, 반환된 URL로 MinIO에 원본 오디오를 직접 업로드할 수 있는지 검증하는 것이다.

```txt
Client
  -> BFF GraphQL createAudioUploadSession
  -> API Gateway gRPC CreateAudioUploadSession
  -> API Gateway SigV4 presigned PUT URL 생성
  -> Client PUT upload to MinIO
  -> MinIO object 존재 확인
```

이 테스트는 업로드 바이트가 BFF/API Gateway를 통과하지 않고 object storage로 직접 들어가는지 확인한다.

## 2. 현재 가능한 E2E 범위

현재 코드 기준으로 바로 검증 가능한 범위는 다음이다.

- BFF GraphQL mutation이 API Gateway gRPC로 연결된다.
- API Gateway가 인증된 사용자 기준으로 upload session을 만든다.
- presigned URL의 object key가 `audio-assets/{audio_asset_id}/original/{filename}` 형식이다.
- 클라이언트가 presigned URL로 MinIO에 `PUT` 업로드한다.
- MinIO bucket에 object가 생성된다.

아직 검증 범위에 포함하지 않는 항목:

- 업로드 완료 API
- object `HEAD` 검증
- `audio.ingest.requested` NATS 이벤트 발행
- `engine-audio-ingest`의 실제 ffprobe/ffmpeg 처리
- `audio.ingest.completed` 또는 `audio.ingest.failed` 발행

## 3. 사전 조건

로컬 Kubernetes/minikube 환경 기준으로 다음이 필요하다.

- Docker Desktop
- minikube
- kubectl
- Node.js/npm
- 유효한 access token
- 테스트 오디오 파일
- MinIO API port-forward

테스트용 오디오 fixture는 다음 파일을 우선 사용한다.

```txt
engine-audio-ingest/samples/audio/mp3-input.mp3
```

## 4. 환경 준비

workspace root에서 서비스 이미지를 빌드한다.

```bash
minikube image build -t vox2vocal/bff-server:local ./vox2vocal-bff-server
minikube image build -t vox2vocal/api-gateway:local ./vox2vocal-api-gateway
minikube image build -t vox2vocal/user-service:local ./vox2vocal-user-service
minikube image build -t vox2vocal/worker:local ./vox2vocal-worker
minikube image build -t vox2vocal/engine-audio-ingest:local ./engine-audio-ingest
```

Kubernetes manifest를 적용한다.

```bash
kubectl apply -k vox2vocal-infra/k8s
```

pod와 bucket 생성 Job을 확인한다.

```bash
kubectl get pods -n vox2vocal
kubectl get job -n vox2vocal minio-create-audio-bucket
kubectl logs -n vox2vocal job/minio-create-audio-bucket
```

로컬에서 presigned URL로 접근할 수 있도록 MinIO API를 연다.

```bash
kubectl port-forward -n vox2vocal svc/minio 19000:9000
```

Ingress 대신 port-forward로 BFF를 호출할 경우 별도 터미널에서 다음을 실행한다.

```bash
kubectl port-forward -n vox2vocal svc/bff-server 4000:4000
```

## 5. Upload Session 발급 테스트

테스트는 기존 로그인 E2E나 개발용 인증 흐름으로 `ACCESS_TOKEN`을 먼저 확보한 뒤 진행한다.

브라우저 요청으로 검증할 때는 CSRF/origin guard를 만족해야 한다.

```bash
curl -s -X POST http://localhost:4000/graphql \
  -H "Content-Type: application/json" \
  -H "Origin: http://localhost:4000" \
  -H "x-vox2vocal-csrf: 1" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  --data '{
    "query": "mutation CreateAudioUploadSession($input: CreateAudioUploadSessionInput!) { createAudioUploadSession(input: $input) { audioAssetId bucket sourceObjectKey uploadUrl method headers { name value } expiresIn expiresAt idempotencyKey } }",
    "variables": {
      "input": {
        "originalFilename": "mp3-input.mp3",
        "contentType": "audio/mpeg"
      }
    }
  }'
```

예상 결과:

- GraphQL response에 `errors`가 없다.
- `method`는 `PUT`이다.
- `uploadUrl` host는 local dev 기준 `localhost:19000`이다.
- `sourceObjectKey`는 `audio-assets/{audioAssetId}/original/mp3-input.mp3` 형식이다.
- `headers`에는 `content-type: audio/mpeg`가 포함된다.
- `expiresIn`은 기본 `900`초다.

## 6. MinIO PUT 업로드 테스트

GraphQL 응답에서 `uploadUrl`을 추출한 뒤 같은 `content-type`으로 업로드한다.

```bash
curl -i -X PUT "${UPLOAD_URL}" \
  -H "Content-Type: audio/mpeg" \
  --data-binary @engine-audio-ingest/samples/audio/mp3-input.mp3
```

예상 결과:

- HTTP status가 `200` 또는 `204`이다.
- API Gateway와 BFF 로그에 upload body 처리 로그가 없어야 한다.
- MinIO에 object가 생성되어야 한다.

MinIO object 확인은 local `mc`가 있으면 다음 방식으로 수행한다.

```bash
mc alias set vox2vocal-local http://localhost:19000 vox2vocal vox2vocal-minio-dev-secret
mc stat "vox2vocal-local/vox2vocal-audio-assets/${SOURCE_OBJECT_KEY}"
```

local `mc`가 없으면 MinIO console port-forward 후 console에서 bucket/object를 확인한다.

```bash
kubectl port-forward -n vox2vocal svc/minio 19001:9001
```

## 7. 필수 Negative Test

업로드 E2E에서 최소 다음 실패 케이스를 함께 확인한다.

| 케이스 | 입력 | 기대 결과 |
| --- | --- | --- |
| 인증 없음 | `Authorization` header 누락 | GraphQL error 또는 unauthorized |
| CSRF 누락 | `x-vox2vocal-csrf` 누락 | forbidden |
| non-audio content type | `contentType: text/plain` | upload session 생성 실패 |
| content-type 불일치 PUT | URL 발급은 `audio/mpeg`, PUT은 다른 content-type | MinIO signature mismatch 또는 upload 실패 |
| 만료 URL | `expiresIn` 이후 PUT | upload 실패 |

## 8. Ingest 요청까지 확장하는 E2E

업로드 후 `audio.ingest.requested` 발행까지 검증하려면 다음 개발이 먼저 필요하다.

- BFF GraphQL `completeAudioUpload` mutation
- API Gateway gRPC `CompleteAudioUpload` RPC
- API Gateway object storage `HEAD` 검증
- object size/content-type 검증
- `audio_asset_id`와 `source_object_key` 소유 경계 검증
- idempotency 처리
- NATS JetStream publisher
- `audio.ingest.requested` payload 생성

확장 후 E2E 흐름:

```txt
createAudioUploadSession
  -> PUT to MinIO
  -> completeAudioUpload
  -> API Gateway HEAD object
  -> publish audio.ingest.requested
  -> engine-audio-ingest receives and ack
```

예상 검증:

- `completeAudioUpload`가 성공한다.
- NATS stream `VOX2VOCAL_AUDIO`에 `audio.ingest.requested`가 저장된다.
- engine log에 `audio_ingest_request_received`가 남는다.
- invalid object 또는 missing object는 ingest 요청을 발행하지 않는다.

## 9. Engine 완료까지 확장하는 E2E

진짜 ingest 완료 E2E는 다음 engine task가 끝난 뒤 가능하다.

- Task 6: FFprobe Wrapper
- Task 7: FFmpeg Canonical Convert Wrapper
- Task 8: Manifest 생성
- Task 9: Ingest Processor 연결

확장 후 최종 E2E 흐름:

```txt
createAudioUploadSession
  -> PUT to MinIO
  -> completeAudioUpload
  -> audio.ingest.requested
  -> engine ffprobe
  -> engine ffmpeg canonical WAV
  -> manifest 생성
  -> canonical WAV와 manifest 저장
  -> audio.ingest.completed 발행
```

예상 검증:

- `audio-assets/{audio_asset_id}/canonical/audio.wav`가 생성된다.
- `audio-assets/{audio_asset_id}/manifest.json`이 생성된다.
- manifest의 source/canonical object key가 event payload와 일치한다.
- `audio.ingest.completed`가 발행된다.
- 실패 fixture는 `audio.ingest.failed`로 끝난다.

## 10. 완료 기준

업로드 E2E 완료 기준:

- upload session 발급 성공
- presigned URL `PUT` 성공
- MinIO object 존재 확인
- 인증/CSRF/content-type 대표 실패 케이스 통과
- 테스트 절차를 반복 실행해도 이전 object와 충돌하지 않음

ingest E2E 완료 기준:

- 업로드 완료 후 `audio.ingest.requested`가 발행됨
- engine consumer가 요청을 수신하고 ack함
- canonical WAV와 manifest가 object storage에 저장됨
- completed/failed 이벤트가 정책대로 발행됨

## 11. 다음 개발 태스크

현재 `engine-audio-ingest/TASKS.md` 기준 다음 engine task는 Task 6 `FFprobe Wrapper`다.

다만 upload E2E를 ingest 요청까지 연결하려면 engine Task 6 전에 cross-service task가 하나 더 필요하다.

```txt
Upload completion boundary
  -> BFF completeAudioUpload mutation
  -> API Gateway CompleteAudioUpload RPC
  -> MinIO HEAD 검증
  -> NATS audio.ingest.requested 발행
```

따라서 진행 순서는 목적에 따라 나눈다.

- 업로드 E2E를 먼저 닫는다: upload session, PUT, MinIO object 확인 자동화
- ingest pipeline을 먼저 전진한다: engine Task 6 FFprobe Wrapper 구현
- 전체 E2E를 목표로 한다: completeAudioUpload boundary를 추가한 뒤 Task 6부터 Task 9까지 진행
