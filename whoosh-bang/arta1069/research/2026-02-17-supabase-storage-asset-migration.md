---
date: 2026-02-17T03:03:47+0000
researcher: arta1069@gmail.com
git_commit: 48d17f6
branch: main
repository: whoosh-bang
topic: "Supabase Storage 기반 에셋 관리 시스템 - DB 스키마 재설계 연구"
tags: [research, codebase, supabase-storage, characters, weapons, asset-management, schema-redesign]
status: complete
last_updated: 2026-02-17
last_updated_by: arta1069
last_updated_note: "배경/사운드 포함 + 마이그레이션 스크립트 구조 — 미해결 질문 4/4 모두 해결"
---

# 연구: Supabase Storage 기반 에셋 관리 시스템 - DB 스키마 재설계

**날짜**: 2026-02-17T03:03:47+0000
**연구자**: arta1069@gmail.com
**Git 커밋**: 48d17f6
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

캐릭터 테이블의 `asset_path`, `asset_prefix`와 무기 테이블의 `sprite_key`, `sprite_frame`, `hold_sprite` 컬럼들을 Supabase Storage 기반으로 재설계하여, 에셋 경로가 로컬 파일시스템 대신 Storage URL을 참조하도록 변경하는 방법.

## 요약

현재 캐릭터/무기 에셋은 Next.js `public/assets/` 디렉토리에 정적 파일로 존재하고, DB 컬럼에 로컬 경로 조각(folder명, 파일 prefix 등)을 저장하여 런타임에 경로를 조립한다. Supabase Storage의 **public bucket**으로 이전하면 CDN 캐싱, 동적 에셋 추가(코드 배포 없이 새 캐릭터/무기 추가), 관리 일원화가 가능하다. Free 플랜 기준 1GB Storage / 10GB 대역폭이므로 현재 에셋 총량(~4.1MB)은 여유롭다.

## 상세 발견 사항

### 1. 현재 캐릭터 에셋 시스템

#### DB 스키마 (characters 테이블)
| 컬럼 | 타입 | 예시 | 역할 |
|------|------|------|------|
| `asset_path` | TEXT NOT NULL | "Player" | 폴더명 (대문자 시작) |
| `asset_prefix` | TEXT NOT NULL | "player" | 파일 prefix (소문자) |

#### 에셋 사용 흐름

**로비 (CharacterSelector.tsx:34)**
```
/assets/characters/PNG/${asset_path}/Poses/${asset_prefix}_idle.png
→ /assets/characters/PNG/Player/Poses/player_idle.png
```
- DB의 `asset_path` + `asset_prefix`를 사용하여 썸네일 경로 조립
- Next.js `<Image>` 컴포넌트로 렌더링

**게임 부팅 (BootScene.ts:55-59, 108-115)**
```typescript
// 하드코딩된 CHARACTERS 배열 (DB 미사용!)
const CHARACTERS = [
  { key: "player", path: "Player", prefix: "player" },
  { key: "adventurer", path: "Adventurer", prefix: "adventurer" },
  { key: "female", path: "Female", prefix: "female" },
]
// 로딩:
this.load.image(`${char.key}_${pose}`,
  `/assets/characters/PNG/${char.path}/Poses/${char.prefix}_${pose}.png`)
```
- BootScene은 **DB를 사용하지 않고** 하드코딩된 배열로 모든 캐릭터의 12개 포즈를 로딩
- 포즈 목록: idle, stand, walk1, walk2, jump, fall, action1, action2, hurt, duck, cheer1, cheer2

**게임 런타임 (Character.ts:90-94, 314-323)**
```typescript
// 이미 로딩된 텍스처를 key로 참조
this.bodySprite = this.scene.add.sprite(x, y, `${characterType}_idle`)
// 애니메이션:
const animKey = `${this.characterType}_${state}` // e.g., "player_walk"
```
- `characterType` (= DB id: "player", "female" 등)을 텍스처 키로 사용
- Phaser registry를 통해 전달: `game.registry.set("player1CharacterType", character)`

