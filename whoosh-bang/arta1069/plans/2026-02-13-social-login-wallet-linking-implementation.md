# 소셜 로그인(Google OAuth) + 선택적 Sui 지갑 연동 구현 계획

## 개요

현재 인증 없이 동작하는 Worms 스타일 포병 게임에 **Supabase 네이티브 Google OAuth**를 주 인증으로, **@mysten/dapp-kit**을 통한 **선택적 Sui 지갑 연동**을 부가 기능으로 구현한다. `/`를 로그인 페이지로 전환하고, 게임 로비를 `/lobby`에 배치한다.

## 현재 상태 분석

### 코드베이스
- **프레임워크**: Next.js 16.1.6 + React 19 + Phaser 3.90 (pnpm 모노레포, Turbo)
- **현재 라우트**: `/` (GameWrapper → MainMenu 또는 PhaserGame)
- **인증 관련 코드**: 없음 (의존성, 미들웨어, API 라우트, 환경변수 모두 미존재)
- **UI**: shadcn/ui + Tailwind CSS 4 사용 중

### Supabase 프로젝트
- **프로젝트**: Whoosh Bang (`fmwjmsfgxhbzhvtnaqta`), ap-northeast-1
- **Google OAuth**: Provider 설정 완료
- **데이터베이스**: public 스키마에 테이블 0개, 마이그레이션 0개

### 핵심 파일 (현재)
- `apps/web/src/app/layout.tsx` — 루트 레이아웃 (Provider 없음)
- `apps/web/src/app/page.tsx` — 홈 페이지 (GameWrapper dynamic import)
- `apps/web/src/components/game/GameWrapper.tsx` — 게임 상태 관리 (menu/playing/result)
- `apps/web/src/components/game/PhaserGame.tsx` — Phaser 초기화
- `apps/web/src/components/ui/MainMenu.tsx` — 메인 메뉴
- `apps/web/package.json` — 인증 관련 의존성 없음

## 원하는 최종 상태

1. **`/`** (로그인 페이지): Google 로그인 버튼. 이미 인증된 사용자는 `/lobby`로 리디렉션
2. **`/auth/callback`**: OAuth PKCE 코드 교환 후 `/lobby`로 리디렉션
3. **`/lobby`**: 프로필 표시 + PLAY 버튼 + 지갑 연동/해제 + 로그아웃. dapp-kit Provider 적용
4. **`/game`**: 기존 게임 플레이 (인증 필수, 미들웨어로 보호)
5. **미들웨어**: `getClaims()`로 토큰 갱신 + 라우트 보호
6. **DB**: player_profiles, wallets, wallet_nonces 테이블 + RLS + 자동 프로필 생성 트리거
7. **지갑 연동**: Nonce 기반 서명 검증 → wallets 테이블 저장 + app_metadata 업데이트

### 검증 방법
- 비인증 사용자가 `/lobby`, `/game` 접근 시 `/`로 리디렉션
- Google 로그인 → 자동 프로필 생성 → `/lobby` 도달
- 지갑 연동 → wallets 테이블 + app_metadata에 주소 저장
- 지갑 해제 → wallets 테이블 + app_metadata에서 제거
- 타입체크, 린트 통과

## 하지 않을 것

- 멀티플레이어/Realtime 구현 (별도 계획)
- NFT/토큰 트랜잭션 로직 (추후 구현)
- 이메일/비밀번호 인증 (Google OAuth만)
- 다중 지갑 지원 (1 유저 = 1 지갑)
- Custom Access Token Hook (app_metadata 사용)
- dapp-kit v2 마이그레이션 (현재 v1 사용)
- 게임 로직 변경 (인증 레이어만 추가)

## 구현 접근 방식

연구 문서 `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md`의 아키텍처를 그대로 따른다. 6단계로 나누어 각 단계가 독립적으로 검증 가능하도록 구성한다.

---

## 1단계: 기반 설정

### 개요
의존성 설치, 환경변수 설정, Supabase 클라이언트 유틸리티(브라우저/서버) 생성.

### 필요한 변경:

#### 1. 의존성 설치
**파일**: `apps/web/package.json`
**변경**: 인증 및 지갑 관련 패키지 추가

```bash
cd apps/web && pnpm add @supabase/supabase-js @supabase/ssr @mysten/dapp-kit @mysten/sui @tanstack/react-query
```

#### 2. 환경변수 파일 생성
**파일**: `apps/web/.env.local` (새로 생성)
**변경**: Supabase 및 Sui 환경변수 설정

```env
# Supabase (공개)
NEXT_PUBLIC_SUPABASE_URL=https://fmwjmsfgxhbzhvtnaqta.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<anon-key>

# Supabase (서버 전용)
SUPABASE_SERVICE_ROLE_KEY=<service-role-key>

# Sui
NEXT_PUBLIC_SUI_NETWORK=testnet
```

#### 3. 브라우저 Supabase 클라이언트
**파일**: `apps/web/src/lib/supabase/client.ts` (새로 생성)

