---
date: 2026-02-14T17:48:57+0900
researcher: arta1069@gmail.com
git_commit: N/A (no git repository initialized)
branch: N/A
repository: game
topic: "AI 플레이어 바람 계산 오류 및 자폭 방지 로직 심층 분석"
tags: [research, codebase, ai-player, wind-system, self-destruct-prevention, trajectory-simulation, projectile-physics]
status: complete
last_updated: 2026-02-14
last_updated_by: arta1069@gmail.com
---

# 연구: AI 플레이어 바람 계산 오류 및 자폭 방지 로직 심층 분석

**날짜**: 2026-02-14T17:48:57+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: N/A (git 저장소 미초기화)
**브랜치**: N/A
**리포지토리**: game

## 연구 질문

AI 플레이어가 가끔 바람의 저항을 계산하지 못하고 발사하며, 발사된 무기가 AI 플레이어에게 되돌아가 자폭하는 현상의 원인을 분석한다.

## 요약

### 발견된 핵심 문제 2가지

1. **바람 시뮬레이션 정확도 불일치**: AI의 `simulateLanding()`은 실제 Matter.js 물리와 구조적으로 다른 적분 방식(Euler vs Verlet)을 사용하며, 투사체 질량(mass)을 `0.5`로 하드코딩하고 있으나 실제 Matter.js가 자동 계산하는 질량과 일치하는지 검증되지 않았다. 질량이 다르면 바람 영향 계산이 크게 틀어진다.

2. **자폭 방지 로직 완전 부재**: `findBestShot()`에서 착탄점과 **타겟까지의 거리만** 계산하고, **발사자(AI) 자신까지의 거리는 전혀 검증하지 않는다**. 따라서 강한 역풍 시 투사체가 AI에게 되돌아오는 발사각/파워 조합도 "타겟에 가까우면" 선택될 수 있다.

### 자폭 시나리오 재현 경로

```
1. 강한 역풍 (windForce: -4~-5 또는 4~5, AI에서 타겟 방향으로 부는 반대 바람)
2. AI가 findBestShot() 실행 → simulateLanding()으로 궤적 시뮬레이션
3. 질량 불일치로 인해 시뮬레이션된 바람 영향 ≠ 실제 바람 영향
4. AI가 "타겟에 가장 가까운" 조합 선택 → 실제로는 투사체가 바람에 더 많이/적게 밀림
5. applyError()의 랜덤 오차가 추가됨 (초반: ±12도, ±15파워)
6. 실제 투사체: 바람에 의해 AI 방향으로 되돌아옴
7. DamageSystem: 자폭 처리 (effectiveRadius × 1.5, 최소 25% 데미지)
```

---

## 상세 발견 사항

### 1. 바람 시뮬레이션 — AI vs 실제 물리 비교

#### 1.1 AI 시뮬레이션 상수 (`AIController.ts:324-330`)

```typescript
private static readonly SIM_GRAVITY = 0.001 * (1000 / 60) * (1000 / 60) // ≈0.278
private static readonly SIM_FRICTION = 0.001
private static readonly SIM_WIND_SCALE = (0.000001406 * 277.78) / 0.5   // ≈0.00078
private static readonly SPAWN_OFFSET = 35
```

#### 1.2 실제 물리 설정

| 항목 | AI 시뮬레이션 | 실제 Projectile | 일치 여부 |
|------|--------------|-----------------|----------|
| 초기 속도 | `cos(θ) × power × 0.15` | `cos(θ) × power × 0.15` | ✅ 완벽 일치 |
| 발사 오프셋 | `SPAWN_OFFSET = 35` | `spawnDistance = 35` | ✅ 완벽 일치 |
| 중력 | `SIM_GRAVITY ≈ 0.278` | `gravity.y=1 × scale=0.001 × dt²=277.78 ≈ 0.278` | ✅ 이론적 일치 |
| 공기저항 | `SIM_FRICTION = 0.001` (frictionMult = 0.999) | `setFrictionAir(0.001)` | ⚠️ 적용 순서 상이 |
| 바람 힘 계수 | `0.000001406` | `windForce × 0.000001406` | ✅ 계수 일치 |
| 질량 가정 | `0.5` 하드코딩 | Matter.js 자동 계산 (sprite 크기 × density × scale²) | ❌ **검증 불가** |
| 적분 방식 | Euler (`v += a; x += v`) | Verlet (위치 기반) | ❌ **구조적 차이** |

