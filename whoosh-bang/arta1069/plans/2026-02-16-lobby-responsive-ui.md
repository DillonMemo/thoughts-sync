---
date: 2026-02-16T21:00:00+0900
author: arta1069
git_commit: 726fe8601543e0ed40363062f06ef43b8a712b9c
branch: main
repository: Whoosh-Bang
topic: "로비 화면 반응형 UI 구현 계획"
tags: [plan, lobby, responsive, mobile, tailwind, flex, grid]
status: draft
---

# 로비 화면 반응형 UI 구현 계획

## 개요

로비 화면(`LobbyContent.tsx`)을 absolute 포지셔닝 기반에서 flex/grid 기반으로 재구성하여 PC와 모바일 환경 모두에서 자연스러운 레이아웃을 제공한다.

- **모바일 세로모드**: 수직 스택 (ProfileCard → CharacterDisplay → CharacterSelector → PlayButton)
- **가로모드 / 데스크탑(≥768px)**: 현재 PC 레이아웃 유지 (CSS Grid)

## 현재 상태 분석

### 레이아웃 구조 (`LobbyContent.tsx:70-119`)

```
[relative h-screen w-screen overflow-hidden]
├── <Image> 배경 (fill, object-cover)
├── 어두운 오버레이 (absolute inset-0 bg-black/30)
├── ProfileCard 래퍼 (relative p-4 z-10, md:max-w-86)
│   ├── <ProfileCard>
│   └── <WalletLinkButton>
├── 빈 div (absolute right-6 top-6 z-10) ← 사용되지 않음
├── <CharacterDisplay> (absolute inset-0, pointer-events-none)
├── <CharacterSelector> (absolute bottom-6 left-1/2 -translate-x-1/2 z-10)
└── <PlayButton> (absolute bottom-12 right-12)
```

### 핵심 문제
- 모든 주요 UI가 absolute 포지셔닝 → 모바일에서 겹침 발생
- CharacterSelector와 PlayButton이 하단에서 충돌
- ProfileCard가 모바일 화면에서 과도한 영역 차지
- 반응형 처리: `md:max-w-86` 1개소만 존재

## 원하는 최종 상태

### 모바일 세로모드 (portrait, < 768px)
```
┌─────────────────────┐
│    ProfileCard      │  shrink-0
│    + WalletLink     │
├─────────────────────┤
│                     │
│  CharacterDisplay   │  flex-1 (남은 공간)
│                     │
├─────────────────────┤
│  CharacterSelector  │  shrink-0, overflow-x-auto
├─────────────────────┤
│     PlayButton      │  shrink-0, 중앙 정렬
└─────────────────────┘
```

### 데스크탑 / 가로모드 (landscape OR ≥ 768px)
```
┌──────────┬────────────────┬─────────┐
│ Profile  │                │         │  row 1: auto
│ Card     │                │         │
├──────────┤  Character     ├─────────┤
│          │  Display       │         │  row 2: 1fr
│          │  (전체 영역)    │         │
├──────────┤                ├─────────┤
│          │ CharSelector   │  PLAY   │  row 3: auto
└──────────┴────────────────┴─────────┘
  col 1:auto  col 2: 1fr    col 3:auto
```

### 검증 방법
- 모바일 세로(375px): 수직 스택, 겹침 없음
- 모바일 가로(667px+): PC 레이아웃
- 태블릿 세로(768px+): PC 레이아웃
- PC(1280px+): 현재와 동일한 시각적 배치

## 하지 않을 것

- Game 화면 변경 (현재 세로모드 오버레이 유지)
- Screen Orientation API / CSS transform rotate 적용
- ProfileCard 내부 구조 변경 (반응형 wrapper만 조정)
- 새로운 모바일 전용 컴포넌트 생성
- 터치 조작 UI 추가

## 구현 접근 방식

### 핵심 전략: Tailwind v4 `@custom-variant` + flex/grid 전환

