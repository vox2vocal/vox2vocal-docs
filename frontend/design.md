# Vox2Vocal Design Reference

최종 업데이트: 2026-06-09

이 문서는 Vox2Vocal 프론트엔드 디자인의 빠른 참조 문서다. 상세 기준은 `design-system-guide.md`를 따른다.

## Source Of Truth

| 영역                    | 기준 위치                                             |
| ----------------------- | ----------------------------------------------------- |
| 프론트엔드 아키텍처     | `vox2vocal-docs/frontend/architecture.md`             |
| 디자인 시스템 상세 기준 | `vox2vocal-docs/frontend/design-system-guide.md`      |
| 코드 토큰               | `vox2vocal-app/src/design-system/tokens.ts`           |
| 인증 공통 UI            | `vox2vocal-app/src/features/auth/auth-components.tsx` |
| 로그인 화면             | `vox2vocal-app/src/features/auth/login-screen.tsx`    |
| 회원가입 화면           | `vox2vocal-app/src/features/auth/signup-screen.tsx`   |
| 브랜드 원본 이미지      | `vox2vocal-app/assets/logo.png`                       |
| 정사각 브랜드 아이콘    | `vox2vocal-app/assets/brand-icon.png`                 |
| UI용 투명 브랜드 마크   | `vox2vocal-app/assets/brand-mark.png`                 |

## Brand Mood

Vox2Vocal의 현재 브랜드 무드는 블랙 앤 레드, 네온 글로우, 오디오/음성 파동, 현대적인 보컬 창작 도구다.

적용 원칙:

- 기본 배경은 거의 검정에 가까운 `#050505`를 사용한다.
- 브랜드가 처음 노출되는 인증/온보딩 화면은 붉은 ambient glow를 사용한다.
- 작업 화면은 정보 판독성을 우선하고, 브랜드 레드는 주요 액션과 상태 강조에 제한적으로 사용한다.
- 글로우는 시선 유도 장치로 쓰되, 텍스트 대비와 입력 상태 인지를 해치면 안 된다.

## Brand Colors

| 이름             | 값        | 사용                             |
| ---------------- | --------- | -------------------------------- |
| Background black | `#050505` | 앱 기본 배경                     |
| Surface          | `#121212` | 입력창, 소셜 버튼                |
| Surface hover    | `#1A1A1A` | hover/pressed surface            |
| Surface focus    | `#1A1010` | auth input focus                 |
| Border           | `#27272A` | 기본 border                      |
| Primary text     | `#FFFFFF` | 제목/본문                        |
| Secondary text   | `#A1A1AA` | 보조 문구                        |
| Muted text       | `#71717A` | placeholder/비활성               |
| Red dark         | `#991B1B` | CTA gradient 시작, pressed       |
| Red              | `#DC2626` | primary action, checkbox checked |
| Red bright       | `#EF4444` | hover, focus, glow highlight     |
| Danger           | `#F87171` | error text                       |

효과 색상:

- CTA glow: `0 0 20px rgba(220, 38, 38, 0.4)`
- Input focus glow: `0 0 15px rgba(220, 38, 38, 0.15)`
- Checkbox checked glow: `0 0 16px rgba(220, 38, 38, 0.35)`
- Text selection: `rgba(239, 68, 68, 0.3)`

## Layout

모바일 퍼스트를 기본으로 한다.

폴더 구조, route/feature/shared 계층, 상태 관리, 플랫폼 분기 기준은 `architecture.md`를 따른다. 이 문서는 색상, 타이포그래피, 인증 컴포넌트 같은 빠른 디자인 참조에 집중한다.

| 항목                    | 기준           |
| ----------------------- | -------------- |
| 대표 모바일 기준        | `375 x 812`    |
| 최소 모바일 QA          | `360 x 640`    |
| iPhone 표준 QA          | `390 x 844`    |
| Auth container          | max `384`      |
| Auth horizontal padding | `24`           |
| Auth route              | `/`, `/signup` |

태블릿/웹에서도 인증 화면은 모바일 폭 컨테이너를 중앙에 유지한다. 제품 대시보드, 업로드, 결과 화면은 viewport class에 따라 2-pane 또는 넓은 레이아웃을 허용한다.

## Typography

기본 폰트는 플랫폼 system font를 사용한다. 한국어 웹/앱 품질이 더 중요해지는 시점에는 Pretendard 또는 Noto Sans KR 적용을 검토한다.

| 용도          | 크기/굵기          |
| ------------- | ------------------ |
| Auth title    | 32-38, weight 700  |
| Auth subtitle | 18, weight 500     |
| Auth input    | 17-18, weight 400  |
| Auth button   | 18, weight 600     |
| Legal/terms   | 14, weight 500-700 |

## Auth Components

### Input

- 기본: background `#121212`, border `#27272A`, radius `16`
- Focus: border `rgba(239, 68, 68, 0.5)`, background `#1A1010`, red glow
- Focus icon: `#EF4444`
- Error: field 아래 문구, danger color
- Password: show/hide icon control 제공

### Primary Button

- Gradient: `#991B1B -> #DC2626 -> #EF4444`
- Glow: red CTA glow
- Hover: scale `1.03-1.05`, arrow `translateX(4px)`
- Disabled: opacity `0.5`
- Loading: 중복 제출 방지, 문구 `확인 중`

### Social Button

- Background `#121212`
- Border `#27272A`
- Hover background `#1A1A1A`
- Google/Apple icon과 텍스트를 함께 사용한다.

### Checkbox

- Unchecked: background `#121212`, border `#27272A`
- Checked: background/border `#DC2626`, check icon, red glow
- Check icon은 scale pop animation을 사용한다.
- 회원가입 CTA는 약관 미동의 시 disabled다.

## Assets

`assets/logo.png`는 원본 브랜드 이미지로 보존한다. 정사각 아이콘이 필요한 곳은 `assets/brand-icon.png`를 사용하고, 화면 배경과 자연스럽게 합성되어야 하는 인증 UI 상단 마크는 투명 배경의 `assets/brand-mark.png`를 사용한다.

사용 기준:

- 앱 런처, splash 등 플랫폼 아이콘은 목적별 전용 자산을 사용한다.
- 화면 상단 브랜드 마크는 `brand-mark.png`를 사용한다.
- 임의로 원본을 늘려 쓰지 않고, 필요한 출력 크기의 파생 자산을 만든다.

## QA

최소 확인:

- `375 x 812`에서 로그인과 회원가입 화면이 깨지지 않는가
- `brand-mark.png`가 화면에서 정상 로드되고 사각 배경이 보이지 않는가
- 입력 포커스 시 border, glow, icon color가 함께 바뀌는가
- 회원가입 checkbox 전/후 CTA disabled 상태가 바뀌는가
- 웹 콘솔 warning/error가 없는가
- `npm run typecheck`, `npm run lint`, `npm run test -- --runInBand --watchman=false`가 통과하는가
