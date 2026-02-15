---
date: 2026-02-13T20:44:20+0900
researcher: arta1069@gmail.com
git_commit: N/A (no git repository initialized)
branch: N/A
repository: game
topic: "AI 플레이어 시스템 - 턴별 학습 AI 설계를 위한 코드베이스 연구"
tags: [research, codebase, ai-player, turn-system, input-controller, game-scene, learning-ai, shot-memory, trajectory-simulation]
status: complete
last_updated: 2026-02-13
last_updated_by: arta1069@gmail.com
---

# 연구: AI 플레이어 시스템 — 턴별 학습 AI 설계

**날짜**: 2026-02-13T20:44:20+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: N/A (git 저장소 미초기화)
**브랜치**: N/A
**리포지토리**: game

## 연구 질문

Play 버튼으로 시작하는 1:1 매치에서, Player 2를 턴별로 학습하는 AI 플레이어로 대체하려면 코드베이스의 어떤 부분을 어떻게 변경해야 하는가?

## 요약

### 현재 상태

현재 코드베이스에는 AI 플레이어 구현이 전혀 없다. 게임은 2인 로컬 멀티플레이어로, 두 플레이어 모두 동일한 `InputController` 하나가 키보드/마우스 입력을 처리한다. GameScene은 `this.inputController`라는 단일 인스턴스로 모든 턴을 제어한다.

### 사용자 요구사항

- **턴별 학습 AI**: 게임 시작 시 "초보자"로 시작하여, 매 턴 발사 결과를 피드백으로 받아 턴이 증가할수록 능숙해짐 (정적 난이도가 아닌 동적 성장)
- **동일 물리**: AI는 Human과 동일한 Character, Projectile, 중력, 바람 시스템을 사용
- **확장 비전**: 1:1 → 1:1:1 → 1:1:1:1 (최대 4명 개인전), 빈 슬롯은 AI로 대체

### 설계 결론

| 결정 사항 | 채택 방향 |
|----------|----------|
| **아키텍처** | `IPlayerController` 인터페이스 — InputController와 AIController가 동일 인터페이스 구현 |
| **플레이어 구분** | `GameScene.controllers[]` 배열, `instanceof`로 분기 (Character에 isAI 플래그 불필요) |
| **학습 시스템** | ShotMemory + 경험치 기반 오차 감소: `errorMult = 1/(1 + exp * 0.2)` |
| **궤적 계산** | 물리 시뮬레이션 반복 탐색 (angle×power 그리드 → simulateLanding) |
| **AI 턴 흐름** | 내부 상태 머신: thinking→moving→weaponSelect→aiming→fire (~2초) |
| **이동 전략** | 조건부 이동 (물/가장자리 회피만, 그 외 바로 발사) |
| **무기 선택** | 거리 기반 (근거리 Shotgun, 원거리 Bazooka) |
| **시각화** | AI는 `AimingUI.update()`, Human은 `AimingUI.updateWithMousePos()` |
| **N-player 확장** | controllers[] 배열 + TurnSystem의 `% playerCount` 순환으로 자연 대응 |

---

## 1. 현재 코드베이스 분석

### 1.1 게임 시작 플로우

```
PlayButton.tsx → game/page.tsx (인증+검증) → PhaserGame.tsx → createGame()
  → BootScene (에셋) → GameScene.create() (게임 초기화)
```

- `index.ts:36-39` — Phaser Registry에 `player1CharacterType` 저장
- `GameScene.ts:65-98` — create() 파이프라인: 지형→플레이어→시스템→UI→게임시작

### 1.2 Player 2 하드코딩

Player 2는 항상 `characterType: "player"`로 하드코딩:
- `GameScene.ts:284-292` — Player 2 생성 (`playerId: 1`, 캐릭터 타입 고정)
- `GameScene.ts:269-270` — Player 1만 Registry에서 캐릭터 타입 읽기

### 1.3 턴 시스템

**상태 머신** (`TurnSystem.ts:4-10`):
```
idle → playerTurn → aiming → firing → animating → playerTurn (다음 플레이어)
```

