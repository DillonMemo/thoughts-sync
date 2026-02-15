# Parallax 배경 기반 게임 시스템 — 구현 완료 문서

> 최종 업데이트: 2026-02-12

## 개요

Freepik 스타일 Parallax PNG 에셋(`assets/background/1~8`)을 기반으로 한 배경 + 이미지 기반 지형 시스템. 기존 Graphics API 절차 생성 배경/데코레이션은 완전 제거되었고, 이미지 alpha 채널에서 충돌 데이터를 추출하는 방식으로 전환됨.

---

## 아키텍처

### 씬 흐름

```
BootScene (에셋 로딩 + 배경 선택) → GameScene (게임 플레이) → BootScene (재시작)
```

### 핵심 시스템

| 시스템 | 파일 | 설명 |
|--------|------|------|
| 에셋 로딩 | `BootScene.ts` | 랜덤 배경 선택, 레이어 로딩, 지형/다리 설정 |
| 배경 렌더링 | `GameScene.ts:130-198` | 이미지 레이어 + 마우스 패럴랙스 + 물 |
| 지형 데이터 | `TerrainData.ts` | 3-state 지형 (0=빈, 1=고체, 2=시각만) |
| 지형 렌더링 | `TerrainRenderer.ts` | 이미지 기반 렌더링, `isVisible()`로 표시 |
| 데미지 | `DamageSystem.ts` | 폭발 데미지 + 자폭 + 익사 + 넉백 |
| 투사체 | `Projectile.ts` | `shooterId` 추적, 무기별 폭발 이벤트 |
| 무기 | `WeaponManager.ts` | 발사 시 `shooterId` 전달 |
| 캐릭터 | `Character.ts` | 물리 바디, 맵 경계 제한, 이동/점프 |

---

## 배경 시스템

### 에셋 구조

- 8세트, 각 5~9개 레이어 (1920x1080 PNG)
- 레이어 번호: 낮을수록 전경(앞), 높을수록 배경(뒤/하늘)
- 에셋 경로: `assets/background/{setId}/{i}_layer.png`
- 심볼릭 링크: `apps/web/public/assets` → `../../../assets`

### 배경 세트 설정 (`BootScene.ts`)

```typescript
const BACKGROUND_LAYER_COUNTS: Record<number, number> = {
  1: 6, 2: 6, 3: 6, 4: 6, 5: 7, 6: 9, 7: 8, 8: 5
}
```

### Depth 맵

```
-25 ~ -10   배경 레이어 (하늘 ~ 가까운 배경, 지형 레이어 제외)
-5          물 본체 (반투명 그라디언트)
-4          물 수면 파동 (애니메이션)
 0          지형 스프라이트
10          전경 레이어 (1_layer.png)
```

### Parallax 공식 (`GameScene.ts:182-198`)

```
offsetX = (pointer.x - centerX) / width    // -0.5 ~ 0.5
offsetY = (pointer.y - centerY) / height

배경: layer.x = centerX - offsetX * parallaxFactor * 15  (최대 ±7.5px)
전경: fg.x = centerX - offsetX * 20  (최대 ±10px)
```

- `oversize = 1.04` — 패럴랙스 이동 시 가장자리 노출 방지

---

## 이미지 기반 지형

### 3-State 지형 시스템 (`TerrainData.ts`)

| 상태 | 값 | 의미 | `isSolid()` | `isVisible()` |
|------|---|------|-------------|----------------|
| 빈 공간 | 0 | 투명 (alpha < 128) | false | false |
| 고체 | 1 | 충돌 + 렌더링 | true | true |
| 시각만 | 2 | 렌더링만 (통과 가능) | false | true |

### 지형 생성 흐름

```
1. BootScene: 배경 세트 선택 → terrainLayer 번호 Registry에 저장
2. GameScene.createTerrain():
   a. 지형 레이어 PNG → 캔버스에 게임 해상도로 그리기
   b. alpha >= 128 → state 1 (고체), 나머지 → state 0
   c. removeNarrowBridges() → 좁은 구간 state 2로 변환
   d. TerrainRenderer에 이미지 데이터 전달
```

### 지형 레이어 매핑 (`BootScene.ts`)

| 세트 | 지형 레이어 | 설명 |
|------|-----------|------|
| 1 | 2 | |
| 2 | 3 | |
| 3 | 2 | 숲 - 나무 줄기 다수 |
| 4 | 2 | |
| 5 | 6 | |
| 6 | 6 | |
| 7 | 3 | 오른쪽 소나무 |
| 8 | 3 | 양쪽 큰 나무 |

### 좁은 다리 제거 설정 (`BootScene.ts`)