#### 에셋 파일 구조
```
apps/web/public/assets/characters/PNG/  (총 ~2.3MB)
├── Player/Poses/       (~184KB, 24개 파일)
│   ├── player_idle.png, player_walk1.png, ...
├── Adventurer/Poses/
├── Female/Poses/
├── Soldier/Poses/      (is_active=false)
└── Zombie/Poses/       (is_active=false)
```

### 2. 현재 무기 에셋 시스템

#### DB 스키마 (weapons 테이블)
| 컬럼 | 타입 | 예시 | 역할 |
|------|------|------|------|
| `sprite_key` | TEXT NOT NULL DEFAULT 'tanks' | "tanks" | Phaser atlas 텍스처 키 |
| `sprite_frame` | TEXT | "tank_bullet1.png" | 투사체 스프라이트 (atlas 프레임명) |
| `hold_sprite` | TEXT | "tanks_turret2.png" | 캐릭터가 들고 있는 무기 (atlas 프레임명) |

#### 에셋 사용 흐름

**게임 부팅 (BootScene.ts:117-122)**
```typescript
// 단일 아틀라스로 모든 무기/폭발 에셋 로딩
this.load.atlasXML("tanks",
  "/assets/resource/Spritesheet/tanks_spritesheetDefault.png",
  "/assets/resource/Spritesheet/tanks_spritesheetDefault.xml")
```

**인게임 투사체 (Bazooka.ts:8-14, Projectile.ts:40-45)**
```typescript
super({ spriteKey: "tanks", frame: WeaponRegistry.bazooka.spriteFrame, mass: ... })
// Projectile.ts:
this.sprite = this.scene.matter.add.sprite(x, y, config.spriteKey, config.frame)
```

**캐릭터 무기 표시 (Character.ts:199-213, 266-273)**
```typescript
// 무기 스프라이트 생성
this.weaponSprite = this.scene.add.image(0, 0, "tanks", weaponStats.holdSprite)
// 무기 변경
this.weaponSprite.setTexture("tanks", weaponStats.holdSprite)
```

**인게임 WeaponSelector UI (WeaponSelector.ts:63-69)**
```typescript
const spriteFrame = stats.holdSprite || stats.spriteFrame
const image = this.scene.add.image(centerX, y, stats.spriteKey, spriteFrame)
```

**로비 WeaponSelector (LobbyWeaponSelector.tsx:34)**
```typescript
// 개별 PNG 파일 사용 (아틀라스가 아님)
const thumbPath = `/assets/resource/PNG/Default size/${weapon.hold_sprite ?? weapon.sprite_frame}`
```

#### 에셋 파일 구조
```
apps/web/public/assets/resource/  (총 ~1.8MB)
├── Spritesheet/
│   ├── tanks_spritesheetDefault.png  (아틀라스 이미지)
│   └── tanks_spritesheetDefault.xml  (프레임 좌표 데이터)
└── PNG/Default size/  (~352KB, 88개 파일)
    ├── tank_bullet1.png  (bazooka 투사체)
    ├── tank_bullet2.png  (shotgun 투사체)
    ├── tank_bullet3.png  (grenade 투사체)
    ├── tanks_turret1.png (shotgun 들기)
    ├── tanks_turret2.png (bazooka 들기)
    └── ... (폭발, 화살표 등 공용 에셋)
```

### 3. Supabase Storage 사양 (Free Plan)

| 항목 | 한도 |
|------|------|
| 총 Storage | 1 GB |
| 파일당 최대 크기 | 50 MB |
| 대역폭 (Egress) | 10 GB/월 |
| Public bucket CDN | Cloudflare CDN 캐시 |

전체 에셋 총량:
| 카테고리 | 크기 | 설명 |
|---------|------|------|
| 배경 (8세트) | 7.5MB | 세트별 5~9개 레이어 PNG |
| 사운드 | 6.4MB | fortress-bgm (3MB), lobby-bgm (3.5MB) |
| 캐릭터 | 2.3MB | 캐릭터별 12포즈 × 5캐릭터 |
| 무기/리소스 | 1.8MB | 아틀라스 + 개별 PNG 88개 |
| **합계** | **~18MB** | **1GB 한도의 1.8%** |

