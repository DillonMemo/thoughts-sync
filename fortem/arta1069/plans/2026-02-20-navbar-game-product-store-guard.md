# Navbar Game Product Store Guard 구현 계획

## 개요

Navbar의 "Create > Game Product" 서브메뉴 버튼에 2단계 가드를 적용한다:
1. **UserType 가드**: `TRADER` 사용자에게는 버튼을 노출하지 않음 (filter로 제외)
2. **Store 유무 가드**: 버튼 클릭 시 `GET /api/v1/stores/my` API를 호출하여:
   - store 있음 → `/store/game/new`로 이동
   - store 없음 → "Store Required" Dialog 노출

## 현재 상태 분석

### 존재하는 것:
- Game Product 서브아이템: `navbar.tsx:426-433`에 `href: "#"`로 placeholder 상태
- `isAuthRequired` 인증 가드 패턴: 미인증 시 `LoginOrConnectButton`으로 교체
- `GET /api/v1/stores/my` 백엔드 API: `MyStoreResponseDto { hasStore: boolean, store: StoreResponseDto | null }` 반환
- KioskModule 패턴: `kiosk.module.ts`에서 동일한 구조의 API 호출 모듈 참고 가능
- Dialog 컴포넌트: Radix UI 기반, `DialogFooter` 포함

### 존재하지 않는 것:
- 프론트엔드 Store 관련 모듈/서비스/훅 (전무)
- `/store/game/new` 페이지 (추후 개발)
- Store 생성 페이지 (추후 개발)
- `subItems`에 대한 filter/hidden 로직

### 핵심 발견:
- `Link` 타입(`navbar.tsx:37-42`)에는 `href`, `label`, `icon`, `isAuthRequired`만 있음. `onClick` 커스텀 핸들러 없음
- `links` useMemo(`navbar.tsx:413-487`)에서 `session?.address`와 `status`가 의존성
- 서브아이템 렌더링(`navbar.tsx:663-721`)에서 `subItem.isAuthRequired` 분기만 존재하며, 필터링/조건부 제외 로직 없음
- `SidebarMenuButton`에서 `{...props}`로 전달되는 `onClick`은 `() => setOpen(false)` (사이드바 닫기)
- Session 타입(`types.ts:112`)에 `userType: UserType` 포함, JWT에는 store 정보 없음

## 원하는 최종 상태

- `TRADER` 사용자: "Create" 메뉴 펼쳤을 때 "Game Product" 항목이 보이지 않음
- `DEVELOPER`/`ADMIN` 사용자 (미인증): `LoginOrConnectButton` 표시 (기존 동작 유지)
- `DEVELOPER`/`ADMIN` 사용자 (인증됨, store 있음): 클릭 시 `/store/game/new`로 이동
- `DEVELOPER`/`ADMIN` 사용자 (인증됨, store 없음): 클릭 시 "Store Required" Dialog 표시
  - Cancel: Dialog 닫기
  - Confirm: 스토어 생성 경로로 이동 (경로는 상수로 정의, 추후 페이지 개발)
- API 호출 중: 버튼에 로딩 스피너 표시

### 검증 방법:
- TRADER로 로그인 후 Create 메뉴에 Game Product가 없는지 확인
- DEVELOPER로 로그인 후 Game Product 클릭 → store 유무에 따른 분기 확인
- 미인증 상태에서 LoginOrConnectButton 동작 확인

## 하지 않을 것

- `/store/game/new` 페이지 구현
- 스토어 생성 페이지 구현
- Store 데이터 캐싱 (React Query useQuery 등) - 버튼 클릭 시 매번 호출
- 다른 서브아이템에 대한 UserType 가드 확장
- `Link` 타입에 범용 `hidden` 속성 추가

## 구현 접근 방식

최소한의 변경으로 2개의 파일만 생성/수정한다:
1. `store.module.ts` 신규 생성 (kiosk.module.ts 패턴)
2. `navbar.tsx` 수정 (Game Product 항목의 filter + onClick + Dialog)

서브아이템 렌더링 구조를 최소한으로 수정하기 위해, Game Product 항목에만 특화된 로직을 `SidebarMenu` 컴포넌트 내부에 인라인으로 구현한다. 범용 가드 시스템을 만들지 않는다.

