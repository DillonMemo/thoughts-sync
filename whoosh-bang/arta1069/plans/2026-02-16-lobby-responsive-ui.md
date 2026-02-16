---
date: 2026-02-16T21:00:00+0900
author: arta1069
git_commit: 726fe8601543e0ed40363062f06ef43b8a712b9c
branch: main
repository: Whoosh-Bang
topic: "로비 반응형 UI + 게임 모바일 조작 구현 계획"
tags: [plan, lobby, responsive, mobile, tailwind, flex, grid, game, touch-controls, phaser]
status: draft
---

# 로비 반응형 UI + 게임 모바일 조작 구현 계획

## 개요

두 가지 작업을 포함한다:

1. **로비 반응형 UI**: `LobbyContent.tsx`를 absolute에서 flex/grid로 재구성하여 PC/모바일 모두 자연스러운 레이아웃 제공
2. **게임 모바일 조작 UI**: 터치 기기에서 캐릭터 이동/점프를 위한 가상 버튼 추가

- **모바일 세로모드 로비**: 수직 스택 (ProfileCard → CharacterDisplay → CharacterSelector → PlayButton)
- **가로모드 / 데스크탑(≥768px) 로비**: 현재 PC 레이아웃 유지 (CSS Grid)
- **게임 화면 (터치 기기)**: 우하단에 [←] [→] [JUMP] 가상 버튼 표시

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

- Screen Orientation API / CSS transform rotate 적용
- ProfileCard 내부 구조 변경 (반응형 wrapper만 조정)
- 게임 세로모드 오버레이 변경 (현재 유도 방식 유지)
- 가상 조이스틱 (아날로그 스틱 방식) - 단순 버튼 방식 채택
- 에이밍 UI 변경 (터치 드래그는 Phaser 자동 변환으로 이미 작동)

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

## 3단계: 게임 모바일 터치 조작 UI 추가

### 개요
터치 기기에서 캐릭터 이동(좌/우)과 점프가 불가능한 문제를 해결한다. Phaser 게임 내에 가상 버튼을 추가하고, InputController가 키보드와 가상 버튼 입력을 동시에 처리하도록 수정한다.

### 현재 문제
- 이동: `A`/`D` 키보드 전용 → 모바일에서 **불가능**
- 점프: `W` 키보드 전용 → 모바일에서 **불가능**
- 에이밍: 마우스 드래그 → Phaser가 터치를 pointer로 변환하므로 **작동함**
- 무기 선택: 키보드 `1/2/3` + 클릭 → 클릭(터치) **작동함**

### 게임 화면 모바일 레이아웃
```
┌──────────────────────────────────────────────┐
│  Round 1                                      │
│                                               │
│              (터치 드래그 에이밍 영역)           │
│                                               │
│                                    Wind: → 3  │
│  [P1 ██████ MOVE]                             │
│  [🔫][🚀][💣]                   [←] [→]      │
│  (WeaponSelector)                [JUMP]       │
└──────────────────────────────────────────────┘
   기존 좌하단 UI                   신규 우하단
```

### 필요한 변경:

#### 1. MobileTouchControls.ts - 신규 파일 생성
**파일**: `packages/game-core/src/ui/MobileTouchControls.ts`
**변경**: 가상 버튼 UI 클래스 신규 생성

