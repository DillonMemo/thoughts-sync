# 무기 아이템화 시스템 구현 계획

## 개요

캐릭터 인벤토리 패턴(마스터 테이블 → 인벤토리 테이블 → 프로필 선택 → 서버 검증)을 무기에 그대로 적용하여, DB 기반 무기 소유/선택 시스템을 구현한다. 로비에서 보유 무기 중 최대 3개를 선택하고, 인게임에서는 선택된 무기만 사용할 수 있도록 한다.

## 현재 상태 분석

**캐릭터 시스템 (완전 구현됨):**
- `characters` 마스터 테이블 → `character_inventory` 소유 → `player_profiles.selected_character` 선택
- 로비 UI: 보유/미보유 구분, 클릭 시 DB 저장, Play 버튼 로딩 상태
- 게임 진입: 서버사이드 소유 검증 → Phaser registry → GameScene

**무기 시스템 (게임 내부만):**
- `WeaponRegistry` 객체에 3종 하드코딩 (bazooka, grenade, shotgun)
- `WeaponManager` 생성 시 전체 무기 즉시 사용 가능
- DB 기반 소유/선택 레이어 전무, 로비 UI 없음

### 핵심 발견:
- `packages/game-core/src/weapons/WeaponTypes.ts:12-43` — WeaponRegistry가 하드코딩된 Record<string, WeaponStats>
- `packages/game-core/src/weapons/WeaponManager.ts:20-24` — initAmmo()가 모든 무기를 무조건 초기화
- `apps/web/src/components/lobby/LobbyContent.tsx:48-68` — 캐릭터 선택 핸들러 + isSaving 상태 → PlayButton loading
- `apps/web/src/app/game/page.tsx:29-38` — 서버사이드 소유 검증 패턴
- `packages/game-core/src/index.ts:38-41` — Phaser registry로 데이터 전달 패턴

## 원하는 최종 상태

1. `weapons` DB 테이블에 게임플레이 속성(damage, explosionRadius, mass 등) 포함하여 마스터 데이터 관리
2. `weapon_inventory` 테이블로 유저별 무기 소유권 관리
3. `player_profiles.selected_weapons` TEXT[] 배열로 최대 3개 무기 선택 저장
4. 로비에서 보유/미보유 구분된 무기 선택 UI (다중 선택, 최대 3개)
5. 게임 진입 시 서버사이드 소유 검증 후 선택 무기만 Phaser에 전달
6. 인게임 WeaponSelector가 선택된 무기만 표시
7. WeaponManager가 DB 데이터 기반으로 동작

### 검증 방법:
- 로비에서 무기 선택/해제가 정상 동작하고 DB에 저장됨
- 미보유 무기는 선택 불가, 최대 3개 제한 동작
- 게임 진입 시 선택한 무기만 인게임에 표시
- 미소유 무기를 URL 조작으로 사용할 수 없음 (서버 검증)

## 하지 않을 것

- ForTem NFT 연동 (추후 별도 작업)
- 새로운 무기 추가 (기존 3종만 DB로 이전)
- 무기 구매/거래 시스템
- 무기별 별도 에셋 로딩 시스템 변경 (기존 tanks 아틀라스 유지)
- 무기 업그레이드/강화 시스템
- AI의 무기 인벤토리 (AI는 기존 로직 유지, 선택 무기 풀에서만 선택)

## 구현 접근 방식

캐릭터 인벤토리 구현 시 확립된 패턴을 최대한 재사용한다:
1. DB 스키마: `characters` → `weapons`, `character_inventory` → `weapon_inventory` 동일 구조
2. 로비 UI: `CharacterSelector` 패턴을 `WeaponSelector` (로비용)에 적용, 단 다중 선택 지원
3. 데이터 플로우: `selected_character` → `selected_weapons[]` 동일 경로
4. 서버 검증: `character_inventory` 소유 검증 → `weapon_inventory` 동일 패턴

---

## 1단계: DB 스키마

### 개요
`weapons` 마스터 테이블, `weapon_inventory` 소유 테이블 생성, `player_profiles`에 `selected_weapons` 배열 컬럼 추가, RLS 정책, 트리거 함수, 시드 데이터, 기존 유저 마이그레이션까지 단일 마이그레이션으로 처리한다.

### 필요한 변경:

#### 1. Supabase 마이그레이션 (MCP apply_migration)

**마이그레이션명**: `create_weapon_tables`

