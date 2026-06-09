# Vox2Vocal Frontend Design System Guide

최종 업데이트: 2026-06-09

이 문서는 Vox2Vocal의 모바일 앱(Android, iOS), 태블릿, 웹 화면을 개발할 때 사용하는 공통 디자인 시스템 기준서다.

목표는 화면마다 감으로 퍼블리싱하지 않고, 제품 원칙, 반응형 레이아웃, 디자인 토큰, 컴포넌트 상태, 접근성, QA 기준을 같은 언어로 판단하게 하는 것이다.

현재 구현 기준:

- 앱 경로: `vox2vocal-app`
- 프론트엔드 아키텍처 기준: `vox2vocal-docs/frontend/architecture.md`
- 토큰 source of truth: `vox2vocal-app/src/design-system/tokens.ts`
- 인증 UI source of truth: `vox2vocal-app/src/features/auth/`
- 브랜드 마크 자산: `vox2vocal-app/assets/brand-mark.png` (`assets/logo.png`에서 투명 배경으로 파생)
- 현재 인증 라우트: `/` 로그인, `/signup` 회원가입

## 1. 문서 목적

### 1.1 이 문서가 해결하는 문제

- 모바일, 태블릿, 웹 화면을 각각 따로 해석하면서 생기는 UI 불일치를 줄인다.
- `px`, `dp`, `pt`, CSS viewport 차이 때문에 생기는 화면 깨짐을 줄인다.
- 버튼, 입력창, 카드, 모달, 리스트 같은 공통 요소를 매번 새로 만들지 않게 한다.
- 디자이너, FE 개발자, 백엔드/엔진 담당자가 화면 상태와 데이터 상태를 같은 기준으로 이해하게 한다.
- Expo React Native App/Web과 Tamagui 기반 코드에서 사용할 수 있는 토큰 구조를 미리 정한다.

### 1.2 적용 범위

| 대상            | 적용 여부 | 설명                                                             |
| --------------- | --------- | ---------------------------------------------------------------- |
| Android app     | 적용      | React Native 네이티브 화면 기준                                  |
| iOS app         | 적용      | React Native 네이티브 화면 기준, safe area와 iOS 관성 반영       |
| Tablet          | 적용      | iPad, Android tablet, foldable expanded viewport 기준            |
| Web mobile      | 적용      | React Native Web 기반 모바일 브라우저                            |
| Web desktop     | 적용      | 데스크톱 브라우저, 넓은 viewport, 키보드/마우스 상호작용         |
| Admin/dashboard | 적용      | 향후 운영/관리 화면이 생길 경우 같은 토큰과 레이아웃 클래스 사용 |

폴더 구조, route/feature/shared 계층, API/state 관리, 플랫폼 분기 기준은 `architecture.md`를 따른다. 이 문서는 시각 언어, 반응형 레이아웃, 토큰, 컴포넌트 상태, 접근성, QA 기준을 담당한다.

### 1.3 현재 앱 기술 기준

현재 `vox2vocal-app`은 다음 기술을 전제로 한다.

| 영역          | 기술                        | 디자인 시스템에 주는 영향                                    |
| ------------- | --------------------------- | ------------------------------------------------------------ |
| App framework | Expo SDK 56                 | Android, iOS, Web을 하나의 앱 모델로 개발                    |
| UI runtime    | React Native 0.85, React 19 | `View`, `Text`, `Pressable`, RN style 기반 UI                |
| Web           | React Native Web            | RN 컴포넌트를 Web에서도 렌더링                               |
| Routing       | Expo Router                 | 파일 기반 라우팅, native/web layout 분기 가능                |
| Styling       | Tamagui v2                  | tokens, themes, variants, responsive media style의 기준      |
| State         | TanStack Query, Zustand     | 서버 상태/클라이언트 상태에 따른 loading/error/empty UI 필요 |
| Forms         | React Hook Form, Zod        | 입력 검증과 에러 메시지 패턴 필요                            |

Expo Router는 Android, iOS, Web에서 같은 navigation 구조를 공유하면서도 web/native layout 분기를 허용한다. Tamagui는 tokens, themes, pseudo state, media query style을 지원하므로, 디자인 시스템 값은 Tamagui config로 옮길 수 있는 형태를 우선한다.

## 2. 디자인 의사결정 원칙

### 2.1 판단 우선순위

화면 개발 중 기준이 충돌하면 아래 순서로 판단한다.

1. 사용자가 핵심 작업을 완료할 수 있는가
2. 접근성 기준을 만족하는가
3. 모바일 compact 화면에서 깨지지 않는가
4. 태블릿과 웹에서 공간을 낭비하지 않는가
5. 기존 토큰과 컴포넌트를 재사용하는가
6. 플랫폼 관습과 맞는가
7. 시각적으로 브랜드 느낌을 강화하는가

### 2.2 제품 성격

Vox2Vocal은 보이스 입력, 분석, 변환, 결과 확인이 중심인 창작/오디오 작업 도구다. 화면은 화려한 랜딩 페이지보다 사용자가 파일, 진행 상태, 결과, 설정을 빠르게 이해하는 쪽이 중요하다.

제품 화면의 기본 성격:

- 어두운 작업 배경을 기본값으로 둔다.
- 브랜드 무드는 블랙 앤 레드, 네온 글로우, 오디오 파동, 현대적인 보컬/음성 이미지를 기준으로 한다.
- 오디오/보컬 작업 상태를 명확히 보여준다.
- 진행률, 시간, 파일 상태, 오류를 숨기지 않는다.
- 모바일에서는 한 번에 하나의 주요 작업에 집중시킨다.
- 태블릿과 웹에서는 목록과 상세, 입력과 미리보기를 병렬로 배치한다.
- 인증, 온보딩, 결과 공유처럼 브랜드 첫인상이 중요한 화면은 붉은 ambient glow를 적극 사용한다.
- 작업 UI는 브랜드 톤을 유지하되, 정보 판독성과 반복 사용성을 우선한다.

### 2.3 플랫폼별 원칙

| 플랫폼  | 기본 원칙                                                                                              | 피해야 할 것                           |
| ------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| Android | Material adaptive layout에 맞춰 compact/medium/expanded를 구분한다. Back 동작과 system bar를 고려한다. | iOS 전용 제스처를 핵심 동작으로 강제   |
| iOS     | safe area, Dynamic Island, home indicator, keyboard avoidance를 우선한다.                              | safe area를 무시한 CTA, 작은 터치 영역 |
| Tablet  | 한 화면에 더 많은 정보를 보여주되 복잡도를 제어한다.                                                   | 모바일 단일 열을 무작정 늘리기         |
| Web     | 키보드, 마우스, hover, focus, URL navigation을 지원한다.                                               | 모바일 UI를 가운데 크게 확대만 하기    |

