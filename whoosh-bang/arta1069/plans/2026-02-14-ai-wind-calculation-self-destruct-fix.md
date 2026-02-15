# AI 바람 계산 수정 및 자폭 방지 구현 계획

## 개요

AI 플레이어의 투사체 궤적 시뮬레이션에서 질량(mass) 불일치로 인해 바람 영향을 최대 8.4배 과소평가하는 문제를 수정하고, `findBestShot()`에 자폭 방지 필터링을 추가하여 자폭 빈도를 감소시킨다.

## 현재 상태 분석

### 핵심 문제 1: 질량 불일치

AI의 `SIM_WIND_SCALE`은 `mass=0.5`를 하드코딩(`AIController.ts:329`)하지만, 실제 Matter.js는 스프라이트 프레임 크기 × scale² × density로 자동 계산한다:

| 무기 | 프레임 | Scale | 실제 Mass | AI 가정 | 바람 오차 |
|------|--------|-------|----------|---------|----------|
| Bazooka | 17×14px | 0.5 | ~0.06 | 0.5 | 8.4× 과소평가 |
| Grenade | 38×26px | 0.5 | ~0.25 | 0.5 | 2× 과소평가 |
| Shotgun | 25×16px | 0.3 | ~0.04 | 0.5 | 14× 과소평가 |

`setMass()`가 어디에서도 호출되지 않아(`Projectile.ts`), 질량이 스프라이트 크기에 의존하는 불안정한 상태.

### 핵심 문제 2: 자폭 방지 부재

`findBestShot()`(`AIController.ts:374-437`)는 착탄점↔타겟 거리만 계산하고, 착탄점↔AI 자신 거리는 전혀 검증하지 않는다. `DamageSystem.ts`의 사후 처리(radius×1.5, 최소 25%)만 존재.

### 핵심 발견:
- `AIController.ts:329` — `SIM_WIND_SCALE = (0.000001406 * 277.78) / 0.5`, mass=0.5 하드코딩
- `Projectile.ts:48` — `setScale(0.5)` 적용, `setMass()` 호출 없음
- `Shotgun.ts:17` — 추가 `setScale(0.3)` 적용 (부모의 0.5 위에)
- `WeaponTypes.ts:1-9` — `WeaponStats` interface에 mass 필드 없음
- `AIController.ts:374-437` — `findBestShot()`에 자폭 거리 체크 없음
- `AIController.ts:592-598` — AI는 Bazooka와 Shotgun만 사용

## 원하는 최종 상태

1. **각 무기가 명시적 mass를 선언** → `WeaponStats.mass`
2. **실제 물리가 선언된 mass를 사용** → `Projectile.setMass()`
3. **AI 시뮬레이션이 무기별 mass를 반영** → `simulateLanding(mass)`
4. **자폭 위험 조합이 findBestShot()에서 필터링** → 빈도 감소 (완전 방지 아님)
5. **applyError()로 인한 가끔의 자폭은 AI "실수"로 허용**

### 검증 방법:
- 강풍(±4~5)에서 AI 발사 시 투사체가 예측 궤적에 가깝게 비행
- AI 자폭 빈도가 현저히 감소 (가끔은 허용)
- 기존 게임플레이 느낌 유지 (투사체 바람 영향 변화 최소)

## 하지 않을 것

- `applyError()` 이후 자폭 재검증 (AI 실수 허용)
- Euler vs Verlet 적분 차이 수정 (영향도 중간, 별도 작업)
- Grenade AI 지원 추가 (범위 밖)
- Shotgun 스프레드 시뮬레이션 (범위 밖)
- `simulateBestFromPos()` (이동 평가용) 정밀 mass 반영 — default mass로 충분

## 구현 접근 방식

**하이브리드 접근**: `WeaponStats`에 `mass` 속성을 추가하고, `Projectile`에서 `setMass()`로 적용하며, AI 시뮬레이션에 동일한 mass를 전달한다. 이렇게 하면:
- 무기별 질량을 설계 의도대로 선언 가능 (추후 무기 다양성 지원)
- 스프라이트 크기와 무관한 예측 가능한 물리
- AI와 실제 물리의 mass 일치 보장

