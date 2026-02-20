---
date: 2026-02-20T00:41:33+0900
researcher: arta1069@gmail.com
git_commit: e536cb1
branch: feat/web/store
repository: fortem
topic: "Navbar Game Product 버튼 - Store 조건부 네비게이션 연구"
tags: [research, codebase, navbar, store, dialog, userType, game-product]
status: complete
last_updated: 2026-02-20
last_updated_by: arta1069
---

# 연구: Navbar Game Product 버튼 - Store 조건부 네비게이션

**날짜**: 2026-02-20T00:41:33+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: e536cb1
**브랜치**: feat/web/store
**리포지토리**: fortem

## 연구 질문

navbar.tsx의 Game Product 서브메뉴 버튼(426-433행)에 대해:
1. `session.userType`이 `TRADER`가 아닐 때만 버튼을 노출
2. 버튼 노출 시 회원의 store 유무에 따라:
   - store 없음: "Store Required" 타이틀의 Dialog 노출
   - store 있음: `/store/game/new`로 이동

이 기능 구현을 위해 관련 코드베이스의 현재 상태를 조사함.

## 요약

- `UserType` enum에는 `TRADER`, `DEVELOPER`, `ADMIN` 3가지 값이 존재하며, Session 타입에 `userType` 필드가 포함됨
- Session(JWT)에는 store 정보가 포함되지 않아, store 유무 확인을 위해 별도 API 호출 필요
- 백엔드에 `GET /api/v1/stores/my` 엔드포인트가 존재하며, `MyStoreResponseDto`에 `hasStore: boolean` 필드가 포함됨
- 프론트엔드에는 store 관련 서비스/훅/페이지가 아직 존재하지 않음
- `/store/game/new` 경로도 아직 존재하지 않음
- Dialog 컴포넌트는 Radix UI 기반으로 잘 구축되어 있으며, 다양한 사용 패턴이 존재함
- 네비게이션 인증 가드 패턴으로 `isAuthRequired` + `LoginOrConnectButton` 조합이 사용됨

## 상세 발견 사항

### 1. UserType과 Session 타입

`UserType` enum은 3가지 값을 가짐:
- `TRADER` - 일반 거래자
- `DEVELOPER` - 개발자 (스토어 생성 가능)
- `ADMIN` - 관리자

Session 타입에 `userType: UserType` 필드가 포함됨 (`apps/web/services/auth/types.ts:112`).

프론트엔드 `useAuth()` 훅은 `useSession()`을 래핑하여 `{ session, status, update }`를 반환함 (`apps/web/hooks/useAuth.tsx:6-31`).

### 2. Session에 Store 정보 부재

JWT 페이로드(`packages/shared/src/types/auth.ts:15`)에는 `id`, `providerAccountId`, `provider`, `userType`, `walletAddress` 필드만 포함됨. store 관련 필드는 없음.

프론트엔드 Session 타입(`apps/web/services/auth/types.ts:105`)도 `accessToken`, `provider`, `address`, `nickname`, `profileImage`, `userType` 등만 포함하고 store 관련 필드 없음.

### 3. 백엔드 Store API - "내 스토어" 조회

`StoreController`에 `GET /api/v1/stores/my` 엔드포인트가 존재함 (`apps/api/src/feature/store/store.controller.ts:46`). `AuthGuard` 보호.

`StoreService.getMyStore(userId)`는 `storeRepository.findByUserId(userId)`를 호출하여 조회하고, `MyStoreResponseDto`를 반환함:
- `store: StoreResponseDto | null`
- `hasStore: boolean`

Store 엔티티(`packages/core/src/entities/store/store.entity.ts:14`)에서 `userId`는 unique 인덱스로, 유저 1명당 스토어 최대 1개.

### 4. 프론트엔드 Store 관련 코드 부재

현재 프론트엔드에 store 관련 코드가 존재하지 않음:
- `apps/web/modules/` 에 store 모듈 없음
- `apps/web/hooks/` 에 store 관련 훅 없음
- `apps/web/app/[locale]/store/` 라우트 없음
- `/store/game/new` 경로 없음

기존 모듈 패턴 예시: `collection.module.ts`, `item.module.ts` 등에서 API 호출 함수를 정의하고 컴포넌트에서 `useMutation`/`useQuery`로 호출하는 패턴.

### 5. 현재 Navbar의 Game Product 항목

`navbar.tsx:426-433`에서 Game Product 항목의 현재 상태:

```typescript
{
  label: "Game Product",
  href: "#",              // placeholder - 비활성 상태
  icon: <Grid2X2Plus size={20} className="shrink-0 text-foreground" />,
  isAuthRequired: status === "unauthenticated",
}
```

- `href: "#"`로 클릭해도 이동하지 않는 placeholder 상태
- `isAuthRequired`로 미인증 사용자는 `LoginOrConnectButton`으로 교체됨
- `userType` 기반 필터링이나 store 유무 확인 로직은 없음

### 6. 네비게이션 인증 가드 패턴

`SidebarMenuButton` 컴포넌트(`navbar.tsx:572-780`)에서:
- 최상위 링크: `link.isAuthRequired`가 `true`이면 `<LoginOrConnectButton>`으로 교체 (`navbar.tsx:731-753`)
- 서브아이템: `subItem.isAuthRequired`가 `true`이면 `<LoginOrConnectButton>`으로 교체 (`navbar.tsx:663-689`)

### 7. Dialog 컴포넌트 패턴