## 3. 반응형 레이아웃 기준

### 3.1 화면 비율보다 viewport class를 우선한다

특정 기기 비율 하나를 기준으로 잡지 않는다. Android, iOS, 태블릿, 브라우저 창은 모두 가변적이므로 width 기반 class를 우선하고, height와 orientation은 보조 기준으로 사용한다.

| Class       |    Width 기준 | 대표 환경                                  | 기본 레이아웃                               |
| ----------- | ------------: | ------------------------------------------ | ------------------------------------------- |
| Compact     |       `< 600` | 모바일 세로, 작은 foldable pane            | 단일 열, bottom navigation, full-width form |
| Medium      |   `600 - 839` | 큰 모바일 가로, 작은 태블릿 세로, foldable | 단일 열 또는 보조 패널 제한적 사용          |
| Expanded    |  `840 - 1199` | 태블릿 가로, 작은 웹, 큰 foldable          | 2-pane, navigation rail/sidebar 가능        |
| Large       | `1200 - 1599` | 노트북, 데스크톱 웹                        | 12-column grid, sidebar + content           |
| Extra Large |       `1600+` | 큰 모니터, wide desktop                    | 최대 너비 제한, 다중 패널 선택적 사용       |

참고: Android window size class는 width를 compact, medium, expanded, large, extra-large로 나누며, 대부분의 앱은 width class를 중심으로 adaptive UI를 설계한다.

### 3.2 Height 기준

height는 세로 스크롤이 가능하기 때문에 width보다 우선순위가 낮다. 단, 키보드, 모바일 landscape, 짧은 노트북 viewport에서는 중요하다.

| Height class    | Height 기준 | 처리 기준                                          |
| --------------- | ----------: | -------------------------------------------------- |
| Compact height  |     `< 480` | 상단 설명 축소, sticky CTA 지양, 스크롤 우선       |
| Medium height   | `480 - 899` | 일반 모바일/웹 화면                                |
| Expanded height |      `900+` | 태블릿 세로, 넓은 웹, 미리보기/보조 정보 확장 가능 |

### 3.3 대표 QA viewport

개발과 디자인 QA에서 최소 아래 크기를 확인한다.

| 구분             |      Viewport | 목적                                  |
| ---------------- | ------------: | ------------------------------------- |
| Mobile minimum   |   `360 x 640` | 작은 Android 최소 대응                |
| iPhone standard  |   `390 x 844` | 기본 iPhone급 세로 화면               |
| Mobile large     |   `412 x 915` | 큰 Android/iPhone급 화면              |
| Mobile landscape |   `844 x 390` | 짧은 height, 가로 회전 또는 웹 모바일 |
| Tablet portrait  |  `768 x 1024` | iPad/Android tablet 세로              |
| Tablet landscape |  `1024 x 768` | 태블릿 가로, 2-pane 검증              |
| Laptop           |  `1366 x 768` | 낮은 height의 일반 노트북             |
| Desktop          |  `1440 x 900` | 표준 데스크톱 웹                      |
| Wide desktop     | `1920 x 1080` | 넓은 웹, max width 검증               |

현재 `app.json`은 native orientation을 `portrait`로 설정한다. 따라서 Android/iOS native 앱은 세로 화면을 우선 QA한다. 태블릿/웹의 landscape 대응이 제품 요구사항이 되면 `app.json` orientation 정책과 화면 설계를 함께 재검토한다.

## 4. 레이아웃 시스템

### 4.1 Grid

Vox2Vocal은 모바일 앱과 웹을 모두 지원하므로, 화면 전체는 class별 grid로 설계하되 컴포넌트 내부는 flex/gap을 우선 사용한다.

| Class       | Columns | Margin | Gutter | 주요 사용                    |
| ----------- | ------: | -----: | -----: | ---------------------------- |
| Compact     |       4 |     16 |     12 | 모바일 단일 열               |
| Medium      |       8 |     24 |     16 | 태블릿 세로, 큰 모바일       |
| Expanded    |      12 |     32 |     20 | 태블릿 가로, 작은 웹         |
| Large       |      12 |     40 |     24 | 데스크톱                     |
| Extra Large |      12 |     48 |     24 | wide desktop, max width 사용 |

개발 규칙:

- 모바일에서는 하나의 핵심 작업을 한 열에 둔다.
- 태블릿부터 목록/상세, 입력/미리보기, 설정/내용 같은 2-pane 구성을 허용한다.
- 웹에서는 전체 콘텐츠가 무한히 넓어지지 않도록 container max width를 둔다.
- 고정 px width는 카드나 폼의 최대 너비에만 사용한다.
- 페이지 섹션을 카드처럼 띄우지 않는다. 카드 안에 카드를 중첩하지 않는다.

### 4.2 Container width

| 용도              |    Compact |    Medium |   Expanded | Large 이상 |
| ----------------- | ---------: | --------: | ---------: | ---------: |
| Page content      |       100% |      100% |       100% | max `1280` |
| Reading content   |       100% | max `680` |  max `720` |  max `760` |
| Auth/form panel   |       100% | max `384` |  max `384` |  max `384` |
| Dashboard content |       100% |      100% | max `1120` | max `1440` |
| Modal             | full/sheet | max `560` |  max `640` |  max `720` |

### 4.3 Safe area

모바일 앱은 safe area를 화면 구조의 일부로 취급한다.

필수 규칙:

- iOS home indicator 위에 CTA를 붙이지 않는다.
- Dynamic Island, notch, status bar와 제목/버튼이 겹치지 않게 한다.
- Android system navigation bar와 bottom CTA가 겹치지 않게 한다.
- keyboard가 올라오면 입력 필드와 제출 버튼이 가려지지 않아야 한다.
- sticky bottom CTA는 safe area padding을 포함한다.
- Web에서는 browser viewport와 scroll container를 구분한다.

인증 화면 특수 규칙:

- 로그인/회원가입은 모바일 퍼스트 고정폭 컨테이너를 사용한다.
- 태블릿/웹에서도 auth form은 화면 중앙 max `384`를 유지한다.
- 기본 좌우 padding은 `24`이며, 모바일 최소 viewport에서도 가로 스크롤이 없어야 한다.
- 회원가입처럼 필드가 많은 화면은 세로 스크롤을 허용하고 하단 여백을 충분히 둔다.

### 4.4 Scroll

