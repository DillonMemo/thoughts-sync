# 게임 밸런스: 바주카 너프 + 이동 게이지 시스템 구현 계획

## 개요

바주카(기본 무기)의 수치를 하향 조정하고, 건바운드/포트리스 스타일의 이동 게이지 시스템을 도입하여 게임 밸런스를 개선합니다.

**두 가지 변경 사항:**
1. 바주카 너프: damage 50→25, explosionRadius 40→30 (1파일 변경)
2. 이동 게이지: 총량 128, 이동 1/프레임, 점프 20/회 (6파일 변경)

## 현재 상태 분석

### 바주카 (WeaponTypes.ts:13-22)
- damage: 50, explosionRadius: 40, ammo: -1 (무제한)
- 3종 무기 중 가장 높은 대미지 + 무제한 탄약으로 다른 무기 선택 동기 부족
- 넉백(DamageSystem.ts:140)은 `damage * 0.001`로 자동 계산됨

### 이동 시스템
- 게이지 개념 없음 — 30초 턴 타이머 내 무제한 이동/점프
- Character.ts:63-65 — BASE_MOVE_SPEED=1.25, JUMP_VELOCITY=-5.3
- InputController.ts:146-160 — A/D로 이동, W로 점프 (playerTurn 상태에서만)
- AIController.ts:184-244 — 시간 기반 이동 (moveTimer 800~1500ms)

### 핵심 발견:
- 넉백 공식 `damage * 0.001` → 바주카 damage 25로 변경하면 넉백 자동으로 0.05→0.025 (DamageSystem.ts:140)
- 자폭 범위 `radius * 1.5` → explosionRadius 30이면 자폭 범위 45px 자동 반영 (DamageSystem.ts:91)
- AI 이동은 moveTimer(ms 기반)로 제어됨 — 게이지 기반으로 전환 필요
- Character.move()는 매 프레임 호출됨 — 프레임당 게이지 1 차감에 자연스러운 구조
- GameHUD는 텍스트 기반만 있음 — Graphics로 게이지 바 추가 필요
- IPlayerController 인터페이스 변경 불필요 — 게이지는 Character가 소유

## 원하는 최종 상태

1. 바주카가 3종 무기 중 가장 약한 기본 무기로 위치 (damage 25, shotgun과 동일하지만 무제한 탄약)
2. 모든 플레이어(인간/AI)에게 동일한 이동 게이지 적용
3. 턴 시작 시 게이지 128로 리셋, 이동 중 프레임당 1 차감, 점프당 20 차감
4. 게이지 소진 시 이동/점프 불가, 조준은 수동으로 가능
5. HUD에 게이지 바가 시각적으로 표시됨

### 검증 방법:
- 바주카 발사 시 대미지 25 확인 (콘솔 또는 체력바)
- 이동 게이지가 0이 되면 A/D/W 입력에 캐릭터가 반응하지 않음
- AI도 게이지 제한 내에서 이동함
- 점프 6회 후 이동 거리가 극히 짧음 (128 - 120 = 8프레임)

## 하지 않을 것

- Grenade/Shotgun 수치 변경 (바주카만 조정)
- 턴 타이머(30초) 변경
- 새 무기 추가
- 게이지 소진 시 자동 조준 전환 (수동 유지)
- IPlayerController 인터페이스 변경

## 구현 접근 방식

단계별로 독립적인 변경을 수행하여 각 단계가 완료될 때마다 게임이 정상 동작하도록 합니다. 바주카 너프는 단 1파일 변경이므로 먼저 처리하고, 이동 게이지는 데이터 모델 → 입력 차단 → AI 전환 → 턴 연동 → UI 순으로 진행합니다.

---

## 1단계: 바주카 너프

### 개요
WeaponTypes.ts의 바주카 수치 2개를 변경합니다. 넉백/자폭 범위는 공식에 의해 자동 반영됩니다.

### 필요한 변경:

#### 1. WeaponTypes.ts (바주카 수치)
**파일**: `packages/game-core/src/weapons/WeaponTypes.ts`
**변경**: Line 15-16

