# User Service Testing Guide

이 문서는 `user-service`에서 테스트 코드를 작성할 때 따를 기준을 정리한다.
현재 코드의 임시 형태보다 NestJS, CQRS, Prisma, gRPC, PostgreSQL 조합에서 권장되는 테스트 경계를 우선한다.

## 1. 현재 스택 인벤토리

테스트를 설계할 때 아래 스택을 모두 고려한다.

| 영역 | 사용 스택 |
| --- | --- |
| Runtime | Node.js 22, Docker `node:22-alpine` |
| Framework | NestJS 11 |
| Architecture | Nest Module, CQRS Command/Query Handler, Repository, Policy, Mapper |
| Transport | HTTP health endpoint, gRPC microservice |
| gRPC | `@nestjs/microservices`, `@grpc/grpc-js`, `@grpc/proto-loader`, `proto/user.proto` |
| Config | `@nestjs/config`, `.env`, `PORT`, `GRPC_URL`, `DATABASE_URL` |
| Database | PostgreSQL, Prisma ORM 6, Prisma Migrate |
| Prisma lifecycle | `PrismaService extends PrismaClient`, `OnModuleInit`, `OnModuleDestroy` |
| Security | Argon2 password hash/verify, login failure policy, temporary lock |
| Test | Jest, ts-jest, `@nestjs/testing`, `jest-mock-extended` |
| Quality | TypeScript strict mode, ESLint, Prettier |

## 2. 기본 원칙

- 테스트의 중심은 HTTP/gRPC controller가 아니라 CQRS command/query handler에 둔다.
- Prisma Client를 사용하는 코드는 unit test에서 실제 DB에 연결하지 않는다.
- repository의 Prisma query, relation include, unique constraint, transaction 동작은 integration test에서 실제 테스트 DB로 검증한다.
- policy, mapper, value 계산 로직은 Nest TestingModule 없이 순수 unit test로 검증한다.
- controller 테스트는 transport adapter 역할만 확인한다. 비즈니스 케이스를 controller 테스트에서 반복하지 않는다.
- gRPC contract는 request/response field mapping과 proto package/service name을 별도 관점으로 검증한다.
- ConfigModule/env 의존 코드는 기본값과 override 값을 모두 검증하되, 현재처럼 기본값이 `main.ts` bootstrap 내부에 있으면 ConfigModule 테스트만으로 검증하지 않는다.
- 날짜가 결과에 영향을 주는 코드는 고정된 `Date` 값을 사용한다.
- 실패 케이스는 성공 케이스와 같은 우선순위로 작성한다.
- 테스트 데이터는 각 spec 안의 fixture helper로 만든다. 테스트 간 공유 mutable fixture를 두지 않는다.

## 3. 권장 테스트 피라미드

```txt
많음  Unit test
      - policy
      - mapper
      - security service
      - command/query handler with mocked dependencies

중간  Integration test
      - Nest provider wiring
      - controller to CommandBus/QueryBus mapping
      - ConfigModule/env behavior
      - Prisma repository with real test database

적음  E2E test
      - HTTP health check
      - gRPC happy path
      - one or two critical auth flows
```

## 4. 파일 배치

신규 테스트는 목적별로 분리한다.

```txt
user-service/
  test/
    unit/
      users/
        policies/
        commands/
        queries/
        mappers/
        security/
    integration/
      config/
      users/
        repositories/
        controllers/
    e2e/
      health.e2e-spec.ts
      user-service.grpc.e2e-spec.ts
    helpers/
      fixtures.ts
      prisma-test-client.ts
```

기존 `test/*.spec.ts` 파일이 있어도 신규 테스트는 위 구조를 우선한다.
기존 테스트는 기회가 있을 때 같은 기준으로 이동한다.

## 5. Unit Test 작성 기준

### 5.1 Policy와 Mapper

다음 유형은 Nest TestingModule을 사용하지 않는다.

- `LoginPolicy`
- `toUserResponse` 같은 mapper
- 입력값 정규화 함수
- 순수 계산 로직

예시:

```ts
describe('LoginPolicy', () => {
  const now = new Date('2026-06-05T00:00:00.000Z')

  it('locks password login after max failures', () => {
    const policy = new LoginPolicy()

    expect(policy.nextFailureState(credential, now)).toEqual({
      failedLoginCount: 5,
      lockedUntil: new Date('2026-06-05T00:05:00.000Z'),
    })
  })
})
```

### 5.2 Security Service

Argon2 기반 password hash/verify는 보안 경계로 취급한다.

검증 대상:

- plain password를 hash로 변환하는지
- 올바른 password는 verify `true`
- 틀린 password는 verify `false`
- 지원하지 않는 hash format은 throw 대신 `false`
- generated hash가 plain password와 다른지

