---
date: "2026-02-19T15:28:16+0900"
researcher: arta1069@gmail.com
git_commit: ea6a086
branch: main
repository: whoosh-bang
topic: "아이템 Import(Redeem Code) 시스템 구현을 위한 코드베이스 연구"
tags: [research, codebase, import, redeem-code, fortem-sdk, inventory, ownership-transfer]
status: complete
last_updated: "2026-02-19"
last_updated_by: arta1069
---

# 연구: 아이템 Import(Redeem Code) 시스템

**날짜**: 2026-02-19T15:28:16+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: ea6a086
**브랜치**: main
**리포지토리**: whoosh-bang

## 연구 질문

ForTem 마켓플레이스를 통해 다른 유저가 구매한 아이템을 Redeem Code로 Import하는 시스템 구현에 필요한 현재 코드베이스 상태 파악:
1. Import 흐름의 전체 데이터 경로 (UI → API → DB)
2. `fortem.items.get(collectionId, code)` API 시그니처 및 응답 구조
3. 소유권 이전(user_id 변경) 시 DB 처리 방식
4. Input OTP 컴포넌트 사용 가능 여부
5. auth.users 삭제 시 Export된 아이템(redeem_code IS NOT NULL)의 cascade 동작 및 보존 방안

## 요약

현재 Export 시스템은 `redeem_code` 컬럼 UPDATE 방식으로 구현되어 있으며, Import 시스템은 미구현 상태이다. Import 구현의 핵심은 `fortem.items.get(collectionId, code)` API를 통해 아이템 존재 여부와 `status === "REDEEMED"` 확인 후, 인벤토리 테이블에 새 레코드를 INSERT(소유자가 다를 수 있으므로)하는 것이다. 현재 인벤토리 테이블은 `user_id REFERENCES auth.users(id) ON DELETE CASCADE`로 설정되어 있어, Export된 아이템(redeem_code IS NOT NULL)도 원래 유저가 삭제되면 함께 삭제된다. Input OTP 패키지는 설치되어 있지 않으며 별도 추가가 필요하다.

## 상세 발견 사항

### 1. 현재 Import 버튼 상태

**파일**: `apps/web/src/components/lobby/StoreButton.tsx:356-361`

현재 Import 버튼은 플레이스홀더만 존재한다:
```tsx
<Button
  onClick={() => console.info("handle import")}
  className="flex-1"
>
  Import
</Button>
```

Export 버튼과 나란히 배치되어 있으며, `DrawerFooter` 내 `flex-nowrap flex-row` 레이아웃으로 Export/Import 버튼이 좌우로 나열된다 (`StoreButton.tsx:345`).

### 2. ForTem `items.get` API 상세

**SDK 타입 정의** (`@fortemlabs/sdk-js@0.0.2 dist/index.d.ts:193-195`):
```typescript
get(collectionId: number, code: string): Promise<FortemResponse<Item>>;
```

**SDK 구현체** (`dist/index.mjs:226-232`):
```javascript
async get(collectionId, code) {
  const response = await this._fetch(
    `${this._apiBaseUrl}/api/v1/developers/collections/${collectionId}/items/${code}`,
    { method: "GET" }
  );
  return parseResponse(response);
}
```

**`Item` 응답 타입** (`index.d.ts:71-84`):
```typescript
interface Item {
  id: number;
  objectId: string;
  name: string;
  description: string;
  nftNumber: number;
  itemImage: string;
  quantity: number;
  attributes: ItemAttribute[];  // [{ name: "type", value: "character"|"weapon" }, { name: "Character"|"Weapon", value: itemId }]
  owner: ItemOwner;             // { nickname, walletAddress }
  status: ItemStatus;           // "PROCESSING" | "REDEEMED"
  createdAt: string;
  updatedAt: string;
}
```

**`FortemResponse<T>` 래퍼** (`index.d.ts:13-16`):
```typescript
interface FortemResponse<T> {
  statusCode: number;
  data: T;
}
```

