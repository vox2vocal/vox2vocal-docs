# API Gateway Testing Guide

이 문서는 `api-gateway`에서 테스트 코드를 작성할 때 따를 기준을 정리한다.
`api-gateway`는 단순 HTTP 서버가 아니라 외부에는 gateway gRPC contract를 제공하고, 내부로는 `user-service` gRPC client를 호출하며, JWT token을 발급/검증하는 경계 서비스다.

## 1. 현재 스택 인벤토리

테스트를 설계할 때 아래 스택을 모두 고려한다.

| 영역 | 사용 스택 |
| --- | --- |
| Runtime | Node.js 22, Docker `node:22-alpine` |
| Framework | NestJS 11 |
| Transport | HTTP health endpoint, gRPC microservice |
| Inbound gRPC | `proto/gateway.proto`, `GatewayService` |
| Downstream gRPC | `proto/user.proto`, `UserService` client |
| Nest Microservices | `@nestjs/microservices`, `Transport.GRPC`, `ClientGrpc`, `ClientsModule.registerAsync()` |
| Config | `@nestjs/config`, `.env`, `PORT`, `GRPC_URL`, `USER_SERVICE_GRPC_URL`, JWT secrets |
| Auth | `@nestjs/jwt`, `JwtService.signAsync()`, `JwtService.verifyAsync()` |
| Async | RxJS `Observable`, `firstValueFrom()` |
| Test | Jest, ts-jest, `@nestjs/testing` |
| Quality | TypeScript strict mode, ESLint, Prettier |

## 2. 기본 원칙

- 테스트의 중심은 gateway가 맡는 경계 변환과 orchestration에 둔다.
- `GatewayController`는 user-service 호출, token 발급/검증, protobuf response mapping을 검증한다.
- `TokenService`는 실제 `JwtService`를 사용해 access/refresh token의 secret, type, TTL, 실패 처리를 검증한다.
- `UserClientService`는 실제 user-service에 연결하지 않고 `ClientGrpc`와 gRPC service stub을 mock한다.
- gRPC contract는 inbound gateway proto와 downstream user proto를 분리해서 검증한다.
- ConfigModule/env 기본값은 현재 코드 위치를 기준으로 검증 가능 범위를 명확히 나눈다.
- e2e test는 적게 유지하고, HTTP health와 gRPC happy path 중심으로 둔다.
- downstream user-service의 비즈니스 로직은 api-gateway 테스트에서 재검증하지 않는다.

## 3. 권장 테스트 피라미드

```txt
많음  Unit test
      - TokenService
      - GatewayController
      - UserClientService with mocked ClientGrpc
      - mapper/helper logic

중간  Integration test
      - GatewayModule provider wiring
      - UserClientModule ClientsModule config
      - ConfigModule/env behavior

적음  E2E test
      - HTTP GET /health
      - gRPC GatewayService.Login happy path
      - gRPC GatewayService.GetCurrentUser happy path
```

## 4. 파일 배치

신규 테스트는 목적별로 분리한다.

```txt
api-gateway/
  test/
    unit/
      auth/
        token.service.spec.ts
      gateway/
        gateway.controller.spec.ts
      user-client/
        user-client.service.spec.ts
    integration/
      config/
      gateway/
      user-client/
    e2e/
      health.e2e-spec.ts
      gateway.grpc.e2e-spec.ts
    helpers/
      fixtures.ts
      grpc-stubs.ts
```

기존 `test/*.spec.ts` 또는 `test/<domain>/*.spec.ts`가 있어도 신규 테스트는 위 구조를 우선한다.
기존 테스트는 기회가 있을 때 같은 기준으로 이동한다.

## 5. GatewayController 테스트 기준

`GatewayController`는 inbound gRPC request를 받아 user-service client와 token service를 조합하는 adapter/orchestrator다.

검증 대상:

- `LoginRequest.email`, `LoginRequest.password`를 `UserClientService.authenticateUser()`에 전달
- login 성공 시 `TokenService.issueTokens()` 호출
- login response shape: `access_token`, `refresh_token`, `expires_in`, `user`
- `GetCurrentUserRequest.access_token` 또는 camelCase `accessToken` 처리
- access token payload의 `sub`로 `UserClientService.getUser()` 호출
- user 내부 모델 `displayName`을 protobuf `display_name`으로 변환
- missing login fields는 현재 코드처럼 empty string으로 전달
- missing access token은 demo user fallback 없이 token 검증으로 실패

controller unit test에서는 gRPC network를 띄우지 않는다.
`UserClientService`와 `TokenService`를 mock하고, controller 메서드를 직접 호출한다.

```ts
const userClient = {
  authenticateUser: jest.fn(),
  getUser: jest.fn(),
}

const tokenService = {
  issueTokens: jest.fn(),
  verifyAccessToken: jest.fn(),
}

const controller = new GatewayController(
  userClient as unknown as UserClientService,
  tokenService as unknown as TokenService,
)
```

비즈니스 edge case는 downstream service의 책임이면 여기서 반복하지 않는다.
예를 들어 password 실패 정책은 `user-service` 테스트에서 검증하고, gateway에서는 error propagation만 확인한다.

## 6. TokenService 테스트 기준

`TokenService`는 gateway의 보안 경계다.
단순 mock만으로 끝내지 말고 실제 `JwtService`를 사용해 token round-trip을 검증한다.

검증 대상:

- access token과 refresh token을 모두 발급
- access token과 refresh token이 서로 다름
- `expiresIn`은 access token TTL인 `900`
- access token payload: `sub`, `email`, `role`, `type: 'access'`
- refresh token payload: `sub`, `email`, `role`, `type: 'refresh'`
- access token 검증 경로에서 refresh token은 거부
- refresh token 검증 경로에서 access token은 거부
- missing token은 `UnauthorizedException`
- malformed token은 `UnauthorizedException`
- 잘못된 secret으로 서명된 token은 `UnauthorizedException`
- env secret이 없으면 dev default secret을 사용한다는 현재 동작

권장 패턴:

```ts
const configService = {
  get: jest.fn((key: string, defaultValue: string) => {
    const values: Record<string, string> = {
      JWT_ACCESS_SECRET: 'test-access-secret',
      JWT_REFRESH_SECRET: 'test-refresh-secret',
    }

    return values[key] ?? defaultValue
  }),
}

const tokenService = new TokenService(configService as unknown as ConfigService, new JwtService())
```

`JwtService.signAsync()`와 `JwtService.verifyAsync()`를 사용하는 현재 구조에 맞춰 테스트도 async matcher를 사용한다.

```ts
await expect(tokenService.verifyAccessToken(token)).resolves.toMatchObject({
  sub: 'user-id',
  type: 'access',
})
```

## 7. UserClientService 테스트 기준

`UserClientService`는 downstream `user-service` gRPC client boundary다.
unit test에서는 실제 `user-service`를 띄우지 않고 `ClientGrpc.getService()`가 반환하는 service stub을 mock한다.

검증 대상:

- `onModuleInit()`에서 `client.getService('UserService')` 호출
- `getUser(userId)`가 downstream request `{ user_id: userId }`로 호출
- `authenticateUser(email, password)`가 downstream request `{ email, password }`로 호출
- downstream Observable을 `firstValueFrom()`으로 Promise화한 결과를 반환
- downstream `display_name`을 internal `displayName`으로 변환
- downstream `displayName` camelCase도 허용
- downstream response에 `user`가 없으면 명시적 error
- downstream Observable error가 그대로 전파

권장 stub 패턴:

```ts
import { of, throwError } from 'rxjs'

const userGrpcService = {
  getUser: jest.fn(),
  authenticateUser: jest.fn(),
}

const clientGrpc = {
  getService: jest.fn().mockReturnValue(userGrpcService),
}

const service = new UserClientService(clientGrpc as unknown as ClientGrpc)
service.onModuleInit()

userGrpcService.getUser.mockReturnValue(
  of({
    id: 'user-id',
    email: 'user@example.com',
    display_name: 'User',
    role: 'USER',
  }),
)
```