```typescript
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

#### 4. 서버 Supabase 클라이언트
**파일**: `apps/web/src/lib/supabase/server.ts` (새로 생성)

```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            cookiesToSet.forEach(({ name, value, options }) =>
              cookieStore.set(name, value, options)
            );
          } catch {
            // Server Component에서 호출 시 무시 — middleware가 처리
          }
        },
      },
    }
  );
}
```

> **중요**: `getAll`/`setAll`만 사용. `get`/`set`/`remove` 사용 금지.

### 성공 기준:

#### 자동화된 검증:
- [x] 의존성 설치 성공: `cd apps/web && pnpm install`
- [x] 타입체크 통과: `pnpm typecheck`
- [x] `apps/web/src/lib/supabase/client.ts` 존재
- [x] `apps/web/src/lib/supabase/server.ts` 존재
- [x] `apps/web/.env.local` 존재

#### 수동 검증:
- [ ] `.env.local`에 실제 Supabase URL과 Key가 올바르게 입력됨

**Implementation Note**: 이 단계 완료 후 타입체크 통과를 확인하고 다음 단계로 진행.

---

## 2단계: 데이터베이스 스키마

### 개요
Supabase에 player_profiles, wallets, wallet_nonces 테이블 생성. RLS 정책 설정. 회원가입 시 자동 프로필 생성 트리거 설정.

### 필요한 변경:

#### 1. 마이그레이션: 테이블 생성
**대상**: Supabase 프로젝트 `fmwjmsfgxhbzhvtnaqta`
**방법**: Supabase MCP `apply_migration`

```sql
-- player_profiles: 사용자 프로필 (auth.users 트리거로 자동 생성)
CREATE TABLE player_profiles (
  id UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username TEXT UNIQUE,
  display_name TEXT,
  avatar_url TEXT,
  level INTEGER DEFAULT 1,
  total_experience BIGINT DEFAULT 0,
  total_matches INTEGER DEFAULT 0,
  wins INTEGER DEFAULT 0,
  losses INTEGER DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- wallets: 지갑 연동 (1 유저 = 1 지갑)
CREATE TABLE wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  wallet_address TEXT NOT NULL UNIQUE,
  chain TEXT NOT NULL DEFAULT 'sui',
  verified BOOLEAN DEFAULT FALSE,
  linked_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_wallets_address ON wallets(wallet_address);

-- wallet_nonces: 지갑 서명 검증용 (일회용)
CREATE TABLE wallet_nonces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  wallet_address TEXT NOT NULL,
  nonce TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_nonces_lookup ON wallet_nonces(user_id, wallet_address, nonce);
```

#### 2. 마이그레이션: 자동 프로필 생성 트리거

```sql
-- 회원가입 시 자동 프로필 생성
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER
LANGUAGE plpgsql
SECURITY DEFINER SET search_path = ''
AS $$
BEGIN
  INSERT INTO public.player_profiles (id, display_name, avatar_url, username)
  VALUES (
    NEW.id,
    COALESCE(NEW.raw_user_meta_data ->> 'full_name', NEW.raw_user_meta_data ->> 'name'),
    NEW.raw_user_meta_data ->> 'avatar_url',
    'player_' || substr(NEW.id::text, 1, 8)
  );
  RETURN NEW;
END;
$$;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

#### 3. 마이그레이션: RLS 정책

```sql
-- RLS 활성화
ALTER TABLE player_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE wallets ENABLE ROW LEVEL SECURITY;
ALTER TABLE wallet_nonces ENABLE ROW LEVEL SECURITY;

-- player_profiles: 모든 인증 사용자가 조회 가능 (리더보드 등)
CREATE POLICY "profiles_select_all"
ON player_profiles FOR SELECT TO authenticated
USING (true);

-- player_profiles: 자신만 수정 가능
CREATE POLICY "profiles_update_own"
ON player_profiles FOR UPDATE TO authenticated
USING (id = auth.uid())
WITH CHECK (id = auth.uid());

-- wallets: 자신의 지갑만 조회
CREATE POLICY "wallets_select_own"
ON wallets FOR SELECT TO authenticated
USING (user_id = auth.uid());

-- wallets: 자신의 지갑만 추가
CREATE POLICY "wallets_insert_own"
ON wallets FOR INSERT TO authenticated
WITH CHECK (user_id = auth.uid());

-- wallets: 자신의 지갑만 삭제 (연동 해제)
CREATE POLICY "wallets_delete_own"
ON wallets FOR DELETE TO authenticated
USING (user_id = auth.uid());

-- wallet_nonces: 일반 사용자 접근 차단 (service_role만 접근)
CREATE POLICY "nonces_no_access"
ON wallet_nonces FOR ALL TO authenticated
USING (false);
```

### 성공 기준:

#### 자동화된 검증:
- [x] 마이그레이션 적용 성공 (Supabase MCP apply_migration)
- [x] `player_profiles` 테이블 존재 확인: `SELECT count(*) FROM player_profiles`
- [x] `wallets` 테이블 존재 확인: `SELECT count(*) FROM wallets`
- [x] `wallet_nonces` 테이블 존재 확인: `SELECT count(*) FROM wallet_nonces`
- [x] RLS 활성화 확인: `SELECT tablename, rowsecurity FROM pg_tables WHERE schemaname = 'public'`
- [x] Supabase security advisor 실행하여 경고 없음 확인

#### 수동 검증:
- [ ] Supabase Dashboard에서 테이블과 RLS 정책 확인

**Implementation Note**: 마이그레이션 적용 후 security advisor를 실행하여 보안 이슈가 없는지 확인.

---

## 3단계: Google OAuth 인증 플로우

### 개요
로그인 페이지(`/`), OAuth 콜백(`/auth/callback`), 미들웨어(토큰 갱신 + 라우트 보호), SessionHandler, 로그아웃 기능 구현.

### 필요한 변경:

#### 1. 미들웨어
**파일**: `apps/web/src/middleware.ts` (새로 생성)

```typescript
import { createServerClient } from '@supabase/ssr';
import { type NextRequest, NextResponse } from 'next/server';

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({ request });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) =>
            request.cookies.set(name, value)
          );
          response = NextResponse.next({ request });
          cookiesToSet.forEach(({ name, value, options }) =>
            response.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  // JWT 클레임 로컬 검증 (네트워크 요청 없음)
  const { data: claims, error } = await supabase.auth.getClaims();

  // 보호된 라우트
  const protectedRoutes = ['/lobby', '/game', '/profile', '/settings'];
  const isProtectedRoute = protectedRoutes.some(route =>
    request.nextUrl.pathname.startsWith(route)
  );

  if (isProtectedRoute && (error || !claims?.sub)) {
    return NextResponse.redirect(new URL('/', request.url));
  }

  // 이미 로그인한 사용자가 로그인 페이지(`/`) 접근 시
  if (request.nextUrl.pathname === '/' && claims?.sub) {
    return NextResponse.redirect(new URL('/lobby', request.url));
  }

  return response;
}

export const config = {
  matcher: [
    '/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)',
  ],
};
```

#### 2. OAuth 콜백 라우트
**파일**: `apps/web/src/app/auth/callback/route.ts` (새로 생성)

```typescript
import { createClient } from '@/lib/supabase/server';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const next = searchParams.get('next') ?? '/lobby';

  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);

    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  return NextResponse.redirect(`${origin}/auth/auth-code-error`);
}
```

#### 3. 인증 오류 페이지
**파일**: `apps/web/src/app/auth/auth-code-error/page.tsx` (새로 생성)

```tsx
import Link from 'next/link';

export default function AuthCodeErrorPage() {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h1 className="text-2xl font-bold mb-4">인증 오류</h1>
      <p className="text-muted-foreground mb-6">로그인 처리 중 문제가 발생했습니다.</p>
      <Link href="/" className="text-primary underline">
        다시 로그인하기
      </Link>
    </div>
  );
}
```

#### 4. 로그인 페이지 (기존 `/` 페이지 교체)
**파일**: `apps/web/src/app/page.tsx` (기존 파일 수정)

```tsx
import { createClient } from '@/lib/supabase/server';
import { redirect } from 'next/navigation';
import { GoogleLoginButton } from '@/components/auth/GoogleLoginButton';

export default async function LoginPage() {
  const supabase = await createClient();
  const { data: { user } } = await supabase.auth.getUser();

  if (user) {
    redirect('/lobby');
  }

  return (
    <div className="flex flex-col items-center justify-center min-h-screen bg-linear-to-b from-gray-900 to-gray-700">
      <h1 className="text-5xl font-bold text-white mb-2">Artillery Wars</h1>
      <p className="text-lg text-gray-300 mb-8">Worms Style Artillery Game</p>
      <GoogleLoginButton />
    </div>
  );
}
```

#### 5. Google 로그인 버튼
**파일**: `apps/web/src/components/auth/GoogleLoginButton.tsx` (새로 생성)

```tsx
'use client';

import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/shadcn/button';

export function GoogleLoginButton() {
  const supabase = createClient();

  const handleLogin = async () => {
    await supabase.auth.signInWithOAuth({
      provider: 'google',
      options: {
        redirectTo: `${window.location.origin}/auth/callback`,
      },
    });
  };

  return (
    <Button
      onClick={handleLogin}
      size="lg"
      className="px-8 py-6 text-xl shadow-lg transition-all hover:scale-105"
    >
      Google로 로그인
    </Button>
  );
}
```

#### 6. SessionHandler 컴포넌트
**파일**: `apps/web/src/components/auth/SessionHandler.tsx` (새로 생성)

```tsx
'use client';

import { useEffect } from 'react';
import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';

export function SessionHandler() {
  const router = useRouter();
  const supabase = createClient();

  useEffect(() => {
    const { data: { subscription } } = supabase.auth.onAuthStateChange(
      (event) => {
        if (event === 'SIGNED_OUT') {
          router.push('/');
        }
      }
    );

    return () => subscription.unsubscribe();
  }, [router, supabase]);

  return null;
}
```

#### 7. 루트 레이아웃에 SessionHandler 추가
**파일**: `apps/web/src/app/layout.tsx` (기존 파일 수정)

```tsx
import type { Metadata } from "next"
import "./globals.css"
import { SessionHandler } from "@/components/auth/SessionHandler"

export const metadata: Metadata = {
  title: "Artillery Wars - Worms Style Game",
  description: "A turn-based artillery game built with Next.js and Phaser",
}

export default function RootLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <html lang="ko" className="dark">
      <body className="bg-background text-foreground">
        <SessionHandler />
        {children}
      </body>
    </html>
  )
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] 타입체크 통과: `pnpm typecheck`
- [x] 린트 통과: `pnpm lint`
- [x] `apps/web/src/middleware.ts` 존재
- [x] `apps/web/src/app/auth/callback/route.ts` 존재
- [x] `apps/web/src/components/auth/GoogleLoginButton.tsx` 존재
- [x] `apps/web/src/components/auth/SessionHandler.tsx` 존재

#### 수동 검증:
- [ ] `/` 접근 시 Google 로그인 버튼이 표시됨
- [ ] Google 로그인 클릭 → Google OAuth 화면 → 콜백 → `/lobby`로 리디렉션
- [ ] 로그인 후 `/` 접근 시 자동으로 `/lobby`로 리디렉션
- [ ] `player_profiles` 테이블에 자동으로 프로필이 생성됨

**Implementation Note**: 이 단계 완료 후 실제 Google 로그인 플로우를 수동 테스트하여 정상 동작 확인.

---

## 4단계: 라우트 재구성 (게임 로비)

### 개요
기존 게임을 `/lobby`와 `/game`으로 분리. 로비 페이지에서 프로필 표시, PLAY 버튼, 로그아웃 기능 제공. 게임 플레이는 `/game`에서 처리.

### 필요한 변경:

#### 1. 로비 페이지
**파일**: `apps/web/src/app/lobby/page.tsx` (새로 생성)

```tsx
import { createClient } from '@/lib/supabase/server';
import { redirect } from 'next/navigation';
import { LobbyContent } from '@/components/lobby/LobbyContent';

export default async function LobbyPage() {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (error || !user) {
    redirect('/');
  }

  // 프로필 조회
  const { data: profile } = await supabase
    .from('player_profiles')
    .select('*')
    .eq('id', user.id)
    .single();

  // 연동된 지갑 조회
  const { data: wallet } = await supabase
    .from('wallets')
    .select('wallet_address, chain')
    .eq('user_id', user.id)
    .single();

  return (
    <LobbyContent
      profile={profile}
      wallet={wallet}
      userEmail={user.email ?? ''}
      avatarUrl={user.user_metadata?.avatar_url}
    />
  );
}
```

#### 2. 로비 콘텐츠 컴포넌트
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx` (새로 생성)

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { createClient } from '@/lib/supabase/client';
import { Button } from '@/components/ui/shadcn/button';

interface LobbyContentProps {
  profile: {
    display_name: string | null;
    username: string;
    level: number;
    wins: number;
    losses: number;
    total_matches: number;
  } | null;
  wallet: { wallet_address: string; chain: string } | null;
  userEmail: string;
  avatarUrl?: string;
}

export function LobbyContent({ profile, wallet, userEmail, avatarUrl }: LobbyContentProps) {
  const router = useRouter();
  const supabase = createClient();

  const handleLogout = async () => {
    await supabase.auth.signOut();
    router.push('/');
  };

  const handlePlay = () => {
    router.push('/game');
  };

  return (
    <div className="flex flex-col items-center justify-center min-h-screen bg-linear-to-b from-gray-900 to-gray-700">
      {/* 프로필 영역 */}
      <div className="mb-8 text-center">
        {avatarUrl && (
          <img
            src={avatarUrl}
            alt="Profile"
            className="w-16 h-16 rounded-full mx-auto mb-2"
          />
        )}
        <h2 className="text-2xl font-bold text-white">
          {profile?.display_name ?? userEmail}
        </h2>
        <p className="text-gray-400">Lv.{profile?.level ?? 1}</p>
        {profile && (
          <p className="text-gray-400 text-sm">
            {profile.wins}W / {profile.losses}L ({profile.total_matches} games)
          </p>
        )}
      </div>

      {/* 지갑 연동 영역 — 5단계에서 WalletLinkButton으로 교체 */}
      <div className="mb-8">
        {wallet ? (
          <p className="text-gray-400 text-sm">
            Wallet: {wallet.wallet_address.slice(0, 6)}...{wallet.wallet_address.slice(-4)}
          </p>
        ) : (
          <p className="text-gray-500 text-sm">지갑 미연동</p>
        )}
      </div>

      {/* 액션 버튼 */}
      <Button
        onClick={handlePlay}
        size="lg"
        className="px-12 py-8 text-3xl font-bold shadow-lg transition-all hover:scale-105 mb-4"
      >
        PLAY
      </Button>

      <Button
        onClick={handleLogout}
        variant="ghost"
        size="sm"
        className="text-gray-400 hover:text-white"
      >
        로그아웃
      </Button>
    </div>
  );
}
```

#### 3. 게임 페이지
**파일**: `apps/web/src/app/game/page.tsx` (새로 생성)

기존 `page.tsx`의 GameWrapper 로직을 이동. 미들웨어가 인증을 보호하므로 서버 측 인증 체크 불필요.

```tsx
'use client';

import dynamic from 'next/dynamic';

const GameWrapper = dynamic(
  () => import('@/components/game/GameWrapper').then((m) => m.GameWrapper),
  { ssr: false }
);

export default function GamePage() {
  return <GameWrapper />;
}
```

#### 4. GameWrapper 수정 — 로비로 돌아가기
**파일**: `apps/web/src/components/game/GameWrapper.tsx` (기존 파일 수정)
**변경**: "Main Menu" 버튼을 `/lobby`로 이동하도록 변경, 초기 화면을 `playing`으로 변경

```tsx
'use client';

import { useCallback, useEffect, useState } from 'react';
import { useRouter } from 'next/navigation';
import { EventBus, GameEvents } from '@repo/game-core';
import { PhaserGame } from './PhaserGame';
import { Button } from '@/components/ui/shadcn/button';

interface GameResult {
  winnerId: number;
}

export function GameWrapper() {
  const router = useRouter();
  const [screen, setScreen] = useState<'playing' | 'result'>('playing');
  const [result, setResult] = useState<GameResult | null>(null);

  useEffect(() => {
    const handleGameOver = (data: GameResult) => {
      setResult(data);
      setScreen('result');
    };

    EventBus.on(GameEvents.GAME_OVER, handleGameOver);
    return () => {
      EventBus.off(GameEvents.GAME_OVER, handleGameOver);
    };
  }, []);

  const handlePlayAgain = useCallback(() => {
    setScreen('playing');
    setResult(null);
  }, []);

  const handleReturnToLobby = useCallback(() => {
    router.push('/lobby');
  }, [router]);

  if (screen === 'result' && result) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen bg-linear-to-b from-gray-900 to-gray-700">
        <h1 className="text-5xl font-bold text-white mb-4">
          {result.winnerId >= 0
            ? `Player ${result.winnerId + 1} Wins!`
            : 'Draw!'}
        </h1>
        <div className="flex gap-4 mt-8">
          <Button
            onClick={handlePlayAgain}
            size="lg"
            className="px-6 py-6 text-xl shadow-lg transition-all hover:scale-105"
          >
            Play Again
          </Button>
          <Button
            onClick={handleReturnToLobby}
            variant="secondary"
            size="lg"
            className="px-6 py-6 text-xl shadow-lg transition-all hover:scale-105"
          >
            Lobby
          </Button>
        </div>
      </div>
    );
  }

  return (
    <div className="relative min-h-screen bg-gray-900">
      <PhaserGame />
      <Button
        onClick={handleReturnToLobby}
        variant="destructive"
        size="sm"
        className="absolute top-4 right-4 z-10"
      >
        Exit
      </Button>
    </div>
  );
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] 타입체크 통과: `pnpm typecheck`
- [x] 린트 통과: `pnpm lint`
- [x] `apps/web/src/app/lobby/page.tsx` 존재
- [x] `apps/web/src/app/game/page.tsx` 존재
- [x] `apps/web/src/components/lobby/LobbyContent.tsx` 존재