```typescript
const TERRAIN_BRIDGE_CONFIG: Record<number, { minWidth: number; searchRange: number }> = {
  3: { minWidth: 35, searchRange: 350 },  // 숲 - 나무 줄기 (5-33px), 높이 600px+
  7: { minWidth: 55, searchRange: 200 },  // 소나무 줄기 (20-48px), 높이 314px
  8: { minWidth: 50, searchRange: 100 },  // 큰 나무 줄기 (~37px)
}
```

`removeNarrowBridges(minWidth, searchRange)`: 각 Y행을 스캔하여 고체 구간의 폭이 `minWidth` 미만이면 state 2로 변환. `searchRange`는 좌우 탐색 범위.

---

## 플레이어 스폰 시스템

### 동적 스폰 배치 (`GameScene.ts:276-319`)

고정 비율(27%/73%)을 사용하지 않고, 실제 지형을 스캔하여 스폰 위치를 결정.

**알고리즘:**
1. 10%~90% 범위를 1% 단위로 스캔
2. 각 X에서 `findSurfaceY()` → 첫 번째 고체(state 1) 픽셀
3. 수면(88%) - 50px 이상 높은 위치만 후보로 수집
4. 가장 멀리 떨어진 두 후보에 플레이어 배치

**세트별 검증 결과:**

| 세트 | P1 위치 | P2 위치 | 거리 | 유효 후보 |
|------|---------|---------|------|----------|
| 1 | 10% (y=729) | 90% (y=675) | 1536px | 81 |
| 2 | 10% (y=828) | 90% (y=684) | 1536px | 81 |
| 3 | 10% (y=456) | 90% (y=504) | 1536px | 77 |
| 4 | 10% (y=665) | 90% (y=646) | 1536px | 81 |
| 5 | 10% (y=681) | 90% (y=442) | 1536px | 81 |
| 6 | 20% (y=891) | 83% (y=896) | 1209px | 53 |
| 7 | 10% (y=695) | 90% (y=430) | 1536px | 46 |
| 8 | 10% (y=160) | 90% (y=345) | 1536px | 47 |

> 세트 7: 이전 고정 73% 배치 시 P2가 수면 아래(y=994 > 수면 950)에 스폰 → 즉사. 동적 배치로 해결.

---

## 물 시스템

### 물 렌더링 (`GameScene.ts:156-175`)

- **수면 위치**: `height * 0.88` (1080px 기준 y=950)
- **본체**: 반투명 그라디언트 (alpha 0.25→0.70), depth -5
- **수면 파동**: 실시간 사인 파 애니메이션, depth -4
  - 메인 파동: `sin(x*0.015 + t*2) * 4 + sin(x*0.035 + t*1.3) * 2`
  - 보조 파동: 8px 아래, 더 연한 흰색
  - 반짝임: 8개 이동하는 점

### 익사 판정 (`DamageSystem.ts:49`)

```typescript
if (pos.y > this.waterLevel) → 즉사
```

캐릭터 중심이 수면에 도달하면 즉사 (반경 18px 고려, 실질적으로 몸의 절반 잠기면 사망).

---

## 데미지 + 자폭 시스템

### 폭발 이벤트 체인

```
GameScene.handleFire(character)
  → WeaponManager.fire(..., character.playerId)
    → new Projectile({ ..., shooterId })
      → 충돌 시 scene.events.emit("explosion", { ..., shooterId })
        → DamageSystem.handleExplosion(event)
```

### ExplosionEvent 인터페이스 (`DamageSystem.ts:6-12`)

```typescript
interface ExplosionEvent {
  x: number; y: number
  radius: number; damage: number
  shooterId: number  // 발사한 플레이어 ID
}
```

### 자폭 데미지 로직 (`DamageSystem.ts:88-100`)

```
발사자(isSelf = true):
  effectiveRadius = radius × 1.5  (확장된 자폭 범위)
  damageMultiplier = max(0.25, 1 - distance/effectiveRadius)  (최소 25% 보장)

다른 플레이어:
  effectiveRadius = radius  (기본 폭발 반경)
  damageMultiplier = 1 - distance/effectiveRadius  (거리 비례 감쇠)
```

### 무기별 자폭 데미지

| 무기 | 데미지 | 폭발 반경 | 자폭 effective | 최소 자폭 데미지 |
|------|--------|----------|---------------|-----------------|
| Bazooka | 50 | 40px | 60px | 13 (25%) |
| Grenade | 35 | 50px | 75px | 9 (25%) |
| Shotgun | 25 | 15px | 22.5px | 7 (25%) |

### 넉백 — 무기 데미지에 비례 (`DamageSystem.ts:126-147`)

