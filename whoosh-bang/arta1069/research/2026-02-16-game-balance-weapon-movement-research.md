---
date: 2026-02-16T01:01:38+0900
researcher: arta1069@gmail.com
git_commit: f637a37
branch: main
repository: whoosh-bang
topic: "게임 밸런스 연구: 바주카 무기 수치 및 이동 제한 시스템"
tags: [research, codebase, game-balance, weapon-system, movement-system, bazooka]
status: complete
last_updated: 2026-02-16
last_updated_by: arta1069
last_updated_note: "밸런스 논의 결과 반영 - 바주카 너프 수치 및 이동 게이지 설계 확정"
---

# 연구: 게임 밸런스 - 바주카 무기 수치 및 이동 제한 시스템

**날짜**: 2026-02-16T01:01:38+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: f637a37
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

1. 바주카(기본 무기)의 대미지, 폭발 반경, 지형 파괴 수치가 향후 무기 다양화를 고려할 때 적절한지 파악
2. 현재 플레이어 이동/점프 시스템에 이동 제한(게이지)이 없는 상태에서, 이동 게이지 시스템 도입을 위한 현재 구현 파악

## 요약

### 바주카 무기 현황
바주카는 현재 3종 무기 중 **가장 높은 대미지(50)**와 **무제한 탄약**을 가진 기본 무기입니다. 폭발 반경(40px)은 중간 수준이며, 지형 파괴도 반경 40px 원형으로 수행합니다. 비교표:

| 무기 | 대미지 | 폭발 반경 | 탄약 | 질량 | 지형 파괴 |
|------|--------|----------|------|------|----------|
| **Bazooka** | **50** | 40 | **무제한** | 0.06 | 반경 40px 원형 |
| Grenade | 35 | **50** | 3발 | 0.25 | 반경 50px 원형 |
| Shotgun | 25 | 15 | 2발 | 0.04 | 반경 15px × 5발 |

바주카는 대미지와 탄약 모두에서 우위이며, Grenade는 폭발 반경만 더 넓습니다.

### 이동 시스템 현황
현재 **이동 게이지 시스템은 존재하지 않습니다**. 플레이어는 30초 턴 시간 내에서 무제한 이동/점프가 가능합니다. 이동 속도는 1.25 픽셀/프레임이고, 점프 속도는 -5.3입니다. 경사면 자동 등반, 코요테 타임(150ms) 등의 보조 메커니즘이 있습니다.

---

## 상세 발견 사항

### 1. 바주카 무기 시스템

#### 1.1 기본 수치 (WeaponTypes.ts:13-22)
```
damage: 50, explosionRadius: 40, ammo: -1, mass: 0.06
```
- `ammo: -1`은 무제한을 의미 (WeaponManager.ts:26-37에서 0이 아니면 선택 가능)
- 기본 무기로 설정됨 (WeaponManager.ts:10 `private currentWeapon: string = "bazooka"`)

#### 1.2 투사체 물리 (Projectile.ts:26-59)
- 파워 변환: `power * 0.15` (POWER_MULTIPLIER)
- 마찰: 0.01, 공기 저항: 0.001
- 중력 스케일: 1.0 (기본 중력 영향)
- 질량 0.06으로 바람 영향을 크게 받음

#### 1.3 폭발 처리 (Bazooka.ts:17-48)
- 충돌 즉시 폭발 (`onCollision` → `explode()`)
- `"explosion"` 이벤트 발생 → DamageSystem이 수신

#### 1.4 대미지 계산 (DamageSystem.ts:77-124)
- **일반 대상**: `damage * (1 - distance / radius)` → 중심 50, 반경 끝 0
- **자폭 범위**: `radius * 1.5` = 60px, 최소 25% 대미지 보장 (12.5 → 12)
- **넉백**: `damage * 0.001 * (1 - distance / effectiveRadius)` → 캐릭터에 `force * 100` 적용

#### 1.5 지형 파괴 (TerrainData.ts:34-47)
- 원형 파괴: `dx² + dy² ≤ radius²` 조건으로 픽셀 제거
- 바주카 파괴 면적: ~5,026 픽셀 (π × 40²)
- 파괴 후 충돌체 전체 재계산 (TerrainRenderer.ts:99-131)

#### 1.6 다른 무기와의 비교