```sql
-- ============================================================
-- 1. weapons 마스터 테이블
-- ============================================================
CREATE TABLE weapons (
  id TEXT PRIMARY KEY,                    -- "bazooka", "grenade", "shotgun"
  name TEXT NOT NULL,                     -- 표시 이름 "Bazooka", "Grenade", "Shotgun"
  damage INTEGER NOT NULL DEFAULT 0,      -- 피해량
  explosion_radius INTEGER NOT NULL DEFAULT 0,  -- 폭발 반경
  ammo INTEGER NOT NULL DEFAULT -1,       -- 탄약 수 (-1 = 무제한)
  mass NUMERIC(6,4) NOT NULL DEFAULT 0.06, -- 투사체 질량 (Matter.js)
  sprite_key TEXT NOT NULL DEFAULT 'tanks', -- 텍스처 아틀라스 키
  sprite_frame TEXT,                      -- 투사체 스프라이트 프레임
  hold_sprite TEXT,                       -- 캐릭터가 들고 있을 때 스프라이트
  is_active BOOLEAN DEFAULT true,         -- 게임에서 선택 가능 여부
  sort_order INTEGER DEFAULT 0,           -- UI 표시 순서
  created_at TIMESTAMPTZ DEFAULT now()
);

-- ============================================================
-- 2. weapon_inventory 소유 테이블
-- ============================================================
CREATE TABLE weapon_inventory (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  weapon_id TEXT NOT NULL REFERENCES weapons(id) ON DELETE CASCADE,
  acquired_at TIMESTAMPTZ DEFAULT now(),
  source TEXT NOT NULL DEFAULT 'default',  -- "default", "fortem", "reward"
  UNIQUE(user_id, weapon_id)
);

CREATE INDEX idx_weapon_inventory_user ON weapon_inventory(user_id);

-- ============================================================
-- 3. player_profiles에 selected_weapons 컬럼 추가
-- ============================================================
ALTER TABLE player_profiles
ADD COLUMN selected_weapons TEXT[] DEFAULT '{bazooka}';

-- ============================================================
-- 4. RLS 정책
-- ============================================================

-- weapons: 모든 인증 사용자 읽기 가능
ALTER TABLE weapons ENABLE ROW LEVEL SECURITY;

CREATE POLICY "weapons_select_all"
ON weapons FOR SELECT TO authenticated
USING (true);

-- weapon_inventory: 자신의 인벤토리만 접근
ALTER TABLE weapon_inventory ENABLE ROW LEVEL SECURITY;

CREATE POLICY "weapon_inventory_select_own"
ON weapon_inventory FOR SELECT TO authenticated
USING (user_id = auth.uid());

CREATE POLICY "weapon_inventory_insert_own"
ON weapon_inventory FOR INSERT TO authenticated
WITH CHECK (user_id = auth.uid());

-- ============================================================
-- 5. 트리거: selected_weapons 검증
-- ============================================================
CREATE OR REPLACE FUNCTION validate_selected_weapons()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = ''
AS $$
BEGIN
  IF NEW.selected_weapons IS DISTINCT FROM OLD.selected_weapons THEN
    -- 최대 3개 제한
    IF array_length(NEW.selected_weapons, 1) > 3 THEN
      RAISE EXCEPTION 'Cannot select more than 3 weapons';
    END IF;

    -- 최소 1개 필수
    IF NEW.selected_weapons IS NULL OR array_length(NEW.selected_weapons, 1) IS NULL THEN
      RAISE EXCEPTION 'Must select at least 1 weapon';
    END IF;

    -- 모든 선택된 무기의 소유권 검증
    IF EXISTS (
      SELECT unnest(NEW.selected_weapons)
      EXCEPT
      SELECT weapon_id FROM public.weapon_inventory WHERE user_id = NEW.id
    ) THEN
      RAISE EXCEPTION 'User does not own all selected weapons';
    END IF;
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER check_selected_weapons_ownership
BEFORE UPDATE OF selected_weapons ON player_profiles
FOR EACH ROW
EXECUTE FUNCTION validate_selected_weapons();

-- ============================================================
-- 6. handle_new_user 트리거 업데이트
-- ============================================================
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = ''
AS $$
BEGIN
  -- 프로필 생성 (기본 캐릭터 + 기본 무기)
  INSERT INTO public.player_profiles (id, display_name, avatar_url, username, selected_character, selected_weapons)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data ->> 'full_name', NEW.raw_user_meta_data ->> 'name'),
    NEW.raw_user_meta_data ->> 'avatar_url',
    'player_' || substr(NEW.id::text, 1, 8),
    'player',
    '{bazooka}'
  );

  -- 기본 캐릭터 부여
  INSERT INTO public.character_inventory (user_id, character_id, source)
  VALUES (NEW.id, 'player', 'default');

  -- 기본 무기 부여
  INSERT INTO public.weapon_inventory (user_id, weapon_id, source)
  VALUES (NEW.id, 'bazooka', 'default');

  RETURN NEW;
END;
$$;

-- ============================================================
-- 7. 시드 데이터
-- ============================================================
INSERT INTO weapons (id, name, damage, explosion_radius, ammo, mass, sprite_key, sprite_frame, hold_sprite, is_active, sort_order) VALUES
  ('bazooka', 'Bazooka', 30, 25, -1, 0.0600, 'tanks', 'tank_bullet1.png', 'tanks_turret2.png', true, 1),
  ('grenade', 'Grenade', 35, 50, 3, 0.2500, 'tanks', 'tank_bullet3.png', 'tank_bullet3.png', true, 2),
  ('shotgun', 'Shotgun', 16, 20, 2, 0.0400, 'tanks', 'tank_bullet2.png', 'tanks_turret1.png', true, 3);

-- ============================================================
-- 8. 기존 유저 마이그레이션
-- ============================================================

-- 기존 유저에게 기본 무기 부여
INSERT INTO weapon_inventory (user_id, weapon_id, source)
SELECT id, 'bazooka', 'default'
FROM auth.users
ON CONFLICT (user_id, weapon_id) DO NOTHING;

-- 기존 유저의 selected_weapons 초기화
UPDATE player_profiles SET selected_weapons = '{bazooka}' WHERE selected_weapons IS NULL;
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 마이그레이션이 Supabase MCP를 통해 성공적으로 적용됨
- [ ] `weapons` 테이블에 3개 레코드 존재: `SELECT count(*) FROM weapons` → 3
- [ ] `weapon_inventory`에 기존 유저 × bazooka 레코드 존재
- [ ] `player_profiles.selected_weapons`에 기본값 확인
- [ ] RLS 정책 동작 확인: 인증 사용자가 weapons 조회 가능
- [ ] security advisor 확인: `get_advisors` 실행하여 누락된 RLS 없음 확인

#### 수동 검증:
- [ ] Supabase 대시보드에서 테이블 구조 확인
- [ ] `validate_selected_weapons` 트리거 테스트: 미소유 무기 선택 시 에러 발생

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: 타입 정의 및 데이터 모델

### 개요
`WeaponData` 타입을 정의하고, `WeaponRegistry`를 DB 데이터 기반으로 전환할 수 있도록 `WeaponStats`를 외부에서 주입받는 구조로 변경한다.

### 필요한 변경:

#### 1. WeaponData 타입 추가
**파일**: `packages/game-core/src/types/index.ts`
**변경**: WeaponData 인터페이스 추가

```typescript
export type CharacterType = "player" | "female" | "adventurer"