#### 수동 검증:
- [ ] 로그인 후 `/lobby`에서 프로필 정보(이름, 아바타) 표시
- [ ] PLAY 버튼 클릭 → `/game`에서 게임 정상 동작
- [ ] 게임 종료/Exit → `/lobby`로 복귀
- [ ] 로그아웃 → `/`로 이동
- [ ] 비인증 상태에서 `/lobby`, `/game` 접근 시 `/`로 리디렉션

**Implementation Note**: 기존 MainMenu 컴포넌트는 더 이상 사용하지 않음 (로비 페이지가 메인 메뉴 역할). 삭제하지 않고 유지하되, 추후 필요 없으면 정리.

---

## 5단계: 지갑 연동

### 개요
dapp-kit Provider를 로비 레이아웃에 배치. 지갑 API 라우트(nonce/verify/unlink) 구현. WalletLinkButton 컴포넌트로 연동/해제 UI 제공. 연동 시 app_metadata 업데이트.

### 필요한 변경:

#### 1. 로비 레이아웃 (dapp-kit Provider)
**파일**: `apps/web/src/app/lobby/layout.tsx` (새로 생성)

```tsx
import { SuiProviders } from '@/components/providers/SuiProviders';

export default function LobbyLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return <SuiProviders>{children}</SuiProviders>;
}
```