```typescript
baseForce = event.damage * 0.001   // damage 50 → 0.05
distanceFalloff = 1 - distance / effectiveRadius
knockbackForce = baseForce * distanceFalloff
```

| 무기 | base force | 비고 |
|------|-----------|------|
| Bazooka (50) | 0.050 | 가장 강한 넉백 |
| Grenade (35) | 0.035 | 중간 |
| Shotgun (25) | 0.025 | 가장 약한 넉백 |

새 무기 추가 시 `damage` 값만 설정하면 넉백이 자동 비례.

---

## 맵 경계 제한 (`Character.ts:476-484`)

```typescript
// 매 프레임 update()에서 체크
const mapWidth = this.scene.scale.width
if (pos.x < 18) → 위치 고정 + X속도 0
if (pos.x > mapWidth - 18) → 위치 고정 + X속도 0
```

캐릭터 반경(18px) 고려하여 좌우 화면 밖 이동 차단. 넉백으로 밀려나도 경계에서 멈춤.

---

## 캐릭터 물리 (`Character.ts`)

| 속성 | 값 | 설명 |
|------|---|------|
| 반경 | 18px | `matter.bodies.circle(..., 18)` |
| friction | 0.8 | |
| frictionStatic | 10 | 경사면 미끄러짐 방지 |
| frictionAir | 0.02 | |
| restitution | 0 | 튕김 없음 |
| MAX_FALL_SPEED | 10 | 최대 낙하 속도 |
| 경사 감쇠 | vx * 0.3 | 정지 시 `\|vx\| < 1.5` 이면 감쇠 |

---

## 무기 시스템

### 무기 스탯 (`WeaponTypes.ts`)

| 무기 | 데미지 | 폭발 반경 | 탄약 | 특징 |
|------|--------|----------|------|------|
| Bazooka | 50 | 40px | 무제한 | 충돌 즉시 폭발 |
| Grenade | 35 | 50px | 3발 | 2.5초 퓨즈, 바운스 |
| Shotgun | 25 | 15px | 2발 | 5발 산탄 (±10° 분산) |

### 투사체 스폰 (`GameScene.ts:488-490`)

- 캐릭터 중심에서 에임 방향으로 **35px** 오프셋
- `Projectile.shooterId`: 발사자 추적 → 폭발 이벤트에 포함

---

## 주요 파일 참조

| 파일 | 라인 | 내용 |
|------|------|------|
| `BootScene.ts` | 4-36 | 상수: LAYER_COUNTS, TERRAIN_LAYERS, BRIDGE_CONFIG |
| `BootScene.ts` | 66-137 | preload(): 에셋 로딩 |
| `GameScene.ts` | 94-128 | createTerrain(): 이미지→지형 변환 |
| `GameScene.ts` | 130-198 | createBackground(): 배경+물+패럴랙스 |
| `GameScene.ts` | 201-248 | updateWater(): 수면 파동 애니메이션 |
| `GameScene.ts` | 276-319 | findSpawnPositions(): 동적 스폰 배치 |
| `GameScene.ts` | 477-500 | handleFire(): shooterId 전달 |
| `DamageSystem.ts` | 6-12 | ExplosionEvent (shooterId 포함) |
| `DamageSystem.ts` | 72-124 | handleExplosion(): 자폭 데미지 + 넉백 |
| `DamageSystem.ts` | 42-69 | checkWaterDeath(): 익사 판정 |
| `Projectile.ts` | 3-12 | ProjectileConfig (shooterId) |
| `WeaponTypes.ts` | 11-39 | WeaponRegistry |
| `Character.ts` | 476-484 | 맵 경계 제한 |
| `TerrainData.ts` | - | 3-state 시스템, removeNarrowBridges() |
| `TerrainRenderer.ts` | - | 이미지 기반 렌더링, isVisible() |

---

## 검증 상태

### 자동화 검증
- [x] 타입 체크 통과: `npx turbo typecheck --filter=@repo/game-core`
- [x] 빌드 성공: `npx turbo build --filter=web`

### 수동 검증
- [x] 8세트 배경 이미지 렌더링 정상
- [x] 마우스 패럴랙스 깊이감 표현
- [x] 전경 레이어 지형/캐릭터 위 표시
- [x] 물 반투명 + 파동 애니메이션 정상
- [x] 익사 메카닉 정상 (수면 도달 시 즉사)
- [x] 재시작 시 랜덤 배경 교체
- [x] 지형 파괴 정상 동작
- [x] 나무 줄기 통과 (세트 3, 7, 8)
- [x] 자폭 데미지 동작 (최소 25% 보장)
- [x] 넉백 무기 데미지에 비례
- [x] 동적 스폰 — 세트 7 P2 즉사 해결
- [x] 맵 경계 밖 이동 차단