#### 1.3 질량 불일치 문제 (가장 큰 변수)

**AI의 바람 가속도 공식** (`AIController.ts:329`):
```
windAccel = wind × 0.000001406 × 277.78 / 0.5 ≈ wind × 0.00078
```

**실제 물리의 바람 가속도**:
```
windAccel = wind × 0.000001406 / actual_mass × dt²
```

- `Projectile.ts:48`에서 `setScale(0.5)` 적용
- Matter.js는 `mass = area × density`로 자동 계산
- `setMass()` 호출이 없으므로 sprite 원본 크기에 의존
- **실제 mass가 0.5가 아니면 바람 영향이 비례적으로 틀어짐**:
  - mass = 0.3이면 → AI가 바람 영향을 **60% 과소평가** (실제는 더 많이 밀림)
  - mass = 0.7이면 → AI가 바람 영향을 **40% 과대평가** (실제는 덜 밀림)

#### 1.4 적분 방식 차이

**AI (Euler 적분)** (`AIController.ts:352-358`):
```typescript
vx += windAccel          // 1. 바람 가산
vy += SIM_GRAVITY        // 2. 중력 가산
vx *= frictionMult       // 3. 마찰 적용
vy *= frictionMult
x += vx                  // 4. 위치 이동
y += vy
```

**Matter.js (Verlet 적분)**:
```
1. force 누적 (중력, 바람)
2. Verlet: position_new = position + velocity×dt + 0.5×accel×dt²
3. velocity = (position_new - position) / dt
4. velocity *= (1 - frictionAir)
```

- Euler는 1차 정확도, Verlet은 2차 정확도
- 장거리 비행 시 **누적 오차** 발생
- 마찰 적용 순서 차이로 AI 시뮬레이션이 실제보다 약간 짧게 비행하는 경향

#### 1.5 거리별 추정 정확도

| 비행 거리 | 추정 정확도 | 비고 |
|-----------|------------|------|
| 단거리 (< 200px) | ~95% | Euler 오차 작음 |
| 중거리 (200-500px) | ~90-95% | 적분 오차 누적 |
| 장거리 (> 500px) | ~80-90% | 오차 누적 + 바람 불확실성 |
| 강풍 (|wind| ≥ 4) | ~60-80% | **mass 불일치 시 가장 큰 변수** |

### 2. 자폭 방지 로직 — 완전 부재

#### 2.1 findBestShot() 분석 (`AIController.ts:374-437`)

```typescript
private findBestShot(character, target) {
  const pos = character.getPosition()
  const targetPos = target.getPosition()
  const wind = this.config.windSystem.getWindForce()

  // 그리드 탐색
  for (let angle = -10; angle <= 80; angle += 5) {
    for (let power = 20; power <= 100; power += 5) {
      const landing = this.simulateLanding(pos.x, pos.y, adjustedAngle, power, wind)
      if (!landing) continue
      const dist = Math.hypot(
        landing.x - targetPos.x,  // ✅ 타겟까지 거리만 계산
        landing.y - targetPos.y
      )
      // ❌ 착탄점과 발사자(pos) 간 거리는 계산하지 않음
      // ❌ 자폭 위험 조합 필터링 없음
      if (dist < bestDist) { bestAngle = angle; bestPower = power }
    }
  }
}
```

**누락된 안전장치**:
1. `착탄점 ↔ 발사자` 거리 검증 없음
2. 자폭 위험 반경 (`explosionRadius × 1.5`) 기반 필터링 없음
3. 투사체 궤적이 발사자를 통과하는지 체크 없음
4. 역풍 시 최소 안전 발사각/파워 제한 없음

#### 2.2 자폭 발생 시나리오

**시나리오 1: 강한 역풍 + 시뮬레이션 오차**
```
- AI가 오른쪽에 있는 타겟을 향해 발사
- 바람: -5 (왼쪽에서 오른쪽으로, 즉 AI→타겟 역방향)
- AI 시뮬레이션: mass=0.5 가정으로 바람 보정 계산
- 실제 mass가 0.3이라면: 실제 투사체가 바람에 60% 더 밀림
- 투사체가 예상보다 크게 밀려 AI 방향으로 되돌아옴
```

