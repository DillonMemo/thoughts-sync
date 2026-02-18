---
date: "2026-02-19T01:34:43+0900"
researcher: arta1069@gmail.com
git_commit: 3b52a9579fdf426e03e58616bbff9d8e32fc23d4
branch: main
repository: whoosh-bang
topic: "Store Drawer UI 및 아이템 내보내기(export) 시스템 구현을 위한 코드베이스 연구"
tags: [research, codebase, store, drawer, fortem-sdk, item-export, inventory, wallet]
status: complete
last_updated: "2026-02-19"
last_updated_by: arta1069
---

# 연구: Store Drawer UI 및 아이템 내보내기(export) 시스템

**날짜**: 2026-02-19T01:34:43+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: 3b52a9579fdf426e03e58616bbff9d8e32fc23d4
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

Store 버튼을 통한 Drawer UI 구현과 Fortem SDK를 활용한 아이템 내보내기(export) 시스템 구현에 필요한 현재 코드베이스 상태 파악:
1. StoreButton 및 Drawer 컴포넌트의 현재 구현 상태
2. 인벤토리 시스템(캐릭터/무기 소유권) 작동 방식
3. Fortem SDK 설치 상태 및 사용 가능한 API
4. vaul 라이브러리의 2중 Drawer(nested) 구현 방법
5. 기본 캐릭터/무기 식별 방식 (export 제외 대상)

## 요약

현재 `StoreButton` 컴포넌트는 지갑이 연결된 경우 Drawer를 열지만 내부 콘텐츠는 "Coming soon..." 플레이스홀더만 있다. 인벤토리 시스템은 `character_inventory`와 `weapon_inventory` 테이블로 관리되며, 기본 캐릭터(`"player"`)와 기본 무기(`"bazooka"`)는 하드코딩된 문자열로 식별된다. Fortem SDK(`@fortemlabs/sdk-js@0.0.2`)는 패키지에 설치되어 있고 환경변수도 설정되어 있으나, 이전 테스트 코드가 롤백되어 현재 소스에서는 사용되지 않는다. vaul 라이브러리는 `Drawer.NestedRoot`를 통해 2중 Drawer를 공식 지원한다.

## 상세 발견 사항

### 1. StoreButton 현재 구현

**파일**: `apps/web/src/components/lobby/StoreButton.tsx`

현재 `StoreButton`은 다음과 같이 동작한다:
- `linkedWallet` prop이 있으면 바로 `showStoreDrawer`를 `true`로 설정 (line 44-45)
- `linkedWallet`이 없으면 `WalletSelectDialog`를 표시하고, 지갑 연결 성공 시 `onWalletLinked` 콜백 호출 후 Drawer를 열음 (line 37-40)
- Drawer 내부는 "Coming soon..." 텍스트만 있음 (line 84)

**Props 인터페이스** (line 16-21):
```ts
interface StoreButtonProps {
  linkedWallet: { wallet_address: string; chain: string } | null
  onWalletLinked: (wallet: { wallet_address: string; chain: string } | null) => void
}
```

`useWalletLink` 훅을 사용하여 지갑 연결 플로우를 처리한다 (line 29-41).

### 2. Drawer 컴포넌트 (vaul 기반 shadcn/ui)

**파일**: `apps/web/src/components/ui/shadcn/drawer.tsx`

vaul 라이브러리(`DrawerPrimitive`)를 래핑한 shadcn/ui Drawer 컴포넌트. 현재 export되는 컴포넌트:
- `Drawer`, `DrawerTrigger`, `DrawerPortal`, `DrawerClose`, `DrawerOverlay`, `DrawerContent`, `DrawerHeader`, `DrawerFooter`, `DrawerTitle`, `DrawerDescription`

**2중 Drawer (Nested Drawer) 구현 방법:**
- vaul은 `Drawer.NestedRoot` 컴포넌트를 통해 nested drawer를 공식 지원
- 현재 shadcn/ui 래퍼에는 `NestedRoot`가 export되지 않음
- 구현 시 vaul의 `DrawerPrimitive.NestedRoot`를 직접 import하여 사용하거나, shadcn 래퍼에 `DrawerNestedRoot`를 추가해야 함
- 패턴:
  ```tsx
  <Drawer>  {/* 부모 (Store 메뉴) */}
    <DrawerContent>
      <DrawerPrimitive.NestedRoot>  {/* 자식 (아이템 내보내기) */}
        <DrawerContent>...</DrawerContent>
      </DrawerPrimitive.NestedRoot>
    </DrawerContent>
  </Drawer>
  ```

### 3. 인벤토리 시스템

#### 소유권 테이블 구조

**`character_inventory`**:
| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | UUID PK | `gen_random_uuid()` |
| `user_id` | UUID | `auth.users(id)` FK |
| `character_id` | TEXT | `characters(id)` FK |
| `acquired_at` | TIMESTAMPTZ | 획득 시각 |
| `source` | TEXT | `"default"` / `"fortem"` / `"reward"` |