```typescript
// 변경 전
damage: 50,
explosionRadius: 40,

// 변경 후
damage: 25,
explosionRadius: 30,
```

### 자동 반영되는 값들 (코드 변경 불필요):
| 항목 | 공식 | 변경 전 | 변경 후 |
|------|------|---------|---------|
| 넉백 force | `damage * 0.001` (DamageSystem.ts:140) | 0.05 | 0.025 |
| 자폭 범위 | `radius * 1.5` (DamageSystem.ts:91) | 60px | 45px |
| 자폭 최소 대미지 | `damage * 0.25` (DamageSystem.ts:98) | 12 | 6 |
| 지형 파괴 면적 | `π × radius²` (TerrainData.ts:34-47) | ~5,026px | ~2,827px |
| AI 자폭 회피 반경 | `explosionRadius * 1.5` (AIController.ts:385) | 60px | 45px |

### AI 영향 검토 (확인 완료):
- **모든 AI 계산 공식은 WeaponRegistry에서 동적으로 읽음** — 하드코딩된 바주카 수치 없음
- `findBestShot()` → `weaponManager.getCurrentWeapon()`으로 mass, explosionRadius 동적 참조 (AIController.ts:383-385)
- `simulateLanding()` → mass 파라미터로 전달 (AIController.ts:338), 궤적 시뮬레이션 정상
- `simulateBestFromPos()` → mass 동적 참조 (AIController.ts:567)
- `selectWeapon()` → 거리 기반 선택 (AIController.ts:630-636), damage 미참조
- 넉백 절반 감소(0.05→0.025)는 **의도된 변경** — 필요시 추후 별도 조정
- 바주카(25) = 샷건(25) 동일 데미지이나, **샷건은 다발 발사**이므로 밸런스 문제 없음

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 타입 체크 통과: `cd packages/game-core && npx tsc --noEmit`
- [x] 빌드 성공: `npm run build` (또는 프로젝트의 빌드 커맨드)

#### 수동 검증:
- [ ] 바주카로 직격 시 대미지가 25인지 확인 (체력바 변화량)
- [ ] 폭발 반경이 이전보다 작아진 것이 시각적으로 확인됨
- [ ] Grenade(35), Shotgun(25) 대미지는 변경 없음

**Implementation Note**: 이 단계는 단순 수치 변경이므로 바로 다음 단계로 진행 가능.

---

## 2단계: Character에 게이지 시스템 추가

### 개요
Character 엔티티에 이동 게이지 상태를 추가하고, moveLeft/moveRight/jump에서 게이지를 소모하도록 합니다. 게이지가 0이면 이동/점프 메서드가 작동하지 않습니다.

### 필요한 변경:

#### 1. Character.ts (게이지 상태 및 소모)
**파일**: `packages/game-core/src/entities/Character.ts`

**1-1. 상수 추가** (Line 62-65 영역, 기존 물리 상수 아래):

```typescript
// 이동 게이지
private readonly MAX_MOVE_GAUGE = 128
private readonly MOVE_COST_PER_FRAME = 1
private readonly JUMP_COST = 20
```

**1-2. 상태 추가** (Line 46-53 영역, 기존 상태 아래):

```typescript
private moveGauge: number = 128
```

**1-3. moveLeft/moveRight에 게이지 차감 추가** (Line 319-331):

```typescript
moveLeft(): void {
  if (this.moveGauge <= 0) return  // 게이지 소진 시 이동 불가
  this.moveGauge -= this.MOVE_COST_PER_FRAME
  this.move(-1)
  this.facingRight = false
  this.isMoving = true
  this.bodySprite.setFlipX(true)
}

moveRight(): void {
  if (this.moveGauge <= 0) return  // 게이지 소진 시 이동 불가
  this.moveGauge -= this.MOVE_COST_PER_FRAME
  this.move(1)
  this.facingRight = true
  this.isMoving = true
  this.bodySprite.setFlipX(false)
}
```

**1-4. jump에 게이지 차감 추가** (Line 404-416):