```typescript
import * as Phaser from "phaser"

export class MobileTouchControls {
  private scene: Phaser.Scene
  private isDestroyed: boolean = false

  // 버튼 상태
  private _isLeftDown: boolean = false
  private _isRightDown: boolean = false
  private _jumpPressed: boolean = false
  private _jumpConsumed: boolean = false

  // UI 요소
  private leftBtn!: Phaser.GameObjects.Container
  private rightBtn!: Phaser.GameObjects.Container
  private jumpBtn!: Phaser.GameObjects.Container

  // 레이아웃 상수 (1280x720 게임 좌표 기준)
  private readonly BTN_SIZE = 56
  private readonly BTN_ALPHA = 0.15
  private readonly BTN_ALPHA_PRESSED = 0.4
  private readonly AREA_RIGHT = 1260  // 우측 기준점
  private readonly AREA_BOTTOM = 710  // 하단 기준점

  constructor(scene: Phaser.Scene) {
    this.scene = scene
    this.createButtons()
  }

  private createButtons(): void {
    const r = this.BTN_SIZE / 2
    const bottom = this.AREA_BOTTOM
    const right = this.AREA_RIGHT

    // [←] 버튼: 우하단 영역 좌측
    this.leftBtn = this.createButton(
      right - this.BTN_SIZE * 2 - 12,
      bottom - this.BTN_SIZE - 8,
      "◀",
      () => { this._isLeftDown = true },
      () => { this._isLeftDown = false }
    )

    // [→] 버튼: 우하단 영역 우측
    this.rightBtn = this.createButton(
      right,
      bottom - this.BTN_SIZE - 8,
      "▶",
      () => { this._isRightDown = true },
      () => { this._isRightDown = false }
    )

    // [JUMP] 버튼: ←→ 사이 아래
    this.jumpBtn = this.createButton(
      right - this.BTN_SIZE - 6,
      bottom,
      "▲",
      () => { this._jumpPressed = true; this._jumpConsumed = false },
      () => { this._jumpPressed = false }
    )
  }

  private createButton(
    x: number,
    y: number,
    label: string,
    onDown: () => void,
    onUp: () => void
  ): Phaser.GameObjects.Container {
    const r = this.BTN_SIZE / 2

    // 배경 원
    const bg = this.scene.add.circle(0, 0, r, 0xffffff, this.BTN_ALPHA)
    bg.setStrokeStyle(2, 0xffffff, 0.3)

    // 라벨
    const text = this.scene.add.text(0, 0, label, {
      fontSize: "22px",
      color: "#ffffff",
    }).setOrigin(0.5)

    // 히트 영역 (투명)
    const hitArea = this.scene.add.circle(0, 0, r)
    hitArea.setInteractive()
    hitArea.setAlpha(0.001)

    hitArea.on("pointerdown", () => {
      bg.setFillStyle(0xffffff, this.BTN_ALPHA_PRESSED)
      onDown()
    })
    hitArea.on("pointerup", () => {
      bg.setFillStyle(0xffffff, this.BTN_ALPHA)
      onUp()
    })
    hitArea.on("pointerout", () => {
      bg.setFillStyle(0xffffff, this.BTN_ALPHA)
      onUp()
    })

    const container = this.scene.add.container(x, y, [bg, text, hitArea])
    container.setDepth(60)
    return container
  }

  // === Public getters ===

  get isLeftDown(): boolean { return this._isLeftDown }
  get isRightDown(): boolean { return this._isRightDown }

  /** 점프는 "한 번만 트리거" 패턴. consume 호출 후 다시 누를 때까지 false */
  consumeJump(): boolean {
    if (this._jumpPressed && !this._jumpConsumed) {
      this._jumpConsumed = true
      return true
    }
    return false
  }

  destroy(): void {
    if (this.isDestroyed) return
    this.isDestroyed = true
    this.leftBtn?.destroy()
    this.rightBtn?.destroy()
    this.jumpBtn?.destroy()
  }
}
```

**설계 결정:**
- 버튼 크기 56px: 모바일 터치 최소 권장 크기(44px)보다 여유있게 설정
- `pointerout` 이벤트: 손가락이 버튼 밖으로 미끄러질 때 해제
- 점프 `consumeJump()` 패턴: `JustDown` 동작 모방 (프레임마다 점프하지 않도록)
- depth 60: WeaponSelector(depth 50-53) 위에 표시
- 반투명 원형 버튼: 게임 화면을 가리지 않음

#### 2. InputController.ts - 가상 버튼 통합
**파일**: `packages/game-core/src/systems/InputController.ts`
**변경**: 터치 기기 감지, MobileTouchControls 생성, 이동/점프에 가상 버튼 상태 통합

**import 추가:**
```typescript
import { MobileTouchControls } from "../ui/MobileTouchControls"
```

**프로퍼티 추가:**
```typescript
private touchControls: MobileTouchControls | null = null
```

**constructor 변경 (라인 27-40):**

Before:
```typescript
constructor(scene: Phaser.Scene) {
  this.scene = scene

  if (scene.input.keyboard) {
    this.keyW = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.W)
    this.keyA = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.A)
    this.keyS = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.S)
    this.keyD = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.D)
  }

  this.setupMouseInput()
}
```

After:
```typescript
constructor(scene: Phaser.Scene) {
  this.scene = scene

  if (scene.input.keyboard) {
    this.keyW = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.W)
    this.keyA = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.A)
    this.keyS = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.S)
    this.keyD = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.D)
  }

  // 터치 기기에서 가상 버튼 생성
  if (scene.sys.game.device.input.touch) {
    this.touchControls = new MobileTouchControls(scene)
  }

  this.setupMouseInput()
}
```

**handleMovement 변경 (라인 146-158):**

Before:
```typescript
private handleMovement(character: Character): void {
  if (character.getMoveGauge() <= 0) {
    character.stopMoving()
    return
  }

  if (this.keyA.isDown) {
    character.moveLeft()
  } else if (this.keyD.isDown) {
    character.moveRight()
  } else {
    character.stopMoving()
  }
}
```

After:
```typescript
private handleMovement(character: Character): void {
  if (character.getMoveGauge() <= 0) {
    character.stopMoving()
    return
  }

  const leftDown = this.keyA?.isDown || this.touchControls?.isLeftDown
  const rightDown = this.keyD?.isDown || this.touchControls?.isRightDown

  if (leftDown) {
    character.moveLeft()
  } else if (rightDown) {
    character.moveRight()
  } else {
    character.stopMoving()
  }
}
```

**handleJump 변경 (라인 161-165):**

Before:
```typescript
private handleJump(character: Character): void {
  if (Phaser.Input.Keyboard.JustDown(this.keyW)) {
    character.jump()
  }
}
```