**`items.get`는 인증 없이 호출 가능**: SDK 구현을 보면 `this._fetch`(일반 fetch)를 사용하며, `this._authenticatedFetch`가 아님. 따라서 `getFortemClient()` (클라이언트 사이드, 공개 API 키)로도 호출 가능하지만, Import의 원자적 처리(DB UPDATE 포함)를 위해 **서버 사이드 API Route**에서 `getFortemServerClient()`로 호출하는 것이 적절하다.

### 3. Export 시 attributes에 저장되는 아이템 메타데이터

**파일**: `apps/web/src/app/api/store/export/route.ts:104-107`

Export 시 ForTem 아이템의 `attributes`에 다음 정보가 저장된다:
```typescript
attributes: [
  { name: "type", value: itemType },       // "character" 또는 "weapon"
  { name: itemLabel, value: itemId },       // "Character": "female" 또는 "Weapon": "shotgun"
]
```

Import 시 이 `attributes`를 파싱하여:
- `type` → 어떤 인벤토리 테이블에 INSERT할지 결정 (`character_inventory` vs `weapon_inventory`)
- `Character`/`Weapon` → `character_id` 또는 `weapon_id` 값

### 4. 인벤토리 테이블 스키마 (Import 관련)

두 테이블 모두 동일한 구조:

| 컬럼 | 타입 | Import 시 역할 |
|------|------|----------------|
| `id` | UUID PK | 자동 생성 |
| `user_id` | UUID FK → auth.users(id) ON DELETE CASCADE | Import하는 유저의 ID |
| `character_id`/`weapon_id` | TEXT FK | ForTem attributes에서 추출 |
| `acquired_at` | TIMESTAMPTZ DEFAULT now() | Import 시점 자동 기록 |
| `source` | TEXT NOT NULL DEFAULT 'default' | `'fortem'`으로 설정 |
| `redeem_code` | TEXT DEFAULT NULL | NULL (Import 완료된 인게임 아이템) |
| `network` | TEXT | ForTem 네트워크 값 |

**UNIQUE 제약 없음**: `remove_inventory_unique_constraints` 마이그레이션으로 제거됨. 동일 유저가 같은 아이템을 여러 개 보유 가능.

**부분 인덱스 존재**:
- `idx_character_inventory_active ON (user_id) WHERE redeem_code IS NULL`
- `idx_character_inventory_active_count ON (user_id, character_id) WHERE redeem_code IS NULL`
- 무기도 동일 패턴

### 5. 소유권 이전 처리 방식

Export 흐름에서 인벤토리 레코드는 DELETE되지 않고 `redeem_code` UPDATE로 상태만 변경된다 (`export/route.ts:118-125`):

```typescript
await admin
  .from(table)
  .update({
    redeem_code: redeemCode,
    network: FORTEM_NETWORK,
    source: "fortem",
  })
  .eq("id", inventoryItem.id)
```

**Import 시 소유권 이전 핵심 사항**:
- Export한 유저(A)의 인벤토리 레코드: `redeem_code`가 채워진 채로 유지됨 (이력 보존)
- Import하는 유저(B): **새 레코드를 INSERT**해야 함 (A의 레코드를 UPDATE하면 안 됨)
- 이유: A와 B는 다른 `user_id`를 가지며, 기존 레코드의 `user_id` FK를 변경하면 A의 이력이 사라짐
- Import된 레코드의 `source`는 `'fortem'`, `redeem_code`는 `NULL` (인게임 활성 상태)

### 6. auth.users 삭제 시 Export된 아이템의 cascade 동작

**현재 구조**:
```
auth.users DELETE → CASCADE →
  ├── player_profiles 삭제
  ├── wallets 삭제
  ├── character_inventory 삭제 (redeem_code 유무 무관, 모두 삭제)
  └── weapon_inventory 삭제 (redeem_code 유무 무관, 모두 삭제)
```