```typescript
jump(): void {
  if (this.moveGauge < this.JUMP_COST) return  // 게이지 부족 시 점프 불가

  const canJump =
    this.isGrounded || Date.now() - this.lastGroundedTime < this.COYOTE_TIME

  if (canJump) {
    this.moveGauge -= this.JUMP_COST
    this.scene.matter.body.setVelocity(this.physicsBody, {
      x: this.physicsBody.velocity.x,
      y: this.JUMP_VELOCITY,
    })
    this.isGrounded = false
    this.lastGroundedTime = 0
  }
}
```

**1-5. 게이지 관련 public 메서드 추가** (getHealth() 근처):

```typescript
getMoveGauge(): number {
  return this.moveGauge
}

getMaxMoveGauge(): number {
  return this.MAX_MOVE_GAUGE
}

resetMoveGauge(): void {
  this.moveGauge = this.MAX_MOVE_GAUGE
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 타입 체크 통과: `cd packages/game-core && npx tsc --noEmit`

#### 수동 검증:
- [ ] 이동 시 게이지 값이 감소함 (콘솔 로그로 확인 가능)
- [ ] 게이지 0 이하에서 A/D 입력 시 캐릭터가 이동하지 않음
- [ ] 게이지 부족 시 W(점프) 입력이 무시됨

**Implementation Note**: 이 단계에서는 아직 게이지가 턴마다 리셋되지 않으므로, 첫 턴의 게이지를 다 쓰면 이후 이동 불가. 4단계에서 리셋 추가.

---

## 3단계: 인간 플레이어 게이지 적용

### 개요
InputController에서 게이지 잔량을 확인하여 이동/점프 시 시각적 피드백을 제공합니다. Character의 moveLeft/moveRight/jump에 이미 게이지 체크가 내장되어 있으므로, InputController 자체에서는 추가 차단이 필수는 아니지만, 게이지 0일 때 `stopMoving()`을 호출하여 관성 이동을 방지합니다.

### 필요한 변경:

#### 1. InputController.ts (게이지 소진 시 정지)
**파일**: `packages/game-core/src/systems/InputController.ts`

**1-1. handleMovement 수정** (Line 146-154):

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

**1-2. handleJump 수정** (Line 156-160):

```typescript
private handleJump(character: Character): void {
  if (Phaser.Input.Keyboard.JustDown(this.keyW)) {
    character.jump()  // Character.jump() 내부에서 게이지 체크
  }
}
```

> 참고: jump()는 Character 내부에서 게이지를 체크하므로 InputController에서 추가 체크 불필요.

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과

#### 수동 검증:
- [ ] A/D 키 입력 중 게이지가 0이 되면 캐릭터가 즉시 정지
- [ ] 게이지 0 상태에서 마우스 클릭으로 조준은 정상 가능
- [ ] 게이지 0 상태에서 W(점프) 입력이 무시됨

---

## 4단계: 턴 시스템 연동 (게이지 리셋)

### 개요
TurnSystem.endTurn() → 다음 턴 시작 시 현재 플레이어의 게이지를 리셋합니다. TurnSystem은 Character를 직접 참조하지 않으므로, GameScene에서 TURN_CHANGED 이벤트를 수신하여 리셋합니다.

### 필요한 변경:

#### 1. GameScene.ts (턴 변경 시 게이지 리셋)
**파일**: `packages/game-core/src/scenes/GameScene.ts`

**1-1. setupEventListeners에 게이지 리셋 추가** (Line 447-468):

```typescript
private setupEventListeners(): void {
  EventBus.on(GameEvents.GAME_OVER, this.onGameOver, this)
  EventBus.on(GameEvents.TURN_CHANGED, this.onTurnChanged, this)  // 추가

  // 기존 폭발 이벤트 코드 유지...
}
```

**1-2. onTurnChanged 핸들러 추가**:

```typescript
private onTurnChanged(data: { currentPlayer: number }): void {
  // 현재 턴 플레이어의 게이지 리셋
  const player = this.players[data.currentPlayer]
  if (player) {
    player.resetMoveGauge()
  }
}
```

**1-3. shutdown에 이벤트 해제 추가** (Line 703):

```typescript
EventBus.off(GameEvents.TURN_CHANGED, this.onTurnChanged, this)
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과