---

## 1단계: `store.module.ts` 생성

### 개요
`GET /api/v1/stores/my` API를 호출하는 `StoreModule`을 `kiosk.module.ts` 패턴에 따라 생성한다.

### 필요한 변경:

#### 1. `apps/web/modules/store.module.ts` (신규 생성)

**파일**: `apps/web/modules/store.module.ts`
**변경**: 신규 파일 생성

```typescript
import NetworkModule, { API_PATH } from "@/lib/network-config"
import type { Session } from "@/services/auth/types"
import type { MyStoreResponseDto } from "@fortem/core"

export const StoreKeys = {
  GET_MY_STORE: "getMyStore",
}

export type GetMyStoreRequest = {
  token: Session["accessToken"]
}
export type GetMyStoreResponse = MyStoreResponseDto

class StoreModule extends NetworkModule {
  constructor(url: string) {
    super(url)
  }

  getMyStore(props: GetMyStoreRequest) {
    return this.http.get<GetMyStoreResponse>(`/api/v1/stores/my`, {
      headers: {
        Authorization: `Bearer ${props.token}`,
      },
    })
  }
}

const storeModule = new StoreModule(API_PATH)
export default storeModule
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `yarn workspace @fortem/web tsc --noEmit` (또는 IDE에서 에러 없음)
- [ ] `MyStoreResponseDto` 임포트가 `@fortem/core`에서 정상 resolve

#### 수동 검증:
- [ ] 파일 구조가 `kiosk.module.ts`와 동일한 패턴임을 확인

---

## 2단계: `navbar.tsx` 수정

### 개요
Game Product 서브아이템에 UserType 필터와 Store 유무 확인 + Dialog 로직을 적용한다.

### 필요한 변경:

#### 1. 임포트 추가
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: 상단에 필요한 임포트 추가

```typescript
import { UserType } from "@fortem/shared"
import storeModule from "@/modules/store.module"
import { Loader2 } from "lucide-react"
import {
  Dialog,
  DialogClose,
  DialogContent,
  DialogDescription,
  DialogFooter,
  DialogHeader,
  DialogTitle,
} from "@/components/ui/dialog"
import { Button } from "@/components/ui/button"
```

#### 2. 경로 상수 정의
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: 컴포넌트 외부 또는 `SidebarMenu` 내부에 상수 정의

```typescript
const STORE_GAME_NEW_PATH = "/store/game/new"
const STORE_CREATE_PATH = "/store/create"
```

#### 3. `SidebarMenu` 컴포넌트에 상태 및 핸들러 추가
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: `SidebarMenu` 컴포넌트 내부, `links` useMemo 근처에 추가

```typescript
// Store Required Dialog 상태
const [storeDialogOpen, setStoreDialogOpen] = React.useState(false)
// Game Product 버튼 로딩 상태
const [gameProductLoading, setGameProductLoading] = React.useState(false)

