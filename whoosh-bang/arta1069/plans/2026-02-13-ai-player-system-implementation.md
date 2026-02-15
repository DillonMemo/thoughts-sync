# AI 플레이어 시스템 구현 계획

## 개요

Play 버튼으로 시작하는 1:1 매치에서, Player 2를 **턴별 학습 AI**로 대체한다.
AI는 게임 시작 시 초보자로 출발하여 매 턴 발사 결과를 피드백으로 받아 점진적으로 능숙해진다.
`IPlayerController` 인터페이스로 Human/AI를 추상화하여 N-player 확장에 대비한다.

## 현재 상태 분석

### 존재하는 것
- 2인 로컬 멀티플레이어, 단일 `InputController`로 모든 턴 처리
- TurnSystem: `(currentPlayerIndex + 1) % playerCount` 순환 (N-player 대응 구조 이미 존재)
- 완성된 물리 시스템: Matter.js (중력, 바람, 공기저항)
- 무기 3종: Bazooka (무제한), Grenade (3발), Shotgun (2발)

### 존재하지 않는 것
- AI 플레이어 구현 일체
- `IPlayerController` 인터페이스
- 플레이어별 컨트롤러 분리 (controllers[] 배열)
- 발사 결과 피드백 시스템 (폭발 위치/명중 추적)

### 핵심 발견:
- `GameScene.ts:28` — `private inputController!: InputController` 단일 인스턴스
- `GameScene.ts:287` — Player 2는 항상 `characterType: "player"` 하드코딩
- `GameScene.ts:428` — `this.inputController.update(currentPlayer, this.turnSystem)` 매 프레임
- `GameScene.ts:466-472` — AimingUI는 `updateWithMousePos()` 사용 (마우스 가이드 라인 포함)
- `AimingUI.ts:40` — `update(character, angle, power)` 메서드도 존재 (마우스 가이드 없이 조준선만)
- `DamageSystem.ts:33` — `"explosion"` scene 이벤트로 폭발 데이터 수신
- `TurnSystem.ts:49-53` — `startAiming()`, `fire()` 메서드가 상태 전환 반환값 제공
- `InputController`에 `destroy()` 메서드 없음

## 원하는 최종 상태

- Play 버튼 → 즉시 Human vs AI 대전 시작 (UI 변경 없음)
- AI가 매 턴 물리 시뮬레이션 기반으로 궤적을 계산하여 발사
- 초반 "못하네" → 중반 "점점 잘하는데?" → 후반 "무섭다" 체감
- AI 턴 시 캐릭터 위 말풍선 표시 (thinking 페이즈)
- AI 턴 시 AimingUI는 `update()` (조준선만), Human 턴 시 `updateWithMousePos()` (마우스 가이드 포함)
- `controllers: IPlayerController[]` 배열로 N-player 확장 구조 확보

### 검증 방법
- 게임 시작 시 Player 2가 자동으로 AI로 동작
- AI가 대상을 향해 발사하고, 턴이 증가할수록 정확도 향상
- Human 플레이어의 입력/조준/발사가 기존과 동일하게 동작
- 30초 턴 타이머가 AI에도 안전장치로 작동
- 게임 오버 시 정상 종료

## 하지 않을 것

- N-player (3인, 4인전) 실제 구현 — 구조적 대비만
- 온라인 멀티플레이어 또는 네트워크 AI
- AI 난이도 선택 UI — 학습 기반 자동 성장만
- Grenade 궤적 예측 (바운스 물리가 복잡) — AI는 Bazooka/Shotgun만 사용
- 이동 AI — 물/가장자리 회피 + 지형 차단 시 재포지셔닝 구현, 전략적 포지셔닝 기본 수준

---

## 1단계: IPlayerController 인터페이스 + InputController 적용

### 개요
공통 인터페이스를 정의하고, 기존 InputController에 적용하여 다형성 기반을 마련한다.

### 필요한 변경:

#### 1. IPlayerController 인터페이스 (신규)
**파일**: `packages/game-core/src/systems/IPlayerController.ts` (신규)

```typescript
import { Character } from "../entities/Character"
import { TurnSystem } from "./TurnSystem"

export interface IPlayerController {
  /** 매 프레임 호출. 컨트롤러 로직 업데이트 */
  update(character: Character, turnSystem: TurnSystem): void

  /** 현재 조준 각도 (-30 ~ 90도) */
  getAimAngle(): number

  /** 현재 조준 파워 (10 ~ 100) */
  getAimPower(): number

  /** 조준 시작 위치 (null이면 조준 안 함) */
  getAimStartPos(): { x: number; y: number } | null

  /** 현재 조준 중인지 여부 */
  isCurrentlyAiming(): boolean

  /** 오른쪽으로 조준 중인지 여부 */
  isAimingRight(): boolean

  /** 조준 상태 리셋 */
  resetAim(): void

  /** 발사 결과 피드백 (AI 전용, Human은 no-op) */
  onShotResult?(landingPos: { x: number; y: number }, hit: boolean): void

  /** 리소스 정리 */
  destroy?(): void
}
```