export interface CharacterData {
  id: string
  name: string
  asset_path: string
  asset_prefix: string
  sort_order: number
}

export interface WeaponData {
  id: string
  name: string
  damage: number
  explosion_radius: number
  ammo: number          // -1 = 무제한
  mass: number
  sprite_key: string
  sprite_frame: string | null
  hold_sprite: string | null
  sort_order: number
}
```

#### 2. WeaponStats 인터페이스 호환 유지 + DB → WeaponStats 변환 함수
**파일**: `packages/game-core/src/weapons/WeaponTypes.ts`
**변경**: `WeaponRegistry`를 하드코딩에서 동적 생성으로 전환

```typescript
import { WeaponData } from "@/types"

export interface WeaponStats {
  name: string
  damage: number
  explosionRadius: number
  ammo: number
  mass: number
  spriteKey: string
  spriteFrame?: string
  holdSprite?: string
}

// DB WeaponData → 게임 내부 WeaponStats 변환
export function weaponDataToStats(data: WeaponData): WeaponStats {
  return {
    name: data.name,
    damage: data.damage,
    explosionRadius: data.explosion_radius,
    ammo: data.ammo,
    mass: data.mass,
    spriteKey: data.sprite_key,
    spriteFrame: data.sprite_frame ?? undefined,
    holdSprite: data.hold_sprite ?? undefined,
  }
}

// 하드코딩된 WeaponRegistry 유지 (fallback + AI용)
export const WeaponRegistry: Record<string, WeaponStats> = {
  bazooka: {
    name: "Bazooka",
    damage: 30,
    explosionRadius: 25,
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
    damage: 16,
    explosionRadius: 20,
    ammo: 2,
    mass: 0.04,
    spriteKey: "tanks",
    spriteFrame: "tank_bullet2.png",
    holdSprite: "tanks_turret1.png",
  },
}
```

#### 3. WeaponManager에 선택된 무기만 초기화하는 기능 추가
**파일**: `packages/game-core/src/weapons/WeaponManager.ts`
**변경**: 생성자에서 선택된 무기 목록을 받아 초기화

```typescript
import * as Phaser from "phaser"
import { EventBus, GameEvents } from "@/EventBus"
import { WeaponRegistry, WeaponStats } from "@/weapons/WeaponTypes"
import { BazookaProjectile } from "@/weapons/Bazooka"
import { GrenadeProjectile } from "@/weapons/Grenade"
import { createShotgunBlast } from "@/weapons/Shotgun"
import { Projectile } from "@/entities/Projectile"