**`weapon_inventory`**:
| 컬럼 | 타입 | 설명 |
|------|------|------|
| `id` | UUID PK | `gen_random_uuid()` |
| `user_id` | UUID | `auth.users(id)` FK |
| `weapon_id` | TEXT | `weapons(id)` FK |
| `acquired_at` | TIMESTAMPTZ | 획득 시각 |
| `source` | TEXT | `"default"` / `"fortem"` / `"reward"` |

#### 소유권 데이터 흐름

1. **로비 서버 컴포넌트** (`apps/web/src/app/lobby/page.tsx:38-64`):
   - `character_inventory`에서 `user_id`로 조회 → `ownedCharacterIds` 배열 생성
   - `weapon_inventory`에서 `user_id`로 조회 → `ownedWeaponIds` 배열 생성
2. **LobbyContent** (`apps/web/src/components/lobby/LobbyContent.tsx:60-61`):
   - `ownedCharacterSet = new Set(ownedCharacterIds)`
   - `ownedWeaponSet = new Set(ownedWeaponIds)`
3. **선택/해제**: `player_profiles.selected_character`와 `selected_weapons[]` UPDATE

#### 인벤토리 조작 API

현재 `character_inventory`나 `weapon_inventory`를 직접 조작하는 API 라우트는 존재하지 않는다. 신규 유저 가입 시 `handle_new_user` DB 트리거가 기본 아이템을 자동 부여한다.

### 4. 기본 캐릭터/무기 식별 (export 제외 대상)

**기본 캐릭터**: `"player"` (하드코딩 문자열)
- `game/page.tsx:25,38`, `lobby/page.tsx:74`, `GameScene.ts:279,298` 등 여러 곳에서 fallback 값으로 사용

**기본 무기**: `"bazooka"` (하드코딩 문자열)
- `game/page.tsx:41,54`, `lobby/page.tsx:77`, `index.ts:10` 등에서 fallback 값으로 사용

기본 아이템은 DB에 `source: "default"` 로 `handle_new_user` 트리거에 의해 부여된다.

### 5. CharacterData 및 WeaponData 타입

**파일**: `packages/game-core/src/types/index.ts`

**CharacterData** (line 3-10):
```ts
export interface CharacterData {
  id: string              // "player", "female", "adventurer"
  name: string
  thumbnail_path: string  // Storage 내 썸네일 경로
  sprite_path: string
  sprite_prefix: string
  sort_order: number
}
```

**WeaponData** (line 12-24):
```ts
export interface WeaponData {
  id: string              // "bazooka", "grenade", "shotgun"
  name: string
  damage: number
  explosion_radius: number
  ammo: number            // -1 = 무제한
  mass: number
  atlas_key: string
  atlas_path: string
  projectile_frame: string | null
  hold_frame: string | null
  sort_order: number
}
```

### 6. Fortem SDK 상태

**패키지**: `@fortemlabs/sdk-js@0.0.2` (`apps/web/package.json:13`)
**환경변수**: `NEXT_PUBLIC_FORTEM_API_KEY=developer_m4c1aj741e_b4b8446a0bff521` (`apps/web/.env.local:12`)
**현재 소스 사용**: 없음 (롤백됨, 커밋 `1e9ad41`)

**SDK 주요 API:**

| 클래스/메서드 | 설명 |
|---|---|
| `createFortemClient({ apiKey, network })` | FortemClient 인스턴스 생성 |
| `client.items.create(collectionId, params)` | 아이템 생성 (NFT 민팅) |
| `client.items.get(collectionId, code)` | 아이템 조회 |
| `client.items.uploadImage(collectionId, file)` | 이미지 업로드 |
| `client.auth.getToken()` | 인증 토큰 발급 |
| `client.users.verify(walletAddress)` | 유저 지갑 검증 |

**`CreateItemParams` 타입:**
```ts
interface CreateItemParams {
  name: string
  quantity: number
  redeemCode: string
  description: string
  recipientAddress: string
  itemImage?: string
  attributes?: ItemAttribute[]  // { name: string; value: string }[]
  redeemUrl?: string
}
```

**`Item` 응답 타입:**
```ts
interface Item {
  id: number
  objectId: string
  name: string
  description: string
  nftNumber: number
  itemImage: string
  quantity: number
  attributes: ItemAttribute[]
  owner: ItemOwner  // { nickname, walletAddress }
  status: "PROCESSING" | "REDEEMED"
  // ...기타 필드
}
```

**네트워크 설정 (testnet):**
- API Base URL: `https://testnet-api.fortem.gg`
- Service URL: `https://testnet.fortem.gg`

### 7. useWalletLink 훅

**파일**: `apps/web/src/lib/hooks/use-wallet-link.ts`

현재 hooks 디렉토리의 유일한 파일. Sui 지갑 연결 → nonce 서명 → 서버 검증 플로우를 캡슐화한다.