Tailwind CSS v4의 `@custom-variant`를 사용하여 `landscape OR ≥ 768px` 조건을 하나의 `desktop:` 변형으로 정의한다. 이렇게 하면 `landscape:` + `md:` 클래스 중복 없이 깔끔하게 처리 가능하다.

```css
/* globals.css에 추가 */
@custom-variant desktop (@media (orientation: landscape), (min-width: 768px));
```

- **기본(모바일 portrait)**: `display: flex; flex-direction: column`
- **desktop 변형 활성 시**: `display: grid` + 3x3 grid template

---

## 1단계: CSS 커스텀 변형 및 LobbyContent 레이아웃 재구성

### 개요
globals.css에 `desktop:` 커스텀 변형을 추가하고, LobbyContent의 루트 레이아웃을 absolute에서 flex/grid로 전환한다.

### 필요한 변경:

#### 1. globals.css - 커스텀 변형 추가
**파일**: `apps/web/src/app/globals.css`
**변경**: `@import "tailwindcss"` 직후에 커스텀 변형 선언

```css
@import "tailwindcss";
@custom-variant desktop (@media (orientation: landscape), (min-width: 768px));
```

#### 2. LobbyContent.tsx - 레이아웃 컨테이너 재구성
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`
**변경**: 배경은 유지하되, 콘텐츠 영역을 flex/grid로 전환

**Before** (라인 70-119):
```tsx
return (
  <div className="relative h-screen w-screen overflow-hidden">
    <Image ... />
    <div className="absolute inset-0 bg-black/30" />

    {/* ProfileCard - relative 포지셔닝 */}
    <div className={cn("relative p-4 z-10 flex flex-col gap-2.5", "md:max-w-86")}>
      <ProfileCard ... />
      <div className="flex flex-nowrap gap-2 items-center">
        <WalletLinkButton ... />
      </div>
    </div>

    {/* 빈 div */}
    <div className="absolute right-6 top-6 z-10"></div>

    <CharacterDisplay ... />
    <CharacterSelector ... />
    <PlayButton ... />
  </div>
)
```

**After**:
```tsx
return (
  <div className="relative h-screen w-screen overflow-hidden">
    {/* 배경 이미지 - 변경 없음 */}
    <Image
      src={`/assets/background/7/7_game_background.png`}
      alt="Lobby Background"
      fill
      className="object-cover"
      priority
    />
    {/* 어두운 오버레이 - 변경 없음 */}
    <div className="absolute inset-0 bg-black/30" />

    {/* 콘텐츠 레이아웃 컨테이너 */}
    <div
      className={cn(
        "relative z-10 flex h-full flex-col gap-3 p-4",
        "desktop:grid desktop:grid-rows-[auto_1fr_auto] desktop:grid-cols-[auto_1fr_auto] desktop:gap-0 desktop:p-6"
      )}
    >
      {/* 좌상단(desktop) / 첫 번째(mobile): 프로필 + 지갑 */}
      <div
        className={cn(
          "z-10 shrink-0 flex flex-col gap-2.5",
          "desktop:col-start-1 desktop:row-start-1 desktop:max-w-86 desktop:self-start"
        )}
      >
        <ProfileCard
          displayName={profile?.display_name ?? null}
          username={profile?.username ?? userEmail}
          level={profile?.level ?? 1}
          totalExperience={profile?.total_experience ?? 0}
          wins={profile?.wins ?? 0}
          losses={profile?.losses ?? 0}
          totalMatches={profile?.total_matches ?? 0}
          avatarUrl={avatarUrl}
        />
        <div className="flex flex-nowrap gap-2 items-center">
          <WalletLinkButton initialWallet={wallet} />
        </div>
      </div>

      {/* 중앙(desktop) / 두 번째(mobile): 캐릭터 표시 */}
      <CharacterDisplay character={selectedCharData} />

      {/* 하단 중앙(desktop) / 세 번째(mobile): 캐릭터 선택 */}
      <CharacterSelector
        characters={allCharacters}
        ownedIds={ownedSet}
        selectedId={selectedCharacter}
        onSelect={handleCharacterSelect}
      />

      {/* 우하단(desktop) / 네 번째(mobile): 플레이 버튼 */}
      <PlayButton loading={isSaving} />
    </div>
  </div>
)
```

**제거 항목**:
- `{/* 우상단: 로그아웃 */}` 빈 div (라인 102-103) 삭제 (사용되지 않음, 로그아웃은 ProfileCard에 있음)

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `cd apps/web && npx tsc --noEmit`
- [ ] 린트 통과: `cd apps/web && npx next lint`
- [ ] 빌드 성공: `cd apps/web && npm run build`

#### 수동 검증:
- [ ] PC 브라우저(1280px+)에서 로비 화면이 현재와 동일하게 보임
- [ ] 콘텐츠 레이아웃 컨테이너가 flex(portrait) / grid(landscape/desktop)로 전환됨

**Implementation Note**: 이 단계 완료 후 브라우저에서 레이아웃 전환이 올바르게 작동하는지 확인합니다.

---

## 2단계: 하위 컴포넌트 반응형 적용

### 개요
CharacterDisplay, CharacterSelector, PlayButton 각 컴포넌트에서 absolute 포지셔닝을 제거하고 flex/grid 레이아웃에 맞는 반응형 스타일을 적용한다.

### 필요한 변경:

#### 1. CharacterDisplay.tsx - absolute → flex/grid 대응
**파일**: `apps/web/src/components/lobby/CharacterDisplay.tsx`
**변경**: absolute 제거, flex-1(모바일) + grid span(데스크탑)

**Before** (라인 16):
```tsx
<div className="pointer-events-none absolute inset-0 flex items-center justify-center">
```

**After**:
```tsx
<div
  className={cn(
    "pointer-events-none flex flex-1 items-center justify-center",
    "desktop:col-span-full desktop:row-span-full"
  )}