export class WeaponManager {
  private scene: Phaser.Scene
  private currentWeapon: string = "bazooka"
  private ammo: Record<string, number> = {}
  private activeProjectiles: Projectile[] = []
  private availableWeapons: Record<string, WeaponStats> = {}

  constructor(scene: Phaser.Scene, selectedWeapons?: Record<string, WeaponStats>) {
    this.scene = scene
    if (selectedWeapons && Object.keys(selectedWeapons).length > 0) {
      this.availableWeapons = selectedWeapons
    } else {
      this.availableWeapons = WeaponRegistry  // fallback
    }
    this.currentWeapon = Object.keys(this.availableWeapons)[0] ?? "bazooka"
    this.initAmmo()
  }

  private initAmmo(): void {
    Object.entries(this.availableWeapons).forEach(([key, stats]) => {
      this.ammo[key] = stats.ammo
    })
  }

  getAvailableWeapons(): Record<string, WeaponStats> {
    return this.availableWeapons
  }

  getAvailableWeaponKeys(): string[] {
    return Object.keys(this.availableWeapons)
  }

  selectWeapon(weaponKey: string): boolean {
    if (!this.availableWeapons[weaponKey]) return false
    if (this.ammo[weaponKey] === 0) return false

    this.currentWeapon = weaponKey
    EventBus.emit(GameEvents.WEAPON_SELECTED, {
      weapon: weaponKey,
      stats: this.availableWeapons[weaponKey],
      ammo: this.ammo[weaponKey],
    })
    return true
  }

  getCurrentWeapon(): WeaponStats {
    return this.availableWeapons[this.currentWeapon]
  }

  getCurrentWeaponKey(): string {
    return this.currentWeapon
  }

  getAmmo(weaponKey: string): number {
    return this.ammo[weaponKey] ?? 0
  }

  // fire(), update(), applyWindToAll(), hasActiveProjectiles(), destroy()는 기존 유지
  // ...
}
```

#### 4. GameOptions 인터페이스 확장
**파일**: `packages/game-core/src/index.ts`
**변경**: `player1Weapons` 추가

```typescript
import { WeaponData } from "./types"

export interface GameOptions {
  parent: HTMLElement
  player1CharacterType: CharacterType
  player1Weapons?: WeaponData[]  // DB에서 조회된 선택 무기 데이터
}
```

`createGame` 함수에서 registry에 저장:
```typescript
const game = new Phaser.Game(config)
game.registry.set("player1CharacterType", options.player1CharacterType ?? "player")
game.registry.set("player1Weapons", options.player1Weapons ?? null)
return game
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 빌드 성공: `cd packages/game-core && npx tsc --noEmit`
- [ ] 기존 게임 로직이 `selectedWeapons` 미전달 시 WeaponRegistry fallback으로 정상 동작

#### 수동 검증:
- [ ] 기존 게임 플레이가 변경 없이 정상 동작 (회귀 없음)

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 3단계: 로비 UI

### 개요
로비 서버 컴포넌트에서 무기 데이터를 fetch하고, `WeaponSelector` 로비 컴포넌트를 생성하여 다중 선택(최대 3개)을 지원한다. 캐릭터 선택과 동일하게 선택 변경 시 `isSaving` 상태로 Play 버튼 로딩 처리한다.

### 필요한 변경:

#### 1. 로비 서버 컴포넌트에 무기 데이터 fetch 추가
**파일**: `apps/web/src/app/lobby/page.tsx`
**변경**: weapons, weapon_inventory 조회 추가

기존 캐릭터 데이터 fetch 패턴과 동일하게:
```typescript
// 활성 무기 전체 목록 조회 (sort_order 순)
const { data: allWeapons } = await supabase
  .from("weapons")
  .select("id, name, damage, explosion_radius, ammo, mass, sprite_key, sprite_frame, hold_sprite, sort_order")
  .eq("is_active", true)
  .order("sort_order")

// 소유 무기 ID 목록 조회
const { data: ownedWeaponInventory } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)

const ownedWeaponIds = new Set(
  (ownedWeaponInventory ?? []).map((item) => item.weapon_id as string)
)
```

