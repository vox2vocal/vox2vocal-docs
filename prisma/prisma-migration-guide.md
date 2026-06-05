# Prisma Migrate 적용 가이드

이 문서는 `user-service`에서 기존 PostgreSQL 데이터베이스를 Prisma Migrate 관리 대상으로 편입하고, 이후 마이그레이션을 안전하게 적용하는 절차를 설명한다.

Prisma Migrate를 처음 도입할 때 데이터베이스가 이미 비어 있지 않으면, Prisma는 기존 객체를 임의로 덮어쓰지 않고 `P3005` 에러를 발생시킨다. 이때는 기존 DB 상태를 baseline migration으로 기록한 뒤, 실제 신규 migration을 별도로 `deploy`해야 한다.

## 1. 개요

작업 기준 경로는 다음과 같다.

```txt
user-service/
  prisma/
    schema.prisma
    migrations/
      0_init/
        migration.sql
      <timestamp>_<migration_name>/
        migration.sql
```

현재 Prisma schema 경로는 `user-service/prisma/schema.prisma`이다.

기존 DB에는 이미 `users` 테이블 등 객체가 존재하지만, Prisma Migrate의 적용 이력을 저장하는 `_prisma_migrations` 기록이 없을 수 있다. 이 상태에서 바로 `npm run prisma:deploy`를 실행하면 Prisma는 DB가 비어 있지 않다고 판단하고 `P3005`를 반환한다.

이 상황은 장애라기보다 정상적인 보호 동작이다. 기존 DB를 Prisma Migrate에 편입하려면 최초 1회 baseline 처리가 필요하다.

## 2. 마이그레이션 파일 구조

Prisma Migrate는 `prisma/migrations` 하위의 디렉터리를 시간순 migration 단위로 관리한다.

```txt
user-service/prisma/migrations/
  0_init/
    migration.sql
  <timestamp>_<migration_name>/
    migration.sql
```

각 디렉터리의 의미는 다음과 같다.

| 디렉터리 | 용도 |
| --- | --- |
| `0_init` | 기존 DB에 이미 존재하는 스키마를 Prisma Migrate에 baseline으로 등록하기 위한 migration |
| `<timestamp>_<migration_name>` | baseline 이후 실제로 DB에 적용해야 하는 신규 migration |

중요한 점은 `0_init`과 신규 migration의 역할이 다르다는 것이다.

`0_init`은 이미 DB에 존재하는 상태를 Prisma에게 "적용 완료"로 알려주는 용도다. 따라서 `0_init`의 SQL을 실제 DB에 다시 실행하는 것이 아니라, `migrate resolve --applied 0_init`으로 적용 이력만 기록한다.

반대로 신규 migration은 실제로 DB에 반영되어야 하는 변경이다. 신규 migration을 baseline으로 resolve하면 안 된다. 그렇게 하면 Prisma는 적용되었다고 기록하지만, 실제 컬럼과 테이블은 생성되지 않는다.

## 3. P1001과 P3005 에러 의미

### P1001

`P1001`은 Prisma가 DB 서버에 연결하지 못했을 때 발생한다.

예시 상황:

- `DATABASE_URL`의 host 또는 port가 잘못됨
- 로컬 DB가 실행 중이지 않음
- Kubernetes port-forward가 연결되어 있지 않음
- 네트워크 또는 방화벽 문제로 DB에 접근할 수 없음

예시 에러 상황:

```txt
Can't reach database server at localhost:15432
```

이 경우 migration 문제가 아니라 DB 연결 문제를 먼저 해결해야 한다.

Kubernetes의 PostgreSQL을 로컬로 연결해야 한다면, 별도 터미널에서 port-forward를 실행한다.

```bash
kubectl port-forward svc/postgres 15432:5432
```

### P3005

`P3005`는 DB schema가 비어 있지 않은데 Prisma Migrate 이력이 없을 때 발생한다.

예시 메시지:

```txt
Error: P3005
The database schema is not empty.
```

이 에러는 기존 DB를 Prisma Migrate에 편입하기 위한 baseline이 필요하다는 의미다.

즉, 다음 상황이면 `P3005`가 정상적으로 발생할 수 있다.

- DB에는 이미 테이블, enum, 인덱스 등 객체가 있음
- 하지만 `_prisma_migrations` 테이블에 적용 이력이 없음
- `npm run prisma:deploy` 또는 `npx prisma migrate deploy`를 실행함

## 4. 기존 DB baseline 절차

baseline은 같은 DB에서 최초 1회만 수행한다. 이미 `_prisma_migrations`에 `0_init`이 적용된 것으로 기록된 DB에서는 반복 실행하지 않는다.

아래 명령은 `user-service` 디렉터리에서 실행한다.

```bash
cd user-service
```

### 4.1 DB 연결 확인

먼저 DB에 연결할 수 있어야 한다.

```bash
export DATABASE_URL='postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users'
npx prisma validate
```

Windows PowerShell에서는 다음처럼 설정한다.