#### 2. InputController 수정
**파일**: `packages/game-core/src/systems/InputController.ts`
**변경**: `implements IPlayerController` 추가 + `destroy()` 메서드 추가

```typescript
import { IPlayerController } from "./IPlayerController"

export class InputController implements IPlayerController {
  // ... 기존 코드 변경 없음 ...

  destroy(): void {
    this.scene.input.off("pointerdown")
    this.scene.input.off("pointermove")
    this.scene.input.off("pointerup")
  }
}
```

#### 3. index.ts export 추가
**파일**: `packages/game-core/src/index.ts`
**변경**: IPlayerController export 추가

```typescript
export * from "./systems/IPlayerController"
```

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 통과 (기존 코드에 영향 없음)
- [x] 기존 게임이 정상 동작 (InputController가 인터페이스를 올바르게 구현)

#### 수동 검증:
- [ ] Play 버튼 → 게임 시작 → 기존과 동일하게 플레이 가능

**Implementation Note**: 이 단계는 기존 동작에 영향을 주지 않는 순수 추가/수정. 완료 후 다음 단계 진행.

---

## 2단계: AIController 핵심 구현

### 개요
AI 상태 머신, 궤적 시뮬레이션, 기본 의사결정 로직을 구현한다. 학습 시스템은 3단계에서 별도 구현.

### 필요한 변경:

#### 1. AIController 클래스 (신규)
**파일**: `packages/game-core/src/systems/AIController.ts` (신규)

**AI 내부 상태 머신:**
```
idle → thinking → [moving] → weaponSelect → aimStart → aiming → fire → done
```

**핵심 구조:**

```typescript
import { IPlayerController } from "./IPlayerController"
import { Character } from "../entities/Character"
import { TurnSystem } from "./TurnSystem"
import { TerrainData } from "../terrain/TerrainData"
import { WeaponManager } from "../weapons/WeaponManager"
import { WindSystem } from "./WindSystem"

type AIPhase = "idle" | "thinking" | "moving" | "weaponSelect" | "aimStart" | "aiming" | "fire" | "done"

interface AIConfig {
  scene: Phaser.Scene
  terrainData: TerrainData
  weaponManager: WeaponManager
  opponents: Character[]
  windSystem: WindSystem
}

export class AIController implements IPlayerController {
  private phase: AIPhase = "idle"
  private phaseTimer: number = 0

  // 조준 상태 (IPlayerController 계약)
  private currentAimAngle: number = 45
  private currentAimPower: number = 50
  private aimingRight: boolean = true
  private aiming: boolean = false

  // 계산된 목표값
  private targetAngle: number = 45
  private targetPower: number = 50

  // 학습용 상태
  private lastTargetPos: { x: number; y: number } = { x: 0, y: 0 }

  // 의존성
  private config: AIConfig

  constructor(config: AIConfig) { ... }

  update(character: Character, turnSystem: TurnSystem): void {
    // TurnSystem 상태에 따라 AI 페이즈 진행
    const state = turnSystem.getState()
    if (state === "playerTurn" && this.phase === "idle") {
      this.startThinking(character)
    }
    // ... 페이즈별 로직 (아래 페이즈별 타이밍 표 참조) ...
  }

  /**
   * thinking 페이즈 진입: 대상 선택 → 이동 필요 판단
   * 실제 발사각/파워 계산은 이동 완료 후 onWeaponSelectComplete에서 수행
   * (이동 후 위치가 바뀌므로 최종 위치에서 재계산해야 정확)
   */
  private startThinking(character: Character): void {
    this.phase = "thinking"
    this.phaseTimer = 600

    // 1. 대상 선택
    const target = this.findTarget(character)
    if (!target) return

    // 2. 학습 피드백용 대상 정보 기록
    this.lastTargetId = target.playerId
    this.lastTargetPos = target.getPosition()
    this.aimingRight = target.getPosition().x > character.getPosition().x

    // 3. 현재 위치에서 발사 품질 평가 (이동 필요 여부 판단용)
    const { bestDist } = this.findBestShot(character, target)
    this.needsRepositioning = bestDist > 80
  }

  // onWeaponSelectComplete에서 최종 계산:
  // findBestShot → applyShotCorrection(pos 포함) → applyError 체이닝

  // IPlayerController 구현
  getAimAngle(): number { return this.currentAimAngle }
  getAimPower(): number { return this.currentAimPower }
  getAimStartPos(): { x: number; y: number } | null { ... }
  isCurrentlyAiming(): boolean { return this.aiming }
  isAimingRight(): boolean { return this.aimingRight }
  resetAim(): void { ... }
  destroy(): void { ... }
}
```