#### 수동 검증:
- [ ] 첫 턴에서 게이지를 모두 소모한 후, 두 번째 턴에서 게이지가 128로 리셋됨
- [ ] 각 플레이어 턴 시작 시 게이지가 독립적으로 리셋됨

---

## 5단계: AI 플레이어 게이지 전환

### 개요
AIController의 시간 기반 이동(moveTimer)을 게이지 기반으로 전환합니다. AI는 게이지 잔량을 확인하여 이동을 종료하고, 점프도 게이지 비용을 고려합니다.

### 필요한 변경:

#### 1. AIController.ts (시간 기반 → 게이지 기반)
**파일**: `packages/game-core/src/systems/AIController.ts`

**1-1. 이동 시작 시 게이지 예산 할당** (startMoving, Line 156-181):

AI는 이동에 전체 게이지를 쓰지 않고, 상황별로 **최대 사용량**을 제한합니다. moveTimer를 게이지 예산(budget)으로 대체합니다.

```typescript
private moveBudget: number = 0  // 새 필드 추가 (Line 69-73 영역)

private startMoving(character: Character): void {
  const pos = character.getPosition()
  this.lastMovePos = { x: pos.x, y: pos.y }
  this.stuckFrames = 0
  const waterLevel = this.config.scene.scale.height * 0.91
  const mapWidth = this.config.scene.scale.width
  const surfaceY = this.getSurfaceY(Math.floor(pos.x))
  const gauge = character.getMoveGauge()

  if (pos.y > waterLevel - 100 || surfaceY > waterLevel - 60) {
    this.moveDirection = this.findSafeDirection(character)
    this.moveBudget = Math.min(gauge, 100)  // 물 탈출: 최대 100 게이지
  } else if (this.needsRepositioning) {
    this.moveDirection = this.evaluateMoveDirection(character)
    this.moveBudget = Math.min(gauge, 80)   // 재배치: 최대 80 게이지
  } else if (pos.x < 80) {
    this.moveDirection = 1
    this.moveBudget = Math.min(gauge, 50)   // 가장자리 회피: 최대 50 게이지
  } else if (pos.x > mapWidth - 80) {
    this.moveDirection = -1
    this.moveBudget = Math.min(gauge, 50)
  } else {
    this.moveDirection = 0
    this.moveBudget = 0
  }
}
```

**1-2. 이동 업데이트에서 게이지 확인** (onMovingUpdate, Line 184-244):

moveTimer 대신 게이지 잔량과 예산을 확인하여 이동 종료를 결정합니다.

```typescript
private onMovingUpdate(character: Character): void {
  const gauge = character.getMoveGauge()
  const budgetUsed = (this.moveBudget > 0)
    ? character.getMaxMoveGauge() - gauge >= (character.getMaxMoveGauge() - this.moveBudget - gauge)
    : true

  if (this.moveDirection !== 0 && gauge > 0 && this.moveBudget > 0) {
    this.moveBudget -= 1  // 프레임당 예산 차감

    if (this.moveDirection > 0) {
      character.moveRight()
    } else {
      character.moveLeft()
    }

    const pos = character.getPosition()

    // 막힘 감지 (기존 로직 유지)
    const moveDelta = Math.abs(pos.x - this.lastMovePos.x)
    if (moveDelta < 0.5) {
      this.stuckFrames++
    } else {
      this.stuckFrames = 0
    }
    this.lastMovePos = { x: pos.x, y: pos.y }

    if (character.getIsGrounded()) {
      const cx = Math.floor(pos.x)
      const aheadNear = Math.floor(pos.x + this.moveDirection * 10)
      const aheadMid = Math.floor(pos.x + this.moveDirection * 25)
      const aheadFar = Math.floor(pos.x + this.moveDirection * 45)
      const currentSurface = this.getSurfaceY(cx)
      const nearSurface = this.getSurfaceY(aheadNear)
      const midSurface = this.getSurfaceY(aheadMid)
      const farSurface = this.getSurfaceY(aheadFar)

      const shouldJump =
        (nearSurface < currentSurface - 8 ||
         midSurface < currentSurface - 15 ||
         farSurface < currentSurface - 25 ||
         this.stuckFrames > 5) &&
        gauge >= 20  // 점프 비용(20) 확인 추가

      if (shouldJump) {
        character.jump()  // Character.jump() 내부에서 게이지 20 차감
        this.stuckFrames = 0
      }
    }

    // 오래 막혀있으면 방향 전환
    if (this.stuckFrames > 30) {
      this.moveDirection = -this.moveDirection
      this.stuckFrames = 0
    }
  } else {
    character.stopMoving()
    this.needsRepositioning = false
    this.stuckFrames = 0
    this.phase = "weaponSelect"
    this.phaseTimer = 300
  }
}
```