Argon2 자체 알고리즘을 재검증하지 않는다. 이 서비스의 책임은 라이브러리 호출과 실패 처리 경계다.

### 5.3 CQRS Command Handler

command handler는 애플리케이션 유스케이스의 핵심 테스트 대상이다.
DB, 외부 API, message broker를 직접 호출하지 않고 dependency를 mock한다.

검증 대상:

- command 입력값 정규화
- repository 호출 인자
- domain policy 적용
- 성공 시 상태 변경 호출
- 실패 시 상태 변경 호출
- exception type과 message
- transaction이 필요한 경우 transaction boundary 호출

`jest-mock-extended`를 사용해 타입 안전한 mock을 만든다.

```ts
import { mock, mockReset, MockProxy } from 'jest-mock-extended'

describe('AuthenticateUserHandler', () => {
  let handler: AuthenticateUserHandler
  let usersRepository: MockProxy<UsersRepository>
  let passwordCredentialsRepository: MockProxy<PasswordCredentialsRepository>
  let passwordHashService: MockProxy<PasswordHashService>

  beforeEach(() => {
    usersRepository = mock<UsersRepository>()
    passwordCredentialsRepository = mock<PasswordCredentialsRepository>()
    passwordHashService = mock<PasswordHashService>()

    handler = new AuthenticateUserHandler(
      usersRepository,
      passwordCredentialsRepository,
      passwordHashService,
      new LoginPolicy(),
    )
  })

  afterEach(() => {
    mockReset(usersRepository)
    mockReset(passwordCredentialsRepository)
    mockReset(passwordHashService)
  })
})
```

핸들러 테스트에서 `Test.createTestingModule()`은 필수로 쓰지 않는다.
DI wiring 자체를 검증해야 하는 경우에만 사용한다.

### 5.4 CQRS Query Handler

query handler는 조회 조건, not found 처리, 반환 모델을 검증한다.

검증 대상:

- query 값을 repository에 올바르게 전달하는지
- 없을 때 `NotFoundException`을 던지는지
- 반환 타입이 read model에 맞는지

### 5.5 Controller

controller는 얇게 테스트한다.
비즈니스 시나리오는 handler test로 이동한다.

검증 대상:

- gRPC request field 변환
- `CommandBus.execute()` 또는 `QueryBus.execute()` 호출
- response DTO mapping
- default/fallback 입력 처리

controller 테스트에서는 `CommandBus`, `QueryBus`를 mock한다.

```ts
const commandBus = mock<CommandBus>()
const queryBus = mock<QueryBus>()
const controller = new UsersController(commandBus, queryBus)
```

## 6. jest-mock-extended 사용 기준

`jest-mock-extended`는 TypeScript 타입을 유지하면서 mock을 만들기 위해 사용한다.

### 6.1 일반 provider mock

Nest provider, repository, policy dependency는 `mock<T>()`를 우선 사용한다.

```ts
const usersRepository = mock<UsersRepository>()

usersRepository.findByEmailWithPasswordCredential
  .calledWith('user@example.com')
  .mockResolvedValue(user)
```

### 6.2 Prisma Client deep mock

Prisma Client 또는 PrismaService delegate를 직접 mock해야 하는 테스트에서는 `mockDeep<T>()`를 사용한다.

```ts
import { PrismaClient } from '@prisma/client'
import { DeepMockProxy, mockDeep, mockReset } from 'jest-mock-extended'

let prisma: DeepMockProxy<PrismaClient>

beforeEach(() => {
  prisma = mockDeep<PrismaClient>()
})

afterEach(() => {
  mockReset(prisma)
})

it('finds a user by id', async () => {
  prisma.user.findUnique.mockResolvedValue(user)

  await expect(prisma.user.findUnique({ where: { id: 'user-id' } })).resolves.toEqual(user)
})
```

handler unit test에서는 Prisma deep mock보다 repository mock을 우선한다.
Prisma deep mock은 repository unit test 또는 PrismaService boundary test에서만 사용한다.

### 6.3 calledWith 사용

특정 인자에 따른 반환값을 구분해야 할 때 `calledWith()`를 사용한다.

```ts
usersRepository.findByEmailWithPasswordCredential
  .calledWith('active@example.com')
  .mockResolvedValue(activeUser)

usersRepository.findByEmailWithPasswordCredential
  .calledWith('disabled@example.com')
  .mockResolvedValue(disabledUser)
```

단순한 케이스에서는 `mockResolvedValue()`만 사용해도 된다.

## 7. Prisma 테스트 기준

### 7.1 Unit Test에서 Prisma를 다루는 법

unit test에서 실제 Prisma Client를 생성하거나 DB에 연결하지 않는다.
Prisma 공식 문서 기준으로는 Prisma Client mock 또는 dependency injection 방식을 사용한다.

권장:

- handler test에서는 Prisma를 직접 mock하지 말고 repository를 mock한다.
- repository unit test가 꼭 필요하면 PrismaService 또는 PrismaClient delegate를 `mockDeep()`으로 mock한다.
- Prisma mock은 `mockReset()`으로 테스트마다 초기화한다.

비권장:

- unit test에서 `DATABASE_URL`을 요구하는 테스트
- unit test에서 `prisma migrate`를 실행하는 테스트
- Prisma delegate 호출 세부사항을 handler test에서 검증하는 것

### 7.2 Repository Integration Test

repository는 실제 DB로 검증한다.
Prisma query는 타입이 맞아도 relation, constraint, transaction 동작은 mock으로 충분히 검증되지 않는다.

검증 대상:

- `findUnique`, `include`, `select` 결과 shape
- unique constraint
- relation cascade
- transaction rollback
- migration과 schema 일치 여부

권장 실행 흐름:

```txt
1. 테스트용 PostgreSQL 실행
2. DATABASE_URL을 테스트 DB로 설정
3. prisma migrate deploy 실행
4. spec별 seed 생성
5. 테스트 실행
6. spec별 data cleanup
7. Prisma connection 종료
```

cleanup은 table별 `deleteMany()` 또는 transaction rollback 전략 중 하나로 통일한다.
운영/개발 DB를 테스트 DB로 사용하지 않는다.

Jest는 기본적으로 spec 파일을 병렬 실행할 수 있으므로 DB integration test는 격리 전략을 명확히 정한 뒤 작성한다.

필수 규칙:

- repository integration test는 기본적으로 `npm.cmd test -- --runInBand` 또는 별도 integration script로 직렬 실행한다.
- `DATABASE_URL`은 반드시 테스트 전용 DB를 가리킨다. 개발 DB와 운영 DB는 절대 사용하지 않는다.
- 여러 spec을 병렬로 돌릴 계획이라면 worker별 DB 또는 worker별 PostgreSQL schema를 사용한다.
- worker별 schema를 쓰는 경우 `search_path` 또는 connection URL schema parameter를 테스트 시작 시 명시한다.
- cleanup은 relation 의존 순서를 고려해 자식 테이블부터 삭제한다.
- cleanup 순서를 직접 관리하기 어렵다면 transaction rollback 또는 schema drop/recreate 전략을 사용한다.
- 테스트 데이터는 spec마다 고유한 email/id/token 값을 사용해 unique constraint 충돌을 피한다.
- migration 검증은 suite 시작 전에 1회 실행하고, 각 spec에서 `prisma migrate`를 반복 실행하지 않는다.

권장 cleanup 순서 예시:

```txt
refresh_tokens
social_accounts
user_password_credentials
users
```

통합 테스트가 많아지면 unit test와 분리된 명령을 둔다.

```bash
npm.cmd test -- --runInBand test/integration
```

### 7.3 Prisma Lifecycle

`PrismaService`는 `OnModuleInit`에서 `$connect()`, `OnModuleDestroy`에서 `$disconnect()`를 호출한다.

검증 기준:

- unit test에서는 lifecycle hook을 직접 호출하되 `$connect`, `$disconnect`는 mock한다.
- Nest application integration/e2e test에서는 `afterAll`에서 반드시 `app.close()`를 호출한다.
- `app.close()`가 lifecycle hook을 실행한다는 전제 아래 DB connection 누수를 방지한다.

## 8. gRPC와 Protobuf 테스트 기준

`user-service`는 HTTP API만 제공하는 서비스가 아니다.
`proto/user.proto`를 contract로 사용하는 gRPC microservice다.

검증 대상:

- proto package name: `user`
- service name: `UserService`
- rpc name: `GetUser`, `AuthenticateUser`
- `GetUserRequest` field: `user_id`
- `GetUser` response shape: `UserResponse`
- `UserResponse` field: `id`, `email`, `display_name`, `role`
- `AuthenticateUserRequest` field: `email`, `password`
- `AuthenticateUser` response shape: `AuthenticateUserResponse`
- `AuthenticateUserResponse` wrapper field: `user`
- `AuthenticateUserResponse.user` shape: `UserResponse`
- TypeScript 내부 camelCase와 protobuf snake_case mapping

controller unit test에서는 gRPC network를 띄우지 않고 field mapping만 검증한다.
gRPC e2e test에서는 실제 microservice를 띄워 proto contract와 transport wiring을 최소 happy path로 검증한다.

## 9. ConfigModule과 Env 테스트 기준

`user-service`는 `@nestjs/config`를 전역으로 사용한다.

검증 대상:

- `PORT`가 없으면 HTTP 기본 포트 `3002`
- `GRPC_URL`이 없으면 기본 URL `0.0.0.0:50051`
- `DATABASE_URL`은 Prisma integration/e2e에서 테스트 DB를 가리켜야 함
- 테스트에서 env를 바꾼 경우 spec 종료 후 원복