- `TurnSystem.ts:67-68` — 턴 순환: `(currentPlayerIndex + 1) % playerCount`
- `GameScene.ts:356` — 턴 제한: 30초
- `GameScene.ts:540` — 턴 종료 시 바람 랜덤화

TurnSystem은 현재 플레이어 인덱스만 추적하며, Human/AI 구분이 없다.

### 1.4 입력 시스템

**InputController** (`InputController.ts:5-188`):
- WASD: 이동/점프 | 마우스 좌클릭 드래그: 조준+파워 | 우클릭: 취소

**GameScene에서의 사용 (단일 인스턴스):**
- `GameScene.ts:30` — `private inputController!: InputController`
- `GameScene.ts:361` — `this.inputController = new InputController(this)`
- `GameScene.ts:430` — `this.inputController.update(currentPlayer, this.turnSystem)`

### 1.5 무기 시스템

| 무기 | 데미지 | 폭발 반경 | 탄약 | 특성 |
|------|--------|-----------|------|------|
| Bazooka | 50 | 40px | ∞ | 충돌 즉시 폭발 |
| Grenade | 35 | 50px | 3발 | 2.5초 퓨즈, 바운스 |
| Shotgun | 25/pellet | 15px | 2발 | 5발 산탄 |

- `WeaponTypes.ts:11-39` — 무기 스탯 레지스트리
- `WeaponManager.ts:26-37` — 선택 로직
- `WeaponManager.ts:51-109` — 발사 로직

### 1.6 물리/환경

AI 궤적 계산에 필요한 상수 (코드베이스에서 추출):

| 상수 | 값 | 출처 |
|------|-----|------|
| 초기 속도 | `power * 0.15` | `Projectile.ts:26` |
| 중력 스케일 | `1` | `Projectile.ts:27` |
| 바람 수평력 | `windForce * 0.00000375` / frame | `Projectile.ts:102` |
| 공기 저항 | `0.001` | `Projectile.ts:50` |
| 바람 범위 | -5 ~ +5, 매 턴 랜덤 | `WindSystem.ts:6-14` |
| 물 (즉사) | 화면 높이 88% | `DamageSystem.ts:30` |

초기 속도 계산 (`Projectile.ts:35-38`):
```
vx = cos(angle_rad) * power * 0.15
vy = -sin(angle_rad) * power * 0.15
```

### 1.7 데미지 시스템

- 거리 기반 감쇠: `1 - distance / effectiveRadius` (`DamageSystem.ts:95`)
- 자폭: 유효 반경 ×1.5, 최소 25% 데미지 (`DamageSystem.ts:90-98`)
- 넉백: `damage * 0.001 * distanceFalloff` (`DamageSystem.ts:139-142`)
- 게임 오버: 생존자 ≤1명 (`DamageSystem.ts:149-158`)

### 1.8 이벤트 시스템

AI 관련 이벤트:
- `TURN_CHANGED` — AI 턴 시작 감지
- `WIND_CHANGED` — 조준 계산용 바람 값
- `WEAPON_SELECTED` — 무기 선택 반영
- `HEALTH_CHANGED` — 전략 판단
- `GAME_OVER` — AI 턴 중단

---

## 2. AI 설계 방향

### 2.1 IPlayerController 인터페이스

InputController의 public 메서드를 기반으로 추출한 공통 인터페이스:

```
IPlayerController (interface)
  ├── InputController (기존, Human)
  └── AIController (신규, AI + ShotMemory)
```

| 메서드 | 반환 타입 | 설명 | 현재 위치 |
|--------|----------|------|----------|
| `update(character, turnSystem)` | `void` | 매 프레임 호출 | `InputController.ts:107` |
| `getAimAngle()` | `number` | 조준 각도 (-30~90) | `InputController.ts:161` |
| `getAimPower()` | `number` | 발사 파워 (10~100) | `InputController.ts:165` |
| `getAimStartPos()` | `{x,y} \| null` | 조준 시작점 | `InputController.ts:169` |
| `isCurrentlyAiming()` | `boolean` | 조준 중 여부 | `InputController.ts:173` |
| `isAimingRight()` | `boolean` | 우측 조준 여부 | `InputController.ts:177` |
| `resetAim()` | `void` | 조준 리셋 | `InputController.ts:181` |
| `onShotResult?(landingPos, hit)` | `void` | 발사 결과 피드백 (AI 전용) | 신규 |
| `destroy?()` | `void` | 리소스 정리 | 신규 |