---

## 1단계: WeaponStats에 mass 속성 추가

### 개요
무기 정의에 `mass` 필드를 추가하여 각 무기의 투사체 질량을 명시적으로 선언한다.

### 필요한 변경:

#### 1. WeaponStats interface 수정
**파일**: `packages/game-core/src/weapons/WeaponTypes.ts`
**변경**: `mass` 필드 추가

```typescript
export interface WeaponStats {
  name: string
  damage: number
  explosionRadius: number
  ammo: number // -1 = 무제한
  mass: number // 투사체 질량 (Matter.js setMass에 사용)
  spriteKey: string
  spriteFrame?: string
  holdSprite?: string
}
```

#### 2. WeaponRegistry에 mass 값 설정
**파일**: `packages/game-core/src/weapons/WeaponTypes.ts`
**변경**: 각 무기에 mass 추가 (현재 물리와 유사한 값)

```typescript
export const WeaponRegistry: Record<string, WeaponStats> = {
  bazooka: {
    name: "Bazooka",
    damage: 50,
    explosionRadius: 40,
    ammo: -1,
    mass: 0.06,
    spriteKey: "tanks",
    spriteFrame: "tank_bullet1.png",
    holdSprite: "tanks_turret2.png",
  },
  grenade: {
    name: "Grenade",
    damage: 35,
    explosionRadius: 50,
    ammo: 3,
    mass: 0.25,
    spriteKey: "tanks",
    spriteFrame: "tank_bullet3.png",
    holdSprite: "tank_bullet3.png",
  },
  shotgun: {
    name: "Shotgun",
    damage: 25,
    explosionRadius: 15,
    ammo: 2,
    mass: 0.04,
    spriteKey: "tanks",
    spriteFrame: "tank_bullet2.png",
    holdSprite: "tanks_turret1.png",
  },
}
```

**mass 값 근거** (현재 자동 계산 값에 근사):
- Bazooka: `17 × 14 × 0.5² × 0.001 = 0.0595 ≈ 0.06`
- Grenade: `38 × 26 × 0.5² × 0.001 = 0.247 ≈ 0.25`
- Shotgun: `25 × 16 × 0.3² × 0.001 = 0.036 ≈ 0.04`

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 에러 없음: `npx tsc --noEmit`
- [x] WeaponRegistry의 모든 무기에 mass 필드 존재

#### 수동 검증:
- [ ] 없음 (인터페이스 변경만)

**Implementation Note**: 이 단계를 완료하고 TypeScript 컴파일이 통과한 후 다음 단계로 진행.

---

## 2단계: Projectile에 명시적 질량 적용

### 개요
`Projectile` 생성자에서 `setMass()`를 호출하여 실제 물리가 선언된 mass를 사용하도록 한다.

### 필요한 변경:

#### 1. ProjectileConfig에 mass 추가
**파일**: `packages/game-core/src/entities/Projectile.ts`
**변경**: config interface에 optional mass 필드 추가, constructor에서 setMass 호출

```typescript
export interface ProjectileConfig {
  scene: Phaser.Scene
  x: number
  y: number
  angle: number
  power: number
  shooterId: number
  spriteKey: string
  frame?: string
  mass?: number // 명시적 질량 설정
}
```

생성자 body에 setMass 호출 추가 (`Projectile.ts:51` 이후, `if (this.spriteBody)` 블록 내):

```typescript
if (this.spriteBody) {
  this.spriteBody.label = "projectile"
  this.spriteBody.gravityScale = { x: 1, y: 1 }
  // 명시적 질량 설정 (스프라이트 크기 의존성 제거)
  if (config.mass !== undefined) {
    this.scene.matter.body.setMass(this.spriteBody, config.mass)
  }
}
```

#### 2. BazookaProjectile에 mass 전달
**파일**: `packages/game-core/src/weapons/Bazooka.ts`
**변경**: super() 호출에 mass 추가

```typescript
constructor(config: Omit<ProjectileConfig, "spriteKey" | "frame">) {
  super({
    ...config,
    spriteKey: "tanks",
    frame: WeaponRegistry.bazooka.spriteFrame,
    mass: WeaponRegistry.bazooka.mass,
  })
}
```

