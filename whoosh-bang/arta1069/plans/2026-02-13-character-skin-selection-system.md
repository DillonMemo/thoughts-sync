# 캐릭터 스킨 선택 시스템 구현 계획

## 개요

캐릭터 소유권과 선택 시스템을 DB 기반으로 구현한다. 로비에서 하단 썸네일 패널을 통해 소유한 캐릭터를 선택하고, 선택된 캐릭터를 Phaser 게임에 전달하여 표시한다. 캐릭터 마스터 데이터와 소유권은 모두 Supabase DB에서 관리하며, 향후 Fortem 플랫폼 연동을 고려한 유연한 설계를 적용한다.

## 현재 상태 분석

### 존재하는 것
- 5종 캐릭터 에셋 (Player, Adventurer, Female, Soldier, Zombie) — 동일한 구조 (24 Poses + 9 Limbs + 1 Tilesheet)
- BootScene에서 3종(player, adventurer, female) 에셋 로딩 및 애니메이션 생성
- `CharacterConfig.characterType: string` — 이미 유연한 설계
- 애니메이션 키 동적 생성: `${characterType}_${state}`
- Supabase Auth (Google OAuth) + 자동 프로필 생성 트리거
- Sui 지갑 연동 완료

### 존재하지 않는 것
- DB에 캐릭터 마스터 데이터 없음
- 캐릭터 소유권 테이블 없음
- `player_profiles`에 선택된 캐릭터 컬럼 없음
- 로비에서 캐릭터 선택 UI 없음
- 로비 → 게임으로 캐릭터 타입 전달 경로 없음

### 핵심 발견
- `CharacterDisplay.tsx:5` — `CHARACTER_PATH` 하드코딩
- `GameScene.ts:259,270` — `characterType: "player"` 하드코딩
- `BootScene.ts:55-59` — CHARACTERS 상수에 3종만 등록
- `BootScene.ts:86-95` — Registry 패턴으로 배경 설정 전달 (동일 패턴 활용 가능)
- `packages/game-core/src/index.ts:5` — `createGame(parent: HTMLElement)` 파라미터 없음
- `PhaserGame.tsx:54` — `createGame(containerRef.current)` 파라미터 없이 호출
- `PlayButton.tsx:13` — `router.push("/game")` 파라미터 없이 이동

## 원하는 최종 상태

1. 로비 화면 중앙에 선택된 캐릭터가 표시됨
2. 화면 하단에 소유한 캐릭터들의 썸네일 패널이 표시됨
3. 썸네일 클릭으로 캐릭터를 선택하면 중앙 표시와 DB가 즉시 업데이트됨
4. "PLAY" 버튼 클릭 시 선택된 캐릭터로 게임이 시작됨
5. Player 2는 기본 캐릭터("player")로 표시됨
6. 신규 유저 가입 시 기본 캐릭터(Player)가 자동 할당됨

### 검증 방법
- DB에서 `characters`, `character_inventory` 테이블 조회
- 로비에서 캐릭터 선택 UI가 동작하고, 선택한 캐릭터가 게임에 반영되는지 확인
- 신규 유저 가입 → 자동으로 기본 캐릭터 할당 확인

## 하지 않을 것

- Soldier/Zombie 캐릭터 활성화 (미활성 상태 유지, DB에만 등록)
- NFT 민팅/온체인 조회 로직 (Fortem 연동은 별도 작업)
- 캐릭터 상점/구매 UI
- 캐릭터별 능력치 차이 (순수 코스메틱)
- 멀티플레이어 네트워킹 (현재 로컬 플레이)
- AI 플레이어 시스템

## 구현 접근 방식

DB 마이그레이션 → 로비 UI → 게임 코어 연동 순서로 진행. 각 단계가 독립적으로 검증 가능하도록 설계.

---

## 1단계: DB 마이그레이션

### 개요
캐릭터 마스터 데이터, 소유권, 선택 상태를 관리하는 DB 스키마를 생성한다.

### 필요한 변경:

#### 1. characters 마스터 테이블 생성

**Supabase 마이그레이션** (MCP `apply_migration` 사용)
**마이그레이션 이름**: `create_character_tables`

