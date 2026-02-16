---
date: 2026-02-17T02:05:04+0900
researcher: arta1069@gmail.com
git_commit: 48d17f6b0fa30b5ab1a52e321144f837b5e466be
branch: main
repository: whoosh-bang
topic: "무기와 캐릭터 아이템화 - 현재 시스템 분석"
tags: [research, codebase, weapons, characters, inventory, database, lobby, itemization]
status: complete
last_updated: 2026-02-17
last_updated_by: arta1069
---

# 연구: 무기와 캐릭터 아이템화 - 현재 시스템 분석

**날짜**: 2026-02-17T02:05:04+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: 48d17f6b0fa30b5ab1a52e321144f837b5e466be
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

무기와 캐릭터 아이템화를 위한 현재 시스템 분석:
- 캐릭터/무기 정보의 DB 저장 구조
- 캐릭터 인벤토리 메커니즘을 무기에도 동일하게 적용하기 위한 기존 패턴 파악
- 로비에서 무기 선택 UI 구현을 위한 현재 로비 시스템 파악
- 최대 3개 무기 선택, 기본 무기 바주카 요구사항 대비 현재 구현 상태

## 요약

현재 코드베이스는 **캐릭터 아이템화가 완전히 구현**되어 있다. `characters` 마스터 테이블, `character_inventory` 소유 테이블, 로비 UI에서의 선택/표시, 게임 진입 시 서버사이드 소유 검증까지 완성된 상태이다.

반면 **무기 시스템은 게임 내부에서만 존재**한다. 3종 무기(bazooka, grenade, shotgun)가 `WeaponRegistry` 객체에 하드코딩되어 있고, 인게임 `WeaponSelector` UI로 선택하며, DB 기반 소유/선택 시스템은 없다. 무기는 로비에서 선택할 수 없고, 게임 시작 시 항상 3종 모두 사용 가능한 상태로 시작한다.

캐릭터 인벤토리 패턴(마스터 테이블 → 인벤토리 테이블 → 프로필 선택 → 서버 검증)을 무기에 그대로 적용할 수 있는 구조가 이미 확립되어 있다.

## 상세 발견 사항

### 1. 현재 무기 시스템 (게임 내부)

#### WeaponRegistry - 무기 정의 (`packages/game-core/src/weapons/WeaponTypes.ts`)

3종 무기가 `Record<string, WeaponStats>` 타입의 객체로 하드코딩되어 있다:

| 무기 | 데미지 | 폭발 반경 | 탄약 | 질량 | 행동 |
|------|--------|----------|------|------|------|
| Bazooka | 30 | 25 | 무제한(-1) | 0.06 | 충돌 시 즉시 폭발 |
| Grenade | 35 | 50 | 3 | 0.25 | 바운스, 2.5초 퓨즈 |
| Shotgun | 16 | 20 | 2 (dev: 무제한) | 0.04 | 5발 산탄 패턴 |

`WeaponStats` 인터페이스 필드:
- `name`, `damage`, `explosionRadius`, `ammo` (-1 = 무제한)
- `mass` (Matter.js 물리), `spriteKey`, `spriteFrame`, `holdSprite`

#### WeaponManager - 무기 관리 (`packages/game-core/src/weapons/WeaponManager.ts`)

- `currentWeapon`: 현재 선택된 무기 (기본값: `"bazooka"`)
- `ammo`: 무기별 잔여 탄약 추적
- `selectWeapon()`: 무기 존재 여부와 탄약 확인 후 선택
- `fire()`: switch 문으로 무기 타입별 프로젝타일 인스턴스 생성
- `applyWindToAll()`: 활성 프로젝타일에 바람 적용

#### Projectile 계층 구조

- **추상 클래스** `Projectile` (`packages/game-core/src/entities/Projectile.ts:15`): 물리, 충돌, 바람 적용 공통 로직
- **BazookaProjectile** (`packages/game-core/src/weapons/Bazooka.ts:5`): `onCollision` → 즉시 폭발
- **GrenadeProjectile** (`packages/game-core/src/weapons/Grenade.ts:5`): bounce 0.6, 2.5초 타이머 폭발
- **ShotgunProjectile** + `createShotgunBlast()` (`packages/game-core/src/weapons/Shotgun.ts:5,70`): 5발 spread (-10°~+10°)

#### 인게임 무기 선택 UI (`packages/game-core/src/ui/WeaponSelector.ts:15`)

- `WeaponRegistry` 키를 순회하며 박스 UI 생성
- 무기 스프라이트, 이름, 잔여 탄약, 핫키(1,2,3) 표시
- 클릭 또는 키보드(0-2)로 선택