Public bucket URL 형식:
```
https://[project_id].supabase.co/storage/v1/object/public/[bucket]/[path]
```

### 4. 핵심 설계 과제

**캐릭터의 이중 참조 문제:**
- BootScene은 DB를 사용하지 않고 하드코딩된 CHARACTERS 배열 사용
- CharacterSelector(로비)는 DB의 `asset_path`/`asset_prefix` 사용
- → Storage 전환 시 BootScene도 DB 데이터 기반으로 전환해야 동적 캐릭터 추가 가능

**무기의 Atlas vs 개별 이미지 문제:**
- 인게임: `tanks` 아틀라스에서 프레임명으로 참조 (spriteKey + frameName)
- 로비: 개별 PNG 파일에서 직접 참조
- → 아틀라스 자체를 Storage로 이전하면 로딩 URL만 변경
- → 로비 썸네일은 별도 경로로 관리

## DB 스키마 재설계안

### Storage Bucket 구조

```
game-assets/  (public bucket)
├── characters/
│   ├── player/
│   │   ├── player_idle.png
│   │   ├── player_stand.png
│   │   ├── player_walk1.png
│   │   ├── player_walk2.png
│   │   ├── player_jump.png
│   │   ├── player_fall.png
│   │   ├── player_action1.png
│   │   ├── player_action2.png
│   │   ├── player_hurt.png
│   │   ├── player_duck.png
│   │   ├── player_cheer1.png
│   │   └── player_cheer2.png
│   ├── adventurer/
│   │   └── ... (동일 패턴)
│   └── female/
│       └── ...
├── weapons/
│   ├── atlas/
│   │   ├── tanks_default.png    (아틀라스 이미지)
│   │   └── tanks_default.xml    (아틀라스 데이터)
│   └── frames/                  (아틀라스 프레임과 동일 이름의 개별 PNG)
│       ├── tank_bullet1.png     (bazooka 투사체 = atlas 프레임명)
│       ├── tank_bullet2.png     (shotgun 투사체)
│       ├── tank_bullet3.png     (grenade 투사체)
│       ├── tanks_turret1.png    (shotgun 들기)
│       └── tanks_turret2.png    (bazooka 들기)
├── backgrounds/
│   ├── 1/
│   │   ├── 1_layer.png
│   │   ├── 2_layer.png ... 6_layer.png
│   │   └── 1_game_background.png  (미사용 원본)
│   ├── 2/ ... 8/                  (세트별 동일 구조)
│   └── config.json                (세트별 레이어 수, 지형 레이어, 다리 설정)
└── sounds/
    ├── fortress-bgm.ogg.mp3      (인게임 BGM, 3MB)
    └── lobby-bgm.ogg             (로비 BGM, 3.5MB)
```

### Characters 테이블 재설계

**현재:**
```sql
asset_path TEXT NOT NULL    -- "Player" (폴더명)
asset_prefix TEXT NOT NULL  -- "player" (파일 prefix)
```

**변경 후:**
```sql
thumbnail_path TEXT NOT NULL  -- Storage 내 썸네일 경로 (로비용)
sprite_path TEXT NOT NULL     -- Storage 내 포즈 폴더 경로
sprite_prefix TEXT NOT NULL   -- 포즈 파일명 prefix
```

**데이터 매핑:**
| id | thumbnail_path | sprite_path | sprite_prefix |
|---|---|---|---|
| player | characters/player/player_idle.png | characters/player | player |
| adventurer | characters/adventurer/adventurer_idle.png | characters/adventurer | adventurer |
| female | characters/female/female_idle.png | characters/female | female |

**경로 조합 패턴:**
- 로비 썸네일: `${STORAGE_URL}/${thumbnail_path}`
- 게임 포즈: `${STORAGE_URL}/${sprite_path}/${sprite_prefix}_${pose}.png`

### Weapons 테이블 재설계

**현재:**
```sql
sprite_key TEXT NOT NULL DEFAULT 'tanks'  -- Phaser atlas 텍스처 키
sprite_frame TEXT                         -- "tank_bullet1.png" (atlas 프레임명)
hold_sprite TEXT                          -- "tanks_turret2.png" (atlas 프레임명)
```