**Bazooka vs Grenade**:
- 바주카가 대미지 43% 더 높음 (50 vs 35)
- Grenade는 폭발 반경 25% 더 넓음 (50 vs 40)
- Grenade는 2.5초 타이머 + 바운스(0.6)로 간접 공격 특화
- Grenade 지형 파괴가 57% 더 넓음 (π×50² vs π×40²)

**Bazooka vs Shotgun**:
- 샷건은 5발 산탄(확산 -10~+10도, 파워 80~120%)
- 이론상 최대 125 대미지(25×5) 가능하나 현실적으로 어려움
- 샷건 폭발 반경 15px × 5 = 개별 작지만 넓은 범위 커버

### 2. 플레이어 이동/점프 시스템

#### 2.1 이동 수치 (Character.ts:63-65)
```
BASE_MOVE_SPEED = 1.25    // 픽셀/프레임
JUMP_VELOCITY = -5.3      // y축 (위)
MAX_FALL_SPEED = 10       // 최대 낙하 속도
```

#### 2.2 현재 이동 제한 방식
- **시간 제한만 존재**: 턴당 30초 (GameScene.ts:367-369)
- **거리/게이지 제한 없음**: 30초 내 무제한 이동 가능
- **상태 기반 차단**: "playerTurn" 상태에서만 이동 가능 (InputController.ts:108-114)
- **조준 중 이동 불가**: showWeapon() 시 물리 바디를 static으로 고정 (Character.ts:210-230)

#### 2.3 경사면 속도 조절 (Character.ts:359-383)
- 오르막: `max(0.4, 1.0 - slopeRatio * 0.75)` → 최대 60% 감속
- 내리막: `min(1.2, 1.0 - slopeRatio * 0.15)` → 최대 20% 가속

#### 2.4 점프 메커니즘 (Character.ts:404-416)
- 단일 점프만 가능 (다중 점프 불가)
- 코요테 타임: 150ms (Character.ts:53)
- 자동 경사 등반: 2px 초과 높이차 시 자동 (Character.ts:336-350)

#### 2.5 물리 바디 설정 (Character.ts:74-81)
- 원형 충돌체 반경 18px
- friction: 0.8, frictionStatic: 10, frictionAir: 0.02, restitution: 0

#### 2.6 이동 관련 데이터 흐름
```
GameScene.update() → InputController.update() → turnSystem.getState()
  └─ "playerTurn" → handleMovement() → character.moveLeft()/moveRight()
                   → handleJump() → character.jump()
  └─ "aiming" → 이동 차단, 조준만 가능
```

#### 2.7 AI의 이동 (AIController.ts:184-244)
- AI는 조건부 이동: 물/가장자리 회피, 지형 차단 시만 이동
- 전방 지형 스캔(10px, 25px, 45px)으로 자동 점프 결정
- 이동 시간 제한: 800~1500ms

### 3. 턴 시스템 (TurnSystem.ts)

#### 3.1 턴 상태 흐름
```
playerTurn (이동 가능) → aiming (이동 불가) → firing → animating → 다음 플레이어 턴
```

#### 3.2 턴 설정
- 턴 시간: 30000ms (30초)
- 상태: "playerTurn" | "aiming" | "firing" | "animating" | "gameOver"

---

## 코드 참조

### 무기 시스템
- `packages/game-core/src/weapons/WeaponTypes.ts:12-43` - 무기 레지스트리 (3종 무기 수치)
- `packages/game-core/src/weapons/Bazooka.ts:17-48` - 바주카 충돌/폭발 처리
- `packages/game-core/src/weapons/WeaponManager.ts:10` - 기본 무기 설정 ("bazooka")
- `packages/game-core/src/weapons/WeaponManager.ts:51-109` - 발사 로직
- `packages/game-core/src/entities/Projectile.ts:26-59` - 투사체 물리 초기화
- `packages/game-core/src/entities/Projectile.ts:99-108` - 바람 영향
- `packages/game-core/src/systems/DamageSystem.ts:77-124` - 폭발 대미지 계산
- `packages/game-core/src/systems/DamageSystem.ts:126-147` - 넉백 계산
- `packages/game-core/src/terrain/TerrainData.ts:34-47` - 지형 파괴 알고리즘