#### 2. Sui Provider 컴포넌트
**파일**: `apps/web/src/components/providers/SuiProviders.tsx` (새로 생성)

```tsx
'use client';

import { createNetworkConfig, SuiClientProvider, WalletProvider } from '@mysten/dapp-kit';
import { getFullnodeUrl } from '@mysten/sui/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import '@mysten/dapp-kit/dist/index.css';

const { networkConfig } = createNetworkConfig({
  mainnet: { url: getFullnodeUrl('mainnet') },
  testnet: { url: getFullnodeUrl('testnet') },
});

const queryClient = new QueryClient();

export function SuiProviders({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <SuiClientProvider networks={networkConfig} defaultNetwork="testnet">
        <WalletProvider
          preferredWallets={['Sui Wallet', 'Suiet', 'Phantom']}
          autoConnect
        >
          {children}
        </WalletProvider>
      </SuiClientProvider>
    </QueryClientProvider>
  );
}
```

#### 3. 지갑 Nonce 발급 API
**파일**: `apps/web/src/app/api/wallet/nonce/route.ts` (새로 생성)

```typescript
import { createClient } from '@/lib/supabase/server';
import { createClient as createAdminClient } from '@supabase/supabase-js';
import { randomBytes } from 'crypto';

export async function POST(req: Request) {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (error || !user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { wallet_address } = await req.json();
  const nonce = randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 5 * 60 * 1000); // 5분

  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  await admin.from('wallet_nonces').insert({
    user_id: user.id,
    wallet_address,
    nonce,
    expires_at: expiresAt.toISOString(),
  });

  return Response.json({ nonce });
}
```