LobbyContent에 새로운 props 추가:
```typescript
<LobbyContent
  // ... 기존 props
  allWeapons={allWeapons ?? []}
  ownedWeaponIds={Array.from(ownedWeaponIds)}
  selectedWeapons={profile?.selected_weapons ?? ["bazooka"]}
/>
```

#### 2. LobbyContent 무기 선택 핸들러 및 레이아웃 업데이트
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`
**변경**: WeaponData import, 새 props, 무기 선택 상태/핸들러, 레이아웃에 WeaponSelector 추가

Props 인터페이스 확장:
```typescript
interface LobbyContentProps {
  // ... 기존
  allWeapons: WeaponData[]
  ownedWeaponIds: string[]
  selectedWeapons: string[]
}
```

상태 및 핸들러 추가:
```typescript
const [selectedWeapons, setSelectedWeapons] = useState<string[]>(initialSelectedWeapons)
const ownedWeaponSet = new Set(ownedWeaponIds)

const handleWeaponToggle = async (weaponId: string) => {
  if (!ownedWeaponSet.has(weaponId)) return

  let newSelection: string[]
  if (selectedWeapons.includes(weaponId)) {
    // 해제: 최소 1개는 유지
    if (selectedWeapons.length <= 1) return
    newSelection = selectedWeapons.filter((id) => id !== weaponId)
  } else {
    // 선택: 최대 3개 제한
    if (selectedWeapons.length >= 3) return
    newSelection = [...selectedWeapons, weaponId]
  }

  setSelectedWeapons(newSelection)
  setIsSaving(true)

  try {
    const supabase = createClient()
    const { data: { user } } = await supabase.auth.getUser()
    if (user) {
      await supabase
        .from("player_profiles")
        .update({ selected_weapons: newSelection })
        .eq("id", user.id)
    }
  } finally {
    setIsSaving(false)
  }
}
```

레이아웃에 WeaponSelector 추가 (CharacterSelector와 같은 행, 또는 그 아래):
```tsx
{/* 하단 중앙(desktop) / 세 번째(mobile): 캐릭터 + 무기 선택 */}
<div className={cn(
  "z-10 shrink-0 self-center flex flex-col gap-2",
  "desktop:col-start-2 desktop:row-start-3 desktop:self-end"
)}>
  <CharacterSelector
    characters={allCharacters}
    ownedIds={ownedSet}
    selectedId={selectedCharacter}
    onSelect={handleCharacterSelect}
  />
  <LobbyWeaponSelector
    weapons={allWeapons}
    ownedIds={ownedWeaponSet}
    selectedIds={selectedWeapons}
    onToggle={handleWeaponToggle}
  />
</div>
```

#### 3. 로비용 WeaponSelector 컴포넌트 생성
**파일**: `apps/web/src/components/lobby/LobbyWeaponSelector.tsx` (신규)
**변경**: CharacterSelector 패턴 기반, 다중 선택 지원

```tsx
"use client"

import Image from "next/image"
import { Lock } from "lucide-react"
import { cn } from "@/lib/utils"
import { WeaponData } from "@repo/game-core"

interface LobbyWeaponSelectorProps {
  weapons: Array<Omit<WeaponData, "sort_order" | "damage" | "explosion_radius" | "ammo" | "mass">>
  ownedIds: Set<string>
  selectedIds: string[]
  onToggle: (weaponId: string) => void
}