>
```

- `cn` 유틸리티 import 추가 필요
- `absolute inset-0` → `flex-1` (모바일에서 남은 공간 차지)
- `desktop:col-span-full desktop:row-span-full` (데스크탑에서 전체 그리드 영역 차지, 다른 요소 뒤에 표시)

#### 2. CharacterSelector.tsx - absolute 제거 + 가로 스크롤
**파일**: `apps/web/src/components/lobby/CharacterSelector.tsx`
**변경**: absolute 포지셔닝 제거, 가로 스크롤 추가, grid 배치

**Before** (라인 22-27):
```tsx
<div className="absolute bottom-6 left-1/2 -translate-x-1/2 z-10">
  <div
    className={cn(
      "flex gap-2 rounded-md",
      "ounded-xl bg-background/40 backdrop-blur-sm p-3"
    )}
  >
```

**After**:
```tsx
<div
  className={cn(
    "z-10 shrink-0 self-center",
    "desktop:col-start-2 desktop:row-start-3 desktop:self-end"
  )}
>
  <div
    className={cn(
      "flex gap-2 overflow-x-auto",
      "rounded-xl bg-background/40 backdrop-blur-sm p-3",
      "max-w-[calc(100vw-2rem)] desktop:max-w-none"
    )}
  >
```

- `absolute bottom-6 left-1/2 -translate-x-1/2` 제거
- `overflow-x-auto` 추가 (모바일에서 캐릭터 많을 때 가로 스크롤)
- `max-w-[calc(100vw-2rem)]` 모바일에서 패딩 고려한 최대 너비 제한
- `desktop:max-w-none` 데스크탑에서 제한 해제
- `"ounded-xl"` 오타를 `"rounded-xl"`로 수정

#### 3. PlayButton.tsx - absolute 제거 + 모바일 크기 조정
**파일**: `apps/web/src/components/lobby/PlayButton.tsx`
**변경**: absolute 포지셔닝 제거, 모바일 크기 축소, grid 배치

**Before** (라인 15-33):
```tsx
<div className="absolute bottom-12 right-12">
  <button
    ...
    className={cn(
      "flex cursor-pointer items-center gap-3 rounded-2xl bg-primary px-14 py-6 text-3xl font-bold text-primary-foreground transition-all duration-300",
      ...
    )}
  >