**장점:**
- InputController 기존 코드 변경 최소화 (`implements` 선언만 추가)
- GameScene에서 `this.inputController` → `this.controllers[playerIdx]`로 교체
- N-player 확장 시 `controllers: IPlayerController[]` 배열로 자연 대응

### 2.2 AI 내부 상태 머신

```
idle → thinking → [moving] → weaponSelect → aimStart → aiming → fire → done
```

| 페이즈 | 시간 | 동작 |
|--------|------|------|
| thinking | ~600ms | 대상 선택, 무기, 궤적 계산, 이동 판단 |
| moving | 가변 | 물/가장자리 회피 (불필요하면 스킵) |
| weaponSelect | ~300ms | 무기 선택 |
| aimStart | 즉시 | `turnSystem.startAiming()` 호출 |
| aiming | ~900ms | 각도/파워를 목표값까지 easeInOutCubic 보간 |
| fire | ~200ms | 조준 안정화 후 `turnSystem.fire()` |

**총 ~2초** (이동 없을 때). 30초 턴 타이머가 안전장치.

### 2.3 턴별 학습 시스템 (ShotMemory)

#### 발사 기록

```typescript
interface ShotRecord {
  turnNumber: number
  targetPos: { x: number; y: number }
  aimAngle: number
  aimPower: number
  wind: number
  landingPos: { x: number; y: number }
  errorDistance: number  // 대상과의 오차 (px)
  hit: boolean
}
```

#### 경험치 적립

```
experience = 0 (게임 시작)

매 턴 종료 시:
  experience += 1                           // 기본
  if (hit) experience += 2                  // 명중 보너스
  if (errorDistance < 50) experience += 1    // 근접 보너스
```

#### 오차 감소 공식

```
errorMultiplier = 1 / (1 + experience * 0.2)
```

| AI 차례 | experience (예상) | errorMultiplier | 의미 |
|---------|-------------------|-----------------|------|
| 1번째 | 0 | 1.00 | 초보 (±20도, ±25파워) |
| 2번째 | 1 | 0.83 | 약간 개선 |
| 3번째 | 3~4 | 0.56~0.63 | 중간 |
| 4번째 | 5~8 | 0.33~0.50 | 능숙 |
| 5번째+ | 8~12+ | 0.19~0.29 | 거의 정확 |

기본 오차: 각도 ±20도, 파워 ±25 → `실제 오차 = BASE × errorMult × random(-1,1)`

#### 발사 보정 (Shot Correction)

직전 착탄 결과로 다음 발사를 보정:
```
correctionRate = min(0.5, experience * 0.08)
angleCorrection = -verticalError * correctionRate * 0.1
powerCorrection = -horizontalError * correctionRate * 0.05
```

**유저 체감**: 초반 "못하네" → 중반 "점점 잘하는데?" → 후반 "무섭다"

### 2.4 궤적 계산 알고리즘

해석적 풀이 불가 (바람+중력+공기저항 조합) → **물리 시뮬레이션 반복 탐색**:

```
1. 대략 탐색: angle(-20~85, step 5) × power(15~95, step 5) = ~357 조합
2. 각 조합 → simulateLanding()으로 착탄점 추정
3. 대상과 오차 최소인 조합 선택
4. 미세 조정: 최적 주변 ±4에서 step 1로 재탐색
5. 학습 보정 (applyShotCorrection) + 경험 기반 랜덤 오차 적용
```

`simulateLanding()`은 `terrainData.isSolid(x, y)`로 지형 충돌 판정, 최대 300프레임(~5초) 시뮬레이션.

### 2.5 의사결정 로직

- **대상 선택**: 살아있는 상대 중 가장 가까운 적
- **이동 판단**: 물(88%) 근처 또는 맵 가장자리(80px 이내)면 안쪽 이동, 그 외 이동 안 함
- **무기 선택**: 거리 < 150px & Shotgun 탄약 → Shotgun, 그 외 → Bazooka
- **AimingUI 분기**: Human은 `updateWithMousePos()` (마우스 가이드), AI는 `update()` (조준선만)