export function LobbyWeaponSelector({
  weapons,
  ownedIds,
  selectedIds,
  onToggle,
}: LobbyWeaponSelectorProps) {
  return (
    <div className={cn("max-w-fit")}>
      <div
        className={cn(
          "flex gap-1.5 justify-center overflow-x-auto",
          "rounded-lg bg-background/40 backdrop-blur-sm p-2",
          "lg:gap-2 lg:rounded-xl lg:p-3"
        )}
      >
        {weapons.map((weapon) => {
          const isOwned = ownedIds.has(weapon.id)
          const isSelected = selectedIds.includes(weapon.id)

          return (
            <button
              key={weapon.id}
              onClick={() => isOwned && onToggle(weapon.id)}
              className={cn(
                "size-14 flex flex-col items-center justify-center gap-0.5",
                "relative rounded-md p-1 transition-all",
                "lg:size-18 lg:gap-1 lg:rounded-lg lg:p-1.5",
                isOwned
                  ? "hover:bg-white/20 cursor-pointer"
                  : "cursor-not-allowed",
                isSelected ? "bg-white/25 ring-2 ring-amber-400" : "bg-white/5"
              )}
            >
              <div className="relative flex size-9 items-center justify-center lg:size-12">
                <Image
                  src={`/assets/resource/Spritesheet/weapon_thumbs/${weapon.id}.png`}
                  alt={weapon.name}
                  fill
                  sizes="100%"
                  className={cn(
                    "object-contain",
                    !isOwned && "brightness-[0.3] grayscale"
                  )}
                />
                {!isOwned && (
                  <Lock className="absolute inset-0 m-auto size-4 text-white/60 lg:size-5" />
                )}
              </div>
              <span className={cn(
                "text-[10px] leading-tight lg:text-xs",
                isSelected ? "text-amber-400 font-semibold" : "text-white/60"
              )}>
                {weapon.name}
              </span>
            </button>
          )
        })}
      </div>
    </div>
  )
}
```

> **Note**: 무기 썸네일 이미지 경로 (`/assets/resource/Spritesheet/weapon_thumbs/{id}.png`)는 실제 에셋 구조에 맞게 조정 필요. tanks 아틀라스에서 개별 이미지를 추출하거나, holdSprite를 아틀라스에서 직접 참조하는 방식으로 구현할 수도 있음. 구현 시점에 에셋 상황에 따라 결정.

#### 4. types export 업데이트
**파일**: `packages/game-core/src/index.ts`
**변경**: `WeaponData` export 확인 (types/index.ts에서 이미 export * 됨)

기존 `export * from "./types"`가 WeaponData를 포함하므로 별도 변경 불필요.

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 빌드 성공: `cd apps/web && npx tsc --noEmit`
- [ ] 로비 페이지 정상 로드 (Next.js dev server)

#### 수동 검증:
- [ ] 로비에서 무기 목록이 보유/미보유 구분되어 표시됨
- [ ] 보유 무기 클릭 시 선택/해제 토글 동작
- [ ] 최대 3개 초과 선택 시 추가 안 됨
- [ ] 최소 1개 미만으로 해제 안 됨
- [ ] 선택 변경 시 Play 버튼에 로딩 표시
- [ ] 선택이 DB에 저장되고 새로고침 후에도 유지됨

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 4단계: 게임 진입 데이터 플로우

### 개요
`game/page.tsx`에서 선택된 무기의 소유 검증 및 무기 데이터 fetch → props 체인을 통해 Phaser에 전달한다.

### 필요한 변경:

#### 1. 게임 서버 컴포넌트에 무기 검증 추가
**파일**: `apps/web/src/app/game/page.tsx`
**변경**: selected_weapons 조회, 소유 검증, 무기 데이터 fetch

```typescript
import { type CharacterType, type WeaponData } from "@repo/game-core"

// 1. player_profiles에서 selected_character + selected_weapons 조회
const { data: profile } = await supabase
  .from("player_profiles")
  .select("selected_character, selected_weapons, display_name, username")
  .eq("id", user.id)
  .single()

const selectedWeaponIds: string[] = profile?.selected_weapons ?? ["bazooka"]

// 2. weapon_inventory에서 선택된 무기들의 소유권 일괄 검증
const { data: ownedWeapons } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)
  .in("weapon_id", selectedWeaponIds)

const ownedWeaponSet = new Set(
  (ownedWeapons ?? []).map((w) => w.weapon_id as string)
)

// 3. 소유 검증: 미소유 무기 제거, 빈 배열이면 기본값 fallback
const verifiedWeaponIds = selectedWeaponIds.filter((id) => ownedWeaponSet.has(id))
const finalWeaponIds = verifiedWeaponIds.length > 0 ? verifiedWeaponIds : ["bazooka"]

// 4. 검증된 무기 데이터 fetch
const { data: weaponDataRows } = await supabase
  .from("weapons")
  .select("id, name, damage, explosion_radius, ammo, mass, sprite_key, sprite_frame, hold_sprite, sort_order")
  .in("id", finalWeaponIds)
  .order("sort_order")

const verifiedWeapons: WeaponData[] = (weaponDataRows ?? []) as WeaponData[]
```

GameClient에 전달:
```typescript
<GameClient
  characterType={verifiedCharacter}
  playerName={playerName}
  matchId={matchId}
  weapons={verifiedWeapons}
/>
```

#### 2. GameClient props 확장
**파일**: `apps/web/src/app/game/GameClient.tsx`
**변경**: weapons prop 추가

```typescript
import { type CharacterType, type WeaponData } from "@repo/game-core"

interface GameClientProps {
  characterType: CharacterType
  playerName: string
  matchId: string | null
  weapons: WeaponData[]
}

export default function GameClient({ characterType, playerName, matchId, weapons }: GameClientProps) {
  return <GameWrapper characterType={characterType} playerName={playerName} matchId={matchId} weapons={weapons} />
}
```

#### 3. GameWrapper props 확장
**파일**: `apps/web/src/components/game/GameWrapper.tsx`
**변경**: weapons prop → PhaserGame에 전달

```typescript
interface GameWrapperProps {
  characterType: CharacterType
  playerName: string
  matchId: string | null
  weapons: WeaponData[]
}

