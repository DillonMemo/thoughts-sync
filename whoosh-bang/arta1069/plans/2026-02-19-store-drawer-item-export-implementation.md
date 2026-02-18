# Store Drawer UI 및 아이템 내보내기(Export) 시스템 구현 계획

## 개요

Store 버튼을 통한 Drawer UI 구현과 Fortem SDK를 활용한 아이템 내보내기(export) 시스템 구현. 사용자가 소유한 캐릭터/무기를 ForTem 마켓플레이스로 내보내는 전체 플로우를 구현한다.

## 현재 상태 분석

- `StoreButton` 컴포넌트: Drawer 내부 "Coming soon..." 플레이스홀더만 존재 (`StoreButton.tsx:84`)
- `@fortemlabs/sdk-js@0.0.2`: 설치됨, 소스에서 미사용 (이전 테스트 코드 롤백됨)
- 인벤토리 테이블: `character_inventory`, `weapon_inventory` — `redeem_code` 컬럼 미존재
- vaul Drawer: `DrawerNestedRoot` 미export 상태 (`drawer.tsx`)
- API 라우트 패턴: SSR 인증 + Admin 클라이언트 2단계 구조 확립됨

### 핵심 발견:
- 기존 API 라우트 5개 모두 동일한 인증 패턴 사용 (`wallet/verify/route.ts:6-11`)
- `LobbyContent`의 `ownedCharacterSet`/`ownedWeaponSet`은 props 직접 파생 (line 60-61) — `router.refresh()`로 갱신 가능
- `selectedCharacter`/`selectedWeapons`는 `useState` (line 50-53) — 별도 `setState` 필요
- DB 트리거 `validate_selected_character`, `validate_selected_weapons`가 소유권 검증 수행
- 기본 아이템(`"player"`, `"bazooka"`)은 `handle_new_user` 트리거로 `source: "default"`로 부여됨

## 원하는 최종 상태

```
StoreButton 클릭
  ├── linkedWallet 없음 → WalletSelectDialog → 지갑 연결 → Drawer 열기
  └── linkedWallet 있음 → Drawer 열기 → fortem.users.verify(walletAddress)
        ├── 로딩 중 → Spinner UI
        ├── isUser === false → ForTem 미가입 안내 + "ForTem 가입하기" 버튼 (https://fortem.gg)
        └── isUser === true → Store 메뉴
              ├── "Store 바로가기" 버튼 → https://fortem.gg (외부 링크)
              └── "아이템 내보내기" 버튼 → Nested Drawer
                    → 보유 아이템 목록 (기본 아이템 제외)
                    → 단일 선택 → Export 버튼
                    → POST /api/store/export → 성공 시 UI 갱신
```

검증 방법: Store Drawer 전체 플로우를 수동으로 테스트하여 verify → 분기 → export → UI 갱신이 정상 작동하는지 확인.

## 하지 않을 것

- mainnet 배포 (현재 testnet 전용)
- ForTem import (되돌리기) 기능
- 배치 export (다중 아이템 동시 내보내기)
- `validate_selected_character`/`validate_selected_weapons` 트리거 수정 (현재 트리거는 `redeem_code` 필터 없이 소유권 검증하지만, export 흐름에서 기본 아이템 fallback이 항상 보장되므로 문제 없음)
- ForTem 아이템 이미지 업로드 (`itemImage` 파라미터 — 추후 작업)
- 테스트 코드 작성 (현재 프로젝트에 테스트 프레임워크 미설정)

## 구현 접근 방식

4단계 점진적 구현: DB 기반 → SDK 유틸리티 + Drawer UI → 서버 API → 프론트엔드 완성. 각 단계가 독립적으로 검증 가능하도록 구성.

---

## 1단계: DB 마이그레이션 + 쿼리 수정

### 개요
두 인벤토리 테이블에 `redeem_code` 컬럼을 추가하고, 기존 인벤토리 조회 쿼리에 필터를 추가하여 export된 아이템을 게임플레이에서 제외한다.

### 필요한 변경:

#### 1. DB 마이그레이션 — `redeem_code` 컬럼 추가
**도구**: Supabase MCP `apply_migration`
**마이그레이션명**: `add_redeem_code_to_inventory`

```sql
-- character_inventory에 redeem_code 컬럼 추가
ALTER TABLE character_inventory
ADD COLUMN redeem_code TEXT DEFAULT NULL;

-- weapon_inventory에 redeem_code 컬럼 추가
ALTER TABLE weapon_inventory
ADD COLUMN redeem_code TEXT DEFAULT NULL;

-- redeem_code IS NULL인 아이템만 빠르게 조회하기 위한 부분 인덱스
CREATE INDEX idx_character_inventory_active
ON character_inventory(user_id)
WHERE redeem_code IS NULL;

CREATE INDEX idx_weapon_inventory_active
ON weapon_inventory(user_id)
WHERE redeem_code IS NULL;
```

`redeem_code IS NULL` → 인게임 사용 중, `redeem_code IS NOT NULL` → ForTem에 export됨.
UNIQUE(user_id, character_id/weapon_id) 제약은 유지 — 동일 아이템 중복 소유 방지.

#### 2. 로비 인벤토리 쿼리 수정
**파일**: `apps/web/src/app/lobby/page.tsx`

line 38-41 (캐릭터 인벤토리):
```ts
const { data: ownedInventory } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)  // 추가
```

line 57-60 (무기 인벤토리):
```ts
const { data: ownedWeaponInventory } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)
  .is("redeem_code", null)  // 추가
```

#### 3. 게임 진입 인벤토리 쿼리 수정
**파일**: `apps/web/src/app/game/page.tsx`

line 30-35 (캐릭터 소유권 검증):
```ts
const { data: ownership } = await supabase
  .from("character_inventory")
  .select("character_id")
  .eq("user_id", user.id)
  .eq("character_id", selectedCharacter)
  .is("redeem_code", null)  // 추가
  .single()
```

line 43-47 (무기 소유권 검증):
```ts
const { data: ownedWeapons } = await supabase
  .from("weapon_inventory")
  .select("weapon_id")
  .eq("user_id", user.id)
  .in("weapon_id", selectedWeaponIds)
  .is("redeem_code", null)  // 추가
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 마이그레이션이 오류 없이 적용됨 (Supabase MCP `apply_migration`)
- [ ] `pnpm --filter web build` 성공 (타입 에러 없음)

#### 수동 검증:
- [ ] 로비에서 기존 보유 아이템이 정상 표시됨
- [ ] 게임 진입 시 캐릭터/무기 소유권 검증 정상 작동
- [ ] DB에서 `SELECT * FROM character_inventory WHERE redeem_code IS NOT NULL`이 빈 결과 반환 (아직 export된 아이템 없음)

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: Fortem SDK 유틸리티 + Store Drawer UI

### 개요
Fortem 클라이언트 초기화 유틸리티를 생성하고, Store Drawer 내부에 `verify` 기반 분기 UI를 구현한다. `DrawerNestedRoot` export를 추가하고, `StoreButton`에 인벤토리 데이터를 전달한다.

### 필요한 변경:

#### 1. 환경변수 추가
**파일**: `apps/web/.env.local`

```
# ForTem (서버 전용)
FORTEM_API_KEY=developer_m4c1aj741e_b4b8446a0bff521
```

기존 `NEXT_PUBLIC_FORTEM_API_KEY`는 클라이언트 `verify` 호출용, 신규 `FORTEM_API_KEY`는 서버 `items.create` 호출용.

#### 2. Fortem 클라이언트 유틸리티
**파일**: `apps/web/src/lib/fortem.ts` (신규)

```ts
import { createFortemClient } from "@fortemlabs/sdk-js"

/** ForTem testnet 컬렉션 ID */
export const FORTEM_COLLECTION_ID = 98

/** 클라이언트 사이드 ForTem SDK 인스턴스 (verify 전용) */
export function getFortemClient() {
  return createFortemClient({
    apiKey: process.env.NEXT_PUBLIC_FORTEM_API_KEY!,
    network: "testnet",
  })
}

