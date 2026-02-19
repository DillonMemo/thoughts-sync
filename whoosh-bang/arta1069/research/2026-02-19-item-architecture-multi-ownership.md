---
date: 2026-02-19T03:19:05+0000
researcher: arta1069@gmail.com
git_commit: b1322ff8119d50412d5f2bd8850ff1beac209c64
branch: main
repository: whoosh-bang
topic: "아이템(무기, 캐릭터) 아키텍처 연구 - 동일 아이템 복수 보유 및 보유 수량 기반 탄약 시스템"
tags: [research, codebase, items, weapons, characters, inventory, architecture, multi-ownership]
status: complete
last_updated: 2026-02-19
last_updated_by: arta1069
last_updated_note: "UI 싱크 결정 사항 추가 - Selector UI 및 Store Drawer UI 방향"
---

# 연구: 아이템(무기, 캐릭터) 아키텍처 - 동일 아이템 복수 보유 및 보유 수량 기반 탄약 시스템

**날짜**: 2026-02-19T03:19:05+0000
**연구자**: arta1069@gmail.com
**Git 커밋**: b1322ff8119d50412d5f2bd8850ff1beac209c64
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

아이템(무기, 캐릭터) 아키텍처를 수정하여:
1. 동일한 캐릭터를 여러 개 보유할 수 있고, 1개라도 보유하면 해당 캐릭터로 게임 활성화 가능
2. 동일한 무기를 여러 개 보유할 수 있으며, 보유 수량에 따라 최대 발사 가능 횟수 결정 (예: Grenade 1개 보유 → 1발, N개 보유 → N발)

## 요약

현재 시스템은 `character_inventory`와 `weapon_inventory` 테이블에 **UNIQUE(user_id, character_id/weapon_id)** 제약이 있어, 한 유저가 동일 아이템을 1개만 보유할 수 있다. 무기 탄약은 DB의 `weapons.ammo` 컬럼(마스터 테이블 고정값)과 `WeaponRegistry` 하드코딩 fallback으로 관리되며, 유저별 보유 수량과는 무관하다.

요청된 변경을 위해서는:
- **DB 계층**: UNIQUE 제약 제거 + 보유 수량 집계 쿼리 추가
- **로비 계층**: 보유 수량 표시 UI + 캐릭터 활성화 조건 변경
- **게임 엔진 계층**: `WeaponManager`의 탄약 초기화 로직을 보유 수량 기반으로 변경
- **Export 계층**: 복수 보유 시 1개씩 export하는 흐름으로 변경

이 4개 계층 각각의 현재 구현을 아래에 상세히 문서화한다.

## 상세 발견 사항

### 1. DB 스키마: 인벤토리 테이블의 UNIQUE 제약

#### character_inventory 테이블

현재 `(user_id, character_id)` 조합에 UNIQUE 제약이 존재한다.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | UUID PK | 인벤토리 레코드 ID |
| `user_id` | UUID FK → auth.users | 유저 ID |
| `character_id` | TEXT FK → characters | 캐릭터 ID |
| `acquired_at` | TIMESTAMPTZ | 획득 시각 |
| `source` | TEXT DEFAULT 'default' | 획득 경로 ('default', 'fortem', 'reward') |
| `redeem_code` | TEXT DEFAULT NULL | ForTem export 시 리딤 코드 |
| `network` | TEXT | export 네트워크 |

**핵심 제약**: `UNIQUE(user_id, character_id)` — 동일 유저가 같은 캐릭터를 2개 이상 보유 불가

**인덱스**:
- `idx_character_inventory_user ON character_inventory(user_id)`
- `idx_character_inventory_active ON character_inventory(user_id) WHERE redeem_code IS NULL` (부분 인덱스)

#### weapon_inventory 테이블

`character_inventory`와 동일한 구조. `(user_id, weapon_id)` UNIQUE 제약 존재.

| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | UUID PK | 인벤토리 레코드 ID |
| `user_id` | UUID FK → auth.users | 유저 ID |
| `weapon_id` | TEXT FK → weapons | 무기 ID |
| `acquired_at` | TIMESTAMPTZ | 획득 시각 |
| `source` | TEXT DEFAULT 'default' | 획득 경로 |
| `redeem_code` | TEXT DEFAULT NULL | ForTem export 시 리딤 코드 |
| `network` | TEXT | export 네트워크 |

**핵심 제약**: `UNIQUE(user_id, weapon_id)` — 동일 유저가 같은 무기를 2개 이상 보유 불가