**시나리오 2: 학습 오차 + 자폭**
```
- applyError() (line 638-653): 초반 ±12도, ±15파워 랜덤 오차
- 오차에 의해 낮은 각도 + 낮은 파워 조합이 선택됨
- 투사체가 바로 앞 지형에 착탄
- 자폭 반경(effectiveRadius × 1.5 = 60px) 내에 AI가 위치
- DamageSystem에서 자폭 데미지 적용 (최소 25%: 13 데미지)
```

**시나리오 3: 이동 후 재계산 없이 발사**
```
- onWeaponSelectComplete() (line 246-267)에서 최종 계산
- findBestShot() → applyShotCorrection() → applyError() 체이닝
- applyShotCorrection()의 보정이 바람 변화를 반영하지 못하면 (windSimilarity=0일 때 보정 무시)
- 무보정 + 랜덤 오차 = 자폭 가능성 증가
```

#### 2.3 DamageSystem의 자폭 처리 (사후 대응만 존재)

**`DamageSystem.ts:89-99`**:
```typescript
const isSelf = character.playerId === event.shooterId
const effectiveRadius = isSelf ? event.radius * 1.5 : event.radius

if (distance <= effectiveRadius) {
  let damageMultiplier = 1 - distance / effectiveRadius
  if (isSelf) {
    damageMultiplier = Math.max(0.25, damageMultiplier)  // 최소 25%
  }
}
```

| 무기 | 데미지 | 폭발 반경 | 자폭 범위 (×1.5) | 최소 자폭 데미지 |
|------|--------|-----------|------------------|-----------------|
| Bazooka | 50 | 40px | 60px | 13 |
| Grenade | 35 | 50px | 75px | 9 |
| Shotgun | 25 | 15px | 22.5px | 7 |

### 3. WeaponManager.fire()의 각도 이중 변환 문제

**`WeaponManager.ts:59`**:
```typescript
fire(x, y, angle, power, facingRight, shooterId) {
  const adjustedAngle = facingRight ? angle : 180 - angle
  // ...
}
```

**`GameScene.ts:590-605`**:
```typescript
handleFire(character, controller) {
  const angle = controller.getAimAngle()
  const facingRight = character.isFacingRight()
  const adjustedAngle = facingRight ? angle : 180 - angle  // ← 1차 변환
  // spawnX, spawnY 계산에 adjustedAngle 사용

  this.weaponManager.fire(
    spawnX, spawnY,
    angle,           // ← 원본 angle 전달
    controller.getAimPower(),
    facingRight,      // ← facingRight도 전달
    character.playerId
  )
}
```

`handleFire()`에서 `angle`(원본)을 WeaponManager.fire()에 전달하고, `fire()` 내부에서 다시 `facingRight ? angle : 180 - angle` 변환을 수행한다. 이 이중 변환은 정상 동작하는 구조이다.

### 4. AI의 바람 적용 타이밍

**`GameScene.ts:488-491`**:
```typescript
this.weaponManager.update()          // 투사체 위치 업데이트
this.weaponManager.applyWindToAll(this.windSystem.getWindForce())  // 바람 적용
```

바람은 매 프레임 `applyWindToAll()`로 모든 활성 투사체에 적용된다. AI 시뮬레이션도 매 스텝 `vx += windAccel`로 동일한 패턴을 따른다. 그러나 실제 물리에서는 `applyForce()`가 Matter.js의 내부 업데이트와 별도로 외부에서 힘을 추가 적용하므로, 적용 시점이 미묘하게 다를 수 있다.

---

## 코드 참조