모바일 화면은 세로 스크롤을 자연스럽게 허용한다. 가로 스크롤은 데이터 테이블이나 waveform timeline처럼 목적이 명확한 경우에만 허용한다.

규칙:

- 화면 전체가 viewport보다 작을 것이라고 가정하지 않는다.
- form, dashboard, result 화면은 content container에 충분한 bottom padding을 둔다.
- fixed footer가 있는 화면은 마지막 콘텐츠가 footer 뒤에 숨지 않게 한다.
- 모바일 landscape에서는 설명/illustration을 줄이고 입력과 CTA를 우선한다.

## 5. 디자인 토큰 구조

### 5.1 토큰 계층

토큰은 세 단계로 관리한다.

| 계층            | 역할             | 예시                                                       |
| --------------- | ---------------- | ---------------------------------------------------------- |
| Primitive token | 원시 값          | `color.blue.500`, `space.4`, `radius.2`                    |
| Semantic token  | 의미 값          | `color.bg.default`, `color.text.primary`, `space.screen.x` |
| Component token | 컴포넌트 전용 값 | `button.primary.bg`, `input.border.focus`                  |

개발자는 컴포넌트 코드에서 primitive token을 직접 쓰지 않는다. 화면/컴포넌트에서는 semantic 또는 component token을 사용한다.

### 5.2 토큰 네이밍

권장 형식:

```txt
category.role.state
category.scale
component.part.property.state
```

예시:

```txt
color.bg.default
color.text.secondary
color.border.focus
space.4
radius.card
button.primary.bg.pressed
input.label.color.error
```

규칙:

- 이름에는 색상 이름보다 역할을 우선한다.
- `primaryBlue`, `grayCard`처럼 값이 바뀌면 틀리는 이름을 피한다.
- 토큰 이름은 소문자와 점 구분을 사용한다.
- Figma, Tamagui, 코드에서 같은 의미의 이름을 유지한다.

### 5.3 브랜드/컬러 토큰

현재 앱은 `userInterfaceStyle: dark`와 `#050505` splash background를 사용한다. Vox2Vocal의 기본 브랜드 팔레트는 블랙 앤 레드이며, 인증 화면은 붉은 네온 글로우를 주요 시각 효과로 사용한다.

코드 기준은 `vox2vocal-app/src/design-system/tokens.ts`의 `colors`, `brandColors`, `authColors`다.

#### Primitive colors

| Token                   | Value     | 용도                               |
| ----------------------- | --------- | ---------------------------------- |
| `color.black.950`       | `#050505` | 앱 최상위 배경                     |
| `color.black.red.950`   | `#0A0505` | 붉은 ambient glow가 깔린 보조 배경 |
| `color.surface.900`     | `#121212` | 입력창/소셜 버튼 surface           |
| `color.surface.850`     | `#1A1A1A` | hover/pressed surface              |
| `color.surface.red.900` | `#1A1010` | input focus surface                |
| `color.zinc.800`        | `#27272A` | 기본 border                        |
| `color.zinc.700`        | `#3F3F46` | 강조 border                        |
| `color.zinc.500`        | `#71717A` | placeholder/비활성 텍스트          |
| `color.zinc.400`        | `#A1A1AA` | 보조 텍스트                        |
| `color.white`           | `#FFFFFF` | primary text                       |
| `color.red.800`         | `#991B1B` | CTA gradient 시작, pressed         |
| `color.red.600`         | `#DC2626` | primary brand/action               |
| `color.red.500`         | `#EF4444` | hover/focus/glow highlight         |
| `color.red.400`         | `#F87171` | error/danger text                  |
| `color.mint.400`        | `#2EE8B6` | success/accent                     |
| `color.amber.400`       | `#F6BF4F` | warning                            |

#### Semantic colors

| Token                         | Value             | 사용 기준               |
| ----------------------------- | ----------------- | ----------------------- |
| `color.bg.default`            | `color.black.950` | 앱 전체 배경            |
| `color.bg.subtle`             | `#0A0505`         | 붉은 ambient 배경       |
| `color.surface.default`       | `#121212`         | 입력창, 소셜 버튼       |
| `color.surface.raised`        | `#1A1A1A`         | hover/pressed surface   |
| `color.surface.overlay`       | `#1A1010`         | input focus surface     |
| `color.text.primary`          | `#FFFFFF`         | 제목/본문 기본          |
| `color.text.secondary`        | `#A1A1AA`         | 보조 문구               |
| `color.text.muted`            | `#71717A`         | placeholder/비활성 보조 |
| `color.text.inverse`          | `#050505`         | 밝은 버튼 위 텍스트     |
| `color.border.subtle`         | `#27272A`         | 카드/입력 기본 border   |
| `color.border.strong`         | `#3F3F46`         | 강조 border             |
| `color.border.focus`          | `#EF4444`         | focus ring/input focus  |
| `color.action.primary`        | `#DC2626`         | primary CTA             |
| `color.action.primaryHover`   | `#EF4444`         | primary hover/glow      |
| `color.action.primaryPressed` | `#991B1B`         | primary pressed         |
| `color.status.success`        | `color.mint.400`  | 성공                    |
| `color.status.warning`        | `color.amber.400` | 경고                    |
| `color.status.danger`         | `#F87171`         | 오류/삭제               |
| `color.audio.waveform`        | `#EF4444`         | waveform 활성 구간      |
| `color.audio.selection`       | `#DC2626`         | 선택 구간               |

#### Brand effect colors

| Token             | Value                     | 사용 기준                       |
| ----------------- | ------------------------- | ------------------------------- |
| `brand.black`     | `#050505`                 | auth/background base            |
| `brand.redDark`   | `#991B1B`                 | CTA gradient 시작점             |
| `brand.red`       | `#DC2626`                 | CTA 중심 색상, checkbox checked |
| `brand.redBright` | `#EF4444`                 | hover, focus, icon active       |
| `brand.glow`      | `rgba(220, 38, 38, 0.4)`  | primary CTA glow                |
| `brand.glowSoft`  | `rgba(220, 38, 38, 0.15)` | input focus glow                |
| `brand.selection` | `rgba(239, 68, 68, 0.3)`  | web text selection              |

컬러 규칙:

- 텍스트와 배경 조합은 WCAG 2.2 AA 기준을 목표로 한다.
- 일반 본문은 최소 4.5:1 대비를 만족한다.
- 큰 텍스트와 아이콘성 텍스트는 최소 3:1 대비를 만족한다.
- 상태를 색상만으로 전달하지 않는다. 아이콘, 문구, 패턴, 위치를 함께 사용한다.
- primary/action 색상은 한 화면에서 CTA 우선순위를 나타낼 때만 사용한다.
- auth/brand 화면에서는 붉은 glow를 허용하되, surface와 border로 기본 구조를 먼저 만든다.
- dark theme elevation은 그림자보다 surface 단계 차이를 우선한다. 단, CTA/focus/checkbox selected는 brand glow를 사용한다.