#### 3. GrenadeProjectile에 mass 전달
**파일**: `packages/game-core/src/weapons/Grenade.ts`
**변경**: super() 호출에 mass 추가 (BazookaProjectile과 동일 패턴)

#### 4. ShotgunProjectile에 mass 전달
**파일**: `packages/game-core/src/weapons/Shotgun.ts`
**변경**: super() 호출에 mass 추가

```typescript
constructor(config: Omit<ProjectileConfig, "spriteKey" | "frame">) {
  super({
    ...config,
    spriteKey: "tanks",
    frame: WeaponRegistry.shotgun.spriteFrame,
    mass: WeaponRegistry.shotgun.mass,
  })

  // 샷건 탄환은 작게 표시 (시각적 scale만 변경, mass는 이미 설정됨)
  if (this.sprite) {
    this.sprite.setScale(0.3)
  }
}
```

**주의**: Shotgun의 `setScale(0.3)`은 시각적 크기만 변경. `setMass()`가 먼저 호출되므로 scale 변경이 mass를 덮어쓰지 않는다. 단, Phaser의 `setScale()`이 Matter.js `Body.scale()`을 호출하면 mass가 재계산될 수 있으므로, `setMass()`를 `setScale()` **이후**에 호출해야 할 수 있다. 이 경우 Projectile 생성자의 setMass 위치를 조정한다.

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 에러 없음: `npx tsc --noEmit`
- [x] 빌드 성공: `npm run build`

#### 수동 검증:
- [x] 게임 실행 후 Bazooka 발사 — 투사체가 기존과 유사하게 비행
- [x] 강풍에서 Bazooka 발사 — 바람 영향이 기존과 유사
- [x] Shotgun 발사 — 펠릿이 정상 비행

**Implementation Note**: setScale과 setMass의 호출 순서를 테스트하여 mass가 올바르게 적용되는지 확인. 필요시 Shotgun에서 setScale 이후 setMass를 재호출하는 방식으로 조정.

---

## 3단계: AI 시뮬레이션 질량 연동

### 개요
AI의 바람 가속도 계산에서 하드코딩된 `mass=0.5`를 제거하고, 현재 무기의 실제 mass를 동적으로 사용한다.

### 필요한 변경:

#### 1. SIM_WIND_SCALE → SIM_WIND_BASE 변경
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: 상수에서 mass 제거 (line 328-329)

```typescript
// 변경 전:
private static readonly SIM_WIND_SCALE = (0.000001406 * 277.78) / 0.5 // ≈0.00078

// 변경 후:
// wind: applyForce → accel = force/mass*dt², mass는 무기별로 동적 전달
private static readonly SIM_WIND_BASE = 0.000001406 * 277.78 // ≈0.000390
```

#### 2. simulateLanding()에 mass 파라미터 추가
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: 함수 시그니처 및 windAccel 계산 수정 (line 332-349)

```typescript
private simulateLanding(
  startX: number,
  startY: number,
  angle: number,
  power: number,
  wind: number,
  mass: number  // 추가
): { x: number; y: number } | null {
  // ... (기존 코드)
  const windAccel = wind * AIController.SIM_WIND_BASE / mass  // mass 반영
  // ... (나머지 동일)
}
```

#### 3. findBestShot()에서 현재 무기 mass 전달
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: `findBestShot()`에서 현재 무기의 mass를 가져와 `simulateLanding()`에 전달 (line 374-437)

```typescript
private findBestShot(
  character: Character,
  target: Character
): { angle: number; power: number; bestDist: number } {
  const pos = character.getPosition()
  const targetPos = target.getPosition()
  const wind = this.config.windSystem.getWindForce()
  const facingRight = targetPos.x > pos.x
  const mass = this.config.weaponManager.getCurrentWeapon().mass  // 추가

  // ... 1차 탐색
  const landing = this.simulateLanding(pos.x, pos.y, adjustedAngle, power, wind, mass)
  // ... 2차 탐색
  const landing = this.simulateLanding(pos.x, pos.y, adjustedAngle, power, wind, mass)
  // ...
}
```