**1-3. moveTimer 필드 제거/미사용 처리**:

기존 `moveTimer` 필드(Line 70)는 더 이상 사용하지 않으므로 `moveBudget`으로 대체합니다.

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과

#### 수동 검증:
- [ ] AI가 게이지 범위 내에서만 이동하는지 확인 (이전보다 이동 거리 제한됨)
- [ ] AI가 점프할 때 게이지가 20 차감되는지 확인
- [ ] AI가 게이지 소진 후 정상적으로 조준/발사하는지 확인
- [ ] AI가 물 근처에서도 게이지 내에서 탈출 시도하는지 확인

**Implementation Note**: 이 단계 완료 후 AI 동작을 관찰하여 moveBudget 값 조정이 필요할 수 있음. 수동 테스트 확인 후 다음 단계 진행.

---

## 6단계: 게이지 바 UI

### 개요
GameHUD에 Phaser Graphics로 이동 게이지 바를 추가합니다. 턴 타이머 아래 또는 현재 플레이어 캐릭터 근처에 수평 바로 표시합니다.

### 필요한 변경:

#### 1. GameHUD.ts (게이지 바 추가)
**파일**: `packages/game-core/src/ui/GameHUD.ts`

**1-1. 게이지 바 Graphics 프로퍼티 추가** (Line 7-13):

```typescript
private gaugeBarBg: Phaser.GameObjects.Graphics | null = null
private gaugeBarFill: Phaser.GameObjects.Graphics | null = null
private gaugeLabel: Phaser.GameObjects.Text | null = null
```

**1-2. createUI에 게이지 바 생성 추가** (Line 40-84, weaponDisplay 이후):

```typescript
// Move Gauge Bar (턴 타이머 아래)
this.gaugeBarBg?.destroy()
this.gaugeBarBg = this.scene.add.graphics()
this.gaugeBarBg.setDepth(50)

this.gaugeBarFill?.destroy()
this.gaugeBarFill = this.scene.add.graphics()
this.gaugeBarFill.setDepth(50)

this.gaugeLabel?.destroy()
this.gaugeLabel = this.scene.add
  .text(width / 2, 80, "MOVE", {
    fontSize: "10px",
    color: "#aaaaaa",
  })
  .setOrigin(0.5)
  .setDepth(50)

// 초기 게이지 바 그리기
this.drawGaugeBar(1.0)
```

**1-3. drawGaugeBar 메서드 추가**:

```typescript
private drawGaugeBar(percentage: number): void {
  if (!this.gaugeBarBg || !this.gaugeBarFill) return

  const { width } = this.scene.scale
  const barWidth = 100
  const barHeight = 6
  const x = width / 2 - barWidth / 2
  const y = 90

  // 배경
  this.gaugeBarBg.clear()
  this.gaugeBarBg.fillStyle(0x000000, 0.5)
  this.gaugeBarBg.fillRect(x - 1, y - 1, barWidth + 2, barHeight + 2)

  // 게이지 (초록 → 노랑 → 빨강)
  this.gaugeBarFill.clear()
  let color = 0x00ff00
  if (percentage < 0.25) color = 0xff4444
  else if (percentage < 0.5) color = 0xffaa00

  this.gaugeBarFill.fillStyle(color, 0.9)
  this.gaugeBarFill.fillRect(x, y, barWidth * Math.max(0, percentage), barHeight)
}
```