Dialog 컴포넌트는 `apps/web/components/ui/dialog.tsx`에 Radix UI 기반으로 정의됨.

**커스텀 props:**
- `showCloseButton?: boolean` (기본값 `true`)
- `preventOutsideClick?: boolean` (기본값 `false`)

**주요 사용 패턴:**

| 패턴 | 예시 파일 | open 제어 | Trigger 위치 |
|------|----------|----------|-------------|
| 래퍼, 외부 state | `dna-dialog.tsx`, `receive-dialog.tsx` | props `open/onOpenChange` | 내부 또는 없음 |
| 래퍼, 내부 state | `collection-share-dialog.tsx` | `useState` 내부 | 내부 |
| 인라인, 내부 state | `manage-api-key.tsx` | `useState` 내부 | 인라인 |
| 파생 state | `image-upload.tsx` | 계산된 boolean | 없음 |
| 복합 (Dialog+Drawer) | `responsive-modal.tsx` | Controlled/Uncontrolled | props `trigger` |

**확인(confirm) 성격 Dialog 패턴** (`manage-api-key.tsx:129-178`):
```tsx
<Dialog open={dialogOpen} onOpenChange={setDialogOpen}>
  <DialogTrigger asChild>
    <Button>트리거 버튼</Button>
  </DialogTrigger>
  <DialogContent>
    <DialogHeader>
      <DialogTitle>타이틀</DialogTitle>
      <DialogDescription>설명 텍스트</DialogDescription>
    </DialogHeader>
    <div className="flex gap-2 justify-end">
      <DialogClose asChild>
        <Button variant="outline">취소</Button>
      </DialogClose>
      <Button onClick={onAction}>확인</Button>
    </div>
  </DialogContent>
</Dialog>
```

- `AlertDialog` 컴포넌트는 코드베이스에 존재하지 않음
- description이 없을 때도 `<DialogDescription />`을 빈 태그로 유지하는 사례 있음 (접근성 확보)

### 8. 유사 패턴 참고 - game-collections/new

`/game-collections/new` 페이지 구현 패턴:

1. **페이지**: `apps/web/app/[locale]/game-collections/new/page.tsx` - Server Component, `requireAuth()` 호출
2. **섹션**: `apps/web/components/sections/collection-form-section.tsx` - Client Component, `mode: "create" | "edit"` 분기
3. **폼**: `apps/web/components/forms/collection-form.tsx` - Client Component, prepare → 서명 → execute 2단계 트랜잭션

### 9. 백엔드 Store API 엔드포인트 (참고)

Store 생성/관리 엔드포인트:
- `GET /api/v1/stores/my` - 내 스토어 조회 (AuthGuard)
- `POST /api/v1/stores/create/prepare` - 스토어 생성 준비 (AuthGuard + `@AllowedUserTypes(UserType.DEVELOPER)`)
- `POST /api/v1/stores/create/execute` - 스토어 생성 실행

Game 엔드포인트:
- `POST /api/v1/stores/games/create/prepare` - 게임 생성 준비 (AuthGuard + DEVELOPER)
- `POST /api/v1/stores/games/create/execute` - 게임 생성 실행

Product 엔드포인트:
- `POST /api/v1/stores/products/create/prepare` - 상품 생성 준비 (AuthGuard + DEVELOPER)
- `POST /api/v1/stores/products/create/execute` - 상품 생성 실행

## 코드 참조

- `apps/web/components/ui/navbar.tsx:426-433` - Game Product 서브메뉴 항목 (현재 href="#")
- `apps/web/services/auth/types.ts:105-125` - Session 타입 정의 (userType 포함, store 없음)
- `packages/shared/src/enums/user.enum.ts:1-5` - UserType enum (TRADER, DEVELOPER, ADMIN)
- `apps/web/hooks/useAuth.tsx:6-31` - useAuth 훅
- `apps/web/components/ui/dialog.tsx:49-90` - DialogContent 커스텀 props
- `apps/web/components/forms/manage-api-key.tsx:129-178` - 확인 Dialog 인라인 패턴
- `apps/web/components/login-or-connect-button.tsx:32-34` - LoginOrConnectButton props
- `apps/web/components/ui/navbar.tsx:572-780` - SidebarMenuButton (인증 가드 패턴)
- `packages/core/src/entities/store/store.entity.ts:14` - Store 엔티티
- `packages/core/src/dto/store/store.dto.ts` - MyStoreResponseDto (hasStore 필드)

## 아키텍처 문서화

### 네비게이션 가드 계층 구조

현재 navbar에는 단일 계층의 인증 가드만 존재:
1. **인증 가드**: `isAuthRequired: status === "unauthenticated"` → `LoginOrConnectButton`으로 교체

요청된 기능 구현 시 추가 필요한 가드 계층:
2. **UserType 가드**: `session.userType !== 'TRADER'` → 버튼 노출/숨김
3. **Store 유무 가드**: store 있으면 이동, 없으면 Dialog 노출

### Store 정보 확인 방식

Session(JWT)에 store 정보가 없으므로, `GET /api/v1/stores/my` API를 호출하여 `hasStore` 여부를 확인해야 함.

## 미해결 질문

1. Store 유무 확인 API 호출 시점: 페이지 로드 시 미리 확인? vs 버튼 클릭 시 확인?
2. Session 타입에 store 관련 필드를 추가할지, 별도 훅으로 관리할지
3. `/store/game/new` 페이지의 구체적인 UI/UX 요구사항
4. "Store Required" Dialog에서 스토어 생성 페이지로의 이동 링크 포함 여부