```sql
-- 캐릭터 마스터 테이블
CREATE TABLE characters (
  id TEXT PRIMARY KEY,           -- "player", "adventurer", "female" 등
  name TEXT NOT NULL,            -- 표시 이름 "Player", "Adventurer" 등
  asset_path TEXT NOT NULL,      -- 에셋 폴더명 (PascalCase) "Player", "Adventurer"
  asset_prefix TEXT NOT NULL,    -- 파일 접두사 (lowercase) "player", "adventurer"
  is_active BOOLEAN DEFAULT false, -- 게임에서 선택 가능 여부
  sort_order INTEGER DEFAULT 0,  -- UI 표시 순서
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 캐릭터 소유권 테이블
CREATE TABLE character_inventory (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  character_id TEXT NOT NULL REFERENCES characters(id) ON DELETE CASCADE,
  acquired_at TIMESTAMPTZ DEFAULT now(),
  source TEXT NOT NULL DEFAULT 'default', -- "default", "fortem", "reward" 등
  UNIQUE(user_id, character_id)
);

CREATE INDEX idx_character_inventory_user ON character_inventory(user_id);

-- player_profiles에 selected_character 컬럼 추가
ALTER TABLE player_profiles
ADD COLUMN selected_character TEXT REFERENCES characters(id) DEFAULT 'player';

-- RLS 활성화
ALTER TABLE characters ENABLE ROW LEVEL SECURITY;
ALTER TABLE character_inventory ENABLE ROW LEVEL SECURITY;

-- characters: 모든 인증 사용자가 조회 가능 (읽기 전용)
CREATE POLICY "characters_select_all"
ON characters FOR SELECT TO authenticated
USING (true);

-- character_inventory: 자신의 인벤토리만 조회 가능
CREATE POLICY "inventory_select_own"
ON character_inventory FOR SELECT TO authenticated
USING (user_id = auth.uid());

-- character_inventory: 자신의 인벤토리에만 추가 가능
CREATE POLICY "inventory_insert_own"
ON character_inventory FOR INSERT TO authenticated
WITH CHECK (user_id = auth.uid());

-- 초기 캐릭터 데이터 시드
INSERT INTO characters (id, name, asset_path, asset_prefix, is_active, sort_order) VALUES
  ('player', 'Player', 'Player', 'player', true, 1),
  ('adventurer', 'Adventurer', 'Adventurer', 'adventurer', true, 2),
  ('female', 'Female', 'Female', 'female', true, 3),
  ('soldier', 'Soldier', 'Soldier', 'soldier', false, 4),
  ('zombie', 'Zombie', 'Zombie', 'zombie', false, 5);

-- 기존 유저에게 기본 캐릭터 부여
INSERT INTO character_inventory (user_id, character_id, source)
SELECT id, 'player', 'default'
FROM auth.users
ON CONFLICT (user_id, character_id) DO NOTHING;

-- 기존 유저의 selected_character 업데이트
UPDATE player_profiles SET selected_character = 'player' WHERE selected_character IS NULL;

-- selected_character 업데이트 시 소유권 검증 (DB 레벨 보안)
-- 유저가 소유하지 않은 캐릭터를 selected_character로 설정하는 것을 방지
CREATE OR REPLACE FUNCTION validate_selected_character()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = ''
AS $$
BEGIN
  -- selected_character가 변경된 경우에만 검증
  IF NEW.selected_character IS DISTINCT FROM OLD.selected_character THEN
    IF NOT EXISTS (
      SELECT 1 FROM public.character_inventory
      WHERE user_id = NEW.id AND character_id = NEW.selected_character
    ) THEN
      RAISE EXCEPTION 'User does not own character: %', NEW.selected_character;
    END IF;
  END IF;
  RETURN NEW;
END;
$$;

CREATE TRIGGER check_selected_character_ownership
BEFORE UPDATE OF selected_character ON player_profiles
FOR EACH ROW
EXECUTE FUNCTION validate_selected_character();
```

#### 2. 자동 프로필 생성 트리거 수정

**마이그레이션 이름**: `update_user_trigger_with_character`

```sql
-- 기존 트리거 함수 교체
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = ''
AS $$
BEGIN
  -- 프로필 생성
  INSERT INTO public.player_profiles (id, display_name, avatar_url, username, selected_character)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data ->> 'full_name', NEW.raw_user_meta_data ->> 'name'),
    NEW.raw_user_meta_data ->> 'avatar_url',
    'player_' || substr(NEW.id::text, 1, 8),
    'player'
  );

  -- 기본 캐릭터 부여
  INSERT INTO public.character_inventory (user_id, character_id, source)
  VALUES (NEW.id, 'player', 'default');

  RETURN NEW;
END;
$$;
```