#### AI 무기 선택 (`packages/game-core/src/systems/AIController.ts:629`)

- 거리 < 150px → shotgun, 그 외 → bazooka
- grenade는 AI가 사용하지 않음

#### 이벤트 시스템 (`packages/game-core/src/EventBus.ts`)

- `WEAPON_SELECTED` (line 11): 무기 선택 시 발행
- `WEAPON_FIRED` (line 13): 무기 발사 시 발행

### 2. 현재 캐릭터 시스템 (완전 구현됨)

#### 캐릭터 타입 정의 (`packages/game-core/src/types/index.ts:1-9`)

```
CharacterType = "player" | "female" | "adventurer"
CharacterData = { id, name, asset_path, asset_prefix, sort_order }
```

#### DB 스키마 (Supabase)

**`characters` 테이블** - 마스터 데이터:
- `id` (TEXT, PK), `name`, `asset_path`, `asset_prefix`, `is_active` (BOOLEAN), `sort_order`, `created_at`
- 5종 등록: player, adventurer, female, soldier(비활성), zombie(비활성)

**`character_inventory` 테이블** - 유저별 보유:
- `id` (UUID, PK), `user_id` (UUID, FK→auth.users), `character_id` (TEXT, FK→characters)
- `acquired_at`, `source` ("default", "fortem", "reward")
- UNIQUE(user_id, character_id)

**`player_profiles` 테이블** - 선택된 캐릭터:
- `selected_character` (TEXT, FK→characters.id, DEFAULT 'player')

#### 트리거 함수

- `validate_selected_character`: `player_profiles.selected_character` UPDATE 전 `character_inventory` 소유 검증
- `handle_new_user`: `auth.users` INSERT 후 프로필 생성 + 기본 캐릭터 인벤토리 부여

#### 데이터 플로우

```
[DB] characters → character_inventory → player_profiles.selected_character
  ↓
[Server: lobby/page.tsx] 전체 캐릭터 + 소유 정보 fetch
  ↓
[Client: LobbyContent] CharacterSelector(썸네일) + CharacterDisplay(프리뷰)
  → 선택 시: player_profiles.selected_character UPDATE
  ↓
[Server: game/page.tsx] selected_character fetch → character_inventory 소유 검증 → 미보유 시 "player" 폴백
  ↓
[Client: PhaserGame] createGame({ player1CharacterType }) → registry 저장
  ↓
[Phaser: GameScene] registry에서 읽어 Character 엔티티 생성
```

### 3. 로비 시스템 현재 상태

#### 로비 페이지 구조 (`apps/web/src/app/lobby/page.tsx`)

Server Component로 데이터 페칭 후 Client Component에 전달:
- `player_profiles` → 프로필 정보 (line 18-20)
- `wallets` → 연결된 지갑 (line 25-27)
- `characters` → 전체 캐릭터 목록 (line 32-34)
- `character_inventory` → 보유 캐릭터 (line 39-41)

#### 로비 UI 컴포넌트 (`apps/web/src/components/lobby/`)

| 컴포넌트 | 위치 | 역할 |
|---------|------|------|
| `LobbyContent.tsx` | 메인 레이아웃 | Grid 기반 전체 배치, 캐릭터 선택 핸들러 |
| `ProfileCard.tsx` | 좌측 상단 | 프로필, 레벨/XP, 승률, BGM, 로그아웃 |
| `CharacterSelector.tsx` | 하단 중앙 | 캐릭터 썸네일 가로 스크롤, 소유/미소유 구분 |
| `CharacterDisplay.tsx` | 중앙 | 선택된 캐릭터 프리뷰 (플로팅 애니메이션) |
| `PlayButton.tsx` | 우측 하단 | PLAY 버튼 → `/game` 라우팅 |
| `WalletLinkButton.tsx` | 프로필 카드 내 | Sui 지갑 연결/해제 |

#### 무기 선택 관련 - 로비에 없음

현재 로비에는 **무기 선택 UI가 전혀 없다**. 무기는 게임 내 `WeaponSelector` UI에서만 선택 가능하고, 게임 시작 시 항상 3종 무기 모두 사용 가능 상태로 시작한다.

### 4. DB 관련 현재 상태

#### Supabase 클라이언트 설정