const handleGameProductClick = React.useCallback(
  async (e: React.MouseEvent) => {
    e.preventDefault()
    if (!session?.accessToken || gameProductLoading) return

    setGameProductLoading(true)
    try {
      const res = await storeModule.getMyStore({ token: session.accessToken })
      if (res.hasStore) {
        router.push(STORE_GAME_NEW_PATH)
        setOpen(false)
      } else {
        setStoreDialogOpen(true)
      }
    } catch (error) {
      console.error("Failed to check store:", error)
    } finally {
      setGameProductLoading(false)
    }
  },
  [session?.accessToken, gameProductLoading, router, setOpen]
)
```

> **참고**: `SidebarMenu`에서 `router` 사용이 필요함. 기존에 `useRouter`가 없다면 추가. `useRouter`는 `next/navigation`에서 import.

#### 4. `links` useMemo에서 TRADER 필터링
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: Game Product 서브아이템의 조건부 포함

```typescript
// Game Product 접근 불가 UserType 목록
const GAME_PRODUCT_EXCLUDED_USER_TYPES: UserType[] = [UserType.TRADER]
```

```typescript
const links = React.useMemo(
  (): Links[] => [
    {
      label: "Create",
      href: "#",
      icon: <PlusIcon size={20} className="shrink-0 text-foreground" />,
      subItems: [
        {
          label: "Game Item",
          href: "/game-collections/new",
          icon: <LayersPlus size={20} className="shrink-0 text-foreground" />,
          isAuthRequired: status === "unauthenticated",
        },
        // 제외 대상 UserType이 아닐 때만 Game Product 포함
        ...(!GAME_PRODUCT_EXCLUDED_USER_TYPES.includes(session?.userType!)
          ? [
              {
                label: "Game Product",
                href: STORE_GAME_NEW_PATH,
                icon: (
                  <Grid2X2Plus
                    size={20}
                    className="shrink-0 text-foreground"
                  />
                ),
                isAuthRequired: status === "unauthenticated",
              },
            ]
          : []),
      ],
    },
    // ... 나머지 링크 동일
  ],
  [session?.address, session?.userType, status]
)
```

> **주의**: `useMemo` 의존성 배열에 `session?.userType` 추가 필요.

#### 5. 서브아이템 렌더링에서 Game Product 특수 처리
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: `SidebarMenuButton` 컴포넌트의 서브아이템 `.map()` 내부 (`navbar.tsx:663-721`)

`subItem.isAuthRequired === false`이고 `subItem.label === "Game Product"`인 경우를 특별히 처리해야 함.

기존 2분기(`isAuthRequired ? LoginOrConnectButton : Link`) 구조에서 3분기로 확장:

```typescript
{link.subItems.map((subItem, idx) =>
  subItem.isAuthRequired ? (
    <LoginOrConnectButton key={idx} ... /> // 기존 동일
  ) : subItem.label === "Game Product" ? (
    // Game Product 전용: 클릭 시 store 확인
    <button
      key={idx}
      onClick={onGameProductClick}
      disabled={isGameProductLoading}
      className={cn(
        className,
        "hover:bg-secondary/70",
        "text-foreground",
        "w-full"
      )}
      role="menuitem"
      tabIndex={0}
    >
      {isGameProductLoading ? (
        <Loader2 className="size-5 shrink-0 animate-spin" />
      ) : (
        subItem.icon && (
          <span className="shrink-0" aria-hidden="true">
            {subItem.icon}
          </span>
        )
      )}
      <motion.span
        animate={textAnimationProps}
        transition={{ duration: 0.15, ease: "easeInOut" }}
        className={cn(
          "text-sm/none will-change-transform",
          "transition duration-150 whitespace-pre inline-block !p-0 !m-0",
          "group-hover/sidebar:translate-x-1"
        )}
      >
        {subItem.label}
      </motion.span>
    </button>
  ) : (
    <Link key={idx} ... /> // 기존 동일
  )
)}
```

> **구현 시 고려사항**: `SidebarMenuButton`은 `SidebarMenu`의 자식 컴포넌트이므로, `onGameProductClick`과 `isGameProductLoading`을 props로 전달해야 함. `SidebarMenuButton`의 props 타입 확장이 필요.

#### 6. `SidebarMenuButton` props 확장
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: `SidebarMenuButton` 컴포넌트의 props에 Game Product 관련 핸들러 추가

```typescript
export function SidebarMenuButton({
  link,
  className,
  locale,
  textAnimationProps,
  onGameProductClick,      // 추가
  isGameProductLoading,     // 추가
  ...props
}: React.ComponentProps<"a"> & {
  link: Links
  locale: string
  textAnimationProps: Pick<React.CSSProperties, "display" | "opacity">
  onGameProductClick?: (e: React.MouseEvent) => void       // 추가
  isGameProductLoading?: boolean                            // 추가
})
```

`SidebarMenu`에서 호출 시:
```typescript
<SidebarMenuButton
  key={`nav-${idx}`}
  link={link}
  locale={locale}
  onClick={() => setOpen(false)}
  className={cn(linkClasses)}
  textAnimationProps={textAnimationProps}
  onGameProductClick={handleGameProductClick}    // 추가
  isGameProductLoading={gameProductLoading}       // 추가