**변경 후:**
```sql
atlas_key TEXT NOT NULL DEFAULT 'tanks'  -- Phaser 런타임 텍스처 키 (변경 없음)
atlas_path TEXT NOT NULL DEFAULT 'weapons/atlas/tanks_default'  -- Storage 내 아틀라스 경로 (.png/.xml)
projectile_frame TEXT          -- 아틀라스 내 투사체 프레임명
hold_frame TEXT                -- 아틀라스 내 들기 프레임명
```

> **설계 결정**: `thumbnail_path` 없음 — 로비와 인게임 모두 동일한 `hold_frame`/`projectile_frame` 값을 사용.
> Storage의 `weapons/frames/` 디렉토리에 아틀라스 프레임과 **동일한 파일명**의 개별 PNG를 배치하여,
> 로비는 `${STORAGE_URL}/weapons/frames/${hold_frame ?? projectile_frame}`으로 이미지를 표시한다.

**데이터 매핑:**
| id | atlas_key | atlas_path | projectile_frame | hold_frame |
|---|---|---|---|---|
| bazooka | tanks | weapons/atlas/tanks_default | tank_bullet1.png | tanks_turret2.png |
| grenade | tanks | weapons/atlas/tanks_default | tank_bullet3.png | tank_bullet3.png |
| shotgun | tanks | weapons/atlas/tanks_default | tank_bullet2.png | tanks_turret1.png |

**경로 조합 패턴:**
- 로비 이미지: `${STORAGE_URL}/weapons/frames/${hold_frame ?? projectile_frame}` (프레임명으로 URL 도출)
- 아틀라스 로딩: `${STORAGE_URL}/${atlas_path}.png` + `${STORAGE_URL}/${atlas_path}.xml`
- 인게임 프레임 참조: `atlas_key` + `projectile_frame` / `hold_frame` (기존과 동일)

### 타입 정의 변경

**CharacterData (packages/game-core/src/types/index.ts):**
```typescript
// 현재
export interface CharacterData {
  id: string
  name: string
  asset_path: string    // 제거
  asset_prefix: string  // 제거
  sort_order: number
}

// 변경 후
export interface CharacterData {
  id: string
  name: string
  thumbnail_path: string   // Storage 내 썸네일 경로
  sprite_path: string      // Storage 내 포즈 폴더 경로
  sprite_prefix: string    // 포즈 파일명 prefix
  sort_order: number
}
```

**WeaponData (packages/game-core/src/types/index.ts):**
```typescript
// 현재
export interface WeaponData {
  id: string
  name: string
  damage: number
  explosion_radius: number
  ammo: number
  mass: number
  sprite_key: string        // 제거
  sprite_frame: string | null  // 제거
  hold_sprite: string | null   // 제거
  sort_order: number
}

// 변경 후
export interface WeaponData {
  id: string
  name: string
  damage: number
  explosion_radius: number
  ammo: number
  mass: number
  atlas_key: string               // Phaser 텍스처 키
  atlas_path: string              // Storage 내 아틀라스 경로
  projectile_frame: string | null // 아틀라스 내 프레임명 (인게임 + 로비 공용)
  hold_frame: string | null       // 아틀라스 내 프레임명 (인게임 + 로비 공용)
  sort_order: number
}
```
> **thumbnail_path 불필요**: 로비는 `hold_frame ?? projectile_frame` 값으로 Storage URL을 도출한다.

### 영향받는 코드 파일