- Browser 클라이언트: `apps/web/src/lib/supabase/client.ts`
- Server 클라이언트: `apps/web/src/lib/supabase/server.ts`
- 환경 변수: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`, `SUPABASE_SERVICE_ROLE_KEY`

#### 현재 DB 테이블 목록

1. `player_profiles` - 유저 프로필, 스탯, 선택 캐릭터
2. `characters` - 캐릭터 마스터 데이터
3. `character_inventory` - 캐릭터 보유 (유저↔캐릭터 N:M)
4. `wallets` - 연결된 지갑 주소
5. `wallet_nonces` - 지갑 서명 검증용 임시 논스
6. `match_history` - 매치 기록 및 결과

#### TypeScript DB 타입

Supabase에서 생성된 TypeScript 타입 파일이 **존재하지 않는다**. 모든 DB 쿼리는 inline으로 추론되고 있다.

#### RPC 함수

- `complete_match`: 매치 완료 처리 및 스탯 업데이트
- `record_abandoned_match`: 포기 매치 처리

### 5. 에셋 로딩 시스템 (`packages/game-core/src/scenes/BootScene.ts`)

#### 캐릭터 에셋 로딩 (line 108-115)

- `CHARACTERS` 배열 × `CHARACTER_POSES` 배열 = 3종 × 12포즈 = 36개 이미지
- 키 패턴: `{characterKey}_{pose}` (예: `player_idle`)
- 경로: `/assets/characters/PNG/{Path}/Poses/{prefix}_{pose}.png`

#### 무기 에셋 로딩 (line 118-122)

- `tanks` 아틀라스 단일 로드 (`tanks_spritesheetDefault.png` + `.xml`)
- 모든 무기가 이 아틀라스의 프레임을 참조

### 6. 프로젝트 구조 개요

```
whoosh-bang/
├── apps/web/                    # Next.js 앱 (로비, 게임, 인증, API)
│   ├── src/app/                 # App Router 페이지
│   │   ├── lobby/               # 로비 (캐릭터 선택, 프로필)
│   │   ├── game/                # 게임 플레이
│   │   └── api/                 # match/, wallet/ API 라우트
│   ├── src/components/          # React 컴포넌트
│   │   ├── lobby/               # 로비 UI (6개 파일)
│   │   └── game/                # 게임 래퍼 (2개 파일)
│   └── src/lib/supabase/        # Supabase 클라이언트
├── packages/game-core/          # Phaser 게임 엔진
│   ├── src/scenes/              # BootScene, GameScene
│   ├── src/entities/            # Character, Projectile
│   ├── src/weapons/             # WeaponTypes, WeaponManager, Bazooka, Grenade, Shotgun
│   ├── src/systems/             # TurnSystem, AIController, DamageSystem, WindSystem
│   ├── src/ui/                  # WeaponSelector, AimingUI, GameHUD
│   └── src/types/               # CharacterType, CharacterData
├── packages/ui/                 # 공유 UI (현재 비어있음)
└── packages/config/             # 공유 ESLint 설정
```

## 코드 참조

### 무기 시스템
- `packages/game-core/src/weapons/WeaponTypes.ts:1-43` - WeaponStats 인터페이스 및 WeaponRegistry
- `packages/game-core/src/weapons/WeaponManager.ts:9-134` - 무기 선택/발사/탄약 관리
- `packages/game-core/src/weapons/Bazooka.ts:5` - 바주카 프로젝타일
- `packages/game-core/src/weapons/Grenade.ts:5` - 수류탄 프로젝타일
- `packages/game-core/src/weapons/Shotgun.ts:5,70` - 샷건 프로젝타일 + createShotgunBlast
- `packages/game-core/src/entities/Projectile.ts:15` - 추상 Projectile 베이스 클래스
- `packages/game-core/src/ui/WeaponSelector.ts:15` - 인게임 무기 선택 UI
- `packages/game-core/src/systems/DamageSystem.ts:69` - 폭발 데미지 계산
- `packages/game-core/src/systems/AIController.ts:629` - AI 무기 선택 로직

### 캐릭터 시스템
- `packages/game-core/src/types/index.ts:1-9` - CharacterType, CharacterData
- `packages/game-core/src/entities/Character.ts:6-13` - CharacterConfig 인터페이스
- `packages/game-core/src/entities/Character.ts:199-274` - 무기 스프라이트 표시
- `packages/game-core/src/scenes/BootScene.ts:38-59` - 캐릭터 포즈/타입 정의
- `packages/game-core/src/scenes/GameScene.ts:272-299` - 캐릭터 생성

### 로비
- `apps/web/src/app/lobby/page.tsx:5-58` - 로비 서버 컴포넌트 (데이터 페칭)
- `apps/web/src/components/lobby/LobbyContent.tsx:32-127` - 로비 메인 레이아웃
- `apps/web/src/components/lobby/CharacterSelector.tsx:15-78` - 캐릭터 선택 UI
- `apps/web/src/components/lobby/CharacterDisplay.tsx:11-40` - 캐릭터 프리뷰

### DB 및 인증
- `apps/web/src/app/game/page.tsx:17-38` - 게임 진입 시 캐릭터 소유 검증
- `apps/web/src/lib/supabase/client.ts` - 브라우저 Supabase 클라이언트
- `apps/web/src/lib/supabase/server.ts` - 서버 Supabase 클라이언트

### 이벤트
- `packages/game-core/src/EventBus.ts:11,13` - WEAPON_SELECTED, WEAPON_FIRED 이벤트

## 아키텍처 문서화

### 캐릭터 인벤토리 패턴 (현재 구현됨)

```
[마스터 테이블] characters (id, name, asset_path, is_active, sort_order)
       ↓