#### 4. simulateBestFromPos()에서 default mass 사용
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: 이동 평가용이므로 현재 무기 mass 사용 (line 524-552)

```typescript
private simulateBestFromPos(
  fromX: number,
  fromY: number,
  targetPos: { x: number; y: number },
  wind: number
): number {
  const mass = this.config.weaponManager.getCurrentWeapon().mass  // 추가
  // ...
  const landing = this.simulateLanding(fromX, fromY, adjustedAngle, power, wind, mass)
  // ...
}
```

#### 5. startThinking()의 findBestShot() 호출 (line 139)
이 호출은 이동 필요 여부 판단용으로, 무기 선택 전에 실행된다. `getCurrentWeapon()`은 이전 턴의 무기 또는 기본 무기(Bazooka)를 반환하므로 문제 없음.

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 에러 없음: `npx tsc --noEmit`
- [x] 빌드 성공: `npm run build`

#### 수동 검증:
- [x] 강풍(±4~5)에서 AI 발사 시 투사체가 타겟 근처에 착탄 (기존보다 정확)
- [x] 무풍~약풍에서 AI 정확도 유사 (기존과 비슷)
- [x] AI가 이전보다 현저히 덜 자폭함 (mass 수정만으로도 개선 기대)

**Implementation Note**: 이 단계에서 mass 수정만으로 자폭 빈도가 크게 줄어들 것. 4단계의 명시적 자폭 방지는 추가 안전장치.

---

## 4단계: findBestShot()에 자폭 방지 필터링 추가

### 개요
`findBestShot()`의 그리드 탐색에서 착탄점이 AI 자신의 폭발 반경 내에 있는 조합을 필터링한다. 안전한 조합이 없으면 자폭 거리가 가장 먼 조합을 fallback으로 선택한다.

### 필요한 변경:

#### 1. findBestShot()에 자폭 거리 체크 추가
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: 그리드 탐색 루프 내에 자폭 필터 추가 (line 374-437)