### 2.6 N-Player 확장 대비

| 컴포넌트 | 현재 (2인) | N-player 확장 시 |
|----------|-----------|------------------|
| `TurnSystem` | `playerCount: 2` | `playerCount: N` (이미 `% playerCount` 순환) |
| `controllers[]` | `[Human, AI]` | `[Human, AI, AI, AI]` 자유 배정 |
| `DamageSystem` | forEach 순회 | 동일 (N명 자동 대응) |
| `checkGameOver()` | `alive ≤ 1` | 동일 |
| `findSpawnPositions()` | 2개 반환 | N개 균등 배치 알고리즘 필요 |

---

## 3. GameScene 수정 지점 매핑

### 3.1 inputController → controllers[] 전환

| 위치 | 현재 | 변경 방향 |
|------|------|----------|
| `:30` | `private inputController!: InputController` | `private controllers: IPlayerController[] = []` |
| `:361` | `new InputController(this)` | 플레이어별 Human/AI 컨트롤러 생성 |
| `:430` | `this.inputController.update(...)` | `controllers[idx].update(...)` |
| `:459` | `this.inputController.isAimingRight()` | `controller.isAimingRight()` |
| `:463` | `this.inputController.getAimAngle()` | `controller.getAimAngle()` |
| `:473` | `this.inputController.getAimPower()` | `controller.getAimPower()` |
| `:506` | `getAimAngle()` (handleFire) | `controller.getAimAngle()` |
| `:519` | `getAimPower()` (handleFire) | `controller.getAimPower()` |
| `:532` | `resetAim()` | `controller.resetAim()` |

### 3.2 기타 수정 지점

- **handleFire()**: 시그니처에 `controller: IPlayerController` 추가
- **onAnimationComplete()** (`:538-542`): AI 학습 피드백 — `controller.onShotResult?()` 호출
  - 이를 위해 WeaponManager에 `lastExplosionPos` 추적, DamageSystem에 `wasLastShotHit()` 추가 필요
- **selectWeaponForCurrentPlayer()**: AI 턴 중 키보드 무기 선택 차단 (`instanceof InputController` 가드)
- **shutdown()** (`:575-601`): `controllers.forEach(c => c.destroy?.())` 추가

### 3.3 AIController 생성자 의존성

```typescript
constructor(config: {
  scene: Phaser.Scene
  terrainData: TerrainData       // isSolid() — 궤적 시뮬레이션
  weaponManager: WeaponManager   // selectWeapon() — 무기 선택
  opponents: Character[]         // 대상 위치 — 조준 계산
  windSystem: WindSystem         // getWindForce() — 바람 보정 (WindSystem.ts:18-20)
})
```

---

## 4. 파일 변경 요약

| 파일 | 변경 | 규모 |
|------|------|------|
| `systems/IPlayerController.ts` | **신규** | ~20줄 |
| `systems/AIController.ts` | **신규** (학습 시스템 포함) | ~350줄 |
| `systems/InputController.ts` | `implements IPlayerController` + `destroy()` | ~5줄 |
| `index.ts` (game-core) | GameOptions 확장, exports | ~10줄 |
| `scenes/GameScene.ts` | 컨트롤러 배열화, 피드백, AimingUI 분기 | ~50줄 변경 |
| `weapons/WeaponManager.ts` | `lastExplosionPos` 추적 | ~15줄 |
| `systems/DamageSystem.ts` | `wasLastShotHit()` | ~10줄 |
| `PhaserGame.tsx` | createGame 옵션 | ~2줄 |

---

## 5. 참조

### 히스토리 컨텍스트

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` — Utility AI 아키텍처, 탄도 계산
- `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md` — "AI 플레이어는 Phase 2" (Line 168)

### 관련 연구

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` — 원본 아키텍처 연구
- `thoughts/shared/research/2026-02-13-character-skin-nft-system-research.md` — 캐릭터 시스템

### 구현 계획

- **Plan 파일**: `.claude/plans/recursive-stirring-wreath.md` — 8단계 구현 절차 포함