/** 서버 사이드 ForTem SDK 인스턴스 (items.create 전용) */
export function getFortemServerClient() {
  return createFortemClient({
    apiKey: process.env.FORTEM_API_KEY!,
    network: "testnet",
  })
}
```

#### 3. DrawerNestedRoot export 추가
**파일**: `apps/web/src/components/ui/shadcn/drawer.tsx`

기존 export 목록에 `DrawerNestedRoot` 추가:

```tsx
function DrawerNestedRoot({
  ...props
}: React.ComponentProps<typeof DrawerPrimitive.NestedRoot>) {
  return <DrawerPrimitive.NestedRoot data-slot="drawer-nested-root" {...props} />
}

export {
  Drawer,
  DrawerClose,
  DrawerContent,
  DrawerDescription,
  DrawerFooter,
  DrawerHeader,
  DrawerNestedRoot,  // 추가
  DrawerOverlay,
  DrawerPortal,
  DrawerTitle,
  DrawerTrigger,
}
```

#### 4. StoreButton Props 확장 + Verify 분기 UI
**파일**: `apps/web/src/components/lobby/StoreButton.tsx`

Props 확장:
```ts
interface StoreButtonProps {
  linkedWallet: { wallet_address: string; chain: string } | null
  onWalletLinked: (
    wallet: { wallet_address: string; chain: string } | null
  ) => void
  allCharacters: CharacterData[]
  ownedCharacterIds: string[]
  allWeapons: WeaponData[]
  ownedWeaponIds: string[]
  onExportComplete: (updates: {
    selectedCharacter: string
    selectedWeapons: string[]
  }) => void
}
```

Drawer 내부 로직 (상태 추가):
```ts
const [fortemUser, setFortemUser] = useState<{
  isUser: boolean
  nickname: string
} | null>(null)
const [isVerifying, setIsVerifying] = useState(false)
```

Drawer `onOpenChange` 시 verify 호출:
```ts
const handleDrawerOpenChange = (open: boolean) => {
  setShowStoreDrawer(open)
  if (open && linkedWallet) {
    verifyFortemUser(linkedWallet.wallet_address)
  }
  if (!open) {
    setFortemUser(null) // Drawer 닫힐 때 상태 초기화
  }
}

const verifyFortemUser = async (walletAddress: string) => {
  setIsVerifying(true)
  try {
    const fortem = getFortemClient()
    const result = await fortem.users.verify(walletAddress)
    setFortemUser({
      isUser: result.data.isUser,
      nickname: result.data.nickname,
    })
  } catch {
    toast.error("Failed to verify ForTem account")
    setFortemUser(null)
  } finally {
    setIsVerifying(false)
  }
}
```

Drawer 내부 콘텐츠 분기:
```tsx
<DrawerContent>
  <DrawerHeader>
    <DrawerTitle>Store</DrawerTitle>
    <DrawerDescription>
      Browse and manage your items
    </DrawerDescription>
  </DrawerHeader>
  <div className="p-4">
    {isVerifying ? (
      <div className="flex justify-center py-8">
        <Loader2 className="animate-spin" />
      </div>
    ) : fortemUser === null ? (
      <p className="text-sm text-muted-foreground text-center py-8">
        Unable to verify ForTem account
      </p>
    ) : !fortemUser.isUser ? (
      /* ForTem 미가입 안내 */
      <div className="flex flex-col items-center gap-4 py-8">
        <p className="text-sm text-muted-foreground text-center">
          ForTem 계정이 필요합니다
        </p>
        <Button asChild>
          <a href="https://fortem.gg" target="_blank" rel="noopener noreferrer">
            ForTem 가입하기
          </a>
        </Button>
      </div>
    ) : (
      /* ForTem 가입 완료 — Store 메뉴 */
      <div className="flex flex-col gap-3">
        <Button asChild variant="outline" className="w-full">
          <a href="https://fortem.gg" target="_blank" rel="noopener noreferrer">
            Store 바로가기
          </a>
        </Button>
        <Button
          className="w-full"
          onClick={() => setShowExportDrawer(true)}
        >
          아이템 내보내기
        </Button>
      </div>
    )}
  </div>