### 성공 기준:

#### 자동화된 검증:
- [x] 마이그레이션이 오류 없이 적용됨 (MCP `apply_migration`)
- [x] `SELECT * FROM characters` — 5개 레코드 반환
- [x] `SELECT * FROM character_inventory` — 기존 유저 수만큼 레코드 존재
- [x] `SELECT selected_character FROM player_profiles` — 모든 행이 'player'

#### 수동 검증:
- [x] 새 유저 가입 시 `character_inventory`에 'player' 자동 추가 확인
- [x] 새 유저의 `player_profiles.selected_character`가 'player'인지 확인
- [x] RLS 정책이 올바르게 적용되는지 확인 (다른 유저의 인벤토리 조회 불가)

---

## 2단계: 로비 UI — 캐릭터 선택 시스템

### 개요
로비에서 소유한 캐릭터를 조회하고, 하단 썸네일 패널로 선택할 수 있는 UI를 구현한다.

### 필요한 변경:

#### 1. 서버 사이드 데이터 로딩 확장
**파일**: `apps/web/src/app/lobby/page.tsx`
**변경**: 캐릭터 인벤토리 + 선택된 캐릭터 정보 추가 조회

```typescript
// 기존 profile/wallet 조회 이후에 추가

// 소유 캐릭터 목록 조회 (characters 테이블 JOIN)
const { data: ownedCharacters } = await supabase
  .from("character_inventory")
  .select("character_id, characters(id, name, asset_path, asset_prefix, sort_order)")
  .eq("user_id", user.id)
  .order("characters(sort_order)")

// LobbyContent에 전달
<LobbyContent
  profile={profile}
  wallet={wallet}
  userEmail={user.email ?? ""}
  avatarUrl={user.user_metadata?.avatar_url}
  ownedCharacters={ownedCharacters ?? []}
  selectedCharacter={profile?.selected_character ?? "player"}
/>
```

#### 2. LobbyContent Props 확장
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`
**변경**: `LobbyContentProps` 인터페이스에 캐릭터 관련 props 추가

```typescript
interface CharacterData {
  id: string
  name: string
  asset_path: string
  asset_prefix: string
  sort_order: number
}

interface LobbyContentProps {
  profile: { /* 기존 필드 */ } | null
  wallet: { wallet_address: string; chain: string } | null
  userEmail: string
  avatarUrl?: string
  ownedCharacters: Array<{
    character_id: string
    characters: CharacterData
  }>
  selectedCharacter: string
}
```

LobbyContent 내부에서 `selectedCharacter` 상태를 관리하고, `CharacterDisplay`와 `CharacterSelector`에 전달:

```typescript
export function LobbyContent({ ..., ownedCharacters, selectedCharacter: initialSelected }: LobbyContentProps) {
  const [selectedCharacter, setSelectedCharacter] = useState(initialSelected)

  // 선택된 캐릭터의 상세 정보 찾기
  const selectedCharData = ownedCharacters.find(
    (c) => c.character_id === selectedCharacter
  )?.characters

  const handleCharacterSelect = async (characterId: string) => {
    // 클라이언트 사이드 소유 여부 확인 (이미 로딩된 목록에서)
    const isOwned = ownedCharacters.some((c) => c.character_id === characterId)
    if (!isOwned) return

    setSelectedCharacter(characterId) // 즉시 UI 반영 (optimistic update)

    const supabase = createClient()
    await supabase
      .from("player_profiles")
      .update({ selected_character: characterId })
      .eq("id", (await supabase.auth.getUser()).data.user?.id)
  }

  return (
    <div className="relative h-screen w-screen overflow-hidden">
      {/* ... 기존 배경, 오버레이, 프로필 카드, 지갑 버튼 ... */}

      {/* 중앙: 캐릭터 (선택된 캐릭터 동적 표시) */}
      <CharacterDisplay character={selectedCharData ?? null} />

      {/* 하단 중앙: 캐릭터 선택 패널 */}
      <CharacterSelector
        characters={ownedCharacters}
        selectedId={selectedCharacter}
        onSelect={handleCharacterSelect}
      />

      {/* 우하단: PLAY 버튼 (기존과 동일, 파라미터 없이 /game 이동) */}
      <PlayButton />
    </div>
  )
}
```

#### 3. CharacterDisplay 동적 캐릭터 표시
**파일**: `apps/web/src/components/lobby/CharacterDisplay.tsx`
**변경**: props로 캐릭터 데이터를 받아 동적으로 이미지 경로 생성

```typescript
interface CharacterDisplayProps {
  character: {
    asset_path: string
    asset_prefix: string
  } | null
}