### 5.4 Typography tokens

기본 폰트는 OS system font를 사용한다. 한국어/영어 혼합 UI에서 줄높이를 충분히 확보한다.

| Token             | Mobile | Tablet/Web | Line height | Weight | 용도                        |
| ----------------- | -----: | ---------: | ----------: | -----: | --------------------------- |
| `type.display`    |     32 |         40 |         1.2 |    700 | 제품명, hero성 제목         |
| `type.h1`         |     28 |         32 |        1.25 |    700 | 화면 최상위 제목            |
| `type.h2`         |     24 |         28 |        1.28 |    700 | 섹션 제목                   |
| `type.h3`         |     20 |         22 |        1.35 |    650 | 카드/패널 제목              |
| `type.body`       |     16 |         16 |         1.5 |    400 | 기본 본문                   |
| `type.bodyStrong` |     16 |         16 |         1.5 |    600 | 강조 본문                   |
| `type.label`      |     14 |         14 |        1.35 |    600 | 입력 라벨, 필드명           |
| `type.caption`    |     12 |         12 |        1.35 |    400 | 보조 메타                   |
| `type.button`     |     15 |         15 |         1.3 |    700 | 버튼                        |
| `type.mono`       |     13 |         13 |        1.35 |    500 | timecode, file size, job id |

Typography 규칙:

- 모바일 입력 필드는 16 이상으로 둔다. Web mobile에서 작은 input text는 zoom 문제를 만들 수 있다.
- 화면 내부 카드/패널 제목에는 hero-scale type을 쓰지 않는다.
- 숫자, 시간, 진행률은 tabular number 또는 mono 스타일을 사용한다.
- 본문 line length는 너무 길지 않게 한다. 웹 reading 영역은 max width를 둔다.
- 문구는 짧고 직접적으로 쓴다. 기능 설명을 화면 안에 길게 쌓지 않는다.

### 5.5 Spacing tokens

4px 기반 spacing scale을 사용한다.

| Token      | Value | 용도                              |
| ---------- | ----: | --------------------------------- |
| `space.0`  |     0 | reset                             |
| `space.1`  |     2 | hairline gap                      |
| `space.2`  |     4 | icon/text micro gap               |
| `space.3`  |     8 | compact gap                       |
| `space.4`  |    12 | small component gap               |
| `space.5`  |    16 | mobile screen margin, default gap |
| `space.6`  |    20 | section inner gap                 |
| `space.7`  |    24 | tablet margin, card padding       |
| `space.8`  |    32 | section gap                       |
| `space.9`  |    40 | desktop block gap                 |
| `space.10` |    48 | large section gap                 |
| `space.11` |    64 | major layout gap                  |
| `space.12` |    80 | web large section gap             |

Spacing 규칙:

- 레이아웃 간격은 `gap`을 우선 사용한다.
- arbitrary spacing을 만들지 않는다. 필요하면 token을 추가한다.
- 모바일 화면의 좌우 기본 padding은 `16`을 최소값으로 하되, 인증 화면은 `24`를 기본값으로 한다.
- 카드 내부 padding은 compact `16`, tablet/web `20-24`를 기본으로 한다.
- 같은 종류의 컴포넌트 사이 간격은 일정해야 한다.

### 5.6 Radius tokens

| Token                | Value | 용도                                   |
| -------------------- | ----: | -------------------------------------- |
| `radius.none`        |     0 | edge-to-edge                           |
| `radius.xs`          |     4 | badge, small input                     |
| `radius.sm`          |     6 | compact control                        |
| `radius.md`          |     8 | card, input, button 기본               |
| `radius.lg`          |    12 | sheet/modal container                  |
| `radius.authControl` |    16 | 로그인/회원가입 입력창, CTA, 소셜 버튼 |
| `radius.full`        |   999 | pill, avatar, circular button          |

Radius 규칙:

- 일반 카드는 `8` 이하를 기본으로 한다.
- 인증 화면의 입력창/버튼은 브랜드 화면 예외로 `16`을 사용한다.
- 페이지 섹션 전체를 둥근 카드처럼 만들지 않는다.
- bottom sheet, modal, floating player처럼 명확한 overlay에는 `12`까지 허용한다.
- icon button은 원형 또는 8px radius 중 하나를 명확히 선택한다.

### 5.7 Border and elevation tokens

| Token                   | Value                             | 용도           |
| ----------------------- | --------------------------------- | -------------- |
| `border.width.hairline` | `1`                               | 기본 border    |
| `border.width.focus`    | `2`                               | focus ring     |
| `elevation.none`        | none                              | 기본 surface   |
| `elevation.raised`      | subtle shadow + raised surface    | 카드 강조      |
| `elevation.overlay`     | overlay surface + stronger shadow | modal, popover |

Dark theme elevation 규칙:

- 그림자만으로 depth를 표현하지 않는다.
- surface 색상, border, spacing으로 먼저 층위를 만든다.
- shadow는 floating overlay, modal, popover, drag 상태에만 제한적으로 사용한다.

### 5.8 Motion tokens

| Token                   | Value | 용도                       |
| ----------------------- | ----: | -------------------------- |
| `motion.instant`        |   0ms | reduced motion, 즉시 전환  |
| `motion.fast`           | 120ms | hover/press                |
| `motion.normal`         | 180ms | sheet, modal, toast        |
| `motion.slow`           | 260ms | page-level transition      |
| `motion.authTransition` | 300ms | auth hover/focus/glow 전환 |

Motion 규칙:

- 상태 변화는 짧고 예측 가능해야 한다.
- 인증 화면의 hover/focus/glow 전환은 약 `300ms`를 기준으로 한다.
- 로딩 skeleton, progress는 과도하게 움직이지 않는다.
- `prefers-reduced-motion` 또는 플랫폼 reduce motion 설정을 존중한다.
- 오디오 분석 진행 상태는 애니메이션보다 실제 진행률/단계를 명확히 보여준다.

## 6. 플랫폼별 레이아웃 전략

### 6.1 Android/iOS 모바일

Compact 화면 기준:

- 단일 열 레이아웃
- 화면 제목은 상단 navigation 영역에 배치
- 핵심 CTA는 콘텐츠 흐름 하단 또는 sticky bottom
- bottom tab은 3-5개 주요 목적지에만 사용
- 입력 화면은 keyboard 대응을 필수로 한다
- 파일 업로드/녹음/분석 시작 같은 핵심 액션은 한 화면에 하나만 primary로 둔다