</DrawerContent>
```

#### 5. LobbyContent에서 StoreButton으로 데이터 전달
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx`

line 160-163의 StoreButton 호출 수정:
```tsx
<StoreButton
  linkedWallet={linkedWallet}
  onWalletLinked={handleWalletChange}
  allCharacters={allCharacters}
  ownedCharacterIds={ownedCharacterIds}
  allWeapons={allWeapons}
  ownedWeaponIds={ownedWeaponIds}
  onExportComplete={handleExportComplete}
/>
```

`handleExportComplete` 핸들러 추가:
```ts
const router = useRouter()

const handleExportComplete = useCallback(
  (updates: { selectedCharacter: string; selectedWeapons: string[] }) => {
    setSelectedCharacter(updates.selectedCharacter)
    setSelectedWeapons(updates.selectedWeapons)
    router.refresh()
  },
  [router]
)
```

`LobbyContent`에서 `ownedCharacterIds`와 `ownedWeaponIds`는 props로 이미 존재하므로 그대로 전달.

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm --filter web build` 성공 (타입 에러 없음)

#### 수동 검증:
- [ ] Store 버튼 클릭 시 Drawer 열림
- [ ] 지갑 연결된 상태에서 verify 호출 → 로딩 스피너 표시 → 결과에 따른 분기 UI 표시
- [ ] ForTem 미가입 시: "ForTem 가입하기" 버튼 클릭 → https://fortem.gg 이동
- [ ] ForTem 가입 완료 시: "Store 바로가기" + "아이템 내보내기" 버튼 표시
- [ ] "Store 바로가기" 클릭 → https://fortem.gg 이동

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 3단계: Export API 라우트

### 개요
`POST /api/store/export` API 라우트를 생성한다. 기존 API 패턴(SSR 인증 + Admin 클라이언트)을 따르며, ForTem `items.create` 호출 + 인벤토리 `redeem_code` UPDATE + `player_profiles` selection fallback UPDATE를 순차 처리한다.

### 필요한 변경:

#### 1. Export API 라우트
**파일**: `apps/web/src/app/api/store/export/route.ts` (신규)

요청 형식:
```ts
{ itemType: "character" | "weapon", itemId: string }
```

응답 형식 (성공):
```ts
{
  success: true,
  redeemCode: string,
  selectedCharacter: string,
  selectedWeapons: string[]
}
```

구현 플로우:
```ts
import { createClient } from "@/lib/supabase/server"
import { createClient as createAdminClient } from "@supabase/supabase-js"
import { getFortemServerClient, FORTEM_COLLECTION_ID } from "@/lib/fortem"

function generateRedeemCode(): string {
  const bytes = crypto.getRandomValues(new Uint8Array(12))
  const chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
  const code = Array.from(bytes, (b) => chars[b % chars.length]).join("")
  return `${code.slice(0, 4)}-${code.slice(4, 8)}-${code.slice(8, 12)}`
}