#### 4. 지갑 서명 검증 + 저장 API
**파일**: `apps/web/src/app/api/wallet/verify/route.ts` (새로 생성)

```typescript
import { createClient } from '@/lib/supabase/server';
import { verifyPersonalMessageSignature } from '@mysten/sui/verify';
import { createClient as createAdminClient } from '@supabase/supabase-js';

export async function POST(req: Request) {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (error || !user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const { wallet_address, signature, message, nonce } = await req.json();

  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  try {
    // 1. Nonce 검증
    const { data: nonceRecord } = await admin
      .from('wallet_nonces')
      .select('*')
      .eq('user_id', user.id)
      .eq('wallet_address', wallet_address)
      .eq('nonce', nonce)
      .gt('expires_at', new Date().toISOString())
      .single();

    if (!nonceRecord) {
      return Response.json({ error: 'Invalid or expired nonce' }, { status: 401 });
    }

    // Nonce 삭제 (일회용)
    await admin.from('wallet_nonces').delete().eq('id', nonceRecord.id);

    // 2. Sui 서명 검증
    const messageBytes = new TextEncoder().encode(message);
    await verifyPersonalMessageSignature(messageBytes, signature, {
      address: wallet_address,
    });

    // 3. 중복 검증
    const { data: existingWallet } = await admin
      .from('wallets')
      .select('user_id')
      .eq('wallet_address', wallet_address)
      .single();

    if (existingWallet && existingWallet.user_id !== user.id) {
      return Response.json(
        { error: 'This wallet is already linked to another account' },
        { status: 409 }
      );
    }

    // 4. 기존 연동 지갑 교체 (1:1 정책)
    await admin.from('wallets').delete().eq('user_id', user.id);

    // 5. 지갑 주소 저장
    const { error: insertError } = await admin.from('wallets').insert({
      user_id: user.id,
      wallet_address,
      chain: 'sui',
      verified: true,
      linked_at: new Date().toISOString(),
    });

    if (insertError) {
      return Response.json({ error: 'Failed to save wallet' }, { status: 500 });
    }

    // 6. app_metadata 업데이트 (JWT에 wallet_address 포함)
    await admin.auth.admin.updateUserById(user.id, {
      app_metadata: { wallet_address, chain: 'sui' },
    });

    return Response.json({ success: true, wallet_address });
  } catch {
    return Response.json({ error: 'Verification failed' }, { status: 401 });
  }
}
```