/>
```

#### 7. "Store Required" Dialog 추가
**파일**: `apps/web/components/ui/navbar.tsx`
**변경**: `SidebarMenu` 컴포넌트 반환 JSX 하단에 Dialog 추가

```typescript
return (
  <div className="flex flex-col items-start overflow-hidden">
    {/* ... 기존 nav 내용 ... */}

    {/* Store Required Dialog */}
    <Dialog open={storeDialogOpen} onOpenChange={setStoreDialogOpen}>
      <DialogContent>
        <DialogHeader>
          <DialogTitle>Store Required</DialogTitle>
          <DialogDescription>
            You need to create a store first before registering a game product.
          </DialogDescription>
        </DialogHeader>
        <DialogFooter>
          <DialogClose asChild>
            <Button variant="outline">Cancel</Button>
          </DialogClose>
          <Button
            onClick={() => {
              setStoreDialogOpen(false)
              router.push(STORE_CREATE_PATH)
            }}
          >
            Create Store
          </Button>
        </DialogFooter>
      </DialogContent>
    </Dialog>
  </div>
)
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `yarn workspace @fortem/web tsc --noEmit`
- [ ] 린트 통과: `yarn workspace @fortem/web lint`
- [ ] 빌드 성공: `yarn build:web`

#### 수동 검증:
- [ ] 미인증 상태: Create > Game Product에 LoginOrConnectButton 표시
- [ ] TRADER로 로그인: Create 메뉴에 Game Product가 보이지 않음
- [ ] DEVELOPER로 로그인 (store 없음): Game Product 클릭 → 로딩 스피너 → "Store Required" Dialog → Cancel로 닫기 가능, Confirm 클릭 시 `/store/create`로 이동
- [ ] DEVELOPER로 로그인 (store 있음): Game Product 클릭 → 로딩 스피너 → `/store/game/new`로 이동
- [ ] API 오류 시: 로딩이 멈추고 콘솔에 에러 로그 출력, UI 깨지지 않음

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 테스트 전략

### 수동 테스트 단계:

1. **미인증 상태 테스트**
   - 로그아웃 상태에서 사이드바 열기
   - Create 메뉴 클릭하여 펼치기
   - Game Product에 LoginOrConnectButton이 표시되는지 확인

2. **TRADER 사용자 테스트**
   - TRADER 계정으로 로그인
   - Create 메뉴 클릭하여 펼치기
   - Game Product 항목이 보이지 않고, Game Item만 보이는지 확인

3. **DEVELOPER 사용자 - Store 없음 테스트**
   - DEVELOPER 계정(store 미생성)으로 로그인
   - Game Product 클릭
   - 로딩 스피너가 버튼 아이콘 위치에 표시되는지 확인
   - "Store Required" Dialog가 나타나는지 확인
   - Dialog의 Cancel 클릭 → Dialog 닫힘 확인
   - 다시 클릭 → Dialog의 "Create Store" 클릭 → `/store/create` 이동 확인

4. **DEVELOPER 사용자 - Store 있음 테스트**
   - DEVELOPER 계정(store 생성 완료)으로 로그인
   - Game Product 클릭
   - `/store/game/new`로 이동하는지 확인

5. **에러 핸들링 테스트**
   - 네트워크 탭에서 `/api/v1/stores/my` 요청 차단
   - Game Product 클릭
   - 로딩이 끝나고 에러가 콘솔에 로그되는지 확인
   - UI가 정상 상태로 돌아오는지 확인

## 성능 고려 사항

- Store 유무 확인은 클릭 시마다 API를 호출함 (캐싱 없음)
- 일반적인 사용 빈도에서는 문제 없음
- 추후 Store 관련 기능이 확장되면 React Query로 캐싱 전환 고려

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-20-navbar-game-product-store-guard.md`
- KioskModule 패턴: `apps/web/modules/kiosk.module.ts`
- 현재 서브아이템 렌더링: `apps/web/components/ui/navbar.tsx:663-721`
- Store DTO: `packages/core/src/dto/store/store.dto.ts:118-121`
- Dialog 컴포넌트: `apps/web/components/ui/dialog.tsx`
- UserType enum: `packages/shared/src/enums/user.enum.ts:1-5`