| 파일 | 변경 이유 |
|------|----------|
| `packages/game-core/src/types/index.ts` | CharacterData, WeaponData 인터페이스 변경 |
| `packages/game-core/src/weapons/WeaponTypes.ts` | WeaponStats 매핑 함수, WeaponRegistry 필드명 변경 |
| `packages/game-core/src/scenes/BootScene.ts` | 하드코딩 제거, DB 데이터 기반 동적 로딩 |
| `packages/game-core/src/entities/Character.ts` | `setWeapon()` → hold_frame 참조 |
| `packages/game-core/src/ui/WeaponSelector.ts` | spriteKey → atlas_key, holdSprite → holdFrame |
| `packages/game-core/src/weapons/Bazooka.ts` | spriteFrame → projectileFrame |
| `packages/game-core/src/weapons/Grenade.ts` | 동일 |
| `packages/game-core/src/weapons/Shotgun.ts` | 동일 |
| `packages/game-core/src/index.ts` | GameOptions에 전체 캐릭터 데이터 추가 |
| `apps/web/src/components/lobby/CharacterSelector.tsx` | thumbnail_path 기반 URL 생성 |
| `apps/web/src/components/lobby/LobbyWeaponSelector.tsx` | thumbnail_path 기반 URL 생성 |
| `apps/web/src/app/lobby/page.tsx` | 새 컬럼명으로 select 변경 |
| `apps/web/src/app/game/page.tsx` | 새 컬럼명으로 select 변경 |

### Storage URL 유틸리티

```typescript
// apps/web/src/lib/storage.ts (신규)
const STORAGE_URL = `${process.env.NEXT_PUBLIC_SUPABASE_URL}/storage/v1/object/public/game-assets`

export function getStorageUrl(path: string): string {
  return `${STORAGE_URL}/${path}`
}
```

## 코드 참조

- `packages/game-core/src/types/index.ts:1-22` — CharacterData, WeaponData 타입 정의
- `packages/game-core/src/weapons/WeaponTypes.ts:14-26` — weaponDataToStats 변환 함수
- `packages/game-core/src/scenes/BootScene.ts:55-59` — 하드코딩된 CHARACTERS 배열
- `packages/game-core/src/scenes/BootScene.ts:108-115` — 캐릭터 포즈 로딩 (하드코딩 경로)
- `packages/game-core/src/scenes/BootScene.ts:117-122` — 아틀라스 로딩 (하드코딩 경로)
- `packages/game-core/src/entities/Character.ts:199-213` — weaponSprite 생성 (atlas "tanks" + holdSprite)
- `packages/game-core/src/entities/Character.ts:266-273` — setWeapon (atlas "tanks" + holdSprite)
- `packages/game-core/src/ui/WeaponSelector.ts:63-69` — 인게임 무기 이미지 (spriteKey + frame)
- `packages/game-core/src/weapons/Bazooka.ts:8-14` — 투사체 생성 (spriteKey + spriteFrame)
- `apps/web/src/components/lobby/CharacterSelector.tsx:34` — 로비 캐릭터 썸네일 (asset_path + asset_prefix)
- `apps/web/src/components/lobby/LobbyWeaponSelector.tsx:34` — 로비 무기 썸네일 (hold_sprite/sprite_frame)
- `apps/web/src/app/lobby/page.tsx:31-35` — 캐릭터 데이터 fetch (asset_path, asset_prefix)
- `apps/web/src/app/lobby/page.tsx:48-54` — 무기 데이터 fetch (sprite_key, sprite_frame, hold_sprite)
- `apps/web/src/app/game/page.tsx:57-61` — 게임 진입 무기 데이터 fetch

## 아키텍처 문서화

### 에셋 로딩 파이프라인 (현재)

```
[DB] characters.asset_path/asset_prefix
  ├─ [로비] CharacterSelector → /assets/characters/PNG/{path}/Poses/{prefix}_idle.png
  └─ [게임] BootScene (하드코딩) → /assets/characters/PNG/{path}/Poses/{prefix}_{pose}.png
       → Phaser textures → Character.ts (key: "{type}_{pose}")

[DB] weapons.sprite_key/sprite_frame/hold_sprite
  ├─ [로비] LobbyWeaponSelector → /assets/resource/PNG/Default size/{hold_sprite}
  └─ [게임] BootScene (하드코딩) → atlas "/assets/resource/Spritesheet/tanks_*.png/.xml"
       → Phaser textures → Character.ts/Projectile.ts/WeaponSelector.ts (key: "tanks", frame: "{frame}")
```

### 에셋 로딩 파이프라인 (Storage 전환 후)

