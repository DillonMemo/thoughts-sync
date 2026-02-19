---
date: "2026-02-19"
author: arta1069
git_commit: ea6a086
branch: main
repository: whoosh-bang
topic: "아이템 Import(Redeem Code) 시스템 구현 계획"
tags: [plan, import, redeem-code, fortem-sdk, inventory, plpgsql, otp-input]
status: approved
---

# 아이템 Import(Redeem Code) 시스템 구현 계획

## 개요

ForTem 마켓플레이스에서 구매한 아이템을 Redeem Code로 Import하는 시스템을 구현한다. 유저가 `XXXX-XXXX-XXXX` 형태의 코드를 입력하면 ForTem API로 검증 후 인벤토리에 아이템을 추가하고, 기존 Export 레코드를 삭제하여 중복 Import를 방지한다.

## 현재 상태 분석

- **Import 버튼**: `StoreButton.tsx:356-361`에 `console.info("handle import")` 플레이스홀더만 존재
- **Export 시스템**: 완전 구현됨 (`/api/store/export` API Route)
- **DB 스키마**: `user_id NOT NULL`, BEFORE DELETE 트리거 미적용
- **UI**: `input-otp` 패키지 미설치, OTP 컴포넌트 미존재
- **트랜잭션 패턴**: 프로젝트에서 원자적 다중 테이블 작업은 plpgsql 함수 + `admin.rpc()` 사용

### 핵심 발견:
- `complete_match`, `record_abandoned_match` 등 기존 plpgsql 함수가 `admin.rpc()` 패턴으로 사용됨 (`match/complete/route.ts:26-32`)
- Export API는 순차적 개별 쿼리 사용 — Import는 원자성 필요하므로 plpgsql 함수가 적절
- Dialog 컴포넌트(`dialog.tsx`) 존재하지만, Import UI는 Drawer 내부 인라인으로 구현
- `fortem.items.get(collectionId, code)`는 인증 없이 호출 가능하지만, DB 작업과 원자적 처리를 위해 서버 사이드에서 처리
- `redeem_code IS NULL` 부분 인덱스가 활성 아이템 필터로 사용됨

## 원하는 최종 상태

- ForTem 계정이 인증된 유저가 Store Drawer 내에서 Redeem Code를 입력하여 아이템을 Import할 수 있음
- Import 시 ForTem API로 아이템 존재/상태 검증 → plpgsql 함수로 원자적 INSERT + DELETE 처리
- 유저 탈퇴 시에도 Export된 아이템 레코드가 보존되어 중복 Import 방지가 유지됨
- Import 완료 후 `router.refresh()`로 소유 목록이 즉시 갱신됨

### 검증 방법:
1. Export된 아이템의 Redeem Code로 다른 유저가 Import 성공
2. 동일 Redeem Code로 재Import 시도 시 에러
3. Export한 유저 탈퇴 후에도 Redeem Code Import 가능
4. Import 후 로비에서 아이템이 즉시 표시

## 하지 않을 것

- ForTem 측 상태 변경 API 호출 (존재하지 않음)
- Import 전용 별도 페이지/라우트 생성 (Drawer 내부 인라인 처리)
- Redeem Code 유효기간/만료 처리 (현재 요구 사항 아님)
- Import 이력 테이블 생성 (현재 단계에서 불필요)

## 구현 접근 방식

Export 시스템의 패턴을 따르되, 원자적 트랜잭션이 필요한 부분은 plpgsql 함수로 구현한다. UI는 Drawer 내부에서 상태 전환 방식으로 Export 아이템 목록과 Import 코드 입력을 전환한다.

---

## 1단계: DB 마이그레이션

### 개요
`user_id` nullable 변경, BEFORE DELETE 트리거 생성, `import_inventory_item()` plpgsql 함수 생성

### 필요한 변경:

#### 1. user_id nullable + BEFORE DELETE 트리거 마이그레이션
**적용 방식**: Supabase MCP `apply_migration`