export async function POST(request: Request) {
  // 1. 인증 확인
  const supabase = await createClient()
  const { data: { user }, error } = await supabase.auth.getUser()
  if (error || !user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 })
  }

  // 2. 요청 파싱 + 검증
  const { itemType, itemId } = await request.json()
  if (!itemType || !itemId || !["character", "weapon"].includes(itemType)) {
    return Response.json({ error: "Invalid parameters" }, { status: 400 })
  }

  // 3. 기본 아이템 export 차단
  if ((itemType === "character" && itemId === "player") ||
      (itemType === "weapon" && itemId === "bazooka")) {
    return Response.json({ error: "Cannot export default items" }, { status: 400 })
  }

  // 4. Admin 클라이언트 생성
  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  // 5. 지갑 주소 조회
  const { data: wallet } = await admin
    .from("wallets")
    .select("wallet_address")
    .eq("user_id", user.id)
    .single()
  if (!wallet) {
    return Response.json({ error: "No linked wallet" }, { status: 400 })
  }

  // 6. 소유권 확인 (redeem_code IS NULL인 아이템만)
  const table = itemType === "character" ? "character_inventory" : "weapon_inventory"
  const idColumn = itemType === "character" ? "character_id" : "weapon_id"

  const { data: inventoryItem } = await admin
    .from(table)
    .select("id")
    .eq("user_id", user.id)
    .eq(idColumn, itemId)
    .is("redeem_code", null)
    .single()
  if (!inventoryItem) {
    return Response.json({ error: "Item not owned or already exported" }, { status: 400 })
  }

  // 7. 마스터 테이블에서 아이템 이름 조회
  const masterTable = itemType === "character" ? "characters" : "weapons"
  const { data: itemData } = await admin
    .from(masterTable)
    .select("name")
    .eq("id", itemId)
    .single()
  if (!itemData) {
    return Response.json({ error: "Item data not found" }, { status: 500 })
  }

  // 8. Redeem 코드 생성
  const redeemCode = generateRedeemCode()

  // 9. ForTem 아이템 생성
  try {
    const fortem = getFortemServerClient()
    await fortem.items.create(FORTEM_COLLECTION_ID, {
      name: itemData.name,
      quantity: 1,
      redeemCode,
      description: `Whoosh Bang ${itemType === "character" ? "Character" : "Weapon"} - ${itemData.name}`,
      recipientAddress: wallet.wallet_address,
      attributes: [
        { name: "type", value: itemType },
        { name: "game_id", value: itemId },
      ],
    })
  } catch (err) {
    console.error("ForTem items.create failed:", err)
    return Response.json({ error: "Failed to create ForTem item" }, { status: 500 })
  }

  // 10. 인벤토리 redeem_code 업데이트
  const { error: updateError } = await admin
    .from(table)
    .update({ redeem_code: redeemCode })
    .eq("id", inventoryItem.id)
  if (updateError) {
    console.error("Inventory update failed:", updateError)
    return Response.json({ error: "Failed to update inventory" }, { status: 500 })
  }

  // 11. player_profiles selection fallback 처리
  const { data: profile } = await admin
    .from("player_profiles")
    .select("selected_character, selected_weapons")
    .eq("id", user.id)
    .single()

  let selectedCharacter = profile?.selected_character ?? "player"
  let selectedWeapons: string[] = profile?.selected_weapons ?? ["bazooka"]

  if (itemType === "character" && selectedCharacter === itemId) {
    selectedCharacter = "player"
    await admin
      .from("player_profiles")
      .update({ selected_character: "player" })
      .eq("id", user.id)
  }

  if (itemType === "weapon" && selectedWeapons.includes(itemId)) {
    selectedWeapons = selectedWeapons.filter((id) => id !== itemId)
    if (selectedWeapons.length === 0) selectedWeapons = ["bazooka"]
    await admin
      .from("player_profiles")
      .update({ selected_weapons: selectedWeapons })
      .eq("id", user.id)
  }

  return Response.json({
    success: true,
    redeemCode,
    selectedCharacter,
    selectedWeapons,
  })
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm --filter web build` 성공 (타입 에러 없음)

#### 수동 검증:
- [ ] 유효한 아이템으로 POST 호출 시 200 + ForTem 아이템 생성 + DB redeem_code 갱신
- [ ] 기본 아이템(`"player"`, `"bazooka"`) export 시도 시 400 반환
- [ ] 미소유 아이템 export 시도 시 400 반환
- [ ] 현재 선택 중인 아이템 export 시 selection이 기본값으로 fallback

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 4단계: 아이템 내보내기 Nested Drawer + UI 갱신

### 개요
Nested Drawer로 보유 아이템 선택 UI를 구현하고, Export API 호출 후 클라이언트 상태 동기화 + `router.refresh()`로 selector UI를 갱신한다.

### 필요한 변경:

#### 1. StoreButton에 Nested Drawer + Export 로직 추가
**파일**: `apps/web/src/components/lobby/StoreButton.tsx`

추가 상태:
```ts
const [showExportDrawer, setShowExportDrawer] = useState(false)
const [selectedExportItem, setSelectedExportItem] = useState<{
  type: "character" | "weapon"
  id: string
  name: string
} | null>(null)
const [isExporting, setIsExporting] = useState(false)
```

Export 가능 아이템 계산:
```ts
const exportableCharacters = allCharacters.filter(
  (c) => c.id !== "player" && ownedCharacterIds.includes(c.id)
)
const exportableWeapons = allWeapons.filter(
  (w) => w.id !== "bazooka" && ownedWeaponIds.includes(w.id)
)
```

Export 실행 함수:
```ts
const handleExport = async () => {
  if (!selectedExportItem) return

  setIsExporting(true)
  try {
    const res = await fetch("/api/store/export", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        itemType: selectedExportItem.type,
        itemId: selectedExportItem.id,
      }),
    })

    if (!res.ok) {
      const data = await res.json()
      toast.error(data.error ?? "Export failed")
      return
    }

    const data = await res.json()
    toast.success(`${selectedExportItem.name} exported successfully`)

    // Nested Drawer + Store Drawer 닫기
    setShowExportDrawer(false)
    setShowStoreDrawer(false)
    setSelectedExportItem(null)

    // 부모 컴포넌트에 갱신 알림
    onExportComplete({
      selectedCharacter: data.selectedCharacter,
      selectedWeapons: data.selectedWeapons,
    })
  } catch {
    toast.error("Export failed")
  } finally {
    setIsExporting(false)
  }
}
```

Nested Drawer UI (Store 메뉴 Drawer 안에서):
```tsx
<DrawerNestedRoot open={showExportDrawer} onOpenChange={setShowExportDrawer}>
  <DrawerContent>
    <DrawerHeader>
      <DrawerTitle>아이템 내보내기</DrawerTitle>
      <DrawerDescription>
        ForTem으로 내보낼 아이템을 선택하세요
      </DrawerDescription>
    </DrawerHeader>
    <div className="p-4 space-y-4 max-h-[50vh] overflow-y-auto">
      {/* 캐릭터 섹션 */}
      {exportableCharacters.length > 0 && (
        <div>
          <h3 className="text-sm font-medium mb-2">Characters</h3>
          <div className="space-y-2">
            {exportableCharacters.map((char) => (
              <button
                key={char.id}
                onClick={() => setSelectedExportItem({
                  type: "character", id: char.id, name: char.name
                })}
                className={cn(
                  "w-full flex items-center gap-3 p-3 rounded-lg border",
                  selectedExportItem?.id === char.id
                    ? "border-primary bg-primary/10"
                    : "border-border"
                )}
              >
                {/* 썸네일 + 이름 */}
                <span className="text-sm">{char.name}</span>
              </button>
            ))}
          </div>
        </div>
      )}

      {/* 무기 섹션 */}
      {exportableWeapons.length > 0 && (
        <div>
          <h3 className="text-sm font-medium mb-2">Weapons</h3>
          <div className="space-y-2">
            {exportableWeapons.map((weapon) => (
              <button
                key={weapon.id}
                onClick={() => setSelectedExportItem({
                  type: "weapon", id: weapon.id, name: weapon.name
                })}
                className={cn(
                  "w-full flex items-center gap-3 p-3 rounded-lg border",
                  selectedExportItem?.id === weapon.id
                    ? "border-primary bg-primary/10"
                    : "border-border"
                )}
              >
                <span className="text-sm">{weapon.name}</span>
              </button>
            ))}
          </div>
        </div>
      )}

      {/* Export 가능 아이템 없음 */}
      {exportableCharacters.length === 0 && exportableWeapons.length === 0 && (
        <p className="text-sm text-muted-foreground text-center py-4">
          내보낼 수 있는 아이템이 없습니다
        </p>
      )}
    </div>

    <DrawerFooter>
      <Button
        onClick={handleExport}
        disabled={!selectedExportItem || isExporting}
      >
        {isExporting ? (
          <Loader2 className="mr-2 h-4 w-4 animate-spin" />
        ) : null}
        Export
      </Button>
    </DrawerFooter>
  </DrawerContent>