**문제점**: 유저 A가 아이템을 Export한 후 탈퇴하면, A의 인벤토리 레코드(redeem_code가 있는 것 포함)가 모두 CASCADE 삭제됨. 그러나 이것이 **Import에 직접적 영향을 주지는 않음**:
- Import는 기존 레코드를 참조하지 않고 **새 레코드를 INSERT**하는 방식
- Import 검증은 ForTem API(`items.get`)의 `status === "REDEEMED"` 응답에 의존
- 동일 `redeem_code`로 중복 Import 방지는 인벤토리 테이블이 아닌 **ForTem 서버 측에서 관리**하거나, Import API에서 별도 검증 필요

**고려 사항**: 동일 `redeem_code`로 중복 Import를 방지하려면:
- 방안 1: Import 시 해당 `redeem_code`가 이미 사용되었는지 인벤토리 테이블에서 확인 (원래 Export한 유저의 레코드가 삭제되었을 수 있으므로 불완전)
- 방안 2: 별도 `used_redeem_codes` 테이블 관리
- 방안 3: ForTem API가 Redeem 상태를 관리하고, Import 후 status가 변경되어 재사용 불가하도록 처리 (SDK의 `status: "REDEEMED"` 활용)

### 7. Input OTP 컴포넌트 현황

**설치 상태**: `input-otp` 패키지 미설치. `apps/web/package.json`에 없음.

**shadcn/ui 컴포넌트 디렉토리** (`apps/web/src/components/ui/shadcn/`):
- `button.tsx` - 존재
- `dialog.tsx` - 존재
- `drawer.tsx` - 존재
- `input-otp.tsx` - **미존재**

**구현 시 필요 작업**:
1. `input-otp` 패키지 설치: `pnpm add input-otp` (apps/web에서)
2. shadcn input-otp 컴포넌트 추가: `apps/web/src/components/ui/shadcn/input-otp.tsx` 생성
3. Redeem Code 형태: `XXXX-XXXX-XXXX` (12자리, 영대문자+숫자, 하이픈 구분)

### 8. Dialog 컴포넌트 현황

**파일**: `apps/web/src/components/ui/shadcn/dialog.tsx`

radix-ui 기반 Dialog 컴포넌트가 존재한다. export되는 컴포넌트:
- `Dialog`, `DialogTrigger`, `DialogPortal`, `DialogClose`
- `DialogOverlay`, `DialogContent` (showCloseButton prop 지원)
- `DialogHeader`, `DialogFooter` (showCloseButton prop 지원)
- `DialogTitle`, `DialogDescription`

`DialogContent`는 `showCloseButton` prop으로 닫기 버튼 표시/숨김 제어 가능 (`dialog.tsx:53-56`).

### 9. LobbyContent → StoreButton 데이터 흐름 (Import 관련)

**현재 Export 흐름**:
```
StoreButton.handleExport()
  → POST /api/store/export
  → 응답: { selectedCharacter, selectedWeapons }
  → onExportComplete({ selectedCharacter, selectedWeapons })
  → LobbyContent.handleExportComplete()
      → setSelectedCharacter()
      → setSelectedWeapons()
      → router.refresh()
```

**Import 후 필요한 데이터 업데이트**:
- Export와 달리 Import는 `ownedCharacterIds`, `ownedWeaponIds`, `characterInventoryRecords`, `weaponInventoryRecords`가 변경됨
- 이 값들은 LobbyContent에서 로컬 상태가 아닌 **props로만** 존재 (`LobbyContent.tsx:31-39`)
- 따라서 `router.refresh()`로 서버 컴포넌트 재실행이 **반드시 필요**
- 기존 `onExportComplete` 콜백을 재사용하거나 별도 `onImportComplete` 콜백 추가 가능

**LobbyContent의 handleExportComplete** (`LobbyContent.tsx:68-75`):
```typescript
const handleExportComplete = useCallback(
  (updates: { selectedCharacter: string; selectedWeapons: string[] }) => {
    setSelectedCharacter(updates.selectedCharacter)
    setSelectedWeapons(updates.selectedWeapons)
    router.refresh()
  },
  [router]
)
```