**페이즈별 타이밍:**

| 페이즈 | 시간 | 동작 |
|--------|------|------|
| thinking | ~600ms | 대상 선택, 궤적 계산, 이동 판단 |
| moving | 가변 | 물/가장자리 회피 (불필요시 스킵) |
| weaponSelect | ~300ms | 무기 선택 |
| aimStart | 즉시 | `turnSystem.startAiming()` 호출 |
| aiming | ~900ms | 각도/파워를 목표값까지 easeInOutCubic 보간 |
| fire | ~200ms | 조준 안정화 후 발사 완료 대기 |

#### 2. 궤적 시뮬레이션 (`simulateLanding`)

AI가 각도×파워 조합의 착탄점을 예측하는 핵심 함수:

```typescript
// Matter.js Verlet 적분 매칭 상수
private static readonly SIM_GRAVITY = 0.001 * (1000 / 60) * (1000 / 60)  // ≈0.278
private static readonly SIM_FRICTION = 0.001  // frictionAir
private static readonly SIM_WIND_SCALE = (0.000001406 * 277.78) / 0.5    // ≈0.00078
private static readonly SPAWN_OFFSET = 35  // handleFire의 spawnDistance와 동일

private simulateLanding(
  startX: number, startY: number,
  angle: number, power: number,
  wind: number
): { x: number; y: number } | null {
  const radians = angle * (Math.PI / 180)
  const cosA = Math.cos(radians)
  const sinA = Math.sin(radians)

  // 발사 위치 오프셋 (handleFire와 동일)
  let x = startX + cosA * AIController.SPAWN_OFFSET
  let y = startY - sinA * AIController.SPAWN_OFFSET
  let vx = cosA * power * 0.15
  let vy = -sinA * power * 0.15

  const windAccel = wind * AIController.SIM_WIND_SCALE
  const frictionMult = 1 - AIController.SIM_FRICTION

  for (let step = 0; step < 300; step++) {
    vx += windAccel
    vy += AIController.SIM_GRAVITY
    vx *= frictionMult
    vy *= frictionMult
    x += vx
    y += vy

    if (this.config.terrainData.isSolid(Math.floor(x), Math.floor(y))) {
      return { x, y }
    }
    if (y > this.config.scene.scale.height * 0.91 + 100) return null
    if (x < -100 || x > this.config.scene.scale.width + 100) return null
  }
  return null
}
```

**설계 결정**: Matter.js의 Verlet 적분 상수를 직접 매칭하여 시뮬레이션 정확도를 높임.
초보자 체감은 `applyError`의 랜덤 오차로 구현하고, 시뮬레이션 자체는 가능한 정확하게 유지.

#### 3. 그리드 탐색 알고리즘

```typescript
private findBestShot(
  character: Character, target: Character
): { angle: number; power: number; bestDist: number } {
  const pos = character.getPosition()
  const targetPos = target.getPosition()
  const wind = this.config.windSystem.getWindForce()
  const facingRight = targetPos.x > pos.x

  let bestAngle = 45, bestPower = 50, bestDist = Infinity

  // 1차 탐색: 대략 (step 5)
  for (let angle = -10; angle <= 80; angle += 5) {
    for (let power = 20; power <= 100; power += 5) {
      const adjustedAngle = facingRight ? angle : 180 - angle
      const landing = this.simulateLanding(pos.x, pos.y, adjustedAngle, power, wind)
      if (!landing) continue
      const dist = Math.hypot(landing.x - targetPos.x, landing.y - targetPos.y)
      if (dist < bestDist) {
        bestDist = dist
        bestAngle = angle
        bestPower = power
      }
    }
  }

  // 2차 탐색: 미세 조정 (step 1, ±6 범위)
  for (let angle = bestAngle - 6; angle <= bestAngle + 6; angle++) {
    for (let power = bestPower - 6; power <= bestPower + 6; power++) {
      if (power < 10 || power > 100) continue
      const adjustedAngle = facingRight ? angle : 180 - angle
      const landing = this.simulateLanding(pos.x, pos.y, adjustedAngle, power, wind)
      if (!landing) continue
      const dist = Math.hypot(landing.x - targetPos.x, landing.y - targetPos.y)
      if (dist < bestDist) {
        bestDist = dist
        bestAngle = angle
        bestPower = power
      }
    }
  }

  return { angle: bestAngle, power: bestPower, bestDist }
}
```

#### 4. 의사결정 로직