```sql
-- user_id nullable 변경
ALTER TABLE character_inventory ALTER COLUMN user_id DROP NOT NULL;
ALTER TABLE weapon_inventory ALTER COLUMN user_id DROP NOT NULL;

-- BEFORE DELETE 트리거: Export된 레코드의 user_id를 NULL로 설정하여 CASCADE에서 보존
CREATE OR REPLACE FUNCTION preserve_exported_inventory()
RETURNS TRIGGER AS $$
BEGIN
  UPDATE character_inventory SET user_id = NULL
  WHERE user_id = OLD.id AND redeem_code IS NOT NULL;
  UPDATE weapon_inventory SET user_id = NULL
  WHERE user_id = OLD.id AND redeem_code IS NOT NULL;
  RETURN OLD;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER preserve_exported_before_user_delete
BEFORE DELETE ON auth.users
FOR EACH ROW
EXECUTE FUNCTION preserve_exported_inventory();
```

#### 2. import_inventory_item() plpgsql 함수 마이그레이션
**적용 방식**: Supabase MCP `apply_migration`

```sql
CREATE OR REPLACE FUNCTION import_inventory_item(
  p_user_id UUID,
  p_item_type TEXT,       -- 'character' 또는 'weapon'
  p_item_id TEXT,         -- character_id 또는 weapon_id
  p_redeem_code TEXT,     -- 입력된 redeem code
  p_network TEXT          -- FORTEM_NETWORK 값
)
RETURNS JSON AS $$
DECLARE
  v_old_record_id UUID;
  v_table TEXT;
  v_id_column TEXT;
  v_new_id UUID;
BEGIN
  -- 테이블/컬럼 결정
  IF p_item_type = 'character' THEN
    v_table := 'character_inventory';
    v_id_column := 'character_id';
  ELSIF p_item_type = 'weapon' THEN
    v_table := 'weapon_inventory';
    v_id_column := 'weapon_id';
  ELSE
    RETURN json_build_object('error', 'Invalid item type');
  END IF;

  -- 기존 Export 레코드 잠금 (race condition 방지)
  EXECUTE format(
    'SELECT id FROM %I WHERE redeem_code = $1 FOR UPDATE',
    v_table
  ) INTO v_old_record_id USING p_redeem_code;

  IF v_old_record_id IS NULL THEN
    RETURN json_build_object('error', 'Redeem code not found in inventory');
  END IF;

  -- 새 레코드 INSERT
  EXECUTE format(
    'INSERT INTO %I (user_id, %I, source, redeem_code, network) VALUES ($1, $2, $3, NULL, $4) RETURNING id',
    v_table, v_id_column
  ) INTO v_new_id USING p_user_id, p_item_id, 'fortem', p_network;

  -- 기존 Export 레코드 DELETE (중복 Import 방지)
  EXECUTE format(
    'DELETE FROM %I WHERE id = $1',
    v_table
  ) USING v_old_record_id;

  RETURN json_build_object('success', true, 'new_inventory_id', v_new_id);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER SET search_path = '';
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 마이그레이션이 에러 없이 적용됨 (Supabase MCP `apply_migration`)
- [ ] `user_id` 컬럼에 NULL 허용 확인: `SELECT is_nullable FROM information_schema.columns WHERE table_name = 'character_inventory' AND column_name = 'user_id'` → `YES`
- [ ] 트리거 존재 확인: `SELECT tgname FROM pg_trigger WHERE tgname = 'preserve_exported_before_user_delete'`
- [ ] 함수 존재 확인: `SELECT proname FROM pg_proc WHERE proname = 'import_inventory_item'`

#### 수동 검증:
- [ ] Export된 레코드가 있는 테스트 유저 삭제 시 Export 레코드가 `user_id = NULL`로 보존되는지 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: Import API Route

### 개요
`POST /api/store/import` API Route를 생성하여 ForTem API 검증 + plpgsql 함수 호출을 처리

### 필요한 변경:

#### 1. Import API Route 생성
**파일**: `apps/web/src/app/api/store/import/route.ts` (신규)

```typescript
import { createClient } from "@/lib/supabase/server"
import { createClient as createAdminClient } from "@supabase/supabase-js"
import {
  FORTEM_COLLECTION_ID,
  FORTEM_NETWORK,
  getFortemServerClient,
} from "@/lib/fortem"