Import의 경우 `selectedCharacter`/`selectedWeapons` 변경이 필요 없으므로(새 아이템 추가일 뿐), `router.refresh()`만 호출하면 충분하다.

### 10. ForTem 클라이언트/서버 구분 패턴

**파일**: `apps/web/src/lib/fortem.ts`

| 함수 | API 키 | 사용 위치 | 메서드 |
|---|---|---|---|
| `getFortemClient()` | `NEXT_PUBLIC_FORTEM_API_KEY` (공개) | 클라이언트 컴포넌트 | `users.verify()` |
| `getFortemServerClient()` | `FORTEM_API_KEY` (비공개) | API Route (서버) | `items.create()` |

**Import에서의 선택**: `items.get`은 인증 없이도 호출 가능하지만, Import API Route에서 DB 조작과 함께 원자적으로 처리해야 하므로 `getFortemServerClient()` 사용이 적절하다.

**상수**:
- `FORTEM_COLLECTION_ID`: development=98, production=4 (`fortem.ts:10-11`)
- `FORTEM_NETWORK`: development="testnet", production="mainnet" (`fortem.ts:4-7`)

## 코드 참조

- `apps/web/src/components/lobby/StoreButton.tsx:356-361` - Import 버튼 (플레이스홀더)
- `apps/web/src/app/api/store/export/route.ts` - Export API Route (Import 참고 패턴)
- `apps/web/src/lib/fortem.ts` - ForTem 클라이언트 팩토리 및 상수
- `apps/web/src/components/ui/shadcn/dialog.tsx` - Dialog 컴포넌트
- `apps/web/src/components/lobby/LobbyContent.tsx:68-75` - handleExportComplete 콜백
- `apps/web/src/components/lobby/LobbyContent.tsx:178-186` - StoreButton props 전달
- `apps/web/src/app/lobby/page.tsx:38-85` - 인벤토리 데이터 조회 (서버 컴포넌트)
- `node_modules/.pnpm/@fortemlabs+sdk-js@0.0.2/node_modules/@fortemlabs/sdk-js/dist/index.d.ts:71-84,193-195` - Item 타입, items.get 시그니처

## 아키텍처 문서화

### Import 목표 흐름
```
Import 버튼 클릭 (StoreButton.tsx)
  → Dialog 열기 (Input OTP UI)
  → 'XXXX-XXXX-XXXX' 형태 코드 입력
  → 'Confirm' 버튼 클릭
  → POST /api/store/import
      1. 인증 확인 (supabase.auth.getUser)
      2. ForTem items.get(FORTEM_COLLECTION_ID, redeemCode) 호출
      3. 아이템 존재 여부 확인 (data가 있는지)
      4. status === "REDEEMED" 확인 (REDEEMED가 아니면 Import 불가)
      5. attributes에서 type/itemId 추출
      6. 해당 아이템이 마스터 테이블(characters/weapons)에 존재하는지 확인
      7. 동일 redeem_code로 중복 Import 방지 검증
      8. 인벤토리 테이블에 새 레코드 INSERT
         (user_id=현재유저, character_id/weapon_id=추출값, source='fortem', redeem_code=NULL)
      9. 성공 응답 반환
  → Dialog 닫기
  → router.refresh() (소유 목록 갱신)
```

### Export된 아이템의 생명주기
```
1. Export (유저 A)
   character_inventory: { user_id: A, character_id: "female", redeem_code: "ABCD-EFGH-IJKL", source: "fortem" }

2. ForTem 마켓에서 유저 B가 구매 → status: "REDEEMED"

3. Import (유저 B)
   character_inventory: { user_id: B, character_id: "female", redeem_code: NULL, source: "fortem" }
   (A의 원래 레코드는 그대로 유지 — 이력 보존)
```