```typescript
// 대상 선택: 가장 가까운 살아있는 적
private findTarget(): Character | null {
  return this.config.opponents
    .filter(c => c.isAlive())
    .sort((a, b) => /* 거리 기준 정렬 */)
    [0] ?? null
}

// 이동 판단: 물/가장자리 위험 + 지형 차단(재포지셔닝) 시 이동
private shouldMove(character: Character): boolean {
  const pos = character.getPosition()
  const waterLevel = this.config.scene.scale.height * 0.91
  const mapWidth = this.config.scene.scale.width
  const surfaceY = this.getSurfaceY(Math.floor(pos.x))
  const nearWater = pos.y > waterLevel - 100 || surfaceY > waterLevel - 60
  return nearWater || pos.x < 80 || pos.x > mapWidth - 80 || this.needsRepositioning
}

// 이동 중 점프 및 막힘 감지 (onMovingUpdate 내부):
// - 다중 거리 지형 스캔: 10px(턱 8px), 25px(경사 15px), 45px(절벽 25px) 앞 확인
// - 막힘 감지: 5프레임 이상 X좌표 변화 없으면 자동 점프
// - 방향 전환: 30프레임 이상 막혀있으면 반대 방향으로 전환

// 무기 선택: 근거리 Shotgun, 원거리 Bazooka
private selectWeapon(distance: number): void {
  if (distance < 150 && this.config.weaponManager.getAmmo("shotgun") > 0) {
    this.config.weaponManager.selectWeapon("shotgun")
  } else {
    this.config.weaponManager.selectWeapon("bazooka")
  }
}
```

#### 5. index.ts export 추가
**파일**: `packages/game-core/src/index.ts`

```typescript
export * from "./systems/AIController"
```

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 통과
- [x] AIController가 IPlayerController 인터페이스를 완전히 구현

#### 수동 검증:
- [x] AIController 인스턴스가 생성 가능 (아직 GameScene에 통합하지 않음)

**Implementation Note**: 이 단계에서는 AIController를 독립적으로 구현. GameScene 통합은 4단계에서 진행.

---

## 3단계: ShotMemory 학습 시스템

### 개요
AI가 매 턴 발사 결과를 기록하고, 경험치 기반으로 정확도를 향상시키는 시스템.

### 필요한 변경:

#### 1. ShotMemory를 AIController에 통합
**파일**: `packages/game-core/src/systems/AIController.ts` (추가)

```typescript
interface ShotRecord {
  turnNumber: number
  targetId: number            // 타겟 playerId (N-player 확장 대비)
  targetPos: { x: number; y: number }
  shooterPos: { x: number; y: number }  // 발사 시점의 AI 위치 (넉백 보정용)
  aimAngle: number
  aimPower: number
  wind: number
  landingPos: { x: number; y: number }
  errorDistance: number
  hit: boolean
}

// AIController 내부에 추가:
private shotHistory: ShotRecord[] = []
private experience: number = 0
private turnNumber: number = 0
private lastTargetId: number = -1  // 현재 턴 타겟 playerId (startThinking에서 설정)
// lastTargetPos는 2단계 핵심 구조에서 이미 선언됨 (startThinking()에서 매 턴 설정)

// 발사 시점의 값 기록 (resetAim 전에 캡처)
private lastFiredAngle: number = 45
private lastFiredPower: number = 50
private lastFiredWind: number = 0
private lastFiredPos: { x: number; y: number } = { x: 0, y: 0 }  // 넉백 후 보정 무효화용
```

#### 2. 경험치 적립

```typescript
// onShotResult() 구현 (IPlayerController 인터페이스)
onShotResult(landingPos: { x: number; y: number } | null, hit: boolean): void {
  // 유효한 착탄 데이터가 있을 때만 기록 (화면 밖 발사는 기록 제외)
  if (landingPos) {
    const record: ShotRecord = {
      turnNumber: this.turnNumber,
      targetId: this.lastTargetId,
      targetPos: this.lastTargetPos,
      shooterPos: { ...this.lastFiredPos },  // 발사 시점의 위치 기록
      aimAngle: this.lastFiredAngle,
      aimPower: this.lastFiredPower,
      wind: this.lastFiredWind,
      landingPos,
      errorDistance: Math.hypot(
        landingPos.x - this.lastTargetPos.x,
        landingPos.y - this.lastTargetPos.y
      ),
      hit,
    }
    this.shotHistory.push(record)

    if (hit) this.experience += 3             // 명중 보너스
    if (record.errorDistance < 50) this.experience += 2  // 근접 보너스
  }

  // 매 턴 기본 경험치 (화면 밖 발사여도 적립)
  this.experience += 2
  this.turnNumber++
}
```

#### 3. 오차 감소 공식

```typescript
private getErrorMultiplier(): number {
  return 1 / (1 + this.experience * 0.44)
}

private applyError(angle: number, power: number): { angle: number; power: number } {
  const errorMult = this.getErrorMultiplier()
  const BASE_ANGLE_ERROR = 12  // ±12도
  const BASE_POWER_ERROR = 15  // ±15

  const angleError = BASE_ANGLE_ERROR * errorMult * (Math.random() * 2 - 1)
  const powerError = BASE_POWER_ERROR * errorMult * (Math.random() * 2 - 1)

  return {
    angle: Math.max(-30, Math.min(90, angle + angleError)),
    power: Math.max(10, Math.min(100, power + powerError)),
  }
}
```