### 이동 시스템
- `packages/game-core/src/entities/Character.ts:63-65` - 이동/점프 상수
- `packages/game-core/src/entities/Character.ts:74-81` - 물리 바디 설정
- `packages/game-core/src/entities/Character.ts:319-331` - moveLeft()/moveRight()
- `packages/game-core/src/entities/Character.ts:336-350` - 자동 경사 등반
- `packages/game-core/src/entities/Character.ts:359-383` - 경사면 속도 계산
- `packages/game-core/src/entities/Character.ts:396-416` - stopMoving()/jump()
- `packages/game-core/src/entities/Character.ts:476-512` - 미끄러짐 방지/경계 제한
- `packages/game-core/src/systems/InputController.ts:108-160` - 키보드 입력 처리
- `packages/game-core/src/systems/TurnSystem.ts:4-10` - 턴 상태 타입
- `packages/game-core/src/systems/AIController.ts:184-244` - AI 이동 실행

### 게임 설정
- `packages/game-core/src/index.ts:18-24` - Matter.js 물리 설정 (gravity y=1)
- `packages/game-core/src/scenes/GameScene.ts:366-370` - 턴 시간 30초

---

## 아키텍처 문서화

### 무기 시스템 패턴
- **데이터 주도 설계**: `WeaponRegistry` 객체에 모든 무기 수치를 중앙 집중
- **추상 클래스**: `Projectile` 추상 클래스에서 공통 물리 로직, 각 무기가 `onCollision()` 구현
- **이벤트 기반**: 투사체 폭발 → `"explosion"` 이벤트 → DamageSystem 수신 (느슨한 결합)
- **팩토리 패턴**: WeaponManager의 switch 문으로 무기별 투사체 생성

### 이동 시스템 패턴
- **상태 기반 입력 제한**: TurnSystem의 state에 따라 이동 가능 여부 결정
- **속도 직접 제어**: Matter.js의 `setVelocity`로 속도 직접 설정 (힘이 아닌 속도)
- **컨트롤러 인터페이스**: `IPlayerController` 인터페이스로 인간/AI 공통 처리
- **동적 속도**: 경사도에 따라 실시간 속도 계산

### 이동 게이지 도입 시 영향 범위
이동 게이지 시스템을 추가하려면 다음 파일들이 영향받음:
- `Character.ts` - 게이지 상태 관리, moveLeft/moveRight/jump에서 게이지 소모
- `InputController.ts` - 게이지 소진 시 이동/점프 입력 차단
- `AIController.ts` - AI 이동 시 게이지 고려
- `TurnSystem.ts` - 턴 시작 시 게이지 리셋
- `GameHUD.ts` - 게이지 UI 표시
- `GameScene.ts` - 게이지 시스템 초기화