</DrawerNestedRoot>
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm --filter web build` 성공 (타입 에러 없음)

#### 수동 검증:
- [ ] "아이템 내보내기" 클릭 시 Nested Drawer 열림
- [ ] 기본 아이템(`"player"`, `"bazooka"`)이 목록에 표시되지 않음
- [ ] 아이템 단일 선택 → Export 버튼 활성화
- [ ] Export 성공 시 toast 메시지 + Drawer 닫힘
- [ ] Export 후 로비의 CharacterSelector/LobbyWeaponSelector에서 해당 아이템이 소유 목록에서 제거됨
- [ ] Export된 아이템이 현재 선택 아이템이었을 경우, 기본 아이템으로 fallback 표시
- [ ] Export 후 Store Drawer를 다시 열고 "아이템 내보내기"를 누르면 방금 export한 아이템이 목록에 없음

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 테스트 전략

### 수동 테스트 시나리오:

1. **기본 플로우**: 지갑 연결 → Store Drawer → verify → "아이템 내보내기" → 캐릭터 선택 → Export → 성공
2. **ForTem 미가입**: verify 결과 `isUser === false` → 안내 UI 표시
3. **기본 아이템 보호**: 기본 캐릭터/무기가 export 목록에 표시되지 않는지 확인
4. **선택 아이템 fallback**: 현재 선택 중인 캐릭터를 export → `selected_character`가 `"player"`로 변경되는지 확인
5. **무기 선택 fallback**: 현재 선택 무기 3개 중 하나를 export → 2개만 남는지 확인. 유일한 비기본 무기를 export → `["bazooka"]`로 fallback
6. **중복 export 방지**: 이미 export된 아이템이 목록에 표시되지 않는지 확인
7. **게임 진입 검증**: export 후 게임 진입 시 export된 아이템이 적용되지 않는지 확인

## 성능 고려 사항

- `redeem_code IS NULL` 부분 인덱스로 인벤토리 조회 성능 유지
- `users.verify()` 호출은 Drawer 열릴 때만 (매번 호출, 캐싱 없음 — ForTem 가입 상태가 변경될 수 있으므로)
- `items.create` 호출은 외부 API이므로 네트워크 지연 가능 — isExporting 로딩 상태로 UX 처리

## 마이그레이션 참고 사항

- `redeem_code` 컬럼 추가는 `DEFAULT NULL`이므로 기존 데이터에 영향 없음
- 기존 인벤토리 레코드는 모두 `redeem_code IS NULL` 상태 유지
- 로비/게임 쿼리의 `.is("redeem_code", null)` 필터는 기존 동작과 동일한 결과 반환

## 파일 변경 요약

| 파일 | 변경 유형 | 설명 |
|------|-----------|------|
| DB 마이그레이션 | 신규 | `redeem_code` 컬럼 + 부분 인덱스 |
| `apps/web/.env.local` | 수정 | `FORTEM_API_KEY` 추가 |
| `apps/web/src/lib/fortem.ts` | 신규 | Fortem 클라이언트 유틸리티 |
| `apps/web/src/components/ui/shadcn/drawer.tsx` | 수정 | `DrawerNestedRoot` export 추가 |
| `apps/web/src/app/lobby/page.tsx` | 수정 | 인벤토리 쿼리에 `redeem_code IS NULL` 필터 |
| `apps/web/src/app/game/page.tsx` | 수정 | 인벤토리 쿼리에 `redeem_code IS NULL` 필터 |
| `apps/web/src/components/lobby/LobbyContent.tsx` | 수정 | StoreButton props 확장 + handleExportComplete |
| `apps/web/src/components/lobby/StoreButton.tsx` | 수정 | verify 분기 UI + Nested Drawer + Export 로직 |
| `apps/web/src/app/api/store/export/route.ts` | 신규 | Export API 라우트 |

## 참조

- 연구 문서: `thoughts/arta1069/research/2026-02-19-store-drawer-item-export-system.md`
- SDK 1단계 계획: `thoughts/arta1069/plans/2026-02-18-fortem-sdk-web-phase1.md`
- 무기 아이템화 계획: `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md`
- Fortem SDK 타입: `node_modules/@fortemlabs/sdk-js/dist/index.d.ts`