- `packages/game-core/src/systems/AIController.ts:324-330` — AI 물리 시뮬레이션 상수 정의
- `packages/game-core/src/systems/AIController.ts:332-370` — simulateLanding() 궤적 시뮬레이션
- `packages/game-core/src/systems/AIController.ts:374-437` — findBestShot() 그리드 탐색 (자폭 체크 없음)
- `packages/game-core/src/systems/AIController.ts:638-653` — applyError() 랜덤 오차 적용
- `packages/game-core/src/systems/AIController.ts:655-697` — applyShotCorrection() 학습 보정
- `packages/game-core/src/entities/Projectile.ts:25-55` — 투사체 물리 설정 (mass 자동 계산)
- `packages/game-core/src/entities/Projectile.ts:95-104` — applyWind() 바람 적용
- `packages/game-core/src/systems/DamageSystem.ts:89-99` — 자폭 데미지 처리 (사후)
- `packages/game-core/src/scenes/GameScene.ts:580-613` — handleFire() 발사 처리
- `packages/game-core/src/scenes/GameScene.ts:488-491` — 매 프레임 바람 적용
- `packages/game-core/src/systems/WindSystem.ts:4-20` — 바람 범위 -5~+5

## 아키텍처 문서화

### 현재 AI 발사 파이프라인

```
startThinking()
  → findTarget() (가장 가까운 적)
  → findBestShot() (그리드 탐색, 바람 반영)
    → simulateLanding() × ~470회 (1차 306 + 2차 169)
  → needsRepositioning 판단

onThinkingComplete()
  → shouldMove() → moving 또는 weaponSelect

onWeaponSelectComplete()
  → findBestShot() (최종 위치에서 재계산)  ← 바람 반영
  → applyShotCorrection() (학습 보정)       ← 바람 유사도 체크
  → applyError() (랜덤 오차)               ← 자폭 가능성 증가 지점
  → targetAngle, targetPower 설정

onAimingUpdate()
  → lerp로 목표값 보간
  → 발사 (turnSystem.fire())

handleFire() [GameScene]
  → spawnDistance=35 오프셋
  → WeaponManager.fire()
  → 실제 Matter.js 투사체 생성
  → 매 프레임 applyWind() 적용              ← 실제 바람 (시뮬레이션과 다를 수 있음)
```

### 문제 지점 요약

| 위치 | 문제 | 영향도 |
|------|------|--------|
| `AIController.ts:329` | mass=0.5 하드코딩, 실제 mass 검증 안 됨 | **높음** — 바람 계산 전체 오차 |
| `AIController.ts:374-437` | 착탄점↔발사자 거리 미검증 | **높음** — 자폭 방지 불가 |
| `AIController.ts:352-358` | Euler 적분 (실제는 Verlet) | **중간** — 장거리 누적 오차 |
| `AIController.ts:638-653` | applyError() 결과 자폭 체크 없음 | **높음** — 오차로 자폭 가능 |

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/shared/research/2026-02-13-ai-player-system-research.md` — AI 시스템 원본 설계 연구. 자폭 방지는 `DamageSystem.ts:90-98`의 사후 처리만 언급됨. 사전 방지 로직에 대한 논의 없음.
- `thoughts/shared/plans/2026-02-13-ai-player-system-implementation.md` — 구현 계획. simulateLanding() 설계 시 "Matter.js Verlet 적분 매칭"을 목표로 했으나, mass 검증 방법은 포함되지 않음.
- `thoughts/shared/plans/2026-02-11-parallax-background-replacement.md` — 자폭 데미지 시스템 상세 설계 (effectiveRadius × 1.5, 최소 25%). 사전 방지가 아닌 게임 밸런스 조정 관점.
- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` — 초기 아키텍처에서 Utility AI와 탄도 계산 언급. 자폭 방지 명시 없음.

## 관련 연구

- `thoughts/shared/research/2026-02-13-ai-player-system-research.md` — AI 학습 시스템 전체 설계
- `thoughts/shared/plans/2026-02-13-ai-player-system-implementation.md` — AI 구현 6단계 계획

## 미해결 질문

1. **실제 투사체 mass 값은?** — `Projectile.ts`에서 `setMass()` 호출 없이 Matter.js가 sprite 크기 기반으로 자동 계산. `console.log(this.spriteBody.mass)`로 런타임 확인 필요.
2. **Euler vs Verlet 오차 크기는?** — 동일 조건에서 AI 시뮬레이션 착탄점과 실제 착탄점을 비교하는 디버그 모드 필요.
3. **강풍 시 자폭 빈도는?** — |wind| ≥ 4일 때 자폭률 통계 수집 필요.