```
[DB] characters.thumbnail_path/sprite_path/sprite_prefix
  ├─ [로비] CharacterSelector → ${STORAGE_URL}/${thumbnail_path}
  └─ [게임] BootScene (DB 데이터) → ${STORAGE_URL}/${sprite_path}/${sprite_prefix}_{pose}.png
       → Phaser textures → Character.ts (동일)

[DB] weapons.atlas_key/atlas_path/projectile_frame/hold_frame
  ├─ [로비] LobbyWeaponSelector → ${STORAGE_URL}/weapons/frames/${hold_frame ?? projectile_frame}
  └─ [게임] BootScene (DB 데이터) → ${STORAGE_URL}/${atlas_path}.png/.xml
       → Phaser textures → 기존과 동일 (atlas_key + frame명)
  ※ 로비와 인게임 모두 동일한 hold_frame/projectile_frame 컬럼을 참조 (통일)
```

### 주요 설계 결정사항

1. **Storage 경로는 상대 경로로 저장**: DB에는 bucket 내 상대 경로만 저장하고, 전체 URL은 런타임에 `STORAGE_URL + path`로 조합. 환경(dev/prod) 변경 시 URL prefix만 바꾸면 됨.

2. **아틀라스 구조 유지**: 무기 인게임 렌더링은 기존 atlas 방식 유지. `atlas_key`로 Phaser 텍스처 키, `atlas_path`로 Storage 위치, `projectile_frame`/`hold_frame`으로 프레임명 참조.

3. **무기 로비/인게임 참조 통일**: 무기는 별도 `thumbnail_path` 없이 `hold_frame`/`projectile_frame` 값을 인게임과 로비 모두에서 사용. Storage의 `weapons/frames/` 디렉토리에 아틀라스 프레임과 동일한 이름의 개별 PNG를 배치하여, 로비가 프레임명에서 URL을 도출. 캐릭터는 `thumbnail_path`로 로비 이미지를 별도 지정.

4. **BootScene 동적화**: 하드코딩된 CHARACTERS 배열 제거. GameOptions를 통해 전달된 DB 데이터로 동적 로딩.

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md` — 무기 DB화 계획 (현재 구현 중). 이 연구는 해당 계획의 에셋 컬럼 부분을 Storage 기반으로 확장.
- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` — 캐릭터/무기 시스템 초기 연구

## 미해결 질문

1. ~~**BootScene 동적 로딩 범위**~~ → **해결됨**: 매치에 참여하는 캐릭터만 로딩. `game/page.tsx`에서 유저 캐릭터(검증 완료) + AI 캐릭터(서버에서 미리 결정)를 `GameOptions.characters[]`로 전달하고, BootScene은 해당 배열만 순회하여 Storage에서 로딩. 현재 AI는 `"player"` 고정(GameScene.ts:299)이므로 유저 캐릭터 + `"player"` 2개만 로딩하면 됨.

2. ~~**무기 썸네일 이미지 소스**~~ → **해결됨**: 로비도 인게임과 동일한 프레임명(`hold_frame`/`projectile_frame`)을 기반으로 Storage URL을 도출. `weapons/frames/` 디렉토리에 아틀라스 프레임과 동일 파일명의 개별 PNG를 배치.

3. ~~**배경/사운드 에셋 포함 범위**~~ → **해결됨**: 배경(8세트, 7.5MB) + 사운드(6.4MB) 모두 이번 재설계에 포함. 전체 ~18MB로 1GB 한도의 1.8%.

4. ~~**에셋 업로드 자동화**~~ → **해결됨**: 아래 후속 연구 참조. `scripts/migrate-assets-to-storage.ts` 스크립트로 로컬 에셋을 Supabase Storage에 일괄 업로드.

## 후속 연구 (2026-02-17)

### 무기 Atlas vs 개별 이미지 통일 결정

**결정**: 로비도 인게임 방식으로 통일한다.

**배경**: 기존에는 인게임은 `tanks` 아틀라스에서 프레임명으로 참조하고, 로비는 `PNG/Default size/` 하위의 개별 PNG 파일을 직접 참조하는 이중 구조였다. 이를 통일하여 **DB의 `hold_frame`/`projectile_frame` 컬럼 하나로** 인게임과 로비 모두를 커버한다.