```typescript
private findBestShot(
  character: Character,
  target: Character
): { angle: number; power: number; bestDist: number } {
  const pos = character.getPosition()
  const targetPos = target.getPosition()
  const wind = this.config.windSystem.getWindForce()
  const facingRight = targetPos.x > pos.x
  const weaponStats = this.config.weaponManager.getCurrentWeapon()
  const mass = weaponStats.mass
  const selfDangerRadius = weaponStats.explosionRadius * 1.5  // DamageSystem과 동일

  let bestAngle = 45
  let bestPower = 50
  let bestDist = Infinity

  // 자폭 위험 fallback (안전한 조합이 없을 때)
  let safeFallbackAngle = 45
  let safeFallbackPower = 50
  let safeFallbackSelfDist = 0  // 자신으로부터 가장 먼 착탄점

  // 1차 탐색: 대략 (step 5)
  for (let angle = -10; angle <= 80; angle += 5) {
    for (let power = 20; power <= 100; power += 5) {
      const adjustedAngle = facingRight ? angle : 180 - angle
      const landing = this.simulateLanding(
        pos.x, pos.y, adjustedAngle, power, wind, mass
      )
      if (!landing) continue

      // 자폭 거리 체크
      const selfDist = Math.hypot(landing.x - pos.x, landing.y - pos.y)

      // 자폭 위험 fallback 갱신 (자신으로부터 가장 먼 조합 기록)
      if (selfDist > safeFallbackSelfDist) {
        safeFallbackSelfDist = selfDist
        safeFallbackAngle = angle
        safeFallbackPower = power
      }

      // 자폭 위험 조합 스킵
      if (selfDist < selfDangerRadius) continue

      const dist = Math.hypot(
        landing.x - targetPos.x,
        landing.y - targetPos.y
      )
      if (dist < bestDist) {
        bestDist = dist
        bestAngle = angle
        bestPower = power
      }
    }
  }

  // 안전한 조합이 없으면 fallback 사용
  if (bestDist === Infinity) {
    bestAngle = safeFallbackAngle
    bestPower = safeFallbackPower
    bestDist = Infinity
  }

  // 2차 탐색: 미세 조정 (step 1, ±6 범위)
  // 2차에서도 동일한 자폭 필터 적용
  for (let angle = bestAngle - 6; angle <= bestAngle + 6; angle++) {
    for (let power = bestPower - 6; power <= bestPower + 6; power++) {
      if (power < 10 || power > 100) continue
      const adjustedAngle = facingRight ? angle : 180 - angle
      const landing = this.simulateLanding(
        pos.x, pos.y, adjustedAngle, power, wind, mass
      )
      if (!landing) continue

      const selfDist = Math.hypot(landing.x - pos.x, landing.y - pos.y)
      if (selfDist < selfDangerRadius) continue

      const dist = Math.hypot(
        landing.x - targetPos.x,
        landing.y - targetPos.y
      )
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

### 설계 결정:

1. **자폭 반경**: `explosionRadius * 1.5` — `DamageSystem.ts:91`의 자폭 판정 반경과 동일
2. **Fallback 전략**: 안전한 조합이 없으면 자신으로부터 가장 먼 착탄점을 가진 조합 선택 (최소한 자폭 피해 최소화)
3. **applyError() 미보호**: 사용자 결정에 따라 applyError()의 랜덤 오차로 인한 자폭은 AI "실수"로 허용

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 에러 없음: `npx tsc --noEmit`
- [x] 빌드 성공: `npm run build`

#### 수동 검증:
- [x] 강풍(±4~5)에서 AI가 자폭하는 빈도가 현저히 감소
- [x] AI가 타겟 근처에 적이 있을 때도 정상적으로 발사 (과도한 필터링 없음)
- [x] 가끔 AI가 실수로 자폭하는 것은 허용 (자연스러운 AI 행동)
- [x] 모든 안전 조합이 막힌 극단적 상황에서도 AI가 발사를 수행 (행동 멈춤 없음)

**Implementation Note**: 이 단계 완료 후 수동 테스트에서 자폭 빈도를 확인. 너무 자주 발생하면 selfDangerRadius를 `explosionRadius * 2.0`으로 조정 가능.

---

## 테스트 전략

### 수동 테스트 시나리오:

1. **강풍 자폭 테스트**:
   - 바람을 ±4~5로 설정하고 AI 턴 반복
   - 기존: 높은 자폭률 → 수정 후: 현저히 감소

2. **AI 정확도 테스트**:
   - 무풍~약풍(±1~2)에서 AI 발사 정확도
   - 기존과 유사하거나 개선되어야 함

3. **근거리 전투 테스트**:
   - 타겟이 AI 근처(< 100px)에 있을 때
   - Shotgun 선택 시 자폭 방지가 과도하게 작동하지 않는지 확인

4. **장거리 강풍 테스트**:
   - 타겟이 멀리 있고 강한 역풍일 때
   - AI가 올바르게 바람 보정하여 발사하는지 확인

5. **Fallback 동작 테스트**:
   - 좁은 골짜기에서 모든 안전 조합이 막힌 경우
   - AI가 멈추지 않고 발사하는지 확인

### setScale/setMass 순서 테스트:
- `console.log(this.spriteBody.mass)` 로 Bazooka 투사체 mass 확인
- 값이 `0.06`인지 확인 (setScale 후 재계산되지 않았는지)
- Shotgun: `setScale(0.3)` 후에도 mass가 `0.04`인지 확인

## 참조

- 연구 문서: `thoughts/shared/research/2026-02-14-ai-wind-calculation-self-destruct-research.md`
- AI 시스템 연구: `thoughts/shared/research/2026-02-13-ai-player-system-research.md`
- AI 구현 계획: `thoughts/shared/plans/2026-02-13-ai-player-system-implementation.md`
- `packages/game-core/src/systems/AIController.ts:324-437` — AI 시뮬레이션 및 그리드 탐색
- `packages/game-core/src/entities/Projectile.ts:27-60` — 투사체 생성 및 물리 설정
- `packages/game-core/src/weapons/WeaponTypes.ts:1-39` — 무기 정의
- `packages/game-core/src/systems/DamageSystem.ts:89-99` — 자폭 데미지 처리