Android/iOS 공통 코드 원칙:

- 같은 화면 로직과 컴포넌트를 공유한다.
- safe area, keyboard, permission, platform convention은 wrapper 또는 hook으로 분리한다.
- iOS/Android를 완전히 다른 화면으로 만들지 않는다.
- platform-specific 분기는 사용자 경험이 달라지는 경우에만 사용한다.

### 6.2 Tablet

Tablet에서는 화면을 넓히는 것이 아니라 정보 구조를 바꾼다.

권장 패턴:

- 목록 + 상세: 프로젝트/파일 목록 왼쪽, 상세 오른쪽
- 입력 + 미리보기: 설정 왼쪽, 결과 preview 오른쪽
- 단계 + 본문: 작업 단계/상태 왼쪽, 현재 작업 본문 오른쪽
- navigation rail 또는 sidebar 사용 가능

Tablet 금지/주의:

- 모바일 card를 단순히 크게 늘리지 않는다.
- 한 줄 길이가 과도하게 길어지지 않게 한다.
- primary CTA가 시야에서 너무 멀어지지 않게 한다.
- split view에서도 각 pane이 독립적으로 스크롤 가능한지 확인한다.

### 6.3 Web

Web은 모바일 앱을 브라우저에 띄운 것이 아니라, keyboard/mouse와 넓은 viewport를 고려한 앱 화면이어야 한다.

Web 기준:

- hover, focus-visible, keyboard tab order를 제공한다.
- URL과 history navigation을 자연스럽게 유지한다.
- desktop에서는 sidebar/header navigation을 사용할 수 있다.
- drag and drop, file picker, shortcut은 보조 수단으로 제공할 수 있다.
- form과 modal은 max width를 둔다.
- 넓은 화면에서 콘텐츠를 가운데만 작게 남기지 않고, 보조 패널이나 context 영역을 활용한다.

Expo Router layout 분기:

- route 구조는 공유한다.
- native에서는 tabs/stack을 사용하고, web에서는 header/sidebar layout을 사용할 수 있다.
- 플랫폼 분기가 필요할 경우 화면 로직은 공유하고 layout shell만 분리한다.

## 7. Navigation

### 7.1 Navigation 구조

권장 최상위 구조:

| 영역              | Mobile                     | Tablet             | Web                   |
| ----------------- | -------------------------- | ------------------ | --------------------- |
| Home/Dashboard    | bottom tab 또는 stack root | sidebar/rail       | sidebar/header        |
| Projects/Library  | tab                        | sidebar item       | sidebar item          |
| Upload/Create     | prominent action           | primary nav action | header/sidebar action |
| Result/Job detail | stack push                 | detail pane        | route detail page     |
| Settings          | stack/modal                | sidebar item       | sidebar item          |

### 7.2 모바일 navigation

- Bottom tab은 3-5개만 둔다.
- 단발성 핵심 액션은 tab보다 primary action button으로 둔다.
- 뒤로가기 동작은 플랫폼 기본 expectation을 따른다.
- 위험하거나 긴 작업은 accidental back을 방지한다.

### 7.3 웹 navigation

- Sidebar는 주요 destination을 안정적으로 보여준다.
- Header는 계정, 알림, 검색, global action을 담당한다.
- Breadcrumb은 deep detail 화면에서만 사용한다.
- Keyboard focus 순서는 화면 시각 구조와 일치해야 한다.

## 8. Component Standards

각 컴포넌트 문서에는 반드시 아래 항목을 포함한다.

- 목적
- 언제 사용하는가
- 언제 사용하지 않는가
- Anatomy
- Variants
- States
- Responsive behavior
- Accessibility
- Example copy
- Design tokens
- QA checklist

### 8.1 Button

Variants:

| Variant        | 용도                       |
| -------------- | -------------------------- |
| Primary        | 화면의 핵심 완료/시작 액션 |
| Secondary      | 보조 액션                  |
| Tertiary/Ghost | 낮은 우선순위 액션         |
| Destructive    | 삭제, 취소, 위험 액션      |
| Icon           | 도구, 닫기, 재생, 다운로드 |

Auth primary CTA:

- 배경은 `#991B1B -> #DC2626 -> #EF4444` 방향의 붉은 그라데이션을 사용한다.
- CTA 주변에는 `0 0 20px rgba(220, 38, 38, 0.4)` 수준의 붉은 glow를 사용한다.
- Web hover에서는 버튼 scale을 약 `1.03-1.05`로 키우고, 화살표 아이콘을 `translateX(4px)` 이동한다.
- Disabled 상태는 opacity `0.5`를 적용한다. 회원가입은 약관 미동의 시 disabled로 둔다.
- 로딩 상태는 같은 버튼 안에서 문구를 `확인 중`으로 바꾸고 중복 제출을 막는다.

Size:

| Size | Height | 사용                  |
| ---- | -----: | --------------------- |
| sm   |     36 | desktop dense toolbar |
| md   |     44 | web/tablet 기본       |
| lg   |     48 | mobile 기본, 주요 CTA |
| auth |  56-62 | 로그인/회원가입 CTA   |

Rules:

- 한 화면에 primary button은 가능한 하나만 둔다.
- 터치 가능한 target은 최소 44x44를 목표로 한다.
- 버튼 텍스트는 동사로 시작한다.
- 로딩 중에는 중복 제출을 막고 loading state를 보여준다.
- icon-only button은 접근성 label과 web tooltip을 제공한다.
- disabled만으로 실패 이유를 숨기지 않는다. 필요한 경우 helper text를 제공한다.

States:

- default
- hover(web)
- pressed
- focused/focus-visible
- disabled
- loading
- danger

### 8.2 Text Input

Anatomy:

- label
- input field
- optional leading/trailing icon
- helper text
- error text
- character count, 필요한 경우

Rules:

- placeholder를 label 대신 사용하지 않는다.
- label은 항상 field 위에 배치한다.
- error text는 field 바로 아래에 둔다.
- required 여부는 문구나 validation으로 명확히 한다.
- password field는 show/hide control을 제공한다.
- mobile keyboard type을 입력 성격에 맞춘다.
- Web에서는 focus-visible outline을 제거하지 않는다.

Auth input state:

- 기본 배경은 `#121212`, border는 `#27272A`, radius는 `16`이다.
- Focus 상태에서는 border를 `rgba(239, 68, 68, 0.5)`로 전환한다.
- Focus 상태에서는 배경을 `#1A1010`으로 바꾸고 `0 0 15px rgba(220, 38, 38, 0.15)` glow를 적용한다.
- Focus 상태의 leading icon은 `#EF4444`로 바뀐다.
- Error 상태는 field 아래에 직접 문구를 노출하고, border/text에 danger 색상을 사용한다.

Default sizes:

| Element            | Compact | Medium+ |
| ------------------ | ------: | ------: |
| Input height       |      48 |   44-48 |
| Horizontal padding |   14-16 |   14-16 |
| Label gap          |       6 |       6 |
| Helper gap         |       6 |       6 |

Auth sizes:

| Element            | Login | Signup |
| ------------------ | ----: | -----: |
| Input height       |    58 |     54 |
| Horizontal padding |    16 |     16 |
| Icon size          |    24 |     22 |
| Text size          |    18 |     17 |

### 8.3 Card

### 8.3 Checkbox

회원가입 약관 동의처럼 명시적 선택이 필요한 경우 커스텀 체크박스를 사용한다.

Anatomy:

- checkbox box
- check icon
- label/legal copy
- optional inline error

States:

- unchecked: background `#121212`, border `#27272A`
- checked: background/border `#DC2626`
- checked glow: `0 0 16px rgba(220, 38, 38, 0.35)`
- check icon: white, scale `0 -> 1` 팝업 애니메이션
- disabled: opacity와 helper text로 이유를 함께 제공

Rules:

- 기본 브라우저 checkbox를 그대로 노출하지 않는다.
- label 전체를 press target으로 취급한다.
- accessibility role/state는 `checkbox`, `checked`를 제공한다.
- 색상만으로 checked를 전달하지 않고 check icon을 함께 표시한다.
- 약관 미동의로 CTA가 disabled일 때는 제출 버튼 opacity `0.5`와 disabled state를 함께 적용한다.

### 8.4 Card

Card는 반복되는 개별 항목이나 독립 정보 묶음에만 사용한다.

Good:

- 프로젝트 카드
- 작업 결과 카드
- 파일 카드
- 요금제 카드
- 알림 카드

Avoid:

- 페이지 전체 섹션을 카드로 감싸기
- 카드 안에 카드 중첩
- 모든 UI를 카드로 분절하기

Rules:

- radius는 기본 `8` 이하.
- dark theme에서는 border + surface 차이를 우선한다.
- hover elevation은 web에서 clickable card에만 적용한다.
- clickable card는 내부 button과 click target이 충돌하지 않게 한다.

### 8.5 List and Table

Mobile:

- 리스트 카드 또는 compact row 사용
- 핵심 정보 2-3개만 노출
- secondary metadata는 아래 줄 또는 detail로 이동

Tablet/Web:

- 스캔이 중요한 데이터는 table 허용
- 비교가 중요한 데이터는 column 유지
- 상세 조작이 많은 경우 list + detail pane 권장

Rules:

- 모바일에서 큰 table을 억지로 넣지 않는다.
- table은 column hide, row detail, horizontal scroll 중 의도를 정한다.
- status는 색상과 텍스트를 함께 사용한다.
- timecode, duration, file size는 tabular/mono 스타일을 사용한다.

### 8.6 Modal, Dialog, Sheet

| Class     | 권장 패턴                            |
| --------- | ------------------------------------ |
| Compact   | bottom sheet 또는 full-screen dialog |
| Medium    | sheet 또는 centered modal            |
| Expanded+ | centered modal, side panel, popover  |

Rules:

- destructive confirmation은 명확한 제목, 영향, 취소 버튼을 제공한다.
- modal 안에 복잡한 multi-step 작업을 넣지 않는다. 필요하면 별도 route로 이동한다.
- Web에서는 focus trap, Escape close, restore focus를 고려한다.
- Native에서는 back gesture/back button 동작을 정의한다.

### 8.7 Toast, Banner, Inline Alert

| Component    | 용도                         |
| ------------ | ---------------------------- |
| Toast        | 짧은 완료/실패 알림          |
| Banner       | 화면 전체에 영향을 주는 상태 |
| Inline alert | 특정 form/section의 오류     |

Rules:

- 중요한 오류를 toast만으로 전달하지 않는다.
- 사용자가 조치해야 하는 문제는 inline alert 또는 banner로 표시한다.
- screen reader가 상태 변화를 인지할 수 있게 accessibility live/status 처리를 고려한다.
- 자동 사라짐 알림은 사용자가 읽을 수 있는 충분한 시간을 둔다.

### 8.8 Audio-specific components

Vox2Vocal 특화 컴포넌트는 별도 기준을 둔다.

#### Waveform

- 모바일 최소 높이: 96
- tablet/web 권장 높이: 120-180
- 현재 재생 위치, 선택 구간, 분석 구간을 구분한다.
- 색상만으로 선택 상태를 전달하지 않는다.
- scrubber target은 터치하기 충분해야 한다.

#### Player controls

- 기본 control: play/pause, seek, current time, duration
- icon button target은 최소 44x44
- web에서는 keyboard control을 고려한다.
- loading/buffering/error 상태를 별도 표시한다.

#### Job progress

- 단계 이름, 진행률, 남은 시간 또는 현재 상태를 함께 보여준다.
- 장기 작업은 background 처리와 재진입 상태를 고려한다.
- 실패 시 재시도, 문의/로그 확인, 파일 교체 중 하나 이상의 다음 행동을 제공한다.

## 9. Screen Patterns

### 9.1 Auth pattern

Mobile:

- 상단 `brand-mark.png`
- 제품명 또는 화면명
- 짧은 설명
- form
- primary login/signup CTA
- social auth buttons, 필요한 경우
- secondary links
- legal text 또는 약관 동의는 하단

Web:

- form panel max `384`
- 현재 인증 화면은 태블릿/웹에서도 모바일 폭을 유지하고 중앙 정렬한다.
- hover, focus, URL navigation을 지원한다.
- 향후 마케팅/브랜드 context 영역이 필요하면 form 로직은 공유하고 layout shell만 분기한다.

Rules:

- auth form은 키보드와 password manager 사용을 방해하지 않는다.
- error는 field-level과 form-level을 구분한다.
- OAuth/SSO가 추가되면 primary auth method 우선순위를 명확히 한다.
- 로그인 route는 `/`, 회원가입 route는 `/signup`이다.
- 공통 인증 컴포넌트는 `src/features/auth/auth-components.tsx`에 둔다.
- 화면별 상태/검증은 `login-screen.tsx`, `signup-screen.tsx`에 둔다.
- ambient glow는 로고 뒤/상단에 배치하고 opacity를 낮게 유지한다.
- 회원가입 CTA는 약관 미동의 시 disabled이며 체크박스 선택 후 활성화된다.