---

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md` - 웜즈 스타일 게임 아키텍처 연구. State Machine 기반 턴 관리, 무기 이펙트/폭발 설계 포함
- `thoughts/arta1069/plans/2026-01-23-worms-game-mvp-implementation.md` - MVP 구현 계획. 무기 시스템(Phase 5), 턴 시스템(Phase 4), 캐릭터 시스템(Phase 3) 모두 완료 상태
- `thoughts/arta1069/research/2026-02-14-ai-wind-calculation-self-destruct-research.md` - AI 바람 계산 및 자폭 방지 연구. 물리 시뮬레이션 정확도 이슈 분석
- `thoughts/arta1069/research/2026-02-13-ai-player-system-research.md` - AI 무기 선택 전략 (거리 기반) 및 이동 로직 분석

**참고**: 이동 게이지에 대한 기존 문서는 발견되지 않았으나, README.md:7에 "이동 게이지를 추가하여 이동에 제한을 추가 (점프시에도 게이지 차감)"이 TODO로 기록되어 있음.

---

## 관련 연구

- `thoughts/arta1069/research/2026-02-14-ai-wind-calculation-self-destruct-research.md` - AI 바람 계산 및 자폭 방지
- `thoughts/arta1069/research/2026-02-13-ai-player-system-research.md` - AI 플레이어 시스템

---

## 미해결 질문

~~1. 바주카 대미지를 하향 조정할 경우, AI의 발사 전략(`findBestShot`)이나 무기 선택 로직(`selectWeapon`)에 대한 재조정이 필요한지~~
→ **해결**: 바주카 너프는 AI/사람 모든 플레이어에 동일하게 적용. AI 발사 전략은 대미지 값을 참조하므로 자동 반영됨.

~~2. 이동 게이지 도입 시 AI의 이동 로직(800~1500ms 타이머 기반)을 게이지 기반으로 전환해야 하는 범위~~
→ **해결**: AI도 동일한 게이지 시스템 적용. 시간 기반이 아닌 게이지 소모 기반으로 전환.

~~3. 게이지 소모량의 이동/점프 비율을 어떻게 설정할 것인지 (예: 이동 1px당 1게이지 vs 점프 1회당 고정 게이지)~~
→ **해결**: 프레임당 소모 방식 채택. 총량 128, 이동 1프레임당 1 게이지, 점프 1회당 20 게이지.

---

## 후속 연구 2026-02-16T01:28:11+0900

### 논의를 통해 확정된 밸런스 변경 사항

#### 1. 바주카 너프 (확정)

바주카는 기본 무기(무제한 탄약)로서 **가장 약한 무기**가 되어야 함. 향후 추가되는 무기들은 바주카보다 강할 것.

| 항목 | 변경 전 | 변경 후 | 변화율 |
|------|--------|--------|--------|
| 대미지 | 50 | **25** | -50% |
| 폭발 반경 | 40px | **30px** | -25% |
| 지형 파괴 면적 | ~5,026px (π×40²) | **~2,827px (π×30²)** | -44% |
| 넉백 | 0.05 | **0.025** | -50% |
| 자폭 범위 | 60px (radius×1.5) | **45px** | -25% |
| 자폭 최소 대미지 | 12 (25%×50) | **6 (25%×25)** | -50% |

변경 후 무기 비교:

| 무기 | 대미지 | 폭발 반경 | 탄약 | 질량 | 지형 파괴 |
|------|--------|----------|------|------|----------|
| **Bazooka (변경 후)** | **25** | **30** | **무제한** | 0.06 | **반경 30px 원형** |
| Grenade | 35 | 50 | 3발 | 0.25 | 반경 50px 원형 |
| Shotgun | 25 | 15 | 2발 | 0.04 | 반경 15px × 5발 |

설계 의도: 바주카는 무제한 탄약의 안정적 기본 무기, Grenade/Shotgun은 상황별 강력한 선택지.

#### 2. 이동 게이지 시스템 (확정)

**레퍼런스**: 건바운드/포트리스 스타일 (거리 제한형 이동 바)
- 웜즈는 이동 게이지가 없음 (턴 타이머만 존재)
- 건바운드/포트리스는 이동 바가 있으며 이동 시 줄어듦 → 이 방식 채택

**소모 방식**: 프레임당 소모 (Per-frame)
- 이동 키 입력 중 매 프레임마다 1 게이지 차감
- 경사면 효과가 자연스럽게 반영됨 (오르막 = 짧은 거리, 내리막 = 먼 거리)
- 구현이 간단하고 경로 선택에 전략성 추가

**수치**:
- 맵 너비: 1280px
- 게이지 총량: **128** (평지에서 128프레임 × 1.25px = 160px ≈ 맵 1/8)
- 이동 소모: **1 게이지/프레임** (이동 키 입력 중)
- 점프 소모: **20 게이지/회** (고정 비용)
- 게이지 리셋: 턴 시작 시 전체 충전

**점프 트레이드오프 예시**:

| 점프 횟수 | 남은 이동 프레임 | 평지 이동 거리 |
|----------|----------------|--------------|
| 0회 | 128프레임 | 160px (전부 이동) |
| 1회 | 108프레임 | 135px + 점프 1회 |
| 2회 | 88프레임 | 110px + 점프 2회 |
| 3회 | 68프레임 | 85px + 점프 3회 |
| 6회 | 8프레임 | 10px + 점프 6회 |

**경사면 영향** (프레임당 소모 특성):
- 평지: 128프레임 = 160px
- 오르막 (최대 감속 0.4x): 128프레임 ≈ 64px
- 내리막 (최대 가속 1.2x): 128프레임 ≈ 192px

**적용 대상**: AI/사람 모든 플레이어에 동일하게 적용
- AI의 기존 시간 기반 이동(800~1500ms)을 게이지 기반으로 전환

#### 3. 영향 받는 파일

**바주카 너프**:
- `WeaponTypes.ts` - damage: 50→25, explosionRadius: 40→30

**이동 게이지 시스템**:
- `Character.ts` - 게이지 상태 관리, moveLeft/moveRight에서 프레임당 차감, jump에서 고정 차감
- `InputController.ts` - 게이지 소진 시 이동/점프 입력 차단
- `AIController.ts` - 시간 기반 이동을 게이지 기반으로 전환
- `TurnSystem.ts` - 턴 시작 시 게이지 리셋
- `GameHUD.ts` - 게이지 바 UI 표시
- `GameScene.ts` - 게이지 시스템 초기화