export async function POST(request: Request) {
  // 1. 인증 확인
  const supabase = await createClient()
  const {
    data: { user },
    error,
  } = await supabase.auth.getUser()
  if (error || !user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 })
  }

  // 2. 요청 파싱
  const { redeemCode } = await request.json()
  if (!redeemCode || typeof redeemCode !== "string") {
    return Response.json({ error: "Invalid redeem code" }, { status: 400 })
  }

  // 3. Redeem Code 형식 정규화 (12자리 → XXXX-XXXX-XXXX)
  const cleaned = redeemCode.replace(/[^A-Z0-9]/gi, "").toUpperCase()
  if (cleaned.length !== 12) {
    return Response.json({ error: "Invalid redeem code format" }, { status: 400 })
  }
  const formattedCode = `${cleaned.slice(0, 4)}-${cleaned.slice(4, 8)}-${cleaned.slice(8, 12)}`

  // 4. ForTem API로 아이템 검증
  let fortemItem
  try {
    const fortem = getFortemServerClient()
    const result = await fortem.items.get(FORTEM_COLLECTION_ID, formattedCode)
    fortemItem = result.data
  } catch {
    return Response.json({ error: "Item not found on ForTem" }, { status: 404 })
  }

  if (!fortemItem) {
    return Response.json({ error: "Item not found on ForTem" }, { status: 404 })
  }

  // 5. status 확인 (REDEEMED만 Import 가능)
  if (fortemItem.status !== "REDEEMED") {
    return Response.json(
      { error: "Item is not available for import" },
      { status: 400 }
    )
  }

  // 6. attributes에서 type/itemId 추출
  const typeAttr = fortemItem.attributes.find((a) => a.name === "type")
  const itemType = typeAttr?.value // "character" 또는 "weapon"
  if (!itemType || !["character", "weapon"].includes(itemType)) {
    return Response.json({ error: "Invalid item type" }, { status: 400 })
  }

  const itemLabel = itemType === "character" ? "Character" : "Weapon"
  const itemAttr = fortemItem.attributes.find((a) => a.name === itemLabel)
  const itemId = itemAttr?.value
  if (!itemId) {
    return Response.json({ error: "Invalid item data" }, { status: 400 })
  }

  // 7. Admin 클라이언트로 마스터 테이블 존재 확인
  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  const masterTable = itemType === "character" ? "characters" : "weapons"
  const { data: masterItem } = await admin
    .from(masterTable)
    .select("id")
    .eq("id", itemId)
    .single()
  if (!masterItem) {
    return Response.json({ error: "Item does not exist in game" }, { status: 400 })
  }

  // 8. plpgsql 함수 호출 (원자적 INSERT + DELETE)
  const { data: rpcResult, error: rpcError } = await admin.rpc(
    "import_inventory_item",
    {
      p_user_id: user.id,
      p_item_type: itemType,
      p_item_id: itemId,
      p_redeem_code: formattedCode,
      p_network: FORTEM_NETWORK,
    }
  )

  if (rpcError) {
    console.error("Import RPC failed:", rpcError)
    return Response.json({ error: "Import failed" }, { status: 500 })
  }

  if (rpcResult?.error) {
    return Response.json({ error: rpcResult.error }, { status: 400 })
  }

  return Response.json({
    success: true,
    itemType,
    itemId,
  })
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `pnpm --filter web typecheck`
- [ ] 린트 통과: `pnpm --filter web lint`

#### 수동 검증:
- [ ] 유효한 Redeem Code로 Import 시 200 응답 + 인벤토리에 아이템 추가됨
- [ ] 무효한 Redeem Code로 Import 시 404 응답
- [ ] 이미 Import된 Redeem Code로 재시도 시 400 응답 (중복 방지)
- [ ] status가 REDEEMED가 아닌 아이템 Import 시 400 응답

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 3단계: UI 구현 (input-otp + StoreButton 인라인 Import)