**반환값:**
- `wallets`: 설치된 지갑 목록
- `isConnecting`: 연결 중 상태
- `connectingWalletName`: 연결 중인 지갑 이름
- `showDialog` / `setShowDialog`: 다이얼로그 상태
- `selectWallet`: 지갑 선택 실행 함수
- `openDialog`: 다이얼로그 열기

### 8. LobbyContent에서 StoreButton으로의 데이터 흐름

**파일**: `apps/web/src/components/lobby/LobbyContent.tsx:160-163`

```tsx
<StoreButton
  linkedWallet={linkedWallet}
  onWalletLinked={handleWalletChange}
/>
```

현재 `StoreButton`은 `linkedWallet`과 `onWalletLinked`만 받는다. 아이템 내보내기 기능 구현 시 다음 데이터가 추가로 필요할 수 있다:
- `allCharacters` / `ownedCharacterIds` (보유 캐릭터 목록)
- `allWeapons` / `ownedWeaponIds` (보유 무기 목록)

이 데이터는 이미 `LobbyContent`의 props로 존재하므로 `StoreButton`에 전달하기만 하면 된다.

## 코드 참조

- `apps/web/src/components/lobby/StoreButton.tsx` - Store 버튼 및 Drawer UI
- `apps/web/src/components/ui/shadcn/drawer.tsx` - vaul 기반 Drawer 컴포넌트
- `apps/web/src/components/lobby/LobbyContent.tsx:160-163` - StoreButton 사용 위치
- `apps/web/src/components/lobby/WalletSelectDialog.tsx` - 지갑 선택 다이얼로그
- `apps/web/src/lib/hooks/use-wallet-link.ts` - 지갑 연결 훅
- `apps/web/src/app/lobby/page.tsx:38-64` - 인벤토리 데이터 조회
- `packages/game-core/src/types/index.ts:3-24` - CharacterData, WeaponData 타입
- `apps/web/package.json:13` - Fortem SDK 패키지 선언
- `apps/web/.env.local:12` - Fortem API 키 환경변수

## 아키텍처 문서화

### 현재 Store 흐름
```
StoreButton 클릭
  ├── linkedWallet 있음 → showStoreDrawer(true) → Drawer("Coming soon...")
  └── linkedWallet 없음 → WalletSelectDialog → 지갑 연결 → onWalletLinked + showStoreDrawer(true)
```

### 인벤토리 소유권 흐름
```
auth.users INSERT → handle_new_user 트리거
  → player_profiles 생성 (selected_character='player', selected_weapons='{bazooka}')
  → character_inventory INSERT ('player', 'default')
  → weapon_inventory INSERT ('bazooka', 'default')
```

### Fortem SDK 사용 패턴 (롤백된 코드에서 확인)
```ts
import { createFortemClient } from "@fortemlabs/sdk-js"

const fortem = createFortemClient({
  apiKey: process.env.NEXT_PUBLIC_FORTEM_API_KEY!,
  network: "testnet",
})

// item.create 호출
await fortem.items.create(collectionId, {
  name: "아이템 이름",
  quantity: 1,
  redeemCode: "고유 코드",
  description: "설명",
  recipientAddress: "지갑 주소",
  attributes: [{ name: "key", value: "value" }],
})
```

### Nested Drawer 패턴 (vaul)
```tsx
import { Drawer as DrawerPrimitive } from "vaul"

// 부모 Drawer 안에서:
<DrawerPrimitive.NestedRoot open={childOpen} onOpenChange={setChildOpen}>
  <DrawerPortal>
    <DrawerOverlay />
    <DrawerContent>
      {/* 자식 Drawer 내용 */}
    </DrawerContent>
  </DrawerPortal>
</DrawerPrimitive.NestedRoot>
```

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/plans/2026-02-18-fortem-sdk-web-phase1.md` - Fortem SDK 1단계 구현 계획
- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md` - SDK 리포 구조 연구
- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md` - 무기 아이템화 시스템 계획 (Fortem 연동 포함)
- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` - 아이템화 연구
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md` - 캐릭터 스킨 시스템 (`source` 컬럼에 `"fortem"` 값)

## 관련 연구

- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md`
- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md`
- `thoughts/arta1069/research/2026-02-13-character-skin-nft-system-research.md`

## 미해결 질문

1. `item.create`의 `redeemCode` 필드에 어떤 값을 매핑해야 하는지 (아이템 id? UUID?)
2. `item.create` 완료 후 인벤토리 제거는 클라이언트에서 직접 Supabase DELETE를 할지, 별도 API 라우트를 만들지
3. `CreateItemParams.recipientAddress`에 현재 연결된 지갑 주소를 넣을지, 별도 수령 주소를 받을지
4. export 후 `selected_character`나 `selected_weapons`가 해당 아이템을 참조하고 있으면 어떻게 처리할지 (fallback으로 기본 아이템 재설정?)