[인벤토리 테이블] character_inventory (user_id, character_id, source)
       ↓
[프로필 선택] player_profiles.selected_character
       ↓
[서버 검증] game/page.tsx에서 소유 확인 → 미보유 시 기본값 폴백
       ↓
[게임 적용] Phaser registry → Character 엔티티
```

이 패턴은 무기 아이템화에 그대로 재사용할 수 있는 구조이다.

### 무기 시스템 현재 아키텍처

```
[하드코딩] WeaponRegistry (WeaponTypes.ts)
       ↓
[게임 내] WeaponManager (선택/발사/탄약)
       ↓
[UI] WeaponSelector (인게임에서만)
       ↓
[물리] Projectile 서브클래스들
```

DB 기반 소유/선택 레이어가 전혀 없다.

### 현재 무기↔캐릭터 차이점

| 측면 | 캐릭터 | 무기 |
|------|--------|------|
| 마스터 데이터 | DB `characters` 테이블 | 코드 `WeaponRegistry` 객체 |
| 보유 시스템 | `character_inventory` 테이블 | 없음 (모두 사용 가능) |
| 선택 저장 | `player_profiles.selected_character` | 없음 |
| 로비 UI | CharacterSelector + CharacterDisplay | 없음 |
| 서버 검증 | game/page.tsx 소유 확인 | 없음 |
| 인게임 UI | 없음 (게임 시작 전 결정) | WeaponSelector (게임 중 선택) |
| 기본값 | "player" 캐릭터 | bazooka |

## 히스토리 컨텍스트 (thoughts/에서)

### 관련 연구 문서
- `thoughts/arta1069/research/2026-02-13-character-skin-nft-system-research.md` - 캐릭터 스킨 NFT 시스템 분석 (ForTem 통합 데이터 모델 포함)
- `thoughts/arta1069/research/2026-02-16-game-balance-weapon-movement-research.md` - 무기 밸런스 연구 (바주카 너프)
- `thoughts/arta1069/research/2026-02-15-match-stats-playtime-active-user-research.md` - 매치 통계 시스템

### 관련 계획 문서
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md` - 캐릭터 스킨 선택 시스템 구현 계획 (DB 스키마, 트리거 함수 포함)
- `thoughts/arta1069/plans/2026-02-16-game-balance-bazooka-nerf-movement-gauge.md` - 바주카 밸런스 계획

### NFT/블록체인 관련
- `thoughts/arta1069/research/2026-02-13-character-skin-nft-system-research.md` - NFT 데이터 구조와 ForTem 통합 모델
- `thoughts/arta1069/plans/2026-02-13-social-login-wallet-linking-implementation.md` - Sui 지갑 연결 구현

## 관련 연구

- `thoughts/arta1069/research/2026-02-13-character-skin-nft-system-research.md` - 캐릭터 NFT 시스템 (가장 직접적으로 관련)
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md` - 캐릭터 선택 시스템 계획 (인벤토리 패턴의 원본)

## 미해결 질문

1. **무기 마스터 데이터 위치**: DB `weapons` 테이블로 옮길 때 `WeaponStats`의 게임플레이 속성(damage, explosionRadius, mass 등)도 DB에 저장할 것인지, 아니면 게임 코드에 유지하고 DB에는 메타데이터(이름, 설명, 에셋 경로)만 저장할 것인지
2. **무기 선택 시점**: 로비에서 최대 3개 무기를 선택하면 인게임 `WeaponSelector` UI는 선택된 3개만 표시하도록 변경해야 하는지, 현재 전체 표시 UI를 유지할 것인지
3. **무기 에셋**: 현재 모든 무기가 `tanks` 아틀라스 하나를 공유하는데, 새 무기 추가 시 별도 에셋 로딩이 필요할 수 있음
4. **기본 무기 부여 시점**: 캐릭터처럼 `handle_new_user` 트리거에서 기본 무기(바주카)를 자동 부여할 것인지
5. **ForTem SDK 통합**: `character_inventory.source`에 "fortem" 값이 이미 정의되어 있어, 무기 인벤토리에도 동일한 source 체계를 적용할 수 있음