### 개요
`input-otp` 패키지 설치, shadcn InputOTP 컴포넌트 추가, StoreButton 내 Drawer에서 Import UI 인라인 표시

### 필요한 변경:

#### 1. input-otp 패키지 설치
**명령어**: `pnpm --filter web add input-otp`

#### 2. shadcn InputOTP 컴포넌트 생성
**파일**: `apps/web/src/components/ui/shadcn/input-otp.tsx` (신규)

shadcn/ui의 input-otp 컴포넌트를 프로젝트 스타일에 맞게 추가한다. 기존 `button.tsx`, `dialog.tsx`, `drawer.tsx`와 동일한 패턴을 따른다.

```typescript
"use client"

import * as React from "react"
import { OTPInput, OTPInputContext } from "input-otp"
import { MinusIcon } from "lucide-react"

import { cn } from "@/lib/utils"

function InputOTP({
  className,
  containerClassName,
  ...props
}: React.ComponentProps<typeof OTPInput> & {
  containerClassName?: string
}) {
  return (
    <OTPInput
      data-slot="input-otp"
      containerClassName={cn(
        "flex items-center gap-2 has-disabled:opacity-50",
        containerClassName
      )}
      className={cn("disabled:cursor-not-allowed", className)}
      {...props}
    />
  )
}

function InputOTPGroup({
  className,
  ...props
}: React.ComponentProps<"div">) {
  return (
    <div
      data-slot="input-otp-group"
      className={cn("flex items-center", className)}
      {...props}
    />
  )
}

function InputOTPSlot({
  index,
  className,
  ...props
}: React.ComponentProps<"div"> & { index: number }) {
  const inputOTPContext = React.useContext(OTPInputContext)
  const { char, hasFakeCaret, isActive } = inputOTPContext.slots[index]

  return (
    <div
      data-slot="input-otp-slot"
      data-active={isActive}
      className={cn(
        "border-input relative flex h-10 w-10 items-center justify-center border-y border-r text-sm shadow-xs transition-all outline-none first:rounded-l-md first:border-l last:rounded-r-md",
        isActive && "ring-ring z-10 ring-1",
        className
      )}
      {...props}
    >
      {char}
      {hasFakeCaret && (
        <div className="pointer-events-none absolute inset-0 flex items-center justify-center">
          <div className="animate-caret-blink bg-foreground h-4 w-px duration-1000" />
        </div>
      )}
    </div>
  )
}

function InputOTPSeparator({
  ...props
}: React.ComponentProps<"div">) {
  return (
    <div data-slot="input-otp-separator" role="separator" {...props}>
      <MinusIcon />
    </div>
  )
}

export { InputOTP, InputOTPGroup, InputOTPSlot, InputOTPSeparator }
```

#### 3. StoreButton Import UI 추가
**파일**: `apps/web/src/components/lobby/StoreButton.tsx` (수정)

**변경 요약**:
- `storeView` 상태 추가: `'browse' | 'import'` (기본값 `'browse'`)
- Import 버튼 클릭 시 `storeView`를 `'import'`로 전환
- `'import'` 뷰: OTP 입력 + Confirm/Back 버튼 렌더링
- Import 성공 시 Drawer 닫기 + 부모에게 알림
- `onImportComplete` 콜백 prop 추가
- Drawer `onOpenChange`에서 뷰 리셋: `setStoreView('browse')`

**구체적 변경 사항**:

a. Props 인터페이스에 `onImportComplete` 추가:
```typescript
interface StoreButtonProps {
  // ... 기존 props
  onImportComplete: () => void
}
```

b. 상태 추가:
```typescript
const [storeView, setStoreView] = useState<"browse" | "import">("browse")
const [redeemCode, setRedeemCode] = useState("")
const [isImporting, setIsImporting] = useState(false)
```

c. Drawer `onOpenChange`에서 리셋:
```typescript
const handleDrawerOpenChange = (open: boolean) => {
  setShowStoreDrawer(open)
  if (!open) {
    setFortemUser(null)
    setSelectedExportItem(null)
    setStoreView("browse")
    setRedeemCode("")
  }
}
```