### DB CASCADE 관계 전체 구조
```
auth.users (루트)
  ├── player_profiles.id          → ON DELETE CASCADE
  ├── wallets.user_id             → ON DELETE CASCADE
  ├── wallet_nonces.user_id       → ON DELETE CASCADE
  ├── character_inventory.user_id → ON DELETE CASCADE (redeem_code 유무 무관)
  └── weapon_inventory.user_id    → ON DELETE CASCADE (redeem_code 유무 무관)
```

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/research/2026-02-19-store-drawer-item-export-system.md` - Export 시스템 연구, `items.get` API 시그니처, DELETE→UPDATE 전환 결정
- `thoughts/arta1069/plans/2026-02-19-store-drawer-item-export-implementation.md` - Export 구현 계획, `redeem_code` 컬럼 마이그레이션
- `thoughts/arta1069/plans/2026-02-19-item-multi-ownership-ammo-system.md` - 복수 보유 구현, "Import 흐름은 별도 연구 후 구현 예정" 명시
- `thoughts/arta1069/research/2026-02-19-item-architecture-multi-ownership.md` - 복수 보유 아키텍처, `redeem_code IS NULL` 패턴
- `thoughts/arta1069/plans/2026-02-18-fortem-sdk-web-phase1.md` - ForTem SDK 1단계 계획
- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` - `source: "fortem"` = ForTem에서 import된 아이템 정의
- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md` - SDK 구조 및 `items.get` 존재 확인

## 관련 연구

- `thoughts/arta1069/research/2026-02-19-store-drawer-item-export-system.md`
- `thoughts/arta1069/research/2026-02-19-item-architecture-multi-ownership.md`
- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md`

## 검토사항 분석

### 검토사항 1: 소유권 이전 (user_id 변경)

인벤토리 테이블의 `user_id`가 소유자를 나타낸다. Export된 아이템이 다른 유저에 의해 Import될 때:
- 기존 레코드(Export한 유저 A)의 `user_id`를 변경하는 것이 **아닌**, 새 레코드를 INSERT하는 방식이 적절하다
- 이유: A의 Export 이력 보존, FK 제약 일관성 유지, 트랜잭션 단순화
- Import된 레코드: `user_id=B`, `source='fortem'`, `redeem_code=NULL`

### 검토사항 2: auth.users 삭제 시 Export된 아이템 보존

현재 `ON DELETE CASCADE` 설정으로 인해 유저 삭제 시 모든 인벤토리 레코드가 삭제된다. 그러나:
- **Import에는 영향 없음**: Import는 ForTem API의 `items.get` 응답으로 검증하며, 원래 Export한 유저의 인벤토리 레코드를 참조하지 않음
- **중복 Import 방지가 핵심**: 동일 `redeem_code`로 여러 번 Import되는 것을 방지해야 함
  - Export한 유저의 레코드가 삭제되면 인벤토리 테이블에서 `redeem_code` 존재 여부로 검증 불가
  - 대안: Import API에서 인벤토리 테이블 전체(`WHERE redeem_code = ?`, user_id 무관)를 검색하여 이미 누군가 Import했는지 확인
  - 또는: Import 시 인벤토리에 `redeem_code`를 함께 기록하여 추적 가능하게 함

## 미해결 질문

1. **중복 Import 방지**: 동일 `redeem_code`로 여러 번 Import하는 것을 어떻게 방지할 것인가? (인벤토리 테이블 검색 vs 별도 테이블 vs ForTem API 상태 활용)
2. **ForTem `status` 값의 의미**: `"REDEEMED"` 상태는 정확히 어떤 시점에 설정되는가? 구매 완료 시점인가, 별도 Redeem 액션이 필요한가?
3. **Import 후 ForTem 상태 변경**: Import 완료 후 ForTem 측에 상태를 업데이트하는 API가 있는가? (현재 SDK에는 `items.create`, `items.get`, `items.uploadImage`만 존재)
4. **redeem_code의 형태 검증**: OTP Input에서 하이픈을 포함하여 입력받을 것인가, 12자리 문자만 입력받고 하이픈은 자동 삽입할 것인가?