#### 5. 지갑 연동 해제 API
**파일**: `apps/web/src/app/api/wallet/unlink/route.ts` (새로 생성)

```typescript
import { createClient } from '@/lib/supabase/server';
import { createClient as createAdminClient } from '@supabase/supabase-js';

export async function DELETE() {
  const supabase = await createClient();
  const { data: { user }, error } = await supabase.auth.getUser();

  if (error || !user) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  );

  const { error: deleteError } = await admin
    .from('wallets')
    .delete()
    .eq('user_id', user.id);

  if (deleteError) {
    return Response.json({ error: 'Failed to unlink wallet' }, { status: 500 });
  }

  // app_metadata에서 wallet_address 제거
  await admin.auth.admin.updateUserById(user.id, {
    app_metadata: { wallet_address: null, chain: null },
  });

  return Response.json({ success: true });
}
```

#### 6. WalletLinkButton 컴포넌트
**파일**: `apps/web/src/components/lobby/WalletLinkButton.tsx` (새로 생성)

```tsx
'use client';

import { ConnectButton, useCurrentAccount, useSignPersonalMessage } from '@mysten/dapp-kit';
import { useState, useCallback } from 'react';
import { Button } from '@/components/ui/shadcn/button';

interface WalletLinkButtonProps {
  initialWallet: { wallet_address: string; chain: string } | null;
}

export function WalletLinkButton({ initialWallet }: WalletLinkButtonProps) {
  const account = useCurrentAccount();
  const { mutateAsync: signMessage } = useSignPersonalMessage();
  const [isLoading, setIsLoading] = useState(false);
  const [linkedWallet, setLinkedWallet] = useState(initialWallet);
  const [error, setError] = useState<string | null>(null);

  const linkWallet = useCallback(async () => {
    if (!account) return;
    setIsLoading(true);
    setError(null);

    try {
      // 1. Nonce 요청
      const nonceRes = await fetch('/api/wallet/nonce', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ wallet_address: account.address }),
      });
      const { nonce } = await nonceRes.json();

      // 2. 메시지 서명
      const message = `Link wallet to Artillery Wars\nAddress: ${account.address}\nNonce: ${nonce}\nTimestamp: ${Date.now()}`;
      const { signature } = await signMessage({
        message: new TextEncoder().encode(message),
      });

      // 3. 서명 검증 요청
      const verifyRes = await fetch('/api/wallet/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ wallet_address: account.address, signature, message, nonce }),
      });

      if (verifyRes.ok) {
        setLinkedWallet({ wallet_address: account.address, chain: 'sui' });
      } else {
        const data = await verifyRes.json();
        setError(data.error);
      }
    } catch {
      setError('지갑 연동에 실패했습니다');
    } finally {
      setIsLoading(false);
    }
  }, [account, signMessage]);

  const unlinkWallet = useCallback(async () => {
    setIsLoading(true);
    setError(null);

    try {
      const res = await fetch('/api/wallet/unlink', { method: 'DELETE' });
      if (res.ok) {
        setLinkedWallet(null);
      } else {
        const data = await res.json();
        setError(data.error);
      }
    } catch {
      setError('연동 해제에 실패했습니다');
    } finally {
      setIsLoading(false);
    }
  }, []);

  // 이미 연동된 지갑이 있는 경우
  if (linkedWallet) {
    const addr = linkedWallet.wallet_address;
    return (
      <div className="flex flex-col items-center gap-2">
        <span className="text-gray-300 text-sm">
          {addr.slice(0, 6)}...{addr.slice(-4)}
        </span>
        <Button
          onClick={unlinkWallet}
          disabled={isLoading}
          variant="ghost"
          size="sm"
          className="text-gray-400"
        >
          {isLoading ? '해제 중...' : '연동 해제'}
        </Button>
        {error && <p className="text-red-400 text-xs">{error}</p>}
      </div>
    );
  }

  // 지갑 미연동 + dapp-kit에 지갑 미연결
  if (!account) {
    return <ConnectButton />;
  }

  // 지갑 연결됨 → 연동 버튼 표시
  return (
    <div className="flex flex-col items-center gap-2">
      <Button
        onClick={linkWallet}
        disabled={isLoading}
        variant="outline"
        size="sm"
      >
        {isLoading ? '연동 중...' : '지갑 연동하기'}
      </Button>
      {error && <p className="text-red-400 text-xs">{error}</p>}
    </div>
  );
}
```