// ... (기존 로직 유지)

return (
  <div className="relative min-h-dvh bg-gray-900">
    <PhaserGame characterType={characterType} weapons={weapons} />
    {/* ... */}
  </div>
)
```

#### 4. PhaserGame props 확장 → createGame에 전달
**파일**: `apps/web/src/components/game/PhaserGame.tsx`
**변경**: weapons prop → createGame에 전달

```typescript
import { type CharacterType, type WeaponData } from "@repo/game-core"

interface PhaserGameProps {
  onGameReady?: (game: Game) => void
  characterType: CharacterType
  weapons: WeaponData[]
}

// createGame 호출 부분:
const game = createGame({
  parent: containerRef.current,
  player1CharacterType: characterType,
  player1Weapons: weapons,
})
```

#### 5. createGame에서 registry 저장
**파일**: `packages/game-core/src/index.ts`
**변경**: player1Weapons를 registry에 저장

```typescript
const game = new Phaser.Game(config)
game.registry.set("player1CharacterType", options.player1CharacterType ?? "player")
game.registry.set("player1Weapons", options.player1Weapons ?? null)
return game
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 빌드 성공: `cd apps/web && npx tsc --noEmit`
- [ ] `cd packages/game-core && npx tsc --noEmit` 성공

#### 수동 검증:
- [ ] 게임 진입 시 에러 없음
- [ ] 브라우저 콘솔에서 registry에 weapons 데이터가 저장되었는지 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 5단계: 게임 코어 연동

### 개요
GameScene에서 registry의 무기 데이터를 읽어 WeaponManager에 전달하고, 인게임 WeaponSelector와 키보드 단축키, AIController가 선택된 무기만 사용하도록 한다.

### 필요한 변경:

#### 1. GameScene에서 무기 데이터 읽기 및 WeaponManager 초기화
**파일**: `packages/game-core/src/scenes/GameScene.ts`
**변경**: `createSystems()`에서 registry의 무기 데이터를 WeaponManager에 전달

```typescript
import { WeaponData } from "@/types"
import { weaponDataToStats, WeaponStats } from "@/weapons/WeaponTypes"

private createSystems(): void {
  // 턴 시스템
  this.turnSystem = new TurnSystem(this, {
    turnDuration: 30000,
    playerCount: this.players.length,
  })

  // registry에서 선택된 무기 데이터 읽기
  const weaponDataList = this.registry.get("player1Weapons") as WeaponData[] | null
  let selectedWeapons: Record<string, WeaponStats> | undefined

  if (weaponDataList && weaponDataList.length > 0) {
    selectedWeapons = {}
    for (const data of weaponDataList) {
      selectedWeapons[data.id] = weaponDataToStats(data)
    }
  }

  // 무기 매니저 (선택된 무기로 초기화)
  this.weaponManager = new WeaponManager(this, selectedWeapons)

  // ... (windSystem, damageSystem, controllers 기존 유지)

  this.setupWeaponKeys()
}
```

#### 2. setupWeaponKeys를 선택된 무기 기반으로 변경
**파일**: `packages/game-core/src/scenes/GameScene.ts`
**변경**: `setupWeaponKeys()`가 WeaponRegistry 대신 `weaponManager.getAvailableWeaponKeys()` 사용

```typescript
private setupWeaponKeys(): void {
  const weapons = this.weaponManager.getAvailableWeaponKeys()

  this.input.keyboard?.on("keydown-ONE", () => {
    if (weapons[0]) this.selectWeaponForCurrentPlayer(weapons[0])
  })

  this.input.keyboard?.on("keydown-TWO", () => {
    if (weapons[1]) this.selectWeaponForCurrentPlayer(weapons[1])
  })

  this.input.keyboard?.on("keydown-THREE", () => {
    if (weapons[2]) this.selectWeaponForCurrentPlayer(weapons[2])
  })
}
```

#### 3. 인게임 WeaponSelector가 선택된 무기만 표시
**파일**: `packages/game-core/src/ui/WeaponSelector.ts`
**변경**: `WeaponRegistry` 대신 `weaponManager.getAvailableWeapons()` 사용

`createBoxes()` 변경:
```typescript
private createBoxes(): void {
  const weapons = Object.keys(this.weaponManager.getAvailableWeapons())
  const registry = this.weaponManager.getAvailableWeapons()

  weapons.forEach((weaponKey, index) => {
    const stats = registry[weaponKey]
    // ... (기존 로직 동일, WeaponRegistry → registry 참조만 변경)
  })
}
```