export function CharacterDisplay({ character }: CharacterDisplayProps) {
  const imagePath = character
    ? `/assets/characters/PNG/${character.asset_path}/Poses/${character.asset_prefix}_idle.png`
    : "/assets/characters/PNG/Player/Poses/player_idle.png"

  return (
    <div className="pointer-events-none absolute inset-0 flex items-center justify-center">
      <div className="flex flex-col items-center">
        <div className="animate-float">
          <Image
            src={imagePath}
            alt="Character"
            width={150}
            height={150}
            className="object-contain drop-shadow-[0_8px_24px_rgba(0,0,0,0.5)]"
            priority
          />
        </div>
        <div className="mt-2 h-4 w-32 rounded-full bg-white/30 blur-md" />
      </div>
    </div>
  )
}
```

#### 4. CharacterSelector 썸네일 패널 (새 컴포넌트)
**파일**: `apps/web/src/components/lobby/CharacterSelector.tsx` (신규)
**설명**: 화면 하단 중앙에 소유한 캐릭터 썸네일을 가로로 나열

```typescript
"use client"

import Image from "next/image"
import { cn } from "@/lib/utils"

interface CharacterSelectorProps {
  characters: Array<{
    character_id: string
    characters: {
      id: string
      name: string
      asset_path: string
      asset_prefix: string
    }
  }>
  selectedId: string
  onSelect: (characterId: string) => void
}

export function CharacterSelector({ characters, selectedId, onSelect }: CharacterSelectorProps) {
  return (
    <div className="absolute bottom-6 left-1/2 -translate-x-1/2 z-10">
      <div className="flex gap-3 rounded-xl bg-black/40 backdrop-blur-sm p-3">
        {characters.map(({ character_id, characters: char }) => {
          const isSelected = character_id === selectedId
          const thumbPath = `/assets/characters/PNG/${char.asset_path}/Poses/${char.asset_prefix}_idle.png`

          return (
            <button
              key={character_id}
              onClick={() => onSelect(character_id)}
              className={cn(
                "relative flex flex-col items-center gap-1 rounded-lg p-2 transition-all",
                "hover:bg-white/20 cursor-pointer",
                isSelected
                  ? "bg-white/25 ring-2 ring-primary"
                  : "bg-white/5"
              )}
            >
              <Image
                src={thumbPath}
                alt={char.name}
                width={48}
                height={48}
                className="object-contain"
              />
              <span className={cn(
                "text-xs",
                isSelected ? "text-white font-bold" : "text-white/70"
              )}>
                {char.name}
              </span>
            </button>
          )
        })}
      </div>
    </div>
  )
}
```

#### 5. PlayButton — 변경 없음
**파일**: `apps/web/src/components/lobby/PlayButton.tsx`
**변경 없음**: 기존대로 `/game`으로 이동. URL 파라미터 사용하지 않음.
선택된 캐릭터는 DB에 이미 저장되어 있으므로, 게임 페이지에서 서버 사이드로 조회한다.

> **보안 참고**: URL 파라미터(`?char=...`)를 사용하면 유저가 소유하지 않은 캐릭터를 URL에 직접 입력하여 악용할 수 있다. 따라서 캐릭터 선택은 반드시 서버 사이드 DB 검증을 거쳐야 한다.

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 오류 없음: `npx tsc --noEmit` (apps/web)
- [x] 린팅 오류 없음

#### 수동 검증:
- [x] 로비에서 중앙 캐릭터가 선택된 캐릭터로 표시됨
- [x] 하단 썸네일 패널에 소유한 캐릭터가 표시됨
- [x] 썸네일 클릭 시 중앙 캐릭터와 선택 상태가 즉시 변경됨
- [x] 페이지 새로고침 후에도 선택한 캐릭터가 유지됨
- [x] PLAY 클릭 시 `/game` 페이지로 이동 (URL 파라미터 없음)

---

## 3단계: 게임 코어 연동 (서버 사이드 검증)

### 개요
게임 페이지에서 서버 사이드로 DB의 `selected_character`를 조회하고 소유권을 검증한 후, 검증된 캐릭터 타입만 Phaser 게임에 전달한다.

> **보안 설계**: URL 파라미터(`?char=...`)를 사용하지 않는다. 유저가 URL을 조작하여 소유하지 않은 캐릭터를 사용하는 악용을 원천 차단하기 위해, 반드시 서버 사이드에서 DB 조회 + 소유권 검증을 수행한다.

### 데이터 흐름

```
[서버 사이드]
game/page.tsx (Server Component)
  → Supabase: player_profiles.selected_character 조회
  → Supabase: character_inventory에서 소유권 검증
  → 미소유 시 fallback: "player"
  → 검증된 characterType을 GameClient에 전달