`UserClientService` 테스트에서 검증할 것은 gRPC client adapter 동작이다.
`UserService`의 인증 정책, DB 조회, 계정 잠금 정책은 `user-service` 테스트 범위다.

## 8. gRPC와 Protobuf Contract 테스트 기준

`api-gateway`에는 두 종류의 proto contract가 있다.

Inbound gateway contract:

- package name: `gateway`
- service name: `GatewayService`
- rpc: `Login`, `GetCurrentUser`
- `LoginRequest`: `email`, `password`
- `GetCurrentUserRequest`: `access_token`
- `AuthTokenResponse`: `access_token`, `refresh_token`, `expires_in`, `user`
- `AuthTokenResponse.user`: `UserResponse`
- `UserResponse`: `id`, `email`, `display_name`, `role`

Downstream user-service contract:

- package name: `user`
- service name: `UserService`
- rpc: `GetUser`, `AuthenticateUser`
- `GetUserRequest`: `user_id`
- `AuthenticateUserRequest`: `email`, `password`
- `AuthenticateUserResponse`: wrapper field `user`
- `AuthenticateUserResponse.user`: user response shape

controller unit test에서는 inbound response mapping을 검증한다.
`UserClientService` unit test에서는 downstream request/response mapping을 검증한다.
gRPC e2e test에서는 실제 `GatewayService`를 proto client로 호출해 service/package/rpc wiring을 최소 happy path로 검증한다.

## 9. ConfigModule과 Env 테스트 기준

`api-gateway`는 `@nestjs/config`를 전역으로 사용한다.

검증 대상:

- gateway inbound HTTP 기본 포트: `PORT` 없으면 `3001`
- gateway inbound gRPC 기본 URL: `GRPC_URL` 없으면 `0.0.0.0:50050`
- downstream user-service 기본 URL: `USER_SERVICE_GRPC_URL` 없으면 `localhost:50051`
- JWT access secret: `JWT_ACCESS_SECRET`
- JWT refresh secret: `JWT_REFRESH_SECRET`
- 테스트에서 env를 바꾼 경우 spec 종료 후 원복

현재 코드에서는 `PORT`와 `GRPC_URL` 기본값이 `src/main.ts`의 `bootstrap()` 내부에 직접 정의되어 있다.
ConfigModule만 import하는 테스트로는 이 기본값 경로를 검증할 수 없다.

현재 구조에서 가능한 검증:

- `UserClientModule`의 `ClientsModule.registerAsync()` factory가 `USER_SERVICE_GRPC_URL` 기본값과 override 값을 사용하는지 검증
- `TokenService`가 JWT secret env/default 값을 사용하는지 검증
- `PORT`, `GRPC_URL`은 `main.ts` bootstrap 테스트를 작성하거나, config helper/provider로 분리한 뒤 검증

권장 리팩터링 방향:

```ts
export type ApiGatewayConfig = {
  httpPort: number
  grpcUrl: string
  userServiceGrpcUrl: string
}

export function getApiGatewayConfig(configService: ConfigService): ApiGatewayConfig {
  return {
    httpPort: configService.get<number>('PORT', 3001),
    grpcUrl: configService.get<string>('GRPC_URL', '0.0.0.0:50050'),
    userServiceGrpcUrl: configService.get<string>('USER_SERVICE_GRPC_URL', 'localhost:50051'),
  }
}
```

위처럼 분리한 뒤 helper unit test에서 기본값과 env override를 직접 검증한다.
분리 전에는 `PORT`, `GRPC_URL` 기본값 테스트를 ConfigModule 테스트로 작성하지 않는다.

## 10. Integration Test 기준

integration test는 Nest provider wiring과 module factory 설정을 검증한다.

권장 대상:

- `GatewayModule`이 `GatewayController`, `UserClientService`, `TokenService` dependency를 올바르게 주입하는지
- `AuthModule`이 `TokenService`와 `JwtService`를 함께 제공하는지
- `UserClientModule`이 `USER_PACKAGE` gRPC client provider를 등록하는지
- `ClientsModule.registerAsync()` factory가 `USER_SERVICE_GRPC_URL`을 반영하는지

실제 downstream `user-service`를 호출하는 테스트는 integration test가 아니라 contract/e2e 성격으로 분리한다.
일반 integration test에서는 provider override를 사용해 network를 끊는다.

```ts
const moduleRef = await Test.createTestingModule({
  imports: [GatewayModule],
})
  .overrideProvider(UserClientService)
  .useValue(userClientMock)
  .overrideProvider(TokenService)
  .useValue(tokenServiceMock)
  .compile()
```

Nest app 또는 testing module이 lifecycle을 갖는 경우 `afterAll`에서 정리한다.

```ts
await app.close()
```

## 11. E2E 테스트 기준

E2E 테스트는 수를 적게 유지한다.
목적은 전체 wiring과 transport가 살아 있는지 확인하는 것이다.

권장 대상:

- HTTP `GET /health`
- gRPC `GatewayService.Login` happy path
- gRPC `GatewayService.GetCurrentUser` happy path
- malformed token 대표 케이스 1개

E2E에서 실제 `user-service`를 사용할지 mock gRPC user-service를 사용할지 테스트 목적별로 나눈다.

권장:

- gateway 자체 wiring 검증: fake user-service gRPC server 사용
- 서비스 간 통합 검증: 별도 contract/e2e suite에서 실제 `user-service` 사용

E2E는 포트 충돌을 피해야 한다.
`PORT`, `GRPC_URL`, `USER_SERVICE_GRPC_URL`은 테스트 전용 값으로 설정하고, 테스트 종료 후 app과 fake server를 반드시 닫는다.

## 12. Fixture 작성 규칙

테스트 데이터는 helper function으로 만든다.

```ts
function createGatewayUser(overrides: Partial<GatewayUser> = {}): GatewayUser {
  return {
    id: 'user-id',
    email: 'user@example.com',
    displayName: 'User',
    role: 'USER',
    ...overrides,
  }
}
```

규칙:

- fixture는 기본적으로 valid object를 반환한다.
- 테스트별 차이는 `overrides`로 표현한다.
- token fixture는 가능하면 실제 `TokenService.issueTokens()`로 생성한다.
- 여러 spec에서 반복되면 `test/helpers/`로 이동한다.

## 13. Assertion 기준

좋은 assertion:

```ts
expect(userClient.authenticateUser).toHaveBeenCalledWith('user@example.com', 'password')
expect(tokenService.issueTokens).toHaveBeenCalledWith({
  id: 'user-id',
  email: 'user@example.com',
  role: 'USER',
})
await expect(tokenService.verifyAccessToken(tokens.refreshToken)).rejects.toThrow(
  UnauthorizedException,
)
```

피해야 할 assertion:

```ts
expect(result).toBeTruthy()
expect(mock).toHaveBeenCalled()
```

단순 truthy, 호출 여부만 확인하는 assertion은 실패 원인을 충분히 설명하지 못한다.
가능하면 값, exception type, 호출 인자, response shape를 구체적으로 검증한다.

## 14. 명령어

PowerShell에서는 `npm` 대신 `npm.cmd`를 우선 사용한다.

```bash
npm.cmd test -- --runInBand
npm.cmd run typecheck
npm.cmd run verify
```

문서만 수정한 경우에도 가능한 범위에서 최소 검증을 수행한다.
테스트 코드를 수정했다면 `npm.cmd test -- --runInBand`는 반드시 실행한다.

## 15. 참고 문서

- NestJS Testing: https://docs.nestjs.com/fundamentals/testing
- NestJS gRPC Microservices: https://docs.nestjs.com/microservices/grpc
- NestJS Microservice Basics: https://docs.nestjs.com/microservices/basics
- NestJS JWT: https://github.com/nestjs/jwt