```

**After**:
```tsx
<div
  className={cn(
    "z-10 shrink-0 self-center",
    "desktop:col-start-3 desktop:row-start-3 desktop:self-end desktop:justify-self-end"
  )}
>
  <button
    ...
    className={cn(
      "flex cursor-pointer items-center gap-2 rounded-2xl bg-primary px-8 py-4 text-xl font-bold text-primary-foreground transition-all duration-300",
      "desktop:gap-3 desktop:px-14 desktop:py-6 desktop:text-3xl",
      ...
    )}
  >
```

- `absolute bottom-12 right-12` 제거
- 모바일: `px-8 py-4 text-xl gap-2` (더 작은 버튼)
- 데스크탑: `desktop:px-14 desktop:py-6 desktop:text-3xl desktop:gap-3` (현재 크기 유지)
- `cn` 유틸리티 이미 import 되어 있음

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `cd apps/web && npx tsc --noEmit`
- [ ] 린트 통과: `cd apps/web && npx next lint`
- [ ] 빌드 성공: `cd apps/web && npm run build`

#### 수동 검증:
- [ ] **모바일 세로(375px portrait)**: 수직 스택, 요소 겹침 없음
- [ ] **모바일 가로(667px landscape)**: PC와 동일한 grid 레이아웃
- [ ] **PC(1280px+)**: 현재와 시각적으로 동일
  - ProfileCard 좌상단
  - CharacterDisplay 중앙
  - CharacterSelector 하단 중앙
  - PlayButton 우하단
- [ ] CharacterSelector에 캐릭터 4개 이상일 때 모바일에서 가로 스크롤 작동
- [ ] PlayButton 모바일에서 적절한 크기
- [ ] 캐릭터 선택, 플레이 버튼 등 모든 인터랙션 정상 작동

**Implementation Note**: 이 단계 완료 후 모든 디바이스/방향 조합에서 수동 테스트가 필요합니다.

---

## 테스트 전략

### 자동화 테스트:
- TypeScript 타입 체크
- Next.js 린트
- 프로덕션 빌드 성공

### 수동 테스트 단계:
1. **Chrome DevTools** 모바일 시뮬레이터로 다음 뷰포트 테스트:
   - iPhone SE (375x667) portrait → 수직 스택
   - iPhone SE (667x375) landscape → grid 레이아웃
   - iPhone 14 Pro (393x852) portrait → 수직 스택
   - iPhone 14 Pro (852x393) landscape → grid 레이아웃
   - iPad (768x1024) portrait → grid 레이아웃 (≥768px)
   - Desktop (1280x720) → grid 레이아웃 (현재와 동일)

2. **기능 확인**:
   - 캐릭터 선택 → 캐릭터 이미지 변경
   - PLAY 버튼 → /game 라우팅
   - 지갑 연결 다이얼로그 열기/닫기
   - 프로필 카드 로그아웃

3. **엣지 케이스**:
   - 브라우저 창 크기 동적 변경 시 레이아웃 전환
   - 매우 긴 사용자 이름 (truncate 동작)
   - CharacterSelector에서 캐릭터 수 많을 때 스크롤

## 성능 고려 사항

- CSS-only 레이아웃 전환 (`@media` 쿼리) → JS 오버헤드 없음
- 기존 애니메이션 (`animate-float`, `animate-scale-pulse`) 유지
- 배경 이미지 로딩 전략 변경 없음

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-16-mobile-responsive-orientation-strategy.md`
- 로비 구현: `apps/web/src/components/lobby/LobbyContent.tsx`
- Tailwind v4 커스텀 변형: `@custom-variant` directive
- 초기 아키텍처: `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md`