`onWeaponSelected()` 변경:
```typescript
private onWeaponSelected(data: { weapon: string }): void {
  if (this.isDestroyed) return
  const weapons = Object.keys(this.weaponManager.getAvailableWeapons())
  this.selectedIndex = weapons.indexOf(data.weapon)
  this.updateSelection()
  this.updateAmmo()
}
```

`updateAmmo()` 변경:
```typescript
private updateAmmo(): void {
  const weapons = Object.keys(this.weaponManager.getAvailableWeapons())
  weapons.forEach((weaponKey, index) => {
    // ... (기존 로직 동일)
  })
}
```

#### 4. AIController 무기 선택을 사용 가능한 무기 풀에서만 선택
**파일**: `packages/game-core/src/systems/AIController.ts`
**변경**: `selectWeapon()`에서 사용 가능한 무기 확인

```typescript
private selectWeapon(distance: number): void {
  const available = this.config.weaponManager.getAvailableWeaponKeys()

  if (distance < 150 && available.includes("shotgun") && this.config.weaponManager.getAmmo("shotgun") > 0) {
    this.config.weaponManager.selectWeapon("shotgun")
  } else if (available.includes("bazooka")) {
    this.config.weaponManager.selectWeapon("bazooka")
  } else {
    // fallback: 첫 번째 사용 가능한 무기 선택
    this.config.weaponManager.selectWeapon(available[0])
  }
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 빌드 성공: `cd packages/game-core && npx tsc --noEmit`
- [ ] `cd apps/web && npx tsc --noEmit` 성공

#### 수동 검증:
- [ ] 로비에서 2개 무기만 선택 후 게임 진입 → 인게임 WeaponSelector에 2개만 표시
- [ ] 로비에서 3개 무기 선택 후 게임 진입 → 인게임 WeaponSelector에 3개 표시
- [ ] 키보드 단축키(1, 2, 3)가 선택된 무기에 매핑됨
- [ ] AI가 선택된 무기 풀에서만 무기를 선택함
- [ ] 탄약 소모 및 무기 선택 UI 업데이트 정상 동작
- [ ] 게임 전체 플로우 (로비 → 무기 선택 → 게임 → 발사 → 결과) 정상

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 테스트 전략

### 통합 테스트 시나리오:
1. **신규 유저 플로우**: 회원가입 → 기본 무기(bazooka) 자동 부여 확인 → 로비에서 bazooka 선택 상태 → 게임 진입 → bazooka만 사용 가능
2. **다중 무기 보유 유저**: 3종 무기 보유 → 2개 선택 → 게임 진입 → 2개만 표시 → 키보드/클릭 선택 동작
3. **소유 검증**: DB에서 직접 `weapon_inventory` 삭제 → 게임 진입 시 미소유 무기 제거 → bazooka fallback
4. **AI 무기 선택**: 선택 무기에 shotgun 미포함 시 AI가 bazooka만 사용

### 수동 테스트 단계:
1. 로비에서 무기 선택 UI 표시 확인
2. 무기 선택/해제 토글 동작 확인
3. 최대 3개 / 최소 1개 제한 동작 확인
4. Play 버튼 로딩 상태 확인 (캐릭터/무기 선택 시 모두)
5. 새로고침 후 선택 유지 확인
6. 게임 진입 후 인게임 WeaponSelector에 선택 무기만 표시 확인
7. 키보드 단축키 매핑 확인
8. AI 무기 선택 확인
9. 무기 발사 및 데미지 정상 동작 확인

## 성능 고려 사항

- 무기 데이터는 3개뿐이므로 DB 쿼리 성능 이슈 없음
- 로비 페이지: 무기 관련 2개 쿼리 추가 (weapons, weapon_inventory) → 캐릭터와 동일한 크기
- 게임 진입: 무기 관련 2개 쿼리 추가 (weapon_inventory IN 검증, weapons 데이터 fetch) → 미미한 영향
- Phaser registry에 저장되는 데이터량: WeaponData[] 최대 3개 → 무시 가능

## 마이그레이션 참고 사항

- 기존 유저에게 자동으로 bazooka 기본 부여 (weapon_inventory INSERT)
- `selected_weapons` 기본값 `'{bazooka}'`로 설정
- `handle_new_user` 트리거 업데이트로 신규 유저도 자동 처리
- 롤백 시: `weapons`, `weapon_inventory` 테이블 DROP + `player_profiles.selected_weapons` DROP COLUMN + `handle_new_user` 원복

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md`
- 캐릭터 시스템 계획: `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md`
- 캐릭터 DB 마이그레이션: `20260213091218_create_character_tables`, `20260213091228_update_user_trigger_with_character`