| AI 차례 | experience (예상) | errorMultiplier | 의미 |
|---------|-------------------|-----------------|------|
| 1번째 | 0 | 1.00 | 초보 (±12도, ±15파워) |
| 2번째 | 2 | 0.53 | 눈에 띄게 개선 |
| 3번째 | 4~7 | 0.24~0.34 | 능숙 |
| 4번째 | 6~12 | 0.15~0.26 | 정확 |
| 5번째+ | 10~16+ | 0.10~0.17 | 거의 정확 |

#### 4. 발사 보정 (Shot Correction)

**N-player 대비**: 같은 타겟을 향한 마지막 발사만 참조하여 보정한다.
다른 타겟에 대한 오차를 현재 타겟에 적용하면 역효과이므로, `targetId` 필터링이 필수.

```typescript
private applyShotCorrection(
  angle: number, power: number, currentPos: { x: number; y: number }
): { angle: number; power: number } {
  // 같은 타겟을 향한 마지막 발사 기록만 사용
  const lastShotAtTarget = this.shotHistory
    .filter(s => s.targetId === this.lastTargetId)
    .at(-1)

  if (!lastShotAtTarget) return { angle, power }

  // 위치 유사도: 넉백 등으로 크게 이동했으면 보정 신뢰도 낮음
  const posDist = Math.hypot(
    currentPos.x - lastShotAtTarget.shooterPos.x,
    currentPos.y - lastShotAtTarget.shooterPos.y
  )
  const posSimilarity = Math.max(0, 1 - posDist / 150)
  if (posSimilarity <= 0) return { angle, power }

  // 바람 유사도
  const currentWind = this.config.windSystem.getWindForce()
  const windDiff = Math.abs(currentWind - lastShotAtTarget.wind)
  const windSimilarity = Math.max(0, 1 - windDiff * 5)
  if (windSimilarity <= 0) return { angle, power }

  const correctionRate =
    Math.min(0.85, this.experience * 0.19) * windSimilarity * posSimilarity

  const verticalError = lastShotAtTarget.landingPos.y - lastShotAtTarget.targetPos.y
  const horizontalError = lastShotAtTarget.landingPos.x - lastShotAtTarget.targetPos.x

  return {
    angle: angle - verticalError * correctionRate * 0.3,
    power: power - horizontalError * correctionRate * 0.15,
  }
}
```

**보정 신뢰도 시스템:**
- **위치 유사도** (`posSimilarity`): 넉백으로 150px+ 이동 시 보정 무시, 가까울수록 비례 적용
- **바람 유사도** (`windSimilarity`): 바람 변화가 0.2+ 이면 보정 무시
- 두 유사도가 `correctionRate`에 곱셈으로 적용되어 조건이 나쁠수록 보정 약화

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 통과
- [x] `onShotResult()` 호출 시 경험치가 올바르게 증가

#### 수동 검증:
- [x] AI의 첫 발사는 부정확하고, 2~3턴 후 눈에 띄게 정확해짐
- [x] 넉백으로 크게 날아간 후에도 잘못된 보정 없이 궤적 시뮬레이션 기반으로 정상 발사

**Implementation Note**: ShotMemory는 AIController 내부에 구현. 외부에서는 `onShotResult()` 메서드만 호출.

---

## 4단계: GameScene 통합

### 개요
GameScene의 단일 `inputController`를 `controllers[]` 배열로 전환하고, AI 턴 분기 로직을 추가한다. **가장 많은 수정이 필요한 핵심 단계.**

### 필요한 변경:

#### 1. 멤버 변수 변경
**파일**: `packages/game-core/src/scenes/GameScene.ts`

```typescript
// 변경 전 (line 28):
private inputController!: InputController

// 변경 후:
private controllers: IPlayerController[] = []
```

**import 추가:**
```typescript
import { IPlayerController } from "@/systems/IPlayerController"
import { AIController } from "@/systems/AIController"
```

#### 2. createSystems() 수정 (`GameScene.ts:351-376`)

```typescript
private createSystems(): void {
  this.turnSystem = new TurnSystem(this, {
    turnDuration: 30000,
    playerCount: this.players.length,
  })

  // Player 1: Human (InputController)
  this.controllers.push(new InputController(this))

  // Player 2: AI (AIController)
  this.controllers.push(new AIController({
    scene: this,
    terrainData: this.terrainData,
    weaponManager: this.weaponManager,
    opponents: [this.players[0]],  // Player 1이 상대
    windSystem: this.windSystem,
  }))

  this.weaponManager = new WeaponManager(this)
  this.windSystem = new WindSystem()
  this.damageSystem = new DamageSystem(this, this.terrainRenderer, this.players)
  this.setupWeaponKeys()
}
```