[클라이언트 사이드]
GameClient.tsx (Client Component)
  → GameWrapper (characterType prop)
    → PhaserGame (characterType prop)
      → createGame({ parent, player1CharacterType })
        → Phaser Registry에 저장
          → GameScene.createPlayers()에서 Registry 읽기
```

### 필요한 변경:

#### 1. Game 페이지를 서버 컴포넌트로 변경
**파일**: `apps/web/src/app/game/page.tsx`
**변경**: `'use client'` 제거, 서버 컴포넌트로 변경하여 DB에서 캐릭터를 조회하고 소유권 검증

```typescript
import { redirect } from 'next/navigation';
import { createClient } from '@/lib/supabase/server';
import GameClient from './GameClient';

export default async function GamePage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (!user) {
    redirect('/login');
  }

  // 1. player_profiles에서 selected_character 조회
  const { data: profile } = await supabase
    .from('player_profiles')
    .select('selected_character')
    .eq('id', user.id)
    .single();

  const selectedCharacter = profile?.selected_character ?? 'player';

  // 2. character_inventory에서 소유권 검증
  const { data: ownership } = await supabase
    .from('character_inventory')
    .select('character_id')
    .eq('user_id', user.id)
    .eq('character_id', selectedCharacter)
    .single();

  // 3. 소유하지 않은 캐릭터면 기본값으로 fallback
  const verifiedCharacter = ownership ? selectedCharacter : 'player';

  return <GameClient characterType={verifiedCharacter} />;
}
```

#### 2. GameClient 클라이언트 컴포넌트 (신규)
**파일**: `apps/web/src/app/game/GameClient.tsx` (신규)
**설명**: 서버 컴포넌트에서 전달받은 검증된 캐릭터 타입으로 GameWrapper를 렌더링. `dynamic()` import는 클라이언트 컴포넌트에서만 가능하므로 이 컴포넌트가 필요.

```typescript
'use client';

import dynamic from 'next/dynamic';

const GameWrapper = dynamic(
  () => import('@/components/game/GameWrapper').then((m) => m.GameWrapper),
  { ssr: false }
);

interface GameClientProps {
  characterType: string;
}

export default function GameClient({ characterType }: GameClientProps) {
  return <GameWrapper characterType={characterType} />;
}
```

#### 3. GameWrapper → PhaserGame → createGame 데이터 전달
**파일**: `apps/web/src/components/game/GameWrapper.tsx`
**변경**: characterType props 추가

```typescript
interface GameWrapperProps {
  characterType: string
}

export function GameWrapper({ characterType }: GameWrapperProps) {
  // ... 기존 로직 동일
  return (
    <div className="relative min-h-screen bg-gray-900">
      <PhaserGame characterType={characterType} />
      {/* ... 기존 Exit 버튼 등 */}
    </div>
  )
}
```

**파일**: `apps/web/src/components/game/PhaserGame.tsx`
**변경**: characterType을 받아 createGame에 전달

```typescript
interface PhaserGameProps {
  onGameReady?: (game: Game) => void
  characterType?: string
}

export function PhaserGame({ onGameReady, characterType = "player" }: PhaserGameProps) {
  // ...
  const initGame = async () => {
    const { createGame } = await import("@repo/game-core")
    if (!containerRef.current || gameRef.current) {
      initializingRef.current = false
      return
    }
    const game = createGame({
      parent: containerRef.current,
      player1CharacterType: characterType,
    })
    gameRef.current = game
    onGameReady?.(game)
  }
  // ...
}
```

#### 4. createGame 함수 확장
**파일**: `packages/game-core/src/index.ts`
**변경**: 캐릭터 타입 옵션을 받아 Registry에 저장

```typescript
export interface GameOptions {
  parent: HTMLElement
  player1CharacterType?: string
}