현재 코드에서는 `PORT`와 `GRPC_URL`의 기본값이 별도 config provider가 아니라 `src/main.ts`의 `bootstrap()` 내부에 직접 정의되어 있다.
따라서 ConfigModule만 import하는 테스트로는 이 기본값 경로를 검증할 수 없다.

현재 구조에서 가능한 검증:

- ConfigModule이 env 값을 읽을 수 있는지 검증
- bootstrap 경로를 테스트할 수 있도록 `NestFactory`와 `ConfigService`를 mock한 `main.ts` bootstrap 테스트 작성
- 또는 기본값을 `getUserServiceConfig()` 같은 helper/provider로 분리한 뒤 해당 helper/provider unit test 작성

권장 리팩터링 방향:

```ts
export type UserServiceConfig = {
  httpPort: number
  grpcUrl: string
}

export function getUserServiceConfig(configService: ConfigService): UserServiceConfig {
  return {
    httpPort: configService.get<number>('PORT', 3002),
    grpcUrl: configService.get<string>('GRPC_URL', '0.0.0.0:50051'),
  }
}
```

위처럼 분리한 뒤에는 helper unit test에서 기본값과 env override를 직접 검증한다.
분리 전에는 문서상 기본값 테스트를 추가하더라도 ConfigModule 테스트가 아니라 bootstrap 테스트로 작성한다.

권장 패턴:

```ts
const originalEnv = process.env

beforeEach(() => {
  process.env = { ...originalEnv }
})

afterAll(() => {
  process.env = originalEnv
})
```

ConfigModule behavior를 검증할 때는 `Test.createTestingModule()`을 사용한다.
순수 handler unit test에서는 ConfigModule을 import하지 않는다.

## 10. E2E 테스트 기준

E2E 테스트는 수를 적게 유지한다.
목적은 전체 wiring과 transport가 살아 있는지 확인하는 것이다.

권장 대상:

- HTTP `GET /health`
- gRPC `GetUser` happy path
- gRPC `AuthenticateUser` happy path
- 인증 실패 대표 케이스 1개

E2E에서 모든 edge case를 검증하지 않는다.
edge case는 command/query handler unit test에서 다룬다.

Nest app을 생성한 e2e test는 항상 `afterAll`에서 `await app.close()`를 호출한다.

## 11. Authentication 테스트 체크리스트

인증 관련 command handler를 수정할 때는 아래 케이스를 우선 확인한다.

- email trim/lowercase
- user not found
- password credential not found
- inactive user status
- temporary lock
- wrong password
- failure count increment
- lock threshold reached
- successful login
- failure state reset
- last login timestamp update
- password hash verification error handling
- generated Argon2 hash is not equal to plain password

## 12. Fixture 작성 규칙

테스트 데이터는 helper function으로 만든다.

```ts
function createUser(overrides: Partial<UserWithPasswordCredential> = {}): UserWithPasswordCredential {
  return {
    id: 'user-id',
    email: 'user@example.com',
    displayName: 'User',
    role: 'USER',
    status: UserStatus.ACTIVE,
    passwordCredential: createCredential(),
    ...overrides,
  }
}
```

규칙:

- fixture는 기본적으로 valid object를 반환한다.
- 테스트별 차이는 `overrides`로 표현한다.
- fixture 안에서 현재 시간을 생성하지 않는다.
- fixture helper는 테스트 파일 가까이에 둔다.
- 여러 spec에서 반복되면 `test/helpers/`로 이동한다.

## 13. Assertion 기준

좋은 assertion:

```ts
expect(usersRepository.findByEmailWithPasswordCredential).toHaveBeenCalledWith('user@example.com')
await expect(handler.execute(command)).rejects.toThrow(UnauthorizedException)
expect(passwordCredentialsRepository.recordFailure).toHaveBeenCalledWith(
  'credential-id',
  5,
  new Date('2026-06-05T00:05:00.000Z'),
)
```

피해야 할 assertion:

```ts
expect(result).toBeTruthy()
expect(mock).toHaveBeenCalled()
```

단순 truthy, 호출 여부만 확인하는 assertion은 실패 원인을 충분히 설명하지 못한다.
가능하면 값, exception type, 호출 인자를 구체적으로 검증한다.

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
- NestJS Lifecycle Events: https://docs.nestjs.com/fundamentals/lifecycle-events
- NestJS CQRS: https://docs.nestjs.com/recipes/cqrs
- Prisma Unit Testing: https://www.prisma.io/docs/orm/prisma-client/testing/unit-testing
- Prisma Integration Testing: https://docs.prisma.io/docs/orm/prisma-client/testing/integration-testing
- jest-mock-extended: https://github.com/marchaos/jest-mock-extended