#### 7. LobbyContent에 WalletLinkButton 통합
**파일**: `apps/web/src/components/lobby/LobbyContent.tsx` (기존 파일 수정)
**변경**: 지갑 영역의 플레이스홀더를 WalletLinkButton으로 교체

```tsx
// import 추가
import { WalletLinkButton } from './WalletLinkButton';

// 기존 지갑 플레이스홀더 영역을 교체:
{/* 지갑 연동 영역 */}
<div className="mb-8">
  <WalletLinkButton initialWallet={wallet} />
</div>
```

### 성공 기준:

#### 자동화된 검증:
- [x] 타입체크 통과: `pnpm typecheck`
- [x] 린트 통과: `pnpm lint`
- [x] `apps/web/src/app/lobby/layout.tsx` 존재
- [x] `apps/web/src/components/providers/SuiProviders.tsx` 존재
- [x] `apps/web/src/app/api/wallet/nonce/route.ts` 존재
- [x] `apps/web/src/app/api/wallet/verify/route.ts` 존재
- [x] `apps/web/src/app/api/wallet/unlink/route.ts` 존재
- [x] `apps/web/src/components/lobby/WalletLinkButton.tsx` 존재

#### 수동 검증:
- [ ] 로비에서 ConnectButton 클릭 → Sui 지갑 선택 → 연결
- [ ] "지갑 연동하기" 클릭 → 서명 요청 → 연동 성공
- [ ] wallets 테이블에 지갑 주소 저장 확인
- [ ] app_metadata에 wallet_address 추가 확인
- [ ] "연동 해제" 클릭 → 해제 성공
- [ ] 이미 다른 유저에 연동된 지갑으로 시도 시 409 에러

**Implementation Note**: 이 단계 완료 후 실제 Sui 지갑(Sui Wallet, Suiet, 또는 Phantom)으로 연동/해제 전체 플로우를 수동 테스트.

---

## 6단계: 통합 검증

### 개요
전체 플로우 검증, 보안 점검, 타입/린트 최종 확인.

### 필요한 변경:

#### 1. Supabase Security Advisor 실행
Supabase MCP `get_advisors`로 보안 점검.

#### 2. 최종 검증 체크리스트 실행

### 성공 기준:

#### 자동화된 검증:
- [x] 타입체크 통과: `pnpm typecheck`
- [x] 린트 통과: `pnpm lint`
- [x] 빌드 성공: `pnpm build`
- [x] Supabase security advisor 경고 없음
- [x] Supabase performance advisor 경고 없음 (unused index INFO만 — 데이터 미사용 상태이므로 정상)

#### 수동 검증:
- [ ] **전체 플로우 A (신규 유저)**: `/` → Google 로그인 → `/lobby` (프로필 자동 생성) → PLAY → 게임 → Exit → 로비
- [ ] **전체 플로우 B (지갑 연동)**: 로비 → ConnectButton → 지갑 연동 → 축약 주소 표시 → 연동 해제
- [ ] **보호 라우트**: 로그아웃 후 `/lobby`, `/game` 접근 → `/`로 리디렉션
- [ ] **세션 유지**: 페이지 새로고침 후에도 로그인 상태 유지

---

## 테스트 전략

### 수동 테스트 시나리오:
1. **신규 유저 가입 플로우**: Google 로그인 → 프로필 생성 → 로비 도달
2. **기존 유저 로그인**: Google 로그인 → 기존 프로필로 로비 도달
3. **라우트 보호**: 비인증 `/lobby`, `/game` 접근 → `/` 리디렉션
4. **중복 리디렉션 방지**: 인증 후 `/` 접근 → `/lobby` 리디렉션
5. **지갑 연동**: dapp-kit 연결 → 서명 → 연동 성공
6. **지갑 해제**: 연동 해제 → wallets 테이블 + app_metadata 초기화
7. **중복 지갑 방지**: 이미 연동된 지갑으로 시도 → 409 에러
8. **로그아웃**: 로그아웃 → 세션 삭제 → `/` 이동
9. **세션 갱신**: 1시간+ 후 자동 토큰 갱신 확인

## 성능 고려 사항

- **dapp-kit 번들 (~8MB unpacked)**: `/lobby` 레이아웃에만 배치 → Next.js 자동 코드 스플릿
- **미들웨어 `getClaims()`**: 네트워크 요청 없음 (로컬 JWT 검증) → 지연 최소
- **app_metadata**: Custom Hook 없이 JWT에 자동 포함 → 토큰 발급 시 DB 쿼리 0회

## 마이그레이션 참고 사항