d. `handleImport` 함수:
```typescript
const handleImport = async () => {
  if (redeemCode.length !== 12) return

  setIsImporting(true)
  try {
    const res = await fetch("/api/store/import", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ redeemCode }),
    })

    if (!res.ok) {
      const data = await res.json()
      toast.error(data.error ?? "Import failed")
      return
    }

    const data = await res.json()
    toast.success(`${data.itemType === "character" ? "Character" : "Weapon"} imported successfully`)

    setShowStoreDrawer(false)
    setStoreView("browse")
    setRedeemCode("")
    onImportComplete()
  } catch {
    toast.error("Import failed")
  } finally {
    setIsImporting(false)
  }
}
```

e. Drawer 콘텐츠 영역에서 `storeView` 기반 조건부 렌더링:
- `storeView === 'browse'`: 기존 Export 아이템 목록 (현재 코드)
- `storeView === 'import'`: OTP 입력 UI

```tsx
{storeView === "import" ? (
  <div className="flex flex-col items-center gap-4 py-4">
    <p className="text-sm text-muted-foreground">
      Enter your 12-digit redeem code
    </p>
    <InputOTP
      maxLength={12}
      value={redeemCode}
      onChange={setRedeemCode}
    >
      <InputOTPGroup>
        <InputOTPSlot index={0} />
        <InputOTPSlot index={1} />
        <InputOTPSlot index={2} />
        <InputOTPSlot index={3} />
      </InputOTPGroup>
      <InputOTPSeparator />
      <InputOTPGroup>
        <InputOTPSlot index={4} />
        <InputOTPSlot index={5} />
        <InputOTPSlot index={6} />
        <InputOTPSlot index={7} />
      </InputOTPGroup>
      <InputOTPSeparator />
      <InputOTPGroup>
        <InputOTPSlot index={8} />
        <InputOTPSlot index={9} />
        <InputOTPSlot index={10} />
        <InputOTPSlot index={11} />
      </InputOTPGroup>
    </InputOTP>
  </div>
) : (
  /* 기존 Export 아이템 목록 */
)}
```

f. Footer 영역 조건부 렌더링:
```tsx
{storeView === "import" ? (
  <DrawerFooter>
    <Button
      onClick={handleImport}
      disabled={redeemCode.length !== 12 || isImporting}
    >
      {isImporting ? <Loader2 className="mr-2 size-4 animate-spin" /> : null}
      Confirm Import
    </Button>
    <Button
      variant="outline"
      onClick={() => {
        setStoreView("browse")
        setRedeemCode("")
      }}
    >
      Back
    </Button>
  </DrawerFooter>
) : (
  /* 기존 Export/Import/Visit Store 버튼 Footer */
)}
```

g. 기존 Import 버튼의 onClick 변경:
```typescript
// 기존: onClick={() => console.info("handle import")}
// 변경:
onClick={() => setStoreView("import")}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `input-otp` 패키지 설치 확인: `pnpm --filter web list input-otp`
- [ ] TypeScript 타입 체크 통과: `pnpm --filter web typecheck`
- [ ] 린트 통과: `pnpm --filter web lint`
- [ ] 빌드 성공: `pnpm --filter web build`

#### 수동 검증:
- [ ] Store Drawer에서 Import 버튼 클릭 시 OTP 입력 UI로 전환
- [ ] Back 버튼 클릭 시 Export 아이템 목록으로 복귀
- [ ] 12자리 입력 완료 시 Confirm Import 버튼 활성화
- [ ] 붙여넣기 시 하이픈 자동 제거되어 12자리만 입력됨
- [ ] Import 성공 시 toast + Drawer 닫힘
- [ ] Import 실패 시 에러 toast 표시

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## 4단계: LobbyContent 연결

### 개요
`onImportComplete` 콜백을 LobbyContent에서 StoreButton으로 전달하여 Import 후 데이터 갱신

### 필요한 변경:

#### 1. LobbyContent에 handleImportComplete 추가
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx` (수정)