After:
```typescript
private handleJump(character: Character): void {
  const keyboardJump = this.keyW && Phaser.Input.Keyboard.JustDown(this.keyW)
  const touchJump = this.touchControls?.consumeJump()

  if (keyboardJump || touchJump) {
    character.jump()
  }
}
```

**destroy 변경 (라인 195-199):**

Before:
```typescript
destroy(): void {
  this.scene.input.off("pointerdown")
  this.scene.input.off("pointermove")
  this.scene.input.off("pointerup")
}
```

After:
```typescript
destroy(): void {
  this.scene.input.off("pointerdown")
  this.scene.input.off("pointermove")
  this.scene.input.off("pointerup")
  this.touchControls?.destroy()
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `cd packages/game-core && npx tsc --noEmit`
- [ ] 빌드 성공: `cd apps/web && npm run build`

#### 수동 검증:
- [ ] **PC 브라우저**: 기존 WASD + 마우스 조작이 변함없이 작동 (가상 버튼 미표시)
- [ ] **모바일(터치 기기)**: 우하단에 [←] [→] [JUMP] 버튼 표시
- [ ] **[←] [→] 버튼**: 터치 시 캐릭터 좌/우 이동, 떼면 정지
- [ ] **[JUMP] 버튼**: 터치 시 1회 점프, 꾹 눌러도 연속 점프 안 됨
- [ ] **이동 게이지**: 가상 버튼으로 이동 시에도 게이지 정상 소모
- [ ] **에이밍**: 가상 버튼 영역 외 화면 터치 드래그 시 에이밍 정상 작동
- [ ] **버튼 시각 피드백**: 누를 때 밝아지고, 놓으면 원래 투명도
- [ ] **기존 WeaponSelector**: 좌하단 무기 선택 터치 정상 작동 (충돌 없음)

**Implementation Note**: Chrome DevTools의 "Toggle device toolbar" + touch simulation으로 기본 테스트 가능하나, 실제 모바일 기기에서의 멀티터치 검증이 필요합니다.

---

## 테스트 전략

### 자동화 테스트:
- TypeScript 타입 체크 (`apps/web`, `packages/game-core`)
- Next.js 린트
- 프로덕션 빌드 성공

### 수동 테스트 단계:

#### 로비 반응형 (1-2단계):
1. **Chrome DevTools** 모바일 시뮬레이터로 다음 뷰포트 테스트:
   - iPhone SE (375x667) portrait → 수직 스택
   - iPhone SE (667x375) landscape → grid 레이아웃
   - iPhone 14 Pro (393x852) portrait → 수직 스택
   - iPhone 14 Pro (852x393) landscape → grid 레이아웃
   - iPad (768x1024) portrait → grid 레이아웃 (≥768px)
   - Desktop (1280x720) → grid 레이아웃 (현재와 동일)

2. **로비 기능 확인**:
   - 캐릭터 선택 → 캐릭터 이미지 변경
   - PLAY 버튼 → /game 라우팅
   - 지갑 연결 다이얼로그 열기/닫기
   - 프로필 카드 로그아웃

3. **엣지 케이스**:
   - 브라우저 창 크기 동적 변경 시 레이아웃 전환
   - 매우 긴 사용자 이름 (truncate 동작)
   - CharacterSelector에서 캐릭터 수 많을 때 스크롤

#### 게임 모바일 조작 (3단계):
4. **PC에서 회귀 테스트**:
   - WASD 이동/점프 정상 작동
   - 마우스 드래그 에이밍/발사 정상 작동
   - 가상 버튼이 표시되지 않음 (터치 미지원 기기)

5. **모바일/터치 기기 테스트** (Chrome DevTools touch simulation 또는 실기기):
   - [←] [→] 버튼으로 좌/우 이동
   - [JUMP] 버튼으로 점프
   - 이동 게이지 정상 소모/표시
   - 가상 버튼 영역 외 드래그 시 에이밍 작동
   - WeaponSelector(좌하단) 터치로 무기 변경

6. **멀티터치 시나리오** (실기기 필수):
   - [→] 누른 채 [JUMP] 동시 터치 → 이동하며 점프
   - 이동 중 다른 손가락으로 에이밍 드래그

## 성능 고려 사항

- CSS-only 레이아웃 전환 (`@media` 쿼리) → JS 오버헤드 없음
- 기존 애니메이션 (`animate-float`, `animate-scale-pulse`) 유지
- 배경 이미지 로딩 전략 변경 없음
- 가상 버튼: Phaser GameObjects 3개 + Container → 성능 영향 무시 가능
- 터치 감지: `scene.sys.game.device.input.touch` 한 번 체크 (매 프레임 검사 아님)

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-16-mobile-responsive-orientation-strategy.md`
- 로비 구현: `apps/web/src/components/lobby/LobbyContent.tsx`
- 게임 입력: `packages/game-core/src/systems/InputController.ts`
- 게임 UI: `packages/game-core/src/ui/` (WeaponSelector, GameHUD)
- 컨트롤러 인터페이스: `packages/game-core/src/systems/IPlayerController.ts`
- Tailwind v4 커스텀 변형: `@custom-variant` directive
- 초기 아키텍처: `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md`