**주의**: `WeaponManager`와 `WindSystem`이 AIController보다 먼저 생성되어야 함. 생성 순서 재배치 필요:
```typescript
private createSystems(): void {
  this.turnSystem = new TurnSystem(this, { ... })
  this.weaponManager = new WeaponManager(this)
  this.windSystem = new WindSystem()
  this.damageSystem = new DamageSystem(this, this.terrainRenderer, this.players)

  // 컨트롤러 생성 (의존성이 모두 준비된 후)
  this.controllers.push(new InputController(this))
  this.controllers.push(new AIController({
    scene: this,
    terrainData: this.terrainData,
    weaponManager: this.weaponManager,
    opponents: [this.players[0]],
    windSystem: this.windSystem,
  }))

  this.setupWeaponKeys()
}
```

#### 3. update() 수정 (`GameScene.ts:421-495`)

```typescript
update(): void {
  if (this.isGameOver) return

  const playerIdx = this.turnSystem.getCurrentPlayer()
  const currentPlayer = this.players[playerIdx]
  const controller = this.controllers[playerIdx]
  if (!currentPlayer || !controller) return

  // 컨트롤러 업데이트 (Human: 입력, AI: 상태 머신)
  controller.update(currentPlayer, this.turnSystem)

  // 모든 캐릭터 업데이트
  this.players.forEach((player) => player.update())
  this.damageSystem.update()
  this.weaponManager.update()
  this.weaponManager.applyWindToAll(this.windSystem.getWindForce())

  const state = this.turnSystem.getState()

  // 무기 표시/숨김
  if (state !== this.prevState) {
    if (state === "aiming") currentPlayer.showWeapon()
    else if (this.prevState === "aiming") currentPlayer.hideWeapon()
    this.prevState = state
  }

  if (state === "aiming") {
    const aimingRight = controller.isAimingRight()
    currentPlayer.setFacingDirection(aimingRight)
    const aimAngle = controller.getAimAngle()
    currentPlayer.setWeaponAngle(aimAngle)

    this.aimingUI.show()

    // Human: 마우스 가이드 라인 포함, AI: 조준선만
    if (controller instanceof InputController) {
      const pointer = this.input.activePointer
      this.aimingUI.updateWithMousePos(
        currentPlayer, pointer.x, pointer.y, aimAngle, controller.getAimPower()
      )
    } else {
      this.aimingUI.update(currentPlayer, aimAngle, controller.getAimPower())
    }
  } else {
    this.aimingUI.hide()
  }

  if (state === "firing") {
    this.handleFire(currentPlayer, controller)
  }

  if (state === "animating" && !this.weaponManager.hasActiveProjectiles()) {
    this.onAnimationComplete()
  }

  this.gameHUD.updateTimer(this.turnSystem.getRemainingTime())
  this.windVisualizer.update()
  this.updateWater()
}
```

#### 4. handleFire() 수정 (`GameScene.ts:497-534`)

```typescript
// 변경 전:
private handleFire(character: Character): void

// 변경 후:
private handleFire(character: Character, controller: IPlayerController): void {
  const pos = character.getPosition()
  character.playShootAnimation()

  const angle = controller.getAimAngle()
  const facingRight = character.isFacingRight()
  const adjustedAngle = facingRight ? angle : 180 - angle
  const radians = Phaser.Math.DegToRad(adjustedAngle)
  const spawnDistance = 35
  const spawnX = pos.x + Math.cos(radians) * spawnDistance
  const spawnY = pos.y - Math.sin(radians) * spawnDistance

  this.weaponManager.fire(
    spawnX, spawnY, angle, controller.getAimPower(), facingRight, character.playerId
  )

  controller.resetAim()
  this.turnSystem.startAnimating()
}
```

#### 5. onAnimationComplete() 수정 (`GameScene.ts:536-540`)

AI 학습 피드백을 위해 마지막 폭발 위치와 명중 여부를 전달:

```typescript
private onAnimationComplete(): void {
  // AI 학습 피드백
  const playerIdx = this.turnSystem.getCurrentPlayer()
  // 현재 "animating" 상태이므로, 이전 턴 플레이어를 구해야 함
  const prevPlayerIdx = (playerIdx - 1 + this.players.length) % this.players.length
  const controller = this.controllers[prevPlayerIdx]

  if (controller.onShotResult && this.lastExplosionData) {
    controller.onShotResult(
      { x: this.lastExplosionData.x, y: this.lastExplosionData.y },
      this.lastExplosionData.hit
    )
  }
  this.lastExplosionData = null

  this.windSystem.randomizeWind()
  this.turnSystem.endTurn()
}
```

**주의**: `onAnimationComplete` 시점에서 `getCurrentPlayer()`는 아직 현재(발사한) 플레이어를 가리키고 있음. `endTurn()`이 호출되면서 다음 플레이어로 넘어감. 따라서 `getCurrentPlayer()`를 `endTurn()` 전에 사용해야 함.