export function createGame(options: GameOptions): Phaser.Game {
  const config: Phaser.Types.Core.GameConfig = {
    type: Phaser.AUTO,
    parent: options.parent,
    // ... 나머지 설정 동일
  }

  const game = new Phaser.Game(config)

  // Registry에 캐릭터 타입 저장
  game.registry.set("player1CharacterType", options.player1CharacterType ?? "player")

  return game
}
```

#### 5. GameScene에서 Registry 읽기
**파일**: `packages/game-core/src/scenes/GameScene.ts`
**변경**: `createPlayers()` 메서드에서 하드코딩 대신 Registry에서 캐릭터 타입 읽기

```typescript
private createPlayers(): void {
  const [spawn1, spawn2] = this.findSpawnPositions()

  // Registry에서 서버 검증된 캐릭터 타입 읽기
  const player1Type = (this.registry.get("player1CharacterType") as string) || "player"

  // Player 1 — 서버에서 소유권 검증된 유저 선택 캐릭터
  const player1 = new Character({
    scene: this,
    x: spawn1.x,
    y: spawn1.y - 30,
    playerId: 0,
    characterType: player1Type,
    terrainData: this.terrainData,
  })
  this.players.push(player1)

  // Player 2 — 기본 캐릭터
  const player2 = new Character({
    scene: this,
    x: spawn2.x,
    y: spawn2.y - 30,
    playerId: 1,
    characterType: "player",
    terrainData: this.terrainData,
  })
  this.players.push(player2)
}
```

#### 6. BootScene — 변경 없음
**파일**: `packages/game-core/src/scenes/BootScene.ts`
현재 3종(player, adventurer, female)이 모두 로딩되므로, 이 3종 캐릭터 선택은 추가 변경 없이 동작한다.

### 성공 기준:

#### 자동화된 검증:
- [x] TypeScript 컴파일 오류 없음: `npx tsc --noEmit` (packages/game-core)
- [x] TypeScript 컴파일 오류 없음: `npx tsc --noEmit` (apps/web)

#### 수동 검증:
- [x] 로비에서 "Adventurer" 선택 후 PLAY → 게임 내 Player 1이 Adventurer 캐릭터로 표시
- [x] 로비에서 "Female" 선택 후 PLAY → 게임 내 Player 1이 Female 캐릭터로 표시
- [x] Player 2는 항상 기본 캐릭터("player")로 표시
- [x] 게임 내 애니메이션(walk, jump, shoot 등)이 선택된 캐릭터에 맞게 동작
- [x] 게임 종료 후 재시작 시에도 선택된 캐릭터가 유지됨
- [x] URL을 직접 `/game`으로 접근해도 DB에서 검증된 캐릭터로 시작됨
- [x] **보안 검증**: 브라우저 개발자 도구 등으로 캐릭터 타입을 조작할 수 없음 (서버 사이드 렌더링이므로 클라이언트에서 변경 불가)

---

## 테스트 전략

### 수동 테스트 시나리오:
1. **신규 유저 가입 플로우**: Google 로그인 → 로비 → 기본 캐릭터(Player)만 소유 확인 → 선택된 상태로 표시
2. **캐릭터 전환**: 썸네일에서 다른 캐릭터 선택 → 중앙 캐릭터 변경 확인 → DB 반영 확인
3. **게임 플레이**: 캐릭터 선택 → PLAY → 게임 내 해당 캐릭터로 플레이
4. **상태 유지**: 캐릭터 선택 → 페이지 새로고침 → 선택 유지 확인
5. **게임 재시작**: 게임 종료 → 로비 복귀 → 다시 PLAY → 같은 캐릭터

## 마이그레이션 참고 사항

- 기존 유저에게 마이그레이션으로 기본 캐릭터 부여 (1단계 SQL에 포함)
- `handle_new_user()` 트리거를 `CREATE OR REPLACE`로 업데이트하므로 기존 트리거를 자동 교체
- `player_profiles.selected_character`는 `DEFAULT 'player'`로 설정되므로 기존 유저는 NULL → 마이그레이션에서 'player'로 업데이트

## 참조

- 연구 문서: `thoughts/shared/research/2026-02-13-character-skin-nft-system-research.md`
- 게임 아키텍처 연구: `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md`
- 소셜 로그인 구현 계획: `thoughts/shared/plans/2026-02-13-social-login-wallet-linking-implementation.md`
- Supabase 프로젝트 ID: `fmwjmsfgxhbzhvtnaqta`