- 기존 사용자 데이터 없음 (신규 구현)
- 기존 `MainMenu` 컴포넌트는 삭제하지 않고 유지 (추후 정리 가능)
- 기존 `page.tsx`의 GameWrapper는 `/game/page.tsx`로 이동

## 파일 생성/수정 요약

### 새로 생성하는 파일 (15개):
| 파일 | 용도 |
|------|------|
| `apps/web/.env.local` | 환경변수 |
| `apps/web/src/lib/supabase/client.ts` | 브라우저 Supabase 클라이언트 |
| `apps/web/src/lib/supabase/server.ts` | 서버 Supabase 클라이언트 |
| `apps/web/src/middleware.ts` | 토큰 갱신 + 라우트 보호 |
| `apps/web/src/app/auth/callback/route.ts` | OAuth 콜백 (PKCE) |
| `apps/web/src/app/auth/auth-code-error/page.tsx` | 인증 오류 페이지 |
| `apps/web/src/app/lobby/page.tsx` | 게임 로비 |
| `apps/web/src/app/lobby/layout.tsx` | 로비 레이아웃 (dapp-kit Provider) |
| `apps/web/src/app/game/page.tsx` | 게임 플레이 |
| `apps/web/src/app/api/wallet/nonce/route.ts` | 지갑 Nonce 발급 |
| `apps/web/src/app/api/wallet/verify/route.ts` | 지갑 서명 검증 + 저장 |
| `apps/web/src/app/api/wallet/unlink/route.ts` | 지갑 연동 해제 |
| `apps/web/src/components/auth/GoogleLoginButton.tsx` | Google 로그인 버튼 |
| `apps/web/src/components/auth/SessionHandler.tsx` | 세션 감지 |
| `apps/web/src/components/lobby/LobbyContent.tsx` | 로비 콘텐츠 |
| `apps/web/src/components/lobby/WalletLinkButton.tsx` | 지갑 연동 버튼 |
| `apps/web/src/components/providers/SuiProviders.tsx` | Sui dapp-kit Provider |

### 수정하는 파일 (2개):
| 파일 | 변경 내용 |
|------|----------|
| `apps/web/src/app/page.tsx` | GameWrapper → 로그인 페이지로 교체 |
| `apps/web/src/app/layout.tsx` | SessionHandler 추가 |
| `apps/web/src/components/game/GameWrapper.tsx` | 메뉴 화면 제거, 로비 복귀 네비게이션 |

### Supabase 마이그레이션 (3개):
| 마이그레이션 | 내용 |
|-------------|------|
| `create_core_tables` | player_profiles, wallets, wallet_nonces 테이블 생성 |
| `create_user_trigger` | handle_new_user 트리거 생성 |
| `setup_rls_policies` | RLS 활성화 + 정책 생성 |

## 구현 완료 후 추가 변경사항 (2026-02-13)

6단계 구현 완료 이후 다음 추가 작업이 진행됨:

### A. Supabase Publishable Key 전환

Legacy `ANON_KEY` (JWT) → Publishable Key (`sb_publishable_...`) 형식으로 전환.
- `client.ts`, `server.ts`, `proxy.ts` 모두 `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` 참조로 변경

### B. 로비 UI 리디자인

단순 텍스트 + 버튼 → 게임 스타일 풀스크린 로비:
- `ProfileCard.tsx` — 아바타, 이름, Lv, XP 바, 전적
- `CharacterDisplay.tsx` — `player_idle.png` + float 애니메이션
- `PlayButton.tsx` — scale-pulse 애니메이션 + 글로우
- `lib/level.ts` — XP 레벨 계산 유틸리티
- `globals.css` — 3개 keyframe 애니메이션 + 퍼플 테마 (`oklch(0.59 0.20 277)`)

### C. 커스텀 지갑 선택 다이얼로그

dapp-kit `<ConnectButton />` 제거 → `useWallets()` + `useConnectWallet()` 훅으로 커스텀 다이얼로그 구현:
- shadcn Dialog + 게임 테마 스타일링
- 브라우저 설치 지갑 자동 감지 및 아이콘/이름 표시

### D. 1-step 지갑 연동 플로우

기존 2-step (Connect → Link) → 1-step (Connect + Sign + Verify) 통합:
- 지갑 선택 → 연결 → 즉시 서명 요청 → 서버 검증 → 연동 완료
- 실패 시 `disconnectWallet()` 호출로 깔끔한 상태 복원
- Unlink 시에도 `disconnectWallet()` 함께 호출

### E. Next.js 16 proxy.ts

`middleware.ts` → `proxy.ts` 변경 (Next.js 16 요구사항).

### F. @mysten 패키지 버전

| 패키지 | 계획 시점 | 실제 설치 |
|--------|----------|----------|
| `@mysten/dapp-kit` | v0.19.11 | **v1.0.3** |
| `@mysten/sui` | v2.1.0 | **v2.4.0** |

`getFullnodeUrl()` 제거됨 → 하드코딩 URL + `network` 프로퍼티.

---

## 참조

- 연구 문서: `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md`
- 이전 연구 (대체됨): `thoughts/shared/research/2026-02-12-sui-wallet-login-supabase-auth-research.md`
- 전체 아키텍처: `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md`
- MVP 계획: `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md`
- Supabase 프로젝트: `fmwjmsfgxhbzhvtnaqta` (Whoosh Bang, ap-northeast-1)