수정된 버전:
```typescript
private onAnimationComplete(): void {
  // AI 학습 피드백 (endTurn 호출 전에 현재 플레이어 정보 획득)
  const playerIdx = this.turnSystem.getCurrentPlayer()
  const controller = this.controllers[playerIdx]

  if (controller.onShotResult && this.lastExplosionData) {
    controller.onShotResult(
      { x: this.lastExplosionData.x, y: this.lastExplosionData.y },
      this.lastExplosionData.hit
    )
  }
  this.lastExplosionData = null

  this.windSystem.randomizeWind()
  this.turnSystem.endTurn()
}
```

#### 6. 폭발 데이터 추적용 멤버/리스너 추가

```typescript
// 멤버 변수 추가
private lastExplosionData: { x: number; y: number; hit: boolean } | null = null

// setupEventListeners() 에 추가
private setupEventListeners(): void {
  EventBus.on(GameEvents.GAME_OVER, this.onGameOver, this)

  // 폭발 이벤트 감시 (AI 학습용)
  this.events.on("explosion", (event: ExplosionEvent) => {
    const hitAny = this.players.some(p => {
      if (p.playerId === event.shooterId || !p.isAlive()) return false
      const pos = p.getPosition()
      const dist = Phaser.Math.Distance.Between(event.x, event.y, pos.x, pos.y)
      return dist <= event.radius
    })
    this.lastExplosionData = { x: event.x, y: event.y, hit: hitAny }
  })
}
```

#### 7. setupWeaponKeys() 수정 — AI 턴 중 키보드 무기 선택 차단

```typescript
private selectWeaponForCurrentPlayer(weaponKey: string): void {
  // AI 턴 중에는 키보드 무기 선택 차단
  const playerIdx = this.turnSystem.getCurrentPlayer()
  const controller = this.controllers[playerIdx]
  if (!(controller instanceof InputController)) return

  if (this.weaponManager.selectWeapon(weaponKey)) {
    const currentPlayer = this.players[playerIdx]
    currentPlayer?.setWeapon(weaponKey)
  }
}
```

#### 8. shutdown() 수정 (`GameScene.ts:573-599`)

```typescript
shutdown(): void {
  if (this.isShuttingDown) return
  this.isShuttingDown = true

  this.bgm?.stop()
  this.terrainRenderer?.destroy()
  this.players.forEach((player) => player.destroy())
  this.players = []
  this.turnSystem?.destroy()

  // 컨트롤러 정리
  this.controllers.forEach(c => c.destroy?.())
  this.controllers = []

  this.weaponManager?.destroy()
  this.damageSystem?.destroy()
  this.aimingUI?.destroy()
  this.gameHUD?.destroy()
  this.windVisualizer?.destroy()
  this.weaponSelector?.destroy()
  // ... 나머지 정리 코드 ...
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 통과
- [x] 기존 `inputController` 참조가 모두 `controller`/`controllers[]`로 교체됨

#### 수동 검증:
- [ ] Player 1 (Human)의 입력/조준/발사가 기존과 동일
- [ ] Player 2 (AI)가 자동으로 궤적을 계산하고 발사
- [ ] AI 턴에서 AimingUI가 조준선만 표시 (마우스 가이드 없음)
- [ ] 턴이 정상적으로 교대
- [ ] 30초 턴 타이머가 AI에도 적용
- [ ] AI가 초반에는 부정확하고 후반에는 정확해짐
- [ ] 게임 오버가 정상 처리됨

**Implementation Note**: 이 단계가 가장 큰 변경. 완료 후 반드시 전체 게임 플로우 수동 테스트 필요.

---

## 5단계: AI 시각 피드백

### 개요
AI 턴에 "생각 중" 말풍선을 캐릭터 위에 표시하고, 선택적으로 HUD에 AI 턴 표시를 추가한다.

### 필요한 변경:

#### 1. AIThinkingBubble 표시 (GameScene에서 직접 처리)

AI 전용 UI 컴포넌트를 별도 파일로 만들 수도 있지만, 간단히 GameScene의 update() 내에서 처리:

```typescript
// 멤버 변수 추가
private thinkingBubble?: Phaser.GameObjects.Text