**구현 방식**:
- Storage에 `weapons/frames/` 디렉토리를 만들고, 아틀라스의 각 프레임과 **동일한 파일명**의 개별 PNG를 업로드
- 로비: `${STORAGE_URL}/weapons/frames/${weapon.hold_frame ?? weapon.projectile_frame}`
- 인게임: `atlas_key` + `hold_frame`/`projectile_frame` (기존 Phaser atlas 방식 유지)
- 무기 테이블에 별도 `thumbnail_path` 컬럼이 불필요해짐

**장점**:
- 단일 참조: 프레임명 하나로 로비/인게임 모두 동작
- 컬럼 수 감소: `thumbnail_path` 불필요
- 일관성: 로비에서 보이는 이미지와 인게임 이미지가 항상 동일
- Storage 구조 단순화: `thumbnails/` 디렉토리 불필요

### BootScene 매치 참여 캐릭터만 로딩 결정

**결정**: 전체 캐릭터 로딩 대신, 매치에 참여하는 캐릭터만 로딩한다.

**배경**: 현재 BootScene은 하드코딩된 3개 캐릭터의 모든 포즈(36개 이미지)를 미리 로딩. Storage 전환 후에는 네트워크 요청이 되므로, 불필요한 로딩을 줄이는 것이 중요하다.

**현재 AI 캐릭터 상태**: GameScene.ts:299에서 AI는 `characterType: "player"` 하드코딩. 따라서 현재는 유저 캐릭터 + "player" 최대 2개만 필요.

**구현 방식**:
- `game/page.tsx`(서버): 유저 캐릭터 + AI 캐릭터의 DB 데이터를 `GameOptions.characters[]`에 포함
- `BootScene`: `GameOptions.characters[]` 배열을 순회하며 각 캐릭터의 `sprite_path`/`sprite_prefix`로 Storage에서 12포즈 로딩
- 하드코딩된 `CHARACTERS` 배열 완전 제거

**효과**:
| 시나리오 | 전체 로딩 | 매치 참여만 |
|---------|----------|-----------|
| 캐릭터 3개 (현재) | 36개 이미지 | 24개 이미지 (유저+AI) |
| 캐릭터 10개 (확장) | 120개 이미지 | 24개 이미지 |
| 캐릭터 20개 (장기) | 240개 이미지 | 24개 이미지 |

**확장성**: 멀티플레이어 추가 시에도 참여 플레이어 수 × 12포즈만 로딩.

### 배경/사운드 에셋 Storage 포함 결정

**결정**: 배경과 사운드 모두 이번 재설계에 포함한다.

**현재 배경 시스템** (BootScene.ts:4-36, 81-104):
- 8개 세트, 세트별 5~9개 레이어 PNG (총 7.5MB)
- `BACKGROUND_LAYER_COUNTS`, `TERRAIN_LAYERS`, `TERRAIN_BRIDGE_CONFIG` 모두 하드코딩
- 매 게임 시작 시 랜덤 세트 선택 → 해당 세트 레이어만 로딩
- 경로: `/assets/background/${setId}/${i}_layer.png`

**현재 사운드 시스템**:
- 인게임 BGM: `fortress-bgm.ogg.mp3` (3MB) — BootScene.ts:125에서 로딩
- 로비 BGM: `lobby-bgm.ogg` (3.5MB) — ProfileCard.tsx:105에서 HTML `<audio>` 태그로 직접 참조

**Storage 전환 시 변경 사항**:

배경:
- Storage 경로: `backgrounds/{setId}/{i}_layer.png`
- 배경 설정(레이어 수, 지형 레이어, 다리 설정)을 `backgrounds/config.json`으로 Storage에 관리
- BootScene: 하드코딩 상수 제거 → config.json fetch 후 해당 세트 레이어 로딩
- 또는 DB `background_sets` 테이블로 관리 (향후 배경 추가/비활성화 가능)

