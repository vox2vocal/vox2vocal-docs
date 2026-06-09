# Vox2Vocal Frontend Architecture Guide

최종 업데이트: 2026-06-09

이 문서는 Vox2Vocal 프론트엔드의 폴더 구조, 화면 개발 단위, 상태 관리, 플랫폼 분기, 테스트 기준을 정의한다. 목적은 Android, iOS, Web을 한 제품으로 개발하면서도 화면, 기능, 공통 코드의 경계를 흐리지 않는 것이다.

## 1. 아키텍처 결론

Vox2Vocal 프론트엔드는 Expo Router 기반의 universal app으로 가져간다. 즉 Android, iOS, Web은 하나의 navigation 모델과 TypeScript 코드베이스를 공유하고, 플랫폼별 UX 차이가 명확할 때만 `*.ios.tsx`, `*.android.tsx`, `*.native.tsx`, `*.web.tsx` 파일로 분리한다.

핵심 판단:

- 라우트는 얇게 유지한다.
- 화면 구현은 `src/features/<feature>` 안에 둔다.
- 공통 UI, 토큰, 유틸은 `src/shared` 또는 `src/design-system`으로 승격한다.
- API 서버 상태는 TanStack Query, 앱 내부 클라이언트 상태는 Zustand, 컴포넌트 내부 임시 상태는 React local state로 나눈다.
- 플랫폼 분기는 작은 차이는 `Platform.select`, 큰 차이는 platform-specific file로 분리한다.
- 현재 앱은 루트 `app/`을 사용하지만, 라우트가 늘어나는 시점의 목표 구조는 Expo SDK 55+ 기본 방향인 `src/app/`이다.

## 2. 조사 근거

### 2.1 MCP 확인

Context7 MCP로 Expo와 React Native 최신 문서 기준을 확인했다.

확인한 기준:

- Expo Router는 Expo/React Native 프로젝트의 권장 file-based routing 방식이다.
- Expo Router route는 `app` 또는 `src/app` 디렉터리의 파일 구조에서 만들어진다.
- `_layout.tsx`는 기존 `App.tsx` 역할을 대체하며 provider, theme, splash, 초기화 코드를 두는 위치다.
- non-navigation component는 route 디렉터리 밖에 두는 것이 권장된다.
- React Native는 복잡한 플랫폼 차이를 `.ios`, `.android`, `.native`, `.web` 확장자로 분리할 수 있다.

### 2.2 웹 공식 문서 확인

문서화 근거:

- [Expo Router introduction](https://docs.expo.dev/router/introduction/): Expo Router는 Android, iOS, Web을 공유하는 file-based router이며 route별 deep link와 web static rendering을 지원한다.
- [Expo Router core concepts](https://docs.expo.dev/router/basics/core-concepts/): 모든 screen/page는 `src/app` 파일로 정의되고, non-navigation 코드는 route 디렉터리 밖에 둔다.
- [Expo top-level src directory](https://docs.expo.dev/router/reference/src-directory/): SDK 55+ 기본 템플릿은 `src/app`, `src/components`, `src/constants`, `src/hooks` 구조를 포함하며, `src/app`과 루트 `app`이 모두 있으면 `src/app`이 우선한다.
- [Expo platform-specific modules](https://docs.expo.dev/router/advanced/platform-specific-modules/): `src/app` 안에서는 universal route 보장을 위해 non-platform base file이 함께 있어야 하며, route 밖에서는 platform-specific component를 자유롭게 둘 수 있다.
- [React Native platform-specific code](https://reactnative.dev/docs/platform-specific-code.html): 복잡한 플랫폼 차이는 `.ios`, `.android`, `.native` 확장자로 파일을 분리하고 동일 import로 불러온다.
- [Tamagui tokens](https://tamagui.dev/docs/core/tokens)와 [Tamagui theme](https://tamagui.dev/docs/core/theme): tokens는 정적 값, themes는 React tree에서 바뀌는 동적 값으로 구분한다.
- [TanStack Query](https://tanstack.com/query/latest): 서버 상태 fetch, cache, synchronize, update를 담당한다.
- [Zustand slices pattern](https://zustand.docs.pmnd.rs/learn/guides/slices-pattern): 앱 상태가 커지면 store를 slice로 나눠 모듈성을 확보한다.

## 3. 현재 구조와 목표 구조

### 3.1 현재 구조

현재 `vox2vocal-app`은 아래 구조다.

```txt
vox2vocal-app/
├─ app/
│  ├─ _layout.tsx
│  ├─ index.tsx
│  └─ signup.tsx
├─ src/
│  ├─ design-system/
│  ├─ features/
│  ├─ providers/
│  └─ stores/
├─ assets/
├─ __tests__/
└─ package.json
```

현재 구조에서 지켜야 할 규칙:

- `app/`에는 route entry와 `_layout.tsx`만 둔다.
- route file은 feature screen을 import하거나 re-export하는 얇은 파일로 유지한다.
- 화면 UI와 비즈니스 흐름은 `src/features`에 둔다.
- provider는 `src/providers`, 전역 client state는 `src/stores`, 디자인 기준은 `src/design-system`을 기준으로 한다.

### 3.2 목표 구조

라우트가 인증 외 화면으로 늘어나기 전에 `app/`을 `src/app/`으로 옮기는 것을 목표 구조로 둔다. 이 구조가 Expo 최신 기본 템플릿과 더 잘 맞고, route와 source code를 한 `src` boundary 안에서 관리할 수 있다.

```txt
vox2vocal-app/
├─ src/
│  ├─ app/
│  │  ├─ _layout.tsx
│  │  ├─ (auth)/
│  │  │  ├─ login.tsx
│  │  │  └─ signup.tsx
│  │  ├─ (main)/
│  │  │  ├─ _layout.tsx
│  │  │  ├─ index.tsx
│  │  │  ├─ upload.tsx
│  │  │  └─ projects/
│  │  │     ├─ index.tsx
│  │  │     └─ [projectId].tsx
│  │  └─ +not-found.tsx
│  ├─ features/
│  │  ├─ auth/
│  │  ├─ upload/
│  │  ├─ voice-analysis/
│  │  ├─ conversion/
│  │  └─ projects/
│  ├─ entities/
│  │  ├─ user/
│  │  ├─ project/
│  │  ├─ audio/
│  │  └─ voice/
│  ├─ shared/
│  │  ├─ ui/
│  │  ├─ api/
│  │  ├─ lib/
│  │  ├─ config/
│  │  └─ testing/
│  ├─ design-system/
│  ├─ providers/
│  └─ stores/
├─ assets/
├─ __tests__/
├─ app.json
├─ metro.config.js
├─ tamagui.config.ts
├─ tsconfig.json
└─ package.json
```

목표 구조 전환 시 함께 처리할 것:

- `app/*`를 `src/app/*`로 이동한다.
- `tsconfig.json`의 `@/*` alias를 `./src/*`로 수렴한다.
- root config 파일은 그대로 프로젝트 루트에 둔다.
- `assets/`, `public/`, `app.json`, `metro.config.js`, `tamagui.config.ts`는 루트에 유지한다.
- route import는 `@/features/auth/login-screen`처럼 `src` 기준 alias로 정리한다.

## 4. 폴더별 책임

| 경로                     | 책임                                                     | 금지                                               |
| ------------------------ | -------------------------------------------------------- | -------------------------------------------------- |
| `app/` 또는 `src/app/`   | Expo Router route, layout, route group, modal route      | 공통 컴포넌트, API client, store, 복잡한 화면 구현 |
| `src/features/<feature>` | 사용자 플로우 중심 화면과 기능 구현                      | 다른 feature 내부 파일 직접 import                 |
| `src/entities/<entity>`  | 여러 feature가 공유하는 도메인 모델, 표시 컴포넌트, 타입 | 특정 화면 전용 UX                                  |
| `src/shared/ui`          | 버튼, 입력창, sheet, modal, list row 등 재사용 UI        | 특정 도메인 문구나 API 호출                        |
| `src/shared/api`         | BFF transport, GraphQL/fetch client, error normalization | feature별 query key와 화면 상태                    |
| `src/shared/lib`         | 날짜, 숫자, 파일 크기, platform helper 등 순수 유틸      | React state나 UI side effect                       |
| `src/shared/config`      | env, endpoint, feature flag, runtime config              | secret hardcoding                                  |
| `src/design-system`      | tokens, theme, typography, responsive helper             | feature 전용 style                                 |
| `src/providers`          | app-level provider composition                           | 개별 화면 상태                                     |
| `src/stores`             | 전역 client state, persisted state                       | API response cache                                 |
| `assets/`                | 앱 아이콘, splash, 브랜드 이미지, 정적 이미지            | 화면별 임시 이미지 난립                            |
| `__tests__/`             | cross-cutting test, integration-like unit test           | feature 내부 세부 테스트의 과도한 집중             |

## 5. Feature 구조

새 기능은 기본적으로 vertical slice로 만든다.

```txt
src/features/auth/
├─ screens/
│  ├─ login-screen.tsx
│  └─ signup-screen.tsx
├─ components/
│  ├─ auth-scaffold.tsx
│  ├─ auth-text-field.tsx
│  └─ social-login-button.tsx
├─ hooks/
│  └─ use-login-form.ts
├─ api/
│  ├─ auth-queries.ts
│  └─ auth-mutations.ts
├─ schemas/
│  └─ auth.schema.ts
├─ types.ts
├─ constants.ts
└─ index.ts
```

현재 인증 기능은 작은 규모라 `src/features/auth/auth-components.tsx`, `login-screen.tsx`, `signup-screen.tsx`로 시작했다. 파일이 다음 기준을 넘으면 위 구조로 쪼갠다.

- 한 파일이 250-300줄을 넘는다.
- 한 컴포넌트가 둘 이상의 화면에서 재사용된다.
- form hook, schema, API mutation이 함께 생긴다.
- web/native 분기 파일이 필요해진다.

## 6. Import 의존성 규칙

의존성 방향은 아래로만 흐른다.

```txt
src/app
  -> src/providers
  -> src/features
  -> src/entities
  -> src/shared
  -> src/design-system
```

규칙:

- `src/app`은 route와 layout만 알고 feature 내부 구현을 얇게 연결한다.
- `features/auth`가 `features/projects` 내부 파일을 직접 import하지 않는다.
- 여러 feature가 함께 쓰는 domain model은 `entities`로 올린다.
- 여러 feature가 함께 쓰는 UI primitive는 `shared/ui`로 올린다.
- `shared`는 `features`를 import하지 않는다.
- `design-system`은 어떤 feature도 import하지 않는다.

예시:

```tsx
// route는 얇게 둔다.
export { LoginScreen as default } from "@/features/auth";
```

```tsx
// feature screen은 feature 내부 hook과 shared UI를 조립한다.
import { Button } from "@/shared/ui/button";
import { useLoginForm } from "../hooks/use-login-form";
```

## 7. Route 설계

### 7.1 Route group

권장 route group:

| Group          | 목적                              | 예시                        |
| -------------- | --------------------------------- | --------------------------- |
| `(auth)`       | 로그인, 회원가입, 비밀번호 재설정 | `/login`, `/signup`         |
| `(onboarding)` | 최초 설정, 권한 안내              | `/onboarding/profile`       |
| `(main)`       | 로그인 이후 주요 앱               | `/`, `/upload`, `/projects` |
| `(modal)`      | 전역 modal route                  | `/project-settings`         |

URL에 드러나지 않아야 하는 묶음은 괄호 route group을 사용한다. 반대로 공유 가능한 화면은 URL 의미가 명확해야 한다.

### 7.2 Layout

`_layout.tsx` 기준:

- root layout: TamaguiProvider, QueryProvider, ThemeProvider, Sentry, StatusBar
- auth layout: 인증 전용 배경, safe area, keyboard avoidance
- main layout: tab, drawer, sidebar, navigation rail
- web layout: header/sidebar 등 web 전용 구조가 필요할 때 `.web.tsx`로 분기

## 8. Platform 분기 기준

작은 차이는 `Platform.select`로 처리한다.

적합한 경우:

- padding, shadow, hitSlop 같은 작은 스타일 차이
- keyboard behavior
- OS별 status bar 색상
- 한두 개 prop 차이

큰 차이는 파일을 분리한다.

적합한 경우:

- web은 sidebar, native는 bottom tabs처럼 navigation 구조가 다르다.
- Android와 iOS에서 네이티브 권한/파일 picker UX가 다르다.
- web은 drag and drop upload, native는 media picker를 쓴다.
- 동일 컴포넌트 안에서 조건문이 3개 이상 늘어난다.

권장 파일명:

```txt
audio-picker.native.tsx
audio-picker.web.tsx
share-sheet.ios.tsx
share-sheet.android.tsx
project-layout.tsx
project-layout.web.tsx
```

`src/app` route 안에서 platform-specific file을 만들 때는 universal route 보장을 위해 non-platform base file도 함께 둔다. 복잡한 platform-specific 구현은 route 밖 `src/features` 또는 `src/shared`에 두고, route는 re-export만 한다.

## 9. 상태 관리

### 9.1 서버 상태

TanStack Query를 사용한다.

사용 대상:

- 사용자 정보 조회
- 프로젝트 목록/상세
- 업로드 상태
- 분석 결과
- 변환 작업 상태
- polling, refetch, optimistic update

규칙:

- query key는 feature 또는 entity 기준으로 상수화한다.
- API response를 Zustand에 복사하지 않는다.
- mutation 성공 후 필요한 query만 invalidate한다.
- 사용자 로그아웃 시 auth 관련 query cache를 정리한다.
- React Query provider는 root layout에서 한 번만 생성한다.

권장 위치:

```txt
src/shared/api/bff-client.ts
src/features/projects/api/project-queries.ts
src/features/projects/api/project-mutations.ts
```

### 9.2 클라이언트 상태

Zustand를 사용한다.

사용 대상:

- session bootstrap state
- selected workspace/project id
- local draft
- modal stack
- player UI state
- persisted preference

규칙:

- 서버에서 다시 가져올 수 있는 데이터는 Zustand에 넣지 않는다.
- store가 커지면 slice pattern으로 나눈다.
- persist 대상은 최소화하고 schema/version을 둔다.
- web은 `localStorage`, native는 MMKV 계층을 사용한다.

권장 위치:

```txt
src/stores/session-store.ts
src/stores/player-store.ts
src/stores/slices/session-slice.ts
```

### 9.3 화면 내부 상태

React `useState`, `useReducer`, React Hook Form을 사용한다.

사용 대상:

- 입력 중인 form 값
- password show/hide
- hover/focus visual state
- 특정 component의 expanded/collapsed state

## 10. API 계층

Vox2Vocal 외부 프론트엔드는 BFF 서버만 호출한다.

```txt
RN App / RN Web
  -> GraphQL
vox2vocal-bff-server
  -> gRPC
api-gateway
  -> services
```

프론트 API 구조:

```txt
src/shared/api/
├─ bff-client.ts
├─ graphql-error.ts
├─ auth-token.ts
└─ query-key.ts

src/features/auth/api/
├─ auth-mutations.ts
└─ auth-queries.ts
```

규칙:

- 화면 컴포넌트에서 `fetch`를 직접 호출하지 않는다.
- transport error, GraphQL error, domain error를 구분한다.
- API 타입은 가능하면 generated type으로 관리한다.
- token 저장/주입은 shared API client 또는 auth provider 계층에서 처리한다.
- feature hook은 UI가 이해할 수 있는 loading/error/empty shape로 반환한다.

## 11. 디자인 시스템 계층

현재 source of truth:

```txt
src/design-system/tokens.ts
tamagui.config.ts
```

방향:

- primitive token은 `src/design-system/tokens.ts`에 먼저 정의한다.
- Tamagui token/theme로 옮길 수 있는 형태를 유지한다.
- 화면에서 hex color를 직접 쓰지 않는다.
- feature 전용 컴포넌트가 반복되면 component token으로 승격한다.

공통 UI 승격 기준:

- 두 개 이상의 feature에서 사용한다.
- 접근성/상태/플랫폼 분기를 매번 구현하면 위험하다.
- 디자인 토큰과 연결되어야 한다.

권장 공통 UI:

```txt
src/shared/ui/button/
src/shared/ui/text-field/
src/shared/ui/checkbox/
src/shared/ui/sheet/
src/shared/ui/modal/
src/shared/ui/toast/
src/shared/ui/empty-state/
```

인증 화면처럼 브랜드 표현이 강한 UI는 바로 shared로 올리지 않는다. 먼저 `features/auth/components`에 두고, 제품 전반에서 같은 패턴이 반복될 때 shared로 승격한다.

## 12. Assets

기준:

- 원본 브랜드 이미지는 `assets/logo.png`로 보존한다.
- 플랫폼 아이콘, splash, favicon은 목적별 파일을 사용한다.
- UI 상단 브랜드 마크는 투명 배경 `assets/brand-mark.png`를 사용한다.
- 특정 feature에서만 쓰는 이미지가 늘면 `assets/features/<feature>` 또는 `src/features/<feature>/assets` 중 하나로 정한다. Metro/Expo asset resolution과 배포 편의성을 고려해 기본은 root `assets`를 유지한다.

## 13. 테스트 구조

테스트는 위험도와 재사용 범위에 맞춰 둔다.

권장 위치:

```txt
__tests__/
├─ viewport-class.test.ts
└─ session-store.test.ts

src/features/auth/__tests__/
├─ login-screen.test.tsx
└─ signup-screen.test.tsx

src/shared/ui/button/__tests__/
└─ button.test.tsx
```

기준:

- 순수 함수와 토큰 helper는 unit test를 둔다.
- store는 persistence mock을 포함한 unit test를 둔다.
- feature screen은 주요 상태, 접근성 label, disabled/loading 상태를 검증한다.
- 반응형 QA는 최소 `360x640`, `375x812`, `768x1024`, `1440x900`을 확인한다.
- 시각 회귀가 중요해지는 시점에는 Playwright screenshot 또는 Expo preview 기반 캡처를 추가한다.

## 14. 새 화면 개발 절차

1. route가 필요한지 확인한다.
2. `src/app` 목표 구조 기준으로 URL과 route group을 먼저 정한다.
3. 실제 화면 구현은 `src/features/<feature>/screens`에 만든다.
4. feature 내부에서만 쓰는 컴포넌트는 `src/features/<feature>/components`에 둔다.
5. 두 feature 이상에서 쓰이면 `src/shared/ui`로 승격한다.
6. 서버 데이터는 `src/features/<feature>/api`의 query/mutation hook으로 감싼다.
7. 화면 내부 form은 React Hook Form + Zod schema를 사용한다.
8. 플랫폼 차이가 커지면 platform-specific component로 분리한다.
9. 최소 unit/component test를 추가한다.
10. `npm run typecheck`, `npm run lint`, `npm run test`를 통과시킨다.

## 15. 파일 위치 판단표

| 만들 코드                | 위치                                                     |
| ------------------------ | -------------------------------------------------------- |
| 새 URL 화면              | `src/app/<route>.tsx` 또는 현재 단계의 `app/<route>.tsx` |
| 화면 본문 구현           | `src/features/<feature>/screens`                         |
| feature 전용 UI          | `src/features/<feature>/components`                      |
| feature 전용 hook        | `src/features/<feature>/hooks`                           |
| feature API hook         | `src/features/<feature>/api`                             |
| 여러 feature 공유 도메인 | `src/entities/<entity>`                                  |
| 여러 feature 공유 UI     | `src/shared/ui`                                          |
| API transport            | `src/shared/api`                                         |
| 순수 유틸                | `src/shared/lib`                                         |
| env/config               | `src/shared/config`                                      |
| 토큰/테마                | `src/design-system`                                      |
| app-level provider       | `src/providers`                                          |
| persisted client state   | `src/stores`                                             |
| 이미지/폰트              | `assets`                                                 |

## 16. 마이그레이션 로드맵

### Phase 1: 현재 구조 안정화

- `app/`는 route-only로 유지한다.
- 인증 화면은 `src/features/auth` 기준으로 유지한다.
- 디자인 토큰은 `src/design-system/tokens.ts`를 source of truth로 둔다.
- 문서와 README에서 현재 구조와 목표 구조를 함께 명시한다.

### Phase 2: `src/app` 전환

- `app/_layout.tsx`, `app/index.tsx`, `app/signup.tsx`를 `src/app`으로 이동한다.
- auth route group을 `(auth)`로 정리한다.
- `@/*` alias를 `./src/*`로 수렴한다.
- Jest module mapper도 같은 alias 기준으로 변경한다.
- route별 smoke test를 확인한다.

### Phase 3: feature scale-out

- 업로드, 프로젝트, 분석 결과 feature를 vertical slice로 추가한다.
- shared API client와 query key convention을 도입한다.
- shared UI primitive를 디자인 시스템 기준으로 만든다.
- player, upload draft 등 필요한 client state store를 slice로 나눈다.

### Phase 4: production QA

- route typed link 점검
- web static export 점검
- Android/iOS safe area와 keyboard QA
- Playwright/Expo screenshot regression
- Sentry release/environment 연결

## 17. Architecture Decision Record

| 결정                                      | 상태      | 이유                                                                                          |
| ----------------------------------------- | --------- | --------------------------------------------------------------------------------------------- |
| Expo Router 사용                          | 채택      | Android, iOS, Web route를 하나의 파일 기반 구조로 관리하고 deep link/web URL을 함께 얻기 위함 |
| `src/app` 목표 구조                       | 채택 예정 | Expo SDK 55+ 기본 구조와 문서 기준에 맞고, route와 source boundary가 명확함                   |
| feature-first 구조                        | 채택      | 화면/비즈니스 흐름 중심으로 개발 단위를 분리해 확장과 리뷰가 쉬움                             |
| full FSD 대신 light layered feature 구조  | 채택      | 초기 제품에는 과한 계층을 줄이고, entities/shared는 재사용이 확인될 때 승격                   |
| TanStack Query/Zustand 분리               | 채택      | 서버 상태와 클라이언트 상태의 생명주기와 캐시 정책이 다름                                     |
| Tamagui token/theme 중심                  | 채택      | native/web 공통 토큰, theme, responsive style을 한 계층으로 수렴 가능                         |
| platform-specific file은 필요할 때만 사용 | 채택      | 코드 공유율을 유지하면서 복잡한 플랫폼 분기를 깨끗하게 분리                                   |