**1-4. updateGauge public 메서드 추가**:

```typescript
updateGauge(current: number, max: number): void {
  if (this.isDestroyed) return
  this.drawGaugeBar(current / max)
}
```

**1-5. destroy에 정리 추가** (Line 131-143):

```typescript
this.gaugeBarBg?.destroy()
this.gaugeBarFill?.destroy()
this.gaugeLabel?.destroy()
```

#### 2. GameScene.ts (매 프레임 게이지 업데이트)
**파일**: `packages/game-core/src/scenes/GameScene.ts`

**2-1. update 루프에 게이지 HUD 업데이트 추가** (Line 571 근처, HUD 타이머 업데이트 이후):

```typescript
// HUD 타이머 업데이트
this.gameHUD.updateTimer(this.turnSystem.getRemainingTime())

// 게이지 업데이트 (현재 플레이어)
this.gameHUD.updateGauge(
  currentPlayer.getMoveGauge(),
  currentPlayer.getMaxMoveGauge()
)
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과
- [ ] 빌드 성공

#### 수동 검증:
- [ ] 턴 타이머 아래에 게이지 바가 표시됨
- [ ] 이동 시 게이지 바가 줄어듦
- [ ] 점프 시 게이지 바가 크게 줄어듦 (20/128 ≈ 15.6%)
- [ ] 게이지 25% 미만에서 빨간색으로 변경됨
- [ ] 턴 변경 시 게이지 바가 풀로 리셋됨
- [ ] AI 턴에서도 게이지 바가 정상 표시됨

---

## 테스트 전략

### 수동 테스트 시나리오:

1. **바주카 너프 확인**:
   - 바주카로 직격 시 대미지 25 확인
   - 바주카 폭발 반경이 이전보다 작은지 시각적 확인
   - Grenade(35), Shotgun(25×5) 대미지 변경 없음 확인

2. **게이지 기본 동작**:
   - 평지에서 A/D 키 꾹 누르기 → 약 2.1초(128프레임) 후 정지
   - 점프 없이 이동 → 약 160px 이동 가능 확인
   - 점프 1회 + 이동 → 약 135px 이동 가능 확인

3. **게이지 트레이드오프**:
   - 점프 6회(120 게이지) → 남은 이동 8프레임(10px) 확인
   - 이동만으로 게이지 소진 후 점프 불가 확인

4. **경사면 영향**:
   - 오르막에서 이동 시 같은 게이지로 짧은 거리 이동 확인
   - 내리막에서 이동 시 같은 게이지로 먼 거리 이동 확인

5. **턴 리셋**:
   - 게이지 소진 → 발사 → 다음 턴 → 게이지 128 리셋 확인

6. **AI 동작**:
   - AI가 게이지 내에서만 이동하는지 관찰
   - AI가 물 근처에서 충분히 탈출하는지 확인
   - AI가 이동 후 정상 발사하는지 확인

7. **게이지 UI**:
   - 게이지 바가 이동에 따라 실시간 감소
   - 색상 변화 (초록 → 노랑 → 빨강) 확인
   - 턴 변경 시 풀 리셋 확인

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-16-game-balance-weapon-movement-research.md`
- 무기 시스템: `packages/game-core/src/weapons/WeaponTypes.ts:12-43`
- 대미지 계산: `packages/game-core/src/systems/DamageSystem.ts:77-147`
- 캐릭터 이동: `packages/game-core/src/entities/Character.ts:319-416`
- 입력 처리: `packages/game-core/src/systems/InputController.ts:108-160`
- AI 이동: `packages/game-core/src/systems/AIController.ts:156-244`
- 턴 시스템: `packages/game-core/src/systems/TurnSystem.ts:71-81`
- HUD: `packages/game-core/src/ui/GameHUD.ts`
- 게임 씬: `packages/game-core/src/scenes/GameScene.ts:365-578`