### 9.2 Dashboard/Home pattern

목적:

- 최근 프로젝트
- 진행 중 작업
- 새 변환 시작
- 사용량/상태 요약

Mobile:

- 상단 greeting/status
- primary create/upload action
- 진행 중 작업
- 최근 항목 리스트

Tablet/Web:

- sidebar navigation
- summary cards
- active jobs
- recent projects table/list
- optional right context panel

### 9.3 Upload/Create pattern

Mobile:

- 한 번에 하나의 입력 단계
- 파일 선택/녹음 시작 CTA를 크게 표시
- 권한/파일 오류를 즉시 표시

Tablet/Web:

- drag and drop zone
- 파일 정보 preview
- 변환 옵션 side panel
- 예상 처리 시간/품질 안내

Rules:

- 파일 제한은 업로드 후가 아니라 업로드 전/중에 알려준다.
- 실패 사유와 다음 행동을 반드시 제공한다.
- progress는 실제 상태와 연결한다.

### 9.4 Analysis/Processing pattern

Mobile:

- 현재 단계
- progress
- background 처리 안내
- 취소/나중에 보기

Tablet/Web:

- 단계 timeline
- 로그 또는 상세 상태는 접을 수 있게 제공
- 결과 preview가 가능하면 오른쪽 pane에 배치

Rules:

- 장기 작업 화면은 빈 spinner만 보여주지 않는다.
- 상태가 멈췄을 때 사용자가 다음 행동을 알 수 있어야 한다.

### 9.5 Result pattern

Mobile:

- 결과 요약
- player
- 주요 다운로드/저장 CTA
- 세부 정보는 접기/상세 이동

Tablet/Web:

- player/waveform
- before/after 비교
- parameter summary
- export actions
- history/version list

Rules:

- 결과 저장, 다운로드, 공유 액션은 구분한다.
- audio preview가 로딩/실패했을 때 fallback UI가 있어야 한다.

### 9.6 Settings pattern

Mobile:

- grouped list
- detail은 stack push
- destructive action은 하단 분리

Tablet/Web:

- settings sidebar
- detail pane
- save/cancel 위치 고정

Rules:

- 설정 변경 성공/실패를 명확히 보여준다.
- 위험 설정은 confirmation을 요구한다.

## 10. Interaction States

모든 interactive component는 아래 상태를 설계하고 구현한다.

| State         | 설명                         |
| ------------- | ---------------------------- |
| Default       | 기본 상태                    |
| Hover         | Web pointer hover            |
| Pressed       | touch/click active           |
| Focused       | keyboard/screen reader focus |
| Focus-visible | Web keyboard focus ring      |
| Disabled      | 사용할 수 없는 상태          |
| Loading       | 처리 중, 중복 입력 방지      |
| Error         | 유효성/처리 오류             |
| Success       | 완료/저장 성공               |
| Selected      | 탭, 리스트, 카드 선택        |

Rules:

- focus ring을 숨기지 않는다.
- hover만으로 중요한 정보를 제공하지 않는다.
- disabled 상태의 이유가 중요하면 helper text를 제공한다.
- loading 상태에서도 layout shift가 없어야 한다.
- 선택 상태는 색상 외에도 border, icon, label로 보강한다.

## 11. Accessibility

### 11.1 기본 목표

Vox2Vocal의 FE 화면은 WCAG 2.2 AA 수준을 목표로 한다.

기본 기준:

- 일반 텍스트 대비 4.5:1 이상
- 큰 텍스트 대비 3:1 이상
- pointer target 최소 24x24, 제품 기준은 44x44 이상 권장
- 텍스트 200% 확대 시 콘텐츠/기능 손실 없음
- focus indicator는 시각적으로 명확해야 함
- form error는 screen reader와 시각 사용자 모두 인지 가능해야 함

### 11.2 Mobile accessibility

- touch target은 44x44 이상을 목표로 한다.
- icon-only action은 accessibility label을 제공한다.
- screen reader 순서는 화면 순서와 일치한다.
- keyboard가 올라와도 field와 error가 가려지지 않는다.
- large text 설정에서 버튼 텍스트가 잘리지 않아야 한다.

### 11.3 Web accessibility

- 모든 interactive element는 keyboard로 접근 가능해야 한다.
- focus-visible style을 제공한다.
- modal은 focus trap과 Escape close를 고려한다.
- skip link 또는 main landmark를 고려한다.
- hover interaction에는 keyboard 대체 동작이 있어야 한다.
- browser zoom 200%에서 horizontal scroll 없이 주요 작업을 완료할 수 있어야 한다.

### 11.4 Content accessibility

- 오류 문구는 원인과 해결 방법을 함께 쓴다.
- 색상 이름만으로 상태를 설명하지 않는다.
- 파일 크기, 시간, 진행률은 명확한 단위를 표시한다.
- 전문 오디오 용어는 처음 등장할 때 보조 설명을 제공한다.

## 12. Content and Microcopy

### 12.1 버튼 문구

원칙:

- 동사 중심
- 짧고 명확하게
- 결과를 예측할 수 있게

예시:

| 상황        | 권장                        |
| ----------- | --------------------------- |
| 파일 업로드 | `파일 업로드`               |
| 변환 시작   | `보컬 변환 시작`            |
| 결과 저장   | `결과 저장`                 |
| 다시 시도   | `다시 시도`                 |
| 삭제        | `삭제` 또는 `프로젝트 삭제` |

### 12.2 오류 문구

오류 문구 구조:

```txt
무엇이 실패했는가 + 왜 실패했는가 + 다음 행동
```

예시:

```txt
파일을 업로드하지 못했습니다. 지원하지 않는 형식입니다. WAV 또는 MP3 파일을 선택해 주세요.
```

### 12.3 Empty state

Empty state는 단순히 "없음"을 보여주지 않는다.

필수 요소:

- 현재 상태
- 사용자가 할 수 있는 다음 행동
- 필요한 경우 primary CTA

예시:

```txt
아직 변환한 프로젝트가 없습니다.
첫 음성 파일을 업로드해 보컬 변환을 시작하세요.
```

## 13. Design-to-Code Handoff

### 13.1 Source of truth

최종 디자인 값은 코드에서 다음 위치로 수렴해야 한다.

```txt
vox2vocal-app/tamagui.config.ts
vox2vocal-app/src/design-system/tokens.ts
vox2vocal-app/src/features/auth/
vox2vocal-app/assets/brand-mark.png
```

권장 구조:

```txt
src/
  design-system/
    tokens.ts
    themes.ts
    breakpoints.ts
    typography.ts
    components/
```

현재 `tamagui.config.ts`는 default config만 사용한다. 실제 화면 개발 전에는 이 문서의 token을 반영한 앱 전용 Tamagui config로 확장하는 것을 권장한다.

현재 구현된 인증 화면 기준:

```txt
app/
  index.tsx                 # 로그인 route
  signup.tsx                # 회원가입 route
src/
  design-system/
    tokens.ts               # viewport, color, brand, auth, spacing, radius
  features/
    auth/
      auth-components.tsx   # auth scaffold, input, button, checkbox, social button
      login-screen.tsx
      signup-screen.tsx
assets/
  logo.png                  # 원본 브랜드 이미지
  brand-icon.png            # 정사각 아이콘 파생 자산
  brand-mark.png            # 인증 UI용 투명 배경 브랜드 마크
```

### 13.2 코드 작성 규칙

- 색상, spacing, radius, typography를 hard-code하지 않는다.
- 브랜드/인증 색상은 `colors`, `brandColors`, `authColors`를 우선 사용한다.
- primitive token은 theme/config에서만 직접 사용한다.
- 화면 컴포넌트는 semantic/component token을 사용한다.
- `Platform.OS` 분기는 레이아웃 shell, native permission, web hover/focus처럼 실제 플랫폼 차이가 있는 경우에만 사용한다.
- 모바일/웹을 완전히 다른 컴포넌트로 복제하지 않는다. form logic, validation, data state는 공유한다.
- `.web.tsx`, `.native.tsx` 또는 layout wrapper 분기는 UI 구조 차이가 충분히 클 때만 사용한다.

### 13.3 Tamagui 적용 기준

Tamagui에서 우선 사용할 개념:

- `tokens`: color, size, space, radius, zIndex
- `themes`: dark/light, component sub-theme
- `variants`: button size, input state, card density
- pseudo states: `hoverStyle`, `pressStyle`, `focusStyle`, `disabledStyle`
- responsive media styles: `$sm`, `$gtMd`, `$gtLg` 등 프로젝트 기준으로 명명

프로젝트 breakpoint 이름은 이 문서의 class와 맞춘다.

```txt
compact
medium
expanded
large
xlarge
```

## 14. QA Checklist

### 14.1 Layout QA

- `360 x 640`에서 가로 스크롤이 없는가
- `390 x 844`에서 핵심 CTA가 보이거나 접근 가능한가
- `375 x 812`에서 로그인/회원가입 auth flow가 깨지지 않는가
- `412 x 915`에서 여백이 과도하지 않은가
- `768 x 1024`에서 tablet layout이 자연스러운가
- `1024 x 768`에서 짧은 height 대응이 되는가
- `1366 x 768`에서 desktop layout이 납작해지지 않는가
- `1440 x 900`에서 content max width가 적절한가
- `1920 x 1080`에서 화면이 과도하게 퍼지지 않는가
- 태블릿/웹 auth 화면은 max `384` 컨테이너로 중앙 정렬되는가

### 14.2 Component QA

- 모든 button state가 있는가
- icon-only button에 label이 있는가
- input error가 field와 연결되어 있는가
- auth input focus에서 border, glow, icon active color가 함께 바뀌는가
- 회원가입 checkbox checked 상태에서 check icon, red fill, glow가 함께 보이는가
- 약관 미동의 시 가입 CTA가 disabled/opacity `0.5`인가
- loading state에서 layout shift가 없는가
- modal close/back/focus 동작이 정의되어 있는가
- toast만으로 중요한 오류를 전달하지 않는가

### 14.3 Accessibility QA

- 일반 텍스트 대비 4.5:1 이상인가
- focus indicator가 보이는가
- keyboard만으로 주요 작업을 완료할 수 있는가
- screen reader label이 누락되지 않았는가
- text 200% 확대에서 기능 손실이 없는가
- touch target이 44x44 이상인가
- reduced motion 설정에서 과한 애니메이션이 줄어드는가

### 14.4 Product QA

- 파일 업로드 실패 이유가 명확한가
- 처리 중 상태가 spinner 하나로 끝나지 않는가
- 결과 화면에서 재생/저장/다운로드 흐름이 구분되는가
- 모바일에서 한 화면의 primary action이 하나로 명확한가
- 웹에서 keyboard/mouse 사용성이 보강되어 있는가

## 15. 문서 확장 계획

이 문서는 1차 기준서다. 실제 화면 개발이 시작되면 아래 문서를 추가한다.

| 문서                              | 목적                                             |
| --------------------------------- | ------------------------------------------------ |
| `design.md`                       | 브랜드 컬러, 인증 화면, 자산 사용 기준 빠른 참조 |
| `tokens-spec.md`                  | Tamagui token 실제 값과 코드 매핑                |
| `component-spec.md`               | Button, Input, Card 등 상세 컴포넌트 명세        |
| `screen-patterns.md`              | Auth, Upload, Result 등 화면 패턴 상세           |
| `accessibility-checklist.md`      | QA 실행용 체크리스트                             |
| `frontend-implementation-plan.md` | 앱 코드 리팩터링/구현 순서                       |

## 16. 참고한 기준

- Android Developers, Window size classes: https://developer.android.com/develop/ui/views/layout/use-window-size-classes
- Apple Human Interface Guidelines, Layout: https://developer.apple.com/design/human-interface-guidelines/layout
- web.dev, Responsive web design basics: https://web.dev/responsive-web-design-basics/
- MDN, CSS media queries: https://developer.mozilla.org/en-US/docs/Web/CSS/Media_Queries
- W3C, WCAG 2.2: https://www.w3.org/TR/WCAG22/
- W3C Design Tokens Community Group, Format Module 2025.10: https://www.w3.org/community/reports/design-tokens/CG-FINAL-format-20251028/
- Atlassian Design System, Foundations: https://atlassian.design/foundations/
- GOV.UK Design System, Components and Patterns: https://design-system.service.gov.uk/components/ and https://design-system.service.gov.uk/patterns/
- Microsoft Fluent 2, Design Tokens: https://fluent2.microsoft.design/design-tokens
- Shopify Polaris, Color tokens: https://polaris-react.shopify.com/design/colors/color-tokens
- Expo docs, Expo Router platform-specific modules: https://docs.expo.dev/router/advanced/platform-specific-modules/
- Tamagui docs, Styles/Themes/Variants: https://tamagui.dev/docs/intro/styles, https://tamagui.dev/docs/intro/themes, https://tamagui.dev/docs/core/variants
