---
date: 2026-02-19
author: arta1069
status: draft
ticket: null
related_research: thoughts/arta1069/research/2026-02-19-item-architecture-multi-ownership.md
---

# 아이템 복수 보유 및 보유 수량 기반 탄약 시스템 구현 계획

## 개요

동일한 캐릭터/무기를 여러 개 보유할 수 있도록 DB UNIQUE 제약을 제거하고, 무기 보유 수량에 따라 인게임 최대 발사 횟수를 결정하는 시스템을 구현한다. Store UI는 인벤토리 레코드별 개별 리스팅으로 변경하여 특정 레코드를 선택해 export할 수 있도록 한다.

## 현재 상태 분석

- `character_inventory`, `weapon_inventory` 테이블에 `UNIQUE(user_id, item_id)` 제약 → 1인 1아이템 1개만 보유
- 무기 탄약은 `weapons.ammo` 마스터 테이블 고정값 (bazooka=-1, grenade=3, shotgun=2)
- 로비/게임 진입 시 소유 여부를 boolean(`Set<string>`)으로만 판별
- Export API는 `itemId`로 레코드를 특정하며, 1 유저 = 1 아이템 = 1 레코드를 전제
- `game/page.tsx`와 `export/route.ts`에서 `.single()` 사용 → 복수 row 시 에러 위험