```powershell
$env:DATABASE_URL = 'postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users'
npx prisma validate
```

`P1001`이 발생하면 port-forward 또는 DB 접속 정보를 먼저 확인한다.

### 4.2 0_init 디렉터리 생성

```bash
mkdir -p prisma/migrations/0_init
```

Windows PowerShell에서는 다음 명령을 사용할 수 있다.

```powershell
New-Item -ItemType Directory -Force prisma/migrations/0_init
```

### 4.3 현재 DB 상태를 baseline SQL로 저장

기존 DB의 현재 schema 상태를 빈 DB 기준 diff로 생성해 `0_init/migration.sql`에 저장한다.

```bash
npx prisma migrate diff \
  --from-empty \
  --to-url 'postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users' \
  --script > prisma/migrations/0_init/migration.sql
```

Windows PowerShell에서는 줄바꿈 문자가 다르므로 다음처럼 실행한다.

```powershell
npx prisma migrate diff `
  --from-empty `
  --to-url 'postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users' `
  --script > prisma/migrations/0_init/migration.sql
```

생성된 `0_init/migration.sql`은 "현재 DB에 이미 존재하는 schema"를 표현한다. 이 파일은 Prisma Migrate가 기존 상태를 이해하도록 보관하는 baseline이다.

### 4.4 baseline migration을 적용 완료로 기록

`0_init`은 실제 DB에 다시 적용하지 않는다. 이미 존재하는 상태를 Prisma Migrate 이력에만 등록한다.

```bash
DATABASE_URL='postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users' \
npx prisma migrate resolve --applied 0_init
```

Windows PowerShell에서는 다음처럼 실행한다.

```powershell
$env:DATABASE_URL = 'postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users'
npx prisma migrate resolve --applied 0_init
```

### 4.5 신규 migration 적용

baseline 등록이 끝난 뒤 실제 신규 migration을 적용한다.

```bash
DATABASE_URL='postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users' \
npm run prisma:deploy
```

Windows PowerShell에서는 다음처럼 실행한다.

```powershell
$env:DATABASE_URL = 'postgresql://vox2vocal:vox2vocal@localhost:15432/vox2vocal?schema=users'
npm run prisma:deploy
```

이 단계에서 baseline 이후 생성된 신규 migration이 실제 DB에 적용되어야 한다.

## 5. 이후 마이그레이션 방법

baseline이 완료된 DB에서는 일반적인 Prisma Migrate 흐름을 사용한다.

개발 환경에서 schema 변경 후 migration을 만들 때:

```bash
cd user-service
npx prisma migrate dev --name <migration_name>
```

운영 또는 배포 환경에서 이미 생성된 migration을 적용할 때:

```bash
cd user-service
npm run prisma:deploy
```

Prisma Client를 다시 생성할 때:

```bash
cd user-service
npm run prisma:generate
```

schema 포맷을 정리할 때:

```bash
cd user-service
npx prisma format
```

schema 유효성을 확인할 때:

```bash
cd user-service
npx prisma validate
```

## 6. 주의사항 및 운영 체크리스트

baseline과 deploy를 구분하기 위해 아래 항목을 확인한다.

- `P1001`은 DB 연결 문제다. migration history 문제가 아니므로 DB host, port, port-forward, credential을 먼저 확인한다.
- `P3005`는 비어 있지 않은 DB에 Prisma Migrate 이력이 없다는 의미다.
- baseline은 같은 DB에서 최초 1회만 수행한다.
- 다른 환경의 DB가 이미 객체를 가지고 있지만 `_prisma_migrations` 기록이 없다면, 그 환경에서도 별도 baseline이 필요하다.
- `0_init`은 기존 DB 상태를 "이미 적용됨"으로 기록하는 용도다.
- `0_init` SQL을 기존 DB에 직접 다시 적용하지 않는다.
- 신규 migration을 `migrate resolve --applied`로 baseline 처리하지 않는다.
- 신규 migration을 resolve로만 처리하면 실제 컬럼, 테이블, enum, 인덱스가 생성되지 않는다.
- baseline 후에는 `npm run prisma:deploy`로 신규 migration이 정상 적용되는지 확인한다.
- 배포 전 `npx prisma validate`, `npx prisma format`, `npm run prisma:generate`, `npm run typecheck`를 실행해 schema와 타입을 검증한다.
- 운영 DB에서 baseline을 수행하기 전에는 반드시 DB 백업 또는 복구 수단을 준비한다.

권장 검증 순서:

```bash
cd user-service
npx prisma format
npx prisma validate
npm run prisma:generate
npm run typecheck
npm run prisma:deploy
```

기존 DB를 Prisma Migrate에 편입하는 핵심 순서는 다음과 같다.

```txt
1. DB 연결 확인
2. 현재 DB 상태로 0_init baseline SQL 생성
3. 0_init을 migrate resolve --applied로 기록
4. 신규 migration을 npm run prisma:deploy로 실제 적용
5. 이후부터는 일반 migration 흐름 사용
```