```typescript
const handleImportComplete = useCallback(() => {
  router.refresh()
}, [router])
```

#### 2. StoreButton에 onImportComplete prop 전달
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx` (수정)

```tsx
<StoreButton
  linkedWallet={linkedWallet}
  onWalletLinked={handleWalletChange}
  allCharacters={allCharacters}
  allWeapons={allWeapons}
  onExportComplete={handleExportComplete}
  onImportComplete={handleImportComplete}  // 추가
  characterInventoryRecords={characterInventoryRecords}
  weaponInventoryRecords={weaponInventoryRecords}
/>
```

### 성공 기준:

#### 자동화된 검증:
- [ ] TypeScript 타입 체크 통과: `pnpm --filter web typecheck`
- [ ] 린트 통과: `pnpm --filter web lint`
- [ ] 빌드 성공: `pnpm --filter web build`

#### 수동 검증:
- [ ] Import 성공 후 로비 화면에서 새 아이템이 즉시 표시됨
- [ ] Import 성공 후 캐릭터/무기 선택기에 새 아이템이 반영됨
- [ ] Export 후 Import한 아이템이 정상적으로 인게임에서 사용 가능

---

## 테스트 전략

### 통합 테스트 (수동):
1. **Export → Import 전체 흐름**: 유저A Export → 유저B Import → 유저B 인벤토리에 아이템 표시
2. **중복 Import 방지**: 동일 Redeem Code로 2회 Import 시도 → 2번째 실패
3. **무효 코드 처리**: 존재하지 않는 코드, REDEEMED가 아닌 코드, 형식 오류 코드
4. **유저 삭제 후 Import**: 유저A Export → 유저A 탈퇴 → 유저B Import 성공
5. **동시 Import**: 같은 Redeem Code로 두 유저가 동시 Import → 하나만 성공 (FOR UPDATE)

### 엣지 케이스:
- 존재하지 않는 아이템 ID가 attributes에 있는 경우 (마스터 테이블 검증으로 차단)
- 네트워크 오류로 ForTem API 실패 시 DB 변경 없음 (API 먼저 호출)
- Import 중 Drawer 닫기 → isImporting 상태로 버튼 비활성화

## 성능 고려 사항

- `SELECT ... FOR UPDATE`가 plpgsql 함수 내에서 실행되므로 트랜잭션 지속 시간을 최소화해야 함
- ForTem API 호출은 plpgsql 함수 외부(API Route)에서 처리하여 DB 잠금 시간 단축
- 부분 인덱스 `idx_*_active`가 `redeem_code IS NULL` 필터를 커버하므로 Import 후 조회 성능 영향 없음

## 마이그레이션 참고 사항

- `user_id` nullable 변경은 기존 데이터에 영향 없음 (기존 레코드는 모두 NOT NULL 값 보유)
- BEFORE DELETE 트리거는 향후 유저 삭제에만 영향
- 기존 RLS 정책 (`user_id = auth.uid()`)은 `user_id = NULL`인 Export 레코드에 대해 일반 유저가 접근 불가 — 의도된 동작
- `import_inventory_item()` 함수는 `SECURITY DEFINER`로 실행되므로 RLS 우회 가능 (admin.rpc로만 호출)

## 참조

- 연구 문서: `thoughts/arta1069/research/2026-02-19-item-import-redeem-system.md`
- Export 구현 계획: `thoughts/arta1069/plans/2026-02-19-store-drawer-item-export-implementation.md`
- 복수 보유 계획: `thoughts/arta1069/plans/2026-02-19-item-multi-ownership-ammo-system.md`
- Export API Route: `apps/web/src/app/api/store/export/route.ts`
- StoreButton: `apps/web/src/components/lobby/StoreButton.tsx`
- LobbyContent: `apps/web/src/components/lobby/LobbyContent.tsx`
- ForTem SDK: `apps/web/src/lib/fortem.ts`
- plpgsql 패턴 참고: `match/complete/route.ts:26-32` + `thoughts/arta1069/plans/2026-02-15-match-stats-playtime-active-user-implementation.md:131-254`