### 2. 무기 탄약(Ammo) 시스템: 현재 데이터 소스

#### 2-1. DB 마스터 테이블 (`weapons.ammo`)

`weapons` 테이블의 `ammo` 컬럼이 각 무기의 **기본 탄약 수**를 정의한다.

| 무기 ID | ammo 값 | 의미 |
|---------|---------|------|
| `bazooka` | -1 | 무제한 |
| `grenade` | 3 | 3발 |
| `shotgun` | 2 | 2발 |

이 값은 모든 유저에게 동일하게 적용되며, 유저별 보유 수량과는 무관하다.

#### 2-2. 하드코딩 Fallback (`WeaponRegistry`)

[packages/game-core/src/weapons/WeaponTypes.ts:31-65](packages/game-core/src/weapons/WeaponTypes.ts#L31-L65)에 `WeaponRegistry`가 하드코딩으로 존재한다.

```typescript
export const WeaponRegistry: Record<string, WeaponStats> = {
  bazooka: { damage: 30, explosionRadius: 25, ammo: -1, mass: 0.06, ... },
  grenade: { damage: 35, explosionRadius: 50, ammo: 3, mass: 0.25, ... },
  shotgun: { damage: 16, explosionRadius: 20, ammo: 2, mass: 0.04, ... },
}
```

`WeaponManager`는 DB 데이터가 없을 때 이 Registry를 fallback으로 사용한다.

#### 2-3. `WeaponManager`의 탄약 초기화

[packages/game-core/src/weapons/WeaponManager.ts:27-31](packages/game-core/src/weapons/WeaponManager.ts#L27-L31)

```typescript
initAmmo(): void {
  for (const [key, stats] of Object.entries(this.availableWeapons)) {
    this.ammo[key] = stats.ammo  // weapons.ammo 값 그대로 복사
  }
}
```

현재는 `WeaponData.ammo`(= DB `weapons.ammo` 값)를 그대로 복사한다. 유저별 보유 수량을 반영하는 로직이 없다.

#### 2-4. 탄약 소모 로직

[packages/game-core/src/weapons/WeaponManager.ts:77-79](packages/game-core/src/weapons/WeaponManager.ts#L77-L79)

```typescript
if (this.ammo[this.currentWeapon] > 0) {
  this.ammo[this.currentWeapon]--
}
```

- `ammo > 0`: 1 감소
- `ammo === -1`: 조건 불통과 → 영원히 유지 (무제한)
- `ammo === 0`: 소진 → `selectWeapon()` 거부, UI dim 처리

### 3. 캐릭터 소유 및 활성화 흐름

#### 3-1. 로비 캐릭터 조회

[apps/web/src/app/lobby/page.tsx:38-46](apps/web/src/app/lobby/page.tsx#L38-L46)

```typescript
const { data: ownedInventory } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)
```

결과를 `Set<string>`으로 변환하여 소유 여부를 O(1)으로 판별한다.

#### 3-2. 캐릭터 선택 UI

[apps/web/src/components/lobby/CharacterSelector.tsx:33-64](apps/web/src/components/lobby/CharacterSelector.tsx#L33-L64)

- `ownedIds.has(char.id)` → 소유: 선택 가능, 클릭 시 `onSelect(char.id)` 호출
- `!ownedIds.has(char.id)` → 미소유: `brightness-[0.3] grayscale` + Lock 아이콘, 클릭 비활성화

현재는 "보유 여부"만 boolean으로 판단하며, "보유 수량"은 어디에도 표시되지 않는다.

#### 3-3. 게임 진입 시 캐릭터 소유권 검증

[apps/web/src/app/game/page.tsx:29-39](apps/web/src/app/game/page.tsx#L29-L39)

```typescript
const { data: ownership } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .eq("character_id", selectedCharacter)
  .is("redeem_code", null)
  .single()

const verifiedCharacter = ownership ? selectedCharacter : "player"
```

`.single()` 호출로 1개 row만 기대한다. UNIQUE 제약이 있으므로 현재는 문제 없다.

### 4. 무기 소유 및 선택 흐름

#### 4-1. 로비 무기 조회

[apps/web/src/app/lobby/page.tsx:57-66](apps/web/src/app/lobby/page.tsx#L57-L66)

`character_inventory`와 동일한 패턴으로 `weapon_inventory`에서 `redeem_code IS NULL`인 무기 ID 목록을 `Set<string>`으로 구성한다.

#### 4-2. 로비 무기 선택 (최대 3개)

[apps/web/src/components/lobby/LobbyContent.tsx:98-127](apps/web/src/components/lobby/LobbyContent.tsx#L98-L127)

- `ownedWeaponSet.has(weaponId)` 체크로 미소유 무기 선택 차단
- 선택된 무기가 1개뿐이면 해제 불가 (최소 1개)
- 선택된 무기가 3개 이상이면 추가 불가 (최대 3개)
- 변경 시 `player_profiles.selected_weapons` 배열 즉시 업데이트

#### 4-3. 게임 진입 시 무기 데이터 전달

[apps/web/src/app/game/page.tsx:42-65](apps/web/src/app/game/page.tsx#L42-L65)

1. `weapon_inventory`에서 소유권 재검증
2. `weapons` 마스터 테이블에서 검증된 무기의 `WeaponData` 조회
3. `WeaponData[]` → Phaser registry → `WeaponManager` 주입

이 과정에서 `WeaponData.ammo`는 마스터 테이블의 고정값이며, 유저별 보유 수량은 반영되지 않는다.

### 5. Export 흐름과 인벤토리 상태

#### 5-1. Export API

[apps/web/src/app/api/store/export/route.ts](apps/web/src/app/api/store/export/route.ts)

Export 시 인벤토리 레코드에 `redeem_code` 값을 채워 "게임 내 사용 불가" 상태로 전환한다.

현재 UNIQUE 제약 하에서는 1 유저당 1 아이템 = 1 레코드이므로, export하면 해당 아이템을 게임에서 완전히 잃는다.

복수 보유가 허용되면, export 시 여러 레코드 중 **1개만** redeem_code를 채우고, 나머지 N-1개는 유지해야 한다.

#### 5-2. Export 후 선택 fallback

[apps/web/src/app/api/store/export/route.ts:133-158](apps/web/src/app/api/store/export/route.ts#L133-L158)

Export된 아이템이 현재 선택 중이면:
- 캐릭터: `"player"`로 fallback
- 무기: `selected_weapons`에서 해당 무기 제거 → 빈 배열이면 `["bazooka"]`로 fallback

복수 보유 시, export 후에도 해당 아이템이 1개 이상 남아있으면 fallback이 불필요하다.

### 6. DB 트리거 함수

#### handle_new_user

신규 유저 가입 시 기본 아이템 지급:
- `character_inventory`에 `character_id='player'`, `source='default'` 1개 삽입
- `weapon_inventory`에 `weapon_id='bazooka'`, `source='default'` 1개 삽입

#### validate_selected_character

`player_profiles.selected_character` UPDATE 시 `character_inventory`에서 소유 여부 확인.

#### validate_selected_weapons

`player_profiles.selected_weapons` UPDATE 시:
1. 배열 길이 > 3 체크
2. NULL 체크
3. `weapon_inventory`에서 각 무기 소유 여부 확인

이 트리거들은 `EXISTS` 쿼리로 "1개라도 있는지"를 확인하므로, UNIQUE 제약 제거 후에도 동작에 문제 없다.

### 7. TypeScript 타입 정의

[packages/game-core/src/types/index.ts:12-24](packages/game-core/src/types/index.ts#L12-L24)

```typescript
export interface WeaponData {
  id: string
  name: string
  damage: number
  explosion_radius: number
  ammo: number          // 현재: weapons 테이블 고정값, 변경 후: 보유 수량 기반
  mass: number
  atlas_key: string
  atlas_path: string
  projectile_frame: string | null
  hold_frame: string | null
  sort_order: number
}
```

`ammo` 필드가 현재는 마스터 테이블의 고정값을 담고 있다. 보유 수량 기반으로 변경하려면 이 값을 게임 진입 시 동적으로 계산해야 한다.

## 코드 참조

### DB 스키마 관련
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md:74-83` — characters 테이블 마이그레이션
- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md:76-89` — weapons 테이블 마이그레이션
- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md:114-131` — weapon_inventory RLS 정책

### 인벤토리 조회
- [apps/web/src/app/lobby/page.tsx:38-46](apps/web/src/app/lobby/page.tsx#L38-L46) — 캐릭터 인벤토리 조회
- [apps/web/src/app/lobby/page.tsx:57-66](apps/web/src/app/lobby/page.tsx#L57-L66) — 무기 인벤토리 조회
- [apps/web/src/app/game/page.tsx:29-39](apps/web/src/app/game/page.tsx#L29-L39) — 캐릭터 소유권 재검증
- [apps/web/src/app/game/page.tsx:44-56](apps/web/src/app/game/page.tsx#L44-L56) — 무기 소유권 재검증

### 탄약 시스템
- [packages/game-core/src/weapons/WeaponTypes.ts:31-65](packages/game-core/src/weapons/WeaponTypes.ts#L31-L65) — WeaponRegistry 하드코딩
- [packages/game-core/src/weapons/WeaponManager.ts:16-31](packages/game-core/src/weapons/WeaponManager.ts#L16-L31) — WeaponManager 생성자 및 initAmmo
- [packages/game-core/src/weapons/WeaponManager.ts:77-79](packages/game-core/src/weapons/WeaponManager.ts#L77-L79) — 탄약 소모 로직

### UI 관련
- [apps/web/src/components/lobby/CharacterSelector.tsx](apps/web/src/components/lobby/CharacterSelector.tsx) — 캐릭터 선택 UI
- [apps/web/src/components/lobby/LobbyWeaponSelector.tsx](apps/web/src/components/lobby/LobbyWeaponSelector.tsx) — 무기 선택 UI
- [apps/web/src/components/lobby/LobbyContent.tsx:98-127](apps/web/src/components/lobby/LobbyContent.tsx#L98-L127) — 무기 토글 로직
- [packages/game-core/src/ui/WeaponSelector.ts](packages/game-core/src/ui/WeaponSelector.ts) — 인게임 무기 선택 UI

### Export 관련
- [apps/web/src/app/api/store/export/route.ts](apps/web/src/app/api/store/export/route.ts) — Export API 전체
- [apps/web/src/components/lobby/StoreButton.tsx:111-156](apps/web/src/components/lobby/StoreButton.tsx#L111-L156) — Store UI export 흐름

## 아키텍처 문서화

### 현재 아키텍처: 마스터-인벤토리 4계층 패턴

```
[1] 마스터 테이블 (characters / weapons)
    - 아이템 카탈로그: id, name, stats, assets
    - 모든 유저에게 동일한 읽기 전용 데이터

[2] 인벤토리 테이블 (character_inventory / weapon_inventory)
    - 유저별 소유권: user_id + item_id + redeem_code
    - UNIQUE(user_id, item_id) 제약 → 1인 1개 보유
    - redeem_code IS NULL = 게임 내 사용 가능

[3] 프로필 선택 (player_profiles)
    - selected_character: TEXT (1개)
    - selected_weapons: TEXT[] (최대 3개)
    - DB 트리거로 소유권 검증

[4] 서버사이드 검증 (game/page.tsx)
    - 게임 진입 시 인벤토리에서 소유권 재확인
    - 미소유 시 기본값 fallback
```

### 변경이 필요한 지점 (계층별)

| 계층 | 현재 | 변경 필요 |
|------|------|-----------|
| DB 스키마 | UNIQUE(user_id, item_id) | 제약 제거 → 동일 아이템 복수 row 허용 |
| 인벤토리 조회 | `SELECT character_id` → Set | `SELECT character_id, COUNT(*)` → Map(id→count) |
| 캐릭터 활성화 | `ownedIds.has(id)` (boolean) | `ownedCount.get(id) >= 1` (count 기반) |
| 무기 탄약 | `weapons.ammo` 고정값 | 보유 수량 × 1 (또는 보유 수량 자체) |
| WeaponManager.initAmmo | `stats.ammo` 복사 | 서버에서 전달받은 보유 수량 사용 |
| WeaponData.ammo | 마스터 테이블 값 | 게임 진입 시 동적 계산 값 |
| Export | 단일 레코드 업데이트 | N개 중 1개만 선택 업데이트 |
| 신규 유저 트리거 | 1개 INSERT | 변경 불필요 (기본 1개 지급) |
| 선택 검증 트리거 | EXISTS 쿼리 | 변경 불필요 (1개 이상 확인) |

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` — 무기/캐릭터 아이템화 시스템의 초기 분석. 캐릭터 패턴(마스터→인벤토리→프로필→검증)을 무기에도 동일하게 적용하는 것이 결정됨
- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md` — 무기 아이템화 구현 계획. UNIQUE 제약, DB 트리거, RLS 정책의 설계 근거 기록
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md` — 캐릭터 스킨 선택 시스템 구현 계획. characters 테이블과 character_inventory의 최초 설계
- `thoughts/arta1069/research/2026-02-19-store-drawer-item-export-system.md` — Store Drawer UI 및 Export 시스템 연구. ForTem 마켓플레이스 연동과 redeem_code 기반 상태 전환 패턴 분석
- `thoughts/arta1069/plans/2026-02-19-store-drawer-item-export-implementation.md` — Store/Export 구현 계획. 현재 동작하는 Export 흐름의 전체 설계

## 관련 연구

- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md` — ForTem SDK 아키텍처 (Import 흐름 구현 시 참조 필요)
- `thoughts/arta1069/research/2026-02-17-supabase-storage-asset-migration.md` — Supabase Storage 에셋 관리

## 후속 연구 2026-02-19T03:28:16+0000

### UI 싱크 결정 사항

#### 1. 캐릭터 Selector UI → 변경 없음

[CharacterSelector.tsx](apps/web/src/components/lobby/CharacterSelector.tsx) 현재 UI 유지.
- 동일 캐릭터 N개 보유해도 1개 이상이면 선택 가능 표시
- `ownedIds: Set<string>` 타입 유지 (boolean 소유 여부만 필요)
- 보유 수량은 이 UI에서 표시하지 않음

#### 2. 무기 Selector UI → 발사 가능 수 뱃지 추가

[LobbyWeaponSelector.tsx](apps/web/src/components/lobby/LobbyWeaponSelector.tsx) 변경.
- 무기 이미지 우측 상단에 보유 수량(= 최대 발사 횟수) 숫자 뱃지 추가
- `ownedIds: Set<string>` → `ownedCounts: Map<string, number>` 로 props 타입 변경
- 무기는 마스터 테이블 기준 고유 아이템으로 1개만 표시하되, 뱃지로 수량 표현

#### 3. Store Drawer UI → 인벤토리 레코드 개별 리스팅

[StoreButton.tsx](apps/web/src/components/lobby/StoreButton.tsx) 변경.
- 동일 아이템 N개 보유 시 **N개의 개별 선택 박스**로 리스팅 (그룹핑 아님)
- 예: Grenade 3개 → Grenade 박스 3개 각각 나열
- 각 박스는 인벤토리 레코드의 `id`(UUID)로 식별
- Export 시 선택된 특정 인벤토리 레코드 1개를 대상으로 처리

**데이터 구조 변경:**

| 위치 | 현재 | 변경 후 |
|------|------|---------|
| `StoreButton` props | `ownedCharacterIds: string[]` (유니크 ID 목록) | `ownedCharacterInventory: { inventoryId: string, characterId: string }[]` (인벤토리 row 목록) |
| `StoreButton` props | `ownedWeaponIds: string[]` | `ownedWeaponInventory: { inventoryId: string, weaponId: string }[]` |
| `selectedExportItem` state | `{ type, id, name }` | `{ type, inventoryId, itemId, name }` (inventoryId 추가) |
| Export API 요청 | `{ itemType, itemId }` | `{ itemType, itemId, inventoryId }` (특정 레코드 지정) |
| Export API 서버 | `itemId`로 레코드 특정 | `inventoryId`(UUID)로 레코드 특정 |

**Export 후 fallback 로직 변경:**
- 현재: export하면 해당 아이템 완전 상실 → 선택 중이면 기본값 fallback
- 변경 후: export 후에도 동일 아이템이 1개 이상 남아있으면 fallback 불필요 → 남은 수량 확인 로직 추가

## 미해결 질문

1. **무기 탄약 계산 공식**: 보유 수량 N개 → ammo=N으로 단순 매핑할 것인지, 아니면 무기별 기본 ammo × N 같은 공식을 적용할 것인지?
   - 예: Grenade 2개 보유 → ammo=2? 아니면 기본 ammo(3) × 2 = 6?
   - README의 예시("N개를 가지고 있다면 N발")에 따르면 보유 수량 = 최대 발사 횟수
2. **Bazooka의 무제한 탄약 유지 여부**: Bazooka는 현재 `ammo=-1`(무제한)인데, 복수 보유 시에도 무제한을 유지할 것인지?
3. **Import 흐름**: ForTem 마켓플레이스에서 아이템을 가져오는 Import API는 아직 미구현 상태. 복수 보유 아키텍처와 함께 Import도 구현이 필요한지?
4. **기본 아이템 복수 보유**: 기본 아이템(`"player"`, `"bazooka"`)도 추가 획득이 가능한지?