// update()의 컨트롤러 업데이트 이후에 추가:
if (controller instanceof AIController) {
  const aiPhase = (controller as AIController).getPhase()
  if (aiPhase === "thinking" || aiPhase === "moving" || aiPhase === "weaponSelect") {
    if (!this.thinkingBubble) {
      this.thinkingBubble = this.add.text(0, 0, "🤔", {
        fontSize: "28px",
        backgroundColor: "#ffffff",
        padding: { x: 6, y: 4 },
        // 둥근 배경은 Phaser Text로는 제한적, 간단한 배경색 사용
      }).setOrigin(0.5, 1).setDepth(100)
    }
    const pos = currentPlayer.getPosition()
    this.thinkingBubble.setPosition(pos.x, pos.y - 50)
    this.thinkingBubble.setVisible(true)
  } else {
    this.thinkingBubble?.setVisible(false)
  }
} else {
  this.thinkingBubble?.setVisible(false)
}
```

#### 2. AIController에 getPhase() public 메서드 추가

```typescript
// AIController.ts에 추가:
getPhase(): AIPhase {
  return this.phase
}
```

#### 3. shutdown()에서 정리

```typescript
this.thinkingBubble?.destroy()
```

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 통과

#### 수동 검증:
- [ ] AI thinking 페이즈에서 캐릭터 위에 말풍선 표시
- [ ] AI가 조준/발사 시 말풍선 사라짐
- [ ] Human 턴에서는 말풍선 표시 안 됨
- [ ] 말풍선 위치가 캐릭터를 따라다님

**Implementation Note**: 시각 피드백은 기능에 영향 없는 순수 UI 추가. 간단히 구현 가능.

---

## 6단계: DamageSystem/WeaponManager 확장

### 개요
AI 학습 피드백에 필요한 폭발 위치 추적과 명중 판정 기능을 추가한다.
(4단계에서 GameScene에 `"explosion"` 이벤트 리스너로 이미 처리했으므로, 이 단계는 보조적.)

### 필요한 변경:

#### 1. 확인 및 마무리

4단계에서 이미 구현한 폭발 데이터 추적 (`lastExplosionData`)이 정상 동작하는지 확인.

**확인 사항:**
- `"explosion"` scene 이벤트는 `BazookaProjectile`과 `ShotgunProjectile`의 `onCollision()`에서 emit됨
- Grenade는 퓨즈 타이머로 폭발 → 동일한 이벤트 emit
- 여러 투사체(Shotgun 5발)의 경우, 마지막 폭발만 기록되면 충분 (AI는 Shotgun 사용 시 가장 가까운 적에게 발사)

#### 2. 엣지 케이스 처리

```typescript
// GameScene.ts - 타임아웃으로 턴 종료된 경우 (발사 없이 턴 종료)
// TurnSystem.onTurnTimeout()이 endTurn()을 직접 호출하므로
// onAnimationComplete()가 호출되지 않음 → AI 피드백도 불필요
// → 이미 정상 동작 (lastExplosionData가 null이므로 피드백 스킵)
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 컴파일 통과

#### 수동 검증:
- [ ] AI가 Bazooka 발사 후 다음 턴에 보정이 적용됨
- [ ] Shotgun 발사 시에도 피드백 정상 동작
- [ ] 타임아웃으로 턴 종료 시 에러 없음
- [ ] 물에 빠진 투사체 (화면 밖 이탈) 시 피드백 없이 정상 진행

---

## 테스트 전략

### 자동화 테스트:
- TypeScript 컴파일 (`npx tsc --noEmit`)
- IPlayerController 인터페이스 준수 확인

### 수동 테스트 시나리오:

1. **기본 플로우**: Play → Human 조준/발사 → AI 자동 발사 → 턴 교대 반복
2. **AI 학습 확인**: 2~3턴 후 AI 정확도 향상 체감, 넉백 후에도 잘못된 보정 없이 정상 발사
3. **AI 이동**: AI 캐릭터가 물 근처에 스폰되면 안쪽으로 이동하는지 확인
4. **AI 이동 장애물**: 지형에 막히면 점프, 오래 막히면 방향 전환하는지 확인
5. **무기 선택**: 근거리에서 Shotgun 선택, 원거리에서 Bazooka 선택
6. **게임 오버**: AI 승리 / Human 승리 / 양쪽 동시 사망(무승부)
7. **턴 타이머**: AI 턴 30초 초과 시 자동 턴 종료
8. **재시작**: 게임 오버 후 SPACE → 새 게임에서 AI 경험치 리셋
9. **조준 UI**: Human 턴에 마우스 가이드 표시, AI 턴에 조준선만 표시
10. **말풍선**: AI thinking 시 표시, 발사 시 사라짐

## 성능 고려 사항

- **궤적 시뮬레이션**: 1차 탐색 ~306개 조합(19×17) + 2차 탐색 ~169개 조합(13×13) × 최대 300프레임
  - JavaScript에서 수 밀리초 이내 완료 가능 (단순 산술 연산)
  - `weaponSelect` 페이즈에서 한 번 실행 (이동 후 최종 위치에서 계산)
- **AI 턴 총 시간**: ~2초 (이동 없을 때) — 플레이어 체감에 자연스러움

## 참조

- 연구 문서: `thoughts/shared/research/2026-02-13-ai-player-system-research.md`
- 원본 아키텍처: `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md`
- MVP 구현 계획: `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md`