### 핵심 발견:
- DB 트리거(`validate_selected_character`, `validate_selected_weapons`)는 `EXISTS` 쿼리 사용 → UNIQUE 제거 후 변경 불필요 ([weapon-itemization-system.md:136-169](thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md#L136-L169))
- `handle_new_user` 트리거는 기본 아이템 1개만 INSERT → 변경 불필요
- `WeaponManager.initAmmo()`는 `WeaponData.ammo` 값을 그대로 사용 → ammo 값만 서버에서 올바르게 계산하면 게임 엔진 변경 불필요 ([WeaponManager.ts:27-31](packages/game-core/src/weapons/WeaponManager.ts#L27-L31))
- 인게임 `WeaponSelector` UI는 `WeaponManager.getAmmo()`를 통해 탄약을 표시 → 변경 불필요 ([WeaponSelector.ts:92-100](packages/game-core/src/ui/WeaponSelector.ts#L92-L100))
- 기본 아이템(player, bazooka)은 export 차단(클라이언트+서버 이중 보호) → 복수 보유 자연 불가

## 원하는 최종 상태

1. 동일 캐릭터 N개 보유 가능, 1개 이상이면 게임에서 해당 캐릭터 선택 가능
2. 동일 무기 N개 보유 시 인게임 최대 발사 횟수 = N (bazooka는 항상 무제한)
3. 로비 무기 선택 UI에 보유 수량 뱃지 표시
4. Store Drawer에서 인벤토리 레코드별 개별 리스팅, 특정 레코드 선택 export
5. Export 후 동일 아이템이 1개 이상 남으면 선택 fallback 불필요

## 하지 않을 것

- Import 흐름 구현 (별도 연구 후 구현 예정)
- 캐릭터 Selector UI 변경 (보유 여부 boolean만 필요, 수량 표시 안 함)
- `WeaponManager` 또는 인게임 `WeaponSelector` 코드 변경 (ammo 값만 서버에서 올바르게 전달)
- `WeaponRegistry` 하드코딩 fallback 값 변경
- DB 트리거 함수 변경 (`handle_new_user`, `validate_selected_character`, `validate_selected_weapons`)

## 구현 접근 방식

DB → 서버 조회 → UI 순서로 아래에서 위로 변경한다. 각 단계는 독립적으로 검증 가능하도록 설계한다.

---

## 1단계: DB 마이그레이션 — UNIQUE 제약 제거 + 인덱스 추가

### 개요
`character_inventory`와 `weapon_inventory`의 UNIQUE 제약을 제거하여 동일 아이템 복수 row를 허용한다. 보유 수량 집계를 위한 복합 인덱스를 추가한다.

### 필요한 변경:

#### 1. Supabase 마이그레이션 SQL

마이그레이션 이름: `remove_inventory_unique_constraints`

```sql
-- 1. character_inventory UNIQUE 제약 제거
ALTER TABLE character_inventory
  DROP CONSTRAINT character_inventory_user_id_character_id_key;

-- 2. weapon_inventory UNIQUE 제약 제거
ALTER TABLE weapon_inventory
  DROP CONSTRAINT weapon_inventory_user_id_weapon_id_key;

-- 3. 보유 수량 집계용 복합 인덱스 추가 (활성 아이템만, redeem_code IS NULL)
CREATE INDEX idx_character_inventory_active_count
  ON character_inventory(user_id, character_id)
  WHERE redeem_code IS NULL;

CREATE INDEX idx_weapon_inventory_active_count
  ON weapon_inventory(user_id, weapon_id)
  WHERE redeem_code IS NULL;
```

**참고**: 기존 인덱스 `idx_character_inventory_user`, `idx_weapon_inventory_user`, `idx_character_inventory_active`는 유지한다. 새 부분 인덱스는 `GROUP BY` 집계 쿼리에 최적화된다.

### 성공 기준:

#### 자동화된 검증:
- [ ] 마이그레이션이 에러 없이 적용됨
- [ ] 동일 `(user_id, character_id)` 조합으로 2개 row INSERT 가능 확인:
  ```sql
  INSERT INTO character_inventory (user_id, character_id, source)
  VALUES ('test-uuid', 'grenade', 'test');
  INSERT INTO character_inventory (user_id, character_id, source)
  VALUES ('test-uuid', 'grenade', 'test');
  -- 에러 없이 성공해야 함
  ```
- [ ] 기존 RLS 정책 정상 동작 확인
- [ ] 기존 트리거 함수 정상 동작 확인

#### 수동 검증:
- [ ] Supabase Dashboard에서 테이블 구조 확인: UNIQUE 제약 제거됨
- [ ] 새 인덱스 `idx_character_inventory_active_count`, `idx_weapon_inventory_active_count` 존재 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: 로비 인벤토리 조회 변경 — 수량 기반 쿼리

### 개요
로비 페이지의 인벤토리 조회를 수량 기반으로 변경한다. 캐릭터는 소유 여부만 필요하므로 기존 `Set<string>` 유지하되 중복 제거 처리를 추가한다. 무기는 수량 정보를 `Record<string, number>`로 전달한다.

### 필요한 변경:

#### 1. 로비 페이지 인벤토리 조회
**파일**: `apps/web/src/app/lobby/page.tsx`
**변경**: 캐릭터 조회는 기존 패턴 유지 (Set이 자동으로 중복 제거), 무기 조회는 수량 포함

```typescript
// 캐릭터: 기존과 동일 (Set이 중복 character_id를 자동 제거)
// 변경 없음 — line 38-46 유지

// 무기: 수량 포함 조회
const { data: ownedWeaponInventory } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)

// weapon_id별 보유 수량 계산
const weaponCountMap: Record<string, number> = {}
for (const item of ownedWeaponInventory ?? []) {
  const id = item.weapon_id as string
  weaponCountMap[id] = (weaponCountMap[id] ?? 0) + 1
}

// 기존 ownedWeaponIds도 유지 (하위 호환)
const ownedWeaponIds = new Set(Object.keys(weaponCountMap))
```

LobbyContent에 props 추가:
```typescript
ownedWeaponIds={Array.from(ownedWeaponIds)}
ownedWeaponCounts={weaponCountMap}
```

#### 2. LobbyContent props 타입 변경
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`
**변경**: `ownedWeaponCounts` prop 추가

```typescript
interface LobbyContentProps {
  // ... 기존 props 유지
  ownedWeaponCounts: Record<string, number>  // 추가
}
```

LobbyContent 내부에서 `ownedWeaponCounts`를 `LobbyWeaponSelector`에 전달:
```typescript
<LobbyWeaponSelector
  weapons={allWeapons}
  ownedIds={ownedWeaponSet}
  ownedCounts={ownedWeaponCounts}  // 추가
  selectedIds={selectedWeapons}
  onToggle={handleWeaponToggle}
/>
```

`StoreButton`에도 인벤토리 레코드 목록 전달 (5단계에서 사용):
```typescript
// Store용 인벤토리 레코드 조회를 page.tsx에서 추가
const { data: characterInventoryRecords } = await supabase
  .from("character_inventory")
  .select("id, character_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)

const { data: weaponInventoryRecords } = await supabase
  .from("weapon_inventory")
  .select("id, weapon_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `npx turbo typecheck`
- [ ] 빌드 성공: `npx turbo build`

#### 수동 검증:
- [ ] 로비 페이지가 정상 로드됨
- [ ] 기존 캐릭터/무기 선택 기능이 동일하게 동작
- [ ] 소유/미소유 UI 분기가 정상 표시

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 3단계: 로비 무기 선택 UI — 수량 뱃지 추가

### 개요
`LobbyWeaponSelector`에 보유 수량 뱃지를 추가한다. 무기 이미지 우측 상단에 숫자를 표시한다. `ammo=-1`(무제한)인 bazooka는 "∞" 표시한다.

### 필요한 변경:

#### 1. LobbyWeaponSelector props 및 뱃지 UI
**파일**: `apps/web/src/components/lobby/LobbyWeaponSelector.tsx`
**변경**: `ownedCounts` prop 추가, 수량 뱃지 렌더링

```typescript
interface LobbyWeaponSelectorProps {
  weapons: WeaponData[]
  ownedIds: Set<string>
  ownedCounts: Record<string, number>  // 추가
  selectedIds: string[]
  onToggle: (weaponId: string) => void
}
```

무기 이미지 영역(`div.relative` 내부, line 55)에 뱃지 추가:
```tsx
{isOwned && (
  <span className={cn(
    "absolute -top-1 -right-1 min-w-4 h-4 flex items-center justify-center",
    "rounded-full bg-amber-500 text-[10px] font-bold text-black",
    "lg:min-w-5 lg:h-5 lg:text-xs"
  )}>
    {weapon.id === "bazooka" ? "∞" : (ownedCounts[weapon.id] ?? 0)}
  </span>
)}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `npx turbo typecheck`
- [ ] 빌드 성공: `npx turbo build`

#### 수동 검증:
- [ ] 소유 무기에 수량 뱃지가 표시됨
- [ ] bazooka에 "∞" 표시됨
- [ ] 미소유 무기에는 뱃지가 표시되지 않음
- [ ] 뱃지가 무기 이미지와 겹치지 않고 우측 상단에 위치
- [ ] 모바일/데스크톱 모두에서 뱃지 크기가 적절

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 4단계: 게임 진입 시 탄약 동적 계산

### 개요
게임 진입 시 무기 보유 수량을 조회하여 `WeaponData.ammo`를 동적으로 오버라이드한다. bazooka는 항상 `ammo=-1`(무제한) 유지.

### 필요한 변경:

#### 1. 캐릭터 소유권 검증 변경
**파일**: `apps/web/src/app/game/page.tsx`
**변경**: `.single()` → `.limit(1)` 변경 (복수 row 대응)

현재 (line 30-36):
```typescript
const { data: ownership } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .eq("character_id", selectedCharacter)
  .is("redeem_code", null)
  .single()
```

변경 후:
```typescript
const { data: ownershipRows } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .eq("character_id", selectedCharacter)
  .is("redeem_code", null)
  .limit(1)

const ownership = ownershipRows && ownershipRows.length > 0
```

#### 2. 무기 보유 수량 조회 + ammo 오버라이드
**파일**: `apps/web/src/app/game/page.tsx`
**변경**: 무기 보유 수량 COUNT 후 `WeaponData.ammo` 오버라이드

기존 `ownedWeapons` 쿼리(line 44-49) 유지하되, 수량 집계 추가:

```typescript
// 무기 소유 검증 (기존)
const { data: ownedWeapons } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)
  .in("weapon_id", selectedWeaponIds)
  .is("redeem_code", null)

// 무기별 보유 수량 계산
const weaponCounts: Record<string, number> = {}
for (const w of ownedWeapons ?? []) {
  const id = w.weapon_id as string
  weaponCounts[id] = (weaponCounts[id] ?? 0) + 1
}

const ownedWeaponSet = new Set(Object.keys(weaponCounts))
const verifiedWeaponIds = selectedWeaponIds.filter((id) => ownedWeaponSet.has(id))
const finalWeaponIds = verifiedWeaponIds.length > 0 ? verifiedWeaponIds : ["bazooka"]
```

무기 데이터 fetch 후 ammo 오버라이드 (기존 line 59-65 이후):
```typescript
const { data: weaponDataRows } = await supabase
  .from("weapons")
  .select("id, name, damage, explosion_radius, ammo, mass, atlas_key, atlas_path, projectile_frame, hold_frame, sort_order")
  .in("id", finalWeaponIds)
  .order("sort_order")

// ammo를 보유 수량으로 오버라이드 (bazooka는 -1 유지)
const verifiedWeapons: WeaponData[] = (weaponDataRows ?? []).map((row) => ({
  ...row,
  ammo: row.ammo === -1 ? -1 : (weaponCounts[row.id] ?? row.ammo),
})) as WeaponData[]
```

### 데이터 흐름 (변경 후):

```
[weapon_inventory] SELECT weapon_id WHERE user_id=X AND redeem_code IS NULL
        ↓ (weapon_id별 COUNT)
weaponCounts: { grenade: 2, shotgun: 1 }
        ↓
[weapons 마스터] SELECT ... WHERE id IN (finalWeaponIds)
        ↓ (ammo 오버라이드)
WeaponData[]: [
  { id: "bazooka", ammo: -1 },     // 무제한 유지
  { id: "grenade", ammo: 2 },      // 마스터 값 3 → 보유 수량 2
  { id: "shotgun", ammo: 1 },      // 마스터 값 2 → 보유 수량 1
]
        ↓ (기존 흐름 그대로)
GameClient → PhaserGame → registry → WeaponManager.initAmmo()
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `npx turbo typecheck`
- [ ] 빌드 성공: `npx turbo build`

#### 수동 검증:
- [ ] Grenade N개 보유 시 인게임에서 N발 발사 가능
- [ ] Bazooka는 여전히 무제한 발사 가능
- [ ] 탄약 소진 후 자동 무기 전환이 정상 동작
- [ ] 인게임 WeaponSelector UI에 올바른 탄약 수 표시

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 5단계: Store UI + Export API — 개별 인벤토리 레코드 처리

### 개요
Store Drawer UI를 인벤토리 레코드별 개별 리스팅으로 변경한다. Export API는 `inventoryId`(UUID)로 특정 레코드를 지정한다. Export 후 남은 수량을 확인하여 fallback 조건을 변경한다.

### 필요한 변경:

#### 1. 로비 페이지에서 인벤토리 레코드 조회 추가
**파일**: `apps/web/src/app/lobby/page.tsx`
**변경**: Store용 인벤토리 레코드 목록 조회 추가

```typescript
// Store용 인벤토리 레코드 조회 (id 포함)
const { data: characterInventoryRecords } = await supabase
  .from("character_inventory")
  .select("id, character_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)

const { data: weaponInventoryRecords } = await supabase
  .from("weapon_inventory")
  .select("id, weapon_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)
```

LobbyContent props로 전달:
```typescript
characterInventoryRecords={(characterInventoryRecords ?? []) as { id: string; character_id: string }[]}
weaponInventoryRecords={(weaponInventoryRecords ?? []) as { id: string; weapon_id: string }[]}
```

#### 2. LobbyContent props 추가
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`
**변경**: 인벤토리 레코드 props 추가, StoreButton으로 전달

```typescript
interface LobbyContentProps {
  // ... 기존 props
  ownedWeaponCounts: Record<string, number>
  characterInventoryRecords: { id: string; character_id: string }[]  // 추가
  weaponInventoryRecords: { id: string; weapon_id: string }[]        // 추가
}
```

StoreButton에 전달:
```typescript
<StoreButton
  // ... 기존 props
  characterInventoryRecords={characterInventoryRecords}
  weaponInventoryRecords={weaponInventoryRecords}
/>
```

#### 3. StoreButton — 인벤토리 레코드별 리스팅
**파일**: `apps/web/src/components/lobby/StoreButton.tsx`
**변경**: props 타입 변경, 개별 레코드 리스팅, selectedExportItem에 inventoryId 추가

Props 변경:
```typescript
interface StoreButtonProps {
  linkedWallet: { wallet_address: string; chain: string } | null
  onWalletLinked: (wallet: { wallet_address: string; chain: string } | null) => void
  allCharacters: CharacterData[]
  ownedCharacterIds: string[]        // 유지 (호환)
  allWeapons: WeaponData[]
  ownedWeaponIds: string[]           // 유지 (호환)
  onExportComplete: (updates: { selectedCharacter: string; selectedWeapons: string[] }) => void
  characterInventoryRecords: { id: string; character_id: string }[]  // 추가
  weaponInventoryRecords: { id: string; weapon_id: string }[]        // 추가
}
```

selectedExportItem state 변경:
```typescript
const [selectedExportItem, setSelectedExportItem] = useState<{
  type: "character" | "weapon"
  inventoryId: string    // 추가: 인벤토리 레코드 UUID
  itemId: string         // 기존 id → itemId로 명확화
  name: string
} | null>(null)
```

Exportable 아이템 계산 변경 — 마스터 데이터 기준이 아닌 인벤토리 레코드 기준:
```typescript
// 캐릭터: 인벤토리 레코드별 리스팅 (기본 아이템 제외)
const exportableCharacterRecords = characterInventoryRecords
  .filter((r) => r.character_id !== "player")
  .map((r) => ({
    ...r,
    character: allCharacters.find((c) => c.id === r.character_id),
  }))
  .filter((r) => r.character != null)

// 무기: 인벤토리 레코드별 리스팅 (기본 아이템 제외)
const exportableWeaponRecords = weaponInventoryRecords
  .filter((r) => r.weapon_id !== "bazooka")
  .map((r) => ({
    ...r,
    weapon: allWeapons.find((w) => w.id === r.weapon_id),
  }))
  .filter((r) => r.weapon != null)
```

UI에서 개별 레코드 리스팅 (캐릭터 예시):
```tsx
{exportableCharacterRecords.map((record) => (
  <button
    key={record.id}  // 인벤토리 레코드 UUID를 key로
    onClick={() =>
      setSelectedExportItem({
        type: "character",
        inventoryId: record.id,
        itemId: record.character_id,
        name: record.character!.name,
      })
    }
    // ... 기존 스타일 유지, 선택 비교를 inventoryId로
    className={cn(
      // ...
      selectedExportItem?.inventoryId === record.id
        ? "bg-white/25 ring-2 ring-primary"
        : "bg-white/5 hover:bg-white/20"
    )}
  >
    {/* 기존 이미지/이름 UI 유지 */}
  </button>
))}
```

Export 요청 변경:
```typescript
const handleExport = async () => {
  if (!selectedExportItem) return
  setIsExporting(true)
  try {
    const res = await fetch("/api/store/export", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        itemType: selectedExportItem.type,
        itemId: selectedExportItem.itemId,
        inventoryId: selectedExportItem.inventoryId,  // 추가
      }),
    })
    // ... 기존 응답 처리 유지
  }
}
```

#### 4. Export API — inventoryId 기반 처리
**파일**: `apps/web/src/app/api/store/export/route.ts`
**변경**: `inventoryId` 파라미터 추가, 특정 레코드 지정, 남은 수량 확인 fallback

요청 파싱 변경 (line 28):
```typescript
const { itemType, itemId, inventoryId } = await request.json()
if (!itemType || !itemId || !inventoryId || !["character", "weapon"].includes(itemType)) {
  return Response.json({ error: "Invalid parameters" }, { status: 400 })
}
```

소유권 확인 변경 (line 65-77) — `inventoryId`로 직접 특정:
```typescript
const { data: inventoryItem } = await admin
  .from(table)
  .select("id")
  .eq("id", inventoryId)           // inventoryId(UUID)로 직접 특정
  .eq("user_id", user.id)          // 소유자 확인
  .eq(idColumn, itemId)            // itemId 일치 확인 (보안)
  .is("redeem_code", null)         // 미export 확인
  .single()
```

Export 후 fallback 변경 (line 133-158) — 남은 수량 확인:
```typescript
// 11. Export 후 남은 수량 확인
const { count: remainingCount } = await admin
  .from(table)
  .select("id", { count: "exact", head: true })
  .eq("user_id", user.id)
  .eq(idColumn, itemId)
  .is("redeem_code", null)

const { data: profile } = await admin
  .from("player_profiles")
  .select("selected_character, selected_weapons")
  .eq("id", user.id)
  .single()

let selectedCharacter = profile?.selected_character ?? "player"
let selectedWeapons: string[] = profile?.selected_weapons ?? ["bazooka"]

// 캐릭터: 남은 수량 0이면 fallback
if (itemType === "character" && selectedCharacter === itemId && (remainingCount ?? 0) === 0) {
  selectedCharacter = "player"
  await admin.from("player_profiles").update({ selected_character: "player" }).eq("id", user.id)
}

// 무기: 남은 수량 0이면 selected_weapons에서 제거
if (itemType === "weapon" && selectedWeapons.includes(itemId) && (remainingCount ?? 0) === 0) {
  selectedWeapons = selectedWeapons.filter((id) => id !== itemId)
  if (selectedWeapons.length === 0) selectedWeapons = ["bazooka"]
  await admin.from("player_profiles").update({ selected_weapons: selectedWeapons }).eq("id", user.id)
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `npx turbo typecheck`
- [ ] 빌드 성공: `npx turbo build`

#### 수동 검증:
- [ ] Store Drawer에서 동일 아이템 N개가 N개 박스로 개별 리스팅됨
- [ ] 특정 박스 선택 후 Export 시 해당 인벤토리 레코드만 처리됨
- [ ] Export 후 동일 아이템이 1개 이상 남아있으면 캐릭터/무기 선택이 변경되지 않음
- [ ] Export 후 동일 아이템이 0개가 되면 기본값으로 fallback
- [ ] 기본 아이템(player, bazooka)은 Store에 표시되지 않음

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 테스트 전략

### 통합 테스트 시나리오:
1. **복수 보유 → 게임 진입**: Grenade 3개 보유 유저가 게임 진입 → 인게임 Grenade 탄약 3발 확인
2. **Export 후 수량 감소**: Grenade 3개 중 1개 export → 로비 뱃지 2, 인게임 탄약 2발
3. **전량 Export → fallback**: Grenade 1개 보유 상태에서 export → selected_weapons에서 제거, fallback 동작
4. **Bazooka 무제한 유지**: Bazooka 보유 수량과 무관하게 항상 ammo=-1

### 수동 테스트 단계:
1. DB에 테스트 유저의 weapon_inventory에 grenade row 3개 수동 삽입
2. 로비 접속 → 무기 선택 UI에서 grenade 뱃지 "3" 확인
3. 게임 진입 → grenade 탄약 3발 확인, 3발 발사 후 소진
4. Store Drawer 열기 → grenade 박스 3개 확인
5. 1개 export → 뱃지 "2", 인게임 탄약 2발 확인
6. 2개 더 export → 뱃지 사라짐, selected_weapons에서 grenade 제거 확인

## 성능 고려 사항

- 인벤토리 조회 쿼리가 1개 추가됨 (Store용 레코드 조회), 그러나 유저당 아이템 수가 적어(수십 개 이하) 성능 영향 미미
- 새 부분 인덱스 `idx_*_active_count`가 `WHERE redeem_code IS NULL` 조건의 GROUP BY 집계를 최적화
- 게임 진입 시 무기 보유 수량 계산은 기존 `ownedWeapons` 쿼리 결과를 재사용 (추가 DB 호출 없음)

## 마이그레이션 참고 사항

- UNIQUE 제약 제거는 **비파괴적 변경** — 기존 데이터에 영향 없음
- 현재 모든 유저가 아이템을 1개씩만 보유하므로, 마이그레이션 후 즉시 동작에 변화 없음
- 복수 보유는 Import 기능 구현 후에만 실제로 발생 (별도 구현 예정)
- 롤백: UNIQUE 제약 재추가 가능 (단, 복수 보유 데이터가 있으면 먼저 중복 제거 필요)

## 참조

- 원본 연구: [thoughts/arta1069/research/2026-02-19-item-architecture-multi-ownership.md](thoughts/arta1069/research/2026-02-19-item-architecture-multi-ownership.md)
- 캐릭터 시스템 계획: [thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md](thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md)
- 무기 아이템화 계획: [thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md](thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md)
- Store/Export 계획: [thoughts/arta1069/plans/2026-02-19-store-drawer-item-export-implementation.md](thoughts/arta1069/plans/2026-02-19-store-drawer-item-export-implementation.md)