사운드:
- Storage 경로: `sounds/fortress-bgm.ogg.mp3`, `sounds/lobby-bgm.ogg`
- BootScene: `this.load.audio("bgm", "${STORAGE_URL}/sounds/fortress-bgm.ogg.mp3")`
- ProfileCard: `<audio src="${STORAGE_URL}/sounds/lobby-bgm.ogg">`

### 에셋 업로드 마이그레이션 스크립트 구조

**결정**: `scripts/migrate-assets-to-storage.ts` 스크립트로 자동화.

**스크립트 역할**: 로컬 `public/assets/` 디렉토리의 파일을 Supabase Storage `game-assets` bucket에 업로드.

**실행 흐름**:
```
1. game-assets public bucket 생성 (없으면)
2. 카테고리별 업로드:
   ├─ characters/  → characters/{id}/{prefix}_{pose}.png
   ├─ weapons/atlas/ → weapons/atlas/tanks_default.png/.xml
   ├─ weapons/frames/ → weapons/frames/{frame}.png (PNG/Default size/에서)
   ├─ backgrounds/ → backgrounds/{setId}/{i}_layer.png
   └─ sounds/ → sounds/{filename}
3. 업로드 결과 리포트 (성공/실패/스킵)
```

**의사 코드**:
```typescript
// scripts/migrate-assets-to-storage.ts
import { createClient } from "@supabase/supabase-js"
import * as fs from "fs"
import * as path from "path"

const supabase = createClient(SUPABASE_URL, SERVICE_ROLE_KEY)
const BUCKET = "game-assets"
const ASSETS_DIR = "apps/web/public/assets"

async function main() {
  // 1. bucket 생성
  await supabase.storage.createBucket(BUCKET, { public: true })

  // 2. 캐릭터 업로드
  for (const charDir of ["Player", "Adventurer", "Female", ...]) {
    const prefix = charDir.toLowerCase()
    for (const pose of POSES) {
      const localPath = `${ASSETS_DIR}/characters/PNG/${charDir}/Poses/${prefix}_${pose}.png`
      const storagePath = `characters/${prefix}/${prefix}_${pose}.png`
      await uploadFile(localPath, storagePath)
    }
  }

  // 3. 무기 아틀라스 업로드
  await uploadFile(
    `${ASSETS_DIR}/resource/Spritesheet/tanks_spritesheetDefault.png`,
    "weapons/atlas/tanks_default.png"
  )
  await uploadFile(
    `${ASSETS_DIR}/resource/Spritesheet/tanks_spritesheetDefault.xml`,
    "weapons/atlas/tanks_default.xml"
  )

  // 4. 무기 프레임 개별 PNG 업로드
  for (const file of fs.readdirSync(`${ASSETS_DIR}/resource/PNG/Default size`)) {
    await uploadFile(
      `${ASSETS_DIR}/resource/PNG/Default size/${file}`,
      `weapons/frames/${file}`
    )
  }

  // 5. 배경 업로드
  for (let setId = 1; setId <= 8; setId++) {
    for (const file of fs.readdirSync(`${ASSETS_DIR}/background/${setId}`)) {
      await uploadFile(
        `${ASSETS_DIR}/background/${setId}/${file}`,
        `backgrounds/${setId}/${file}`
      )
    }
  }

  // 6. 사운드 업로드
  for (const file of fs.readdirSync(`${ASSETS_DIR}/sound`)) {
    await uploadFile(`${ASSETS_DIR}/sound/${file}`, `sounds/${file}`)
  }
}

async function uploadFile(localPath: string, storagePath: string) {
  const fileBuffer = fs.readFileSync(localPath)
  const contentType = getContentType(localPath)
  const { error } = await supabase.storage
    .from(BUCKET)
    .upload(storagePath, fileBuffer, { contentType, upsert: true })

  if (error) console.error(`FAIL: ${storagePath}`, error.message)
  else console.log(`OK: ${storagePath}`)
}
```

**실행 방법**: `npx tsx scripts/migrate-assets-to-storage.ts`
**필요 환경변수**: `NEXT_PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY`
**멱등성**: `upsert: true`로 재실행 시 기존 파일 덮어쓰기 (안전하게 반복 가능)
