---
date: 2026-02-12T22:24:05+0900
researcher: arta1069@gmail.com
topic: "소셜 로그인(Google) 우선 + 선택적 Sui 지갑 연동 인증 아키텍처 연구"
tags: [research, supabase, auth, google-oauth, sui, wallet-linking, next-js, ssr]
status: complete
last_updated: 2026-02-13
last_updated_by: arta1069@gmail.com
last_updated_note: "구현 완료 후 최신화 — Publishable Key, proxy.ts, 1-step 지갑 연동, 로비 UI 리디자인 반영"
---

# 연구: 소셜 로그인(Google) 우선 + 선택적 Sui 지갑 연동 인증 아키텍처

**날짜**: 2026-02-12T22:24:05+0900
**연구자**: arta1069@gmail.com
**리포지토리**: whoosh-bang

## 연구 질문

기존 "지갑 로그인 우선" 아키텍처에서 **"소셜 로그인(Google) 우선 + 게임 로비에서 선택적 지갑 연동"** 아키텍처로 변경하는 방법 연구. Supabase Auth 네이티브 Google OAuth를 주 인증으로 사용하고, @mysten/dapp-kit으로 Sui 지갑 연동을 선택적 기능으로 제공.

## 요약

### 핵심 결론

1. **아키텍처 전환**: 지갑 로그인(커스텀 JWT) → **Supabase Auth 네이티브 Google OAuth** (커스텀 JWT 불필요)
2. **@supabase/ssr**: `@supabase/auth-helpers-nextjs`는 deprecated. **`@supabase/ssr`이 유일한 권장 패키지**
3. **PKCE Flow**: SSR 환경에서 기본이며 보안성이 높음. `exchangeCodeForSession`으로 코드 교환
4. **지갑 연동**: 별도 `wallets` 테이블에 저장 (user_metadata 사용 금지 — 사용자가 조작 가능)
5. **Custom Access Token Hook**: JWT에 `wallet_address` 클레임 자동 추가 → RLS에서 활용
6. **보안 핵심**: 서버에서 `getUser()` 사용 필수. `getSession()`은 JWT를 재검증하지 않아 위조 가능
7. **getClaims() vs getUser()**: `getClaims()`는 로컬 JWT 검증(빠름), `getUser()`는 서버 검증(확실함). Middleware에서는 `getClaims()`, API Route에서는 `getUser()` 사용 권장
7. **현재 코드베이스**: 인증 관련 구현 없음. provider, middleware, env 모두 미설정 상태

### 인증 플로우 개요

```
[로그인 화면]
  ↓
사용자 → "Google로 로그인" 클릭
  ↓
Supabase Auth → Google OAuth (PKCE Flow)
  ↓
/auth/callback → exchangeCodeForSession(code)
  ↓
쿠키에 세션 저장 → 게임 로비로 리디렉션
  ↓
[게임 로비] (이미지 참고: 프로필, 탱크, PLAY 버튼)
  ↓
선택: "지갑 연동하기" 클릭
  ↓
dapp-kit 지갑 연결 → 메시지 서명 → 서버 검증
  ↓
wallets 테이블에 저장 → Custom Access Token Hook으로 JWT에 추가
```

---

## 상세 발견 사항

### 1. 아키텍처 비교: 이전 vs 변경

| 항목 | 이전 (지갑 우선) | 변경 (소셜 우선) |
|------|-----------------|-----------------|
| **주 인증** | Sui 지갑 서명 | Google OAuth (Supabase Auth) |
| **JWT 방식** | 커스텀 JWT (직접 발급) | Supabase Auth 네이티브 JWT |
| **지갑 역할** | 로그인 필수 | 선택적 연동 (게임 로비에서) |
| **세션 관리** | localStorage + accessToken | 쿠키 기반 (SSR 호환) |
| **사용자 ID** | 지갑 주소 기반 | Supabase `auth.users.id` (UUID) |
| **계정 복구** | 불가 (지갑 분실 시 계정 손실) | Google 계정으로 복구 가능 |
| **온보딩 마찰** | 높음 (지갑 설치 필수) | 낮음 (Google 로그인만으로 즉시 플레이) |
| **필요 패키지** | @supabase/supabase-js + jsonwebtoken | @supabase/ssr (커스텀 JWT 불필요) |

### 2. Supabase Google OAuth 설정

#### 패키지 현황

| 패키지 | 상태 | 비고 |
|--------|------|------|
| `@supabase/ssr` | **권장** (현재) | 프레임워크 무관, Next.js 16 / React 19 호환 |
| `@supabase/auth-helpers-nextjs` | **Deprecated** | 사용 금지. 동시 사용 시 인증 문제 발생 |
| `@supabase/supabase-js` | **필수** | @supabase/ssr의 의존성 |

**설치**:
```bash
pnpm add @supabase/supabase-js @supabase/ssr
```

#### Google Cloud Console 설정

1. OAuth Client ID 생성 (Web application 타입)
2. **Authorized redirect URIs**: `https://<project-id>.supabase.co/auth/v1/callback`
3. Supabase Dashboard → Authentication → Providers → Google에서 Client ID/Secret 등록
4. 필수 스코프: `openid`, `email`, `profile`

#### Supabase 클라이언트 생성 패턴

**브라우저 클라이언트** (`lib/supabase/client.ts`):
```typescript
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  );
}
```

**서버 클라이언트** (`lib/supabase/server.ts`):
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

> **중요**: 반드시 `getAll`/`setAll`만 사용. 절대 `get`/`set`/`remove` 사용 금지.

### 3. Next.js 미들웨어 (토큰 갱신)

미들웨어가 필요한 이유:
1. Server Components는 쿠키를 **쓸 수 없음**
2. 만료된 Auth 토큰을 자동 갱신해야 함
3. 보호된 라우트 접근 제어

```typescript
// middleware.ts (src/ 루트)
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

  // JWT 클레임 로컬 검증 (네트워크 요청 없음, 빠름)
  // getClaims()는 JWT 서명과 만료시간만 검증 — 라우트 보호용으로 적합
  // 민감한 작업에는 getUser()를 사용할 것 (API Route, Server Component에서)
  const { data: claims, error } = await supabase.auth.getClaims();

  // 보호된 라우트: 로그인 필요 페이지
  const protectedRoutes = ['/lobby', '/game', '/profile', '/settings'];
  const isProtectedRoute = protectedRoutes.some(route =>
    request.nextUrl.pathname.startsWith(route)
  );

  if (isProtectedRoute && (error || !claims?.sub)) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  // 이미 로그인한 사용자가 로그인 페이지 접근 시
  if (request.nextUrl.pathname === '/login' && claims?.sub) {
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

> **보안 참고**:
> - 서버 코드에서 **절대** `getSession()` 사용 금지 — JWT를 재검증하지 않아 쿠키 위조에 취약
> - **Middleware**: `getClaims()` 사용 권장 (로컬 JWT 검증, 네트워크 요청 없음, 빠름)
> - **API Route / Server Component**: `getUser()` 사용 권장 (Auth 서버에서 세션 유효성 완전 검증)
> - 자세한 비교는 아래 [getClaims() vs getUser() 비교](#후속-연구-getclaims-vs-getuser-비교) 섹션 참고

### 4. OAuth 콜백 처리

```typescript
// app/auth/callback/route.ts
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

### 5. 로그인 UI 구현

```typescript
// app/login/page.tsx
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
    <div>
      <h1>Worms Game</h1>
      <GoogleLoginButton />
    </div>
  );
}
```

```typescript
// components/auth/GoogleLoginButton.tsx
'use client';

import { createClient } from '@/lib/supabase/client';

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
    <button onClick={handleLogin}>
      Google로 로그인
    </button>
  );
}
```

### 6. 게임 로비에서 선택적 지갑 연동

#### 지갑 연동 플로우

```
[게임 로비] → "지갑 연동하기" 버튼 클릭
  ↓
1. dapp-kit ConnectButton으로 Sui 지갑 연결
  ↓
2. 서버에 Nonce 요청 (POST /api/wallet/nonce)
  ↓
3. Nonce 포함 메시지에 지갑으로 서명 (signPersonalMessage)
  ↓
4. 서버에 서명 검증 요청 (POST /api/wallet/verify)
  ↓
5. 서버: Sui 서명 검증 → wallets 테이블에 저장
  ↓
6. Custom Access Token Hook이 다음 JWT 갱신 시 wallet_address 추가
```

#### 지갑 연동 API: Nonce 발급

```typescript
// app/api/wallet/nonce/route.ts
import { createClient } from '@/lib/supabase/server';
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

  // Service role client로 nonce 저장
  const { createClient: createAdminClient } = await import('@supabase/supabase-js');
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

#### 지갑 연동 API: 서명 검증 및 저장

```typescript
// app/api/wallet/verify/route.ts
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

    // 3. 중복 검증: 이미 다른 유저에 연동된 지갑인지 확인
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

    // 4. 기존 연동 지갑이 있으면 교체 (1:1 정책)
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

    return Response.json({ success: true, wallet_address });
  } catch (err) {
    return Response.json({ error: 'Verification failed' }, { status: 401 });
  }
}
```

#### 지갑 연동 해제 API

```typescript
// app/api/wallet/unlink/route.ts
import { createClient } from '@/lib/supabase/server';
import { createClient as createAdminClient } from '@supabase/supabase-js';

export async function DELETE(req: Request) {
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

  return Response.json({ success: true });
}
```

#### 게임 로비 지갑 연동/해제 UI

```typescript
// components/lobby/WalletLinkButton.tsx
'use client';

import { ConnectButton, useCurrentAccount, useSignPersonalMessage } from '@mysten/dapp-kit';
import { useState, useCallback, useEffect } from 'react';

interface WalletLinkButtonProps {
  /** 서버에서 조회한 현재 연동된 지갑 정보 */
  initialWallet: { wallet_address: string; chain: string } | null;
}

export function WalletLinkButton({ initialWallet }: WalletLinkButtonProps) {
  const account = useCurrentAccount();
  const { mutateAsync: signMessage } = useSignPersonalMessage();
  const [isLoading, setIsLoading] = useState(false);
  const [linkedWallet, setLinkedWallet] = useState(initialWallet);
  const [error, setError] = useState<string | null>(null);

  // 지갑 연동
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
      const message = `Link wallet to Worms Game\nAddress: ${account.address}\nNonce: ${nonce}\nTimestamp: ${Date.now()}`;
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

  // 지갑 연동 해제
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

  // 이미 연동된 지갑이 있는 경우 → 연동 해제 버튼
  if (linkedWallet) {
    const addr = linkedWallet.wallet_address;
    return (
      <div>
        <span>🔗 {addr.slice(0, 6)}...{addr.slice(-4)}</span>
        <button onClick={unlinkWallet} disabled={isLoading}>
          {isLoading ? '해제 중...' : '연동 해제'}
        </button>
        {error && <p>{error}</p>}
      </div>
    );
  }

  // 지갑 미연동 → dapp-kit ConnectButton으로 지갑 선택 후 연동
  if (!account) {
    return <ConnectButton />;
  }

  return (
    <div>
      <button onClick={linkWallet} disabled={isLoading}>
        {isLoading ? '연동 중...' : '지갑 연동하기'}
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

> **UX 정책**:
> - 지갑 미연동 유저 → "지갑 연동하기" 버튼 (ConnectButton → 서명 → 연동)
> - 지갑 연동 유저 → 축약 주소 표시 + "연동 해제" 버튼
> - 1 유저 = 1 지갑 (다중 지갑 미지원)
> - 이미 다른 유저에 연동된 지갑 → 409 에러와 함께 연동 차단

### 7. Provider 설정 (변경된 구조)

```typescript
// app/providers.tsx
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

export function Providers({ children }: { children: React.ReactNode }) {
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

> **참고**: Supabase 클라이언트는 Provider가 불필요. `createBrowserClient()`가 싱글턴 패턴으로 동작하므로 매번 호출해도 동일 인스턴스 반환. dapp-kit Provider는 지갑 연동 기능이 있는 페이지에서만 필요.

### 8. Supabase 데이터베이스 스키마

#### 테이블 구조

```sql
-- 사용자 프로필 (auth.users 트리거로 자동 생성)
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

-- 회원가입 시 자동 프로필 생성 트리거
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

-- 지갑 연동 테이블 (별도 테이블 — user_metadata 사용 금지)
-- 정책: 1 유저 = 1 지갑 (다중 지갑 미지원)
CREATE TABLE wallets (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE REFERENCES auth.users(id) ON DELETE CASCADE,
  wallet_address TEXT NOT NULL UNIQUE,
  chain TEXT NOT NULL DEFAULT 'sui',
  verified BOOLEAN DEFAULT FALSE,
  linked_at TIMESTAMPTZ DEFAULT NOW()
);

-- user_id UNIQUE: 한 유저당 하나의 지갑만 허용
-- wallet_address UNIQUE: 한 지갑은 하나의 유저만 연동 가능
CREATE INDEX idx_wallets_address ON wallets(wallet_address);

-- Nonce 테이블 (지갑 서명 검증용, 일회용)
CREATE TABLE wallet_nonces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  wallet_address TEXT NOT NULL,
  nonce TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_nonces_lookup ON wallet_nonces(user_id, wallet_address, nonce);

-- 자동 만료 처리 (pg_cron 또는 Supabase Edge Function)
-- DELETE FROM wallet_nonces WHERE expires_at < NOW();
```

#### RLS 정책

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

-- wallets: 자신의 지갑만 추가 (실제 삽입은 service_role로 처리)
CREATE POLICY "wallets_insert_own"
ON wallets FOR INSERT TO authenticated
WITH CHECK (user_id = auth.uid());

-- wallets: 자신의 지갑만 삭제 (연동 해제)
CREATE POLICY "wallets_delete_own"
ON wallets FOR DELETE TO authenticated
USING (user_id = auth.uid());

-- wallet_nonces: service_role로만 접근 (RLS 불필요하지만 안전장치)
CREATE POLICY "nonces_no_access"
ON wallet_nonces FOR ALL TO authenticated
USING (false);
```

#### JWT에 wallet_address 포함 방식: app_metadata 활용

> Custom Access Token Hook 대신 `app_metadata`를 사용하여 JWT에 wallet_address를 포함.
> Hook이 불필요하므로 토큰 발급 시 DB 쿼리 0회, 관리 포인트 감소.

**작동 원리**: `app_metadata`에 저장된 데이터는 JWT의 `app_metadata` 클레임에 자동 포함된다.

```typescript
// 지갑 연동 시 — /api/wallet/verify에서 지갑 저장 후 호출
await admin.auth.admin.updateUserById(user.id, {
  app_metadata: { wallet_address, chain: 'sui' }
});

// 지갑 해제 시 — /api/wallet/unlink에서 지갑 삭제 후 호출
await admin.auth.admin.updateUserById(user.id, {
  app_metadata: { wallet_address: null, chain: null }
});
```

**RLS에서 접근:**
```sql
-- app_metadata에서 wallet_address 읽기
(auth.jwt() -> 'app_metadata' ->> 'wallet_address') IS NOT NULL
```

> **보안**: `app_metadata`는 서버(service_role)에서만 수정 가능. 클라이언트가 조작할 수 없어 `user_metadata`와 달리 안전.
> **wallets 테이블은 유지**: 상세 지갑 정보의 원본(source of truth). `app_metadata`는 JWT 전달용 캐시 역할.

### 9. 세션 관리

#### 쿠키 기반 세션 (SSR 호환)

| 항목 | 값 |
|------|---|
| 저장 위치 | 쿠키 (`sb-<project-ref>-auth-token`) |
| Access Token 만료 | 1시간 (기본) |
| Refresh Token 만료 | 7일 (기본) |
| 자동 갱신 | Middleware에서 `getUser()` 호출 시 자동 |
| PKCE Flow | 기본값, 변경 불필요 |

#### 클라이언트 사이드 세션 감지

```typescript
// components/auth/SessionHandler.tsx
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
          router.push('/login');
        }
      }
    );

    return () => subscription.unsubscribe();
  }, [router, supabase]);

  return null;
}
```

---

## 아키텍처 문서화

### 전체 인증 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Next.js App (Client)                            │
│                                                                     │
│  ┌──────────────────┐  ┌────────────────┐  ┌────────────────────┐  │
│  │ Login Page        │  │ Game Lobby     │  │ @mysten/dapp-kit   │  │
│  │                   │  │                │  │                    │  │
│  │ • Google 로그인    │  │ • 프로필 표시   │  │ • ConnectButton    │  │
│  │ • signInWithOAuth │  │ • PLAY 버튼    │  │ • useSign...       │  │
│  │                   │  │ • 지갑 연동     │  │ • useCurrentAcct   │  │
│  └───────┬──────────┘  └───────┬────────┘  └────────┬───────────┘  │
│          │                     │                     │              │
│          │  ┌──────────────────┘                     │              │
│          │  │  Supabase Client (@supabase/ssr)       │              │
│          │  │  • createBrowserClient                 │              │
│          │  │  • 쿠키 기반 세션                       │              │
│          │  │  • RLS 쿼리                            │              │
│          │  └────────────────────────────────────────┘              │
└──────────┼──────────────────────────────────────────────────────────┘
           │
     ┌─────┴──────┐
     │ Middleware  │ ← 토큰 갱신 + 라우트 보호
     └─────┬──────┘
           │
     ┌─────┴──────────────────────────────────────────────────────┐
     │                    Next.js Server                           │
     │                                                             │
     │  ┌──────────────────┐  ┌────────────────────────────────┐  │
     │  │ /auth/callback   │  │ /api/wallet/nonce              │  │
     │  │ exchangeCode...  │  │ /api/wallet/verify             │  │
     │  │                  │  │ verifyPersonalMessageSignature  │  │
     │  └──────────────────┘  └────────────────────────────────┘  │
     └─────────────────────────┬───────────────────────────────────┘
                               │
     ┌─────────────────────────┴──────────────────────────────────┐
     │                       Supabase                              │
     │                                                             │
     │  ┌──────────────┐  ┌──────────────────────────────────┐   │
     │  │ Auth (GoTrue) │  │ PostgreSQL                       │   │
     │  │               │  │                                  │   │
     │  │ • Google OAuth│  │ • auth.users (Supabase 관리)     │   │
     │  │ • PKCE Flow   │  │ • player_profiles                │   │
     │  │ • JWT 발급    │  │ • wallets                        │   │
     │  │ • app_metadata│  │ • wallet_nonces                  │   │
     │  └──────────────┘  │ • game_lobbies (추후)             │   │
     │                    │ • match_history (추후)             │   │
     │  ┌──────────────┐  └──────────────────────────────────┘   │
     │  │ Realtime     │                                          │
     │  │ (게임 멀티)   │                                          │
     │  └──────────────┘                                          │
     └────────────────────────────────────────────────────────────┘
                               │
     ┌─────────────────────────┴──────────────────────────────────┐
     │                    Sui Blockchain                           │
     │  • 지갑 서명 검증 (서버 측 @mysten/sui/verify)               │
     │  • NFT/토큰 트랜잭션 (지갑 연동 사용자만)                     │
     └────────────────────────────────────────────────────────────┘
```

### 라우트 구조

```
app/
├── layout.tsx              ← 루트 레이아웃 (SessionHandler 포함)
├── login/
│   └── page.tsx            ← 로그인 페이지 (Google OAuth)
├── auth/
│   ├── callback/
│   │   └── route.ts        ← OAuth 콜백 (PKCE code exchange)
│   └── auth-code-error/
│       └── page.tsx         ← 인증 오류 페이지
├── lobby/
│   ├── layout.tsx           ← 로비 레이아웃 (dapp-kit Providers)
│   └── page.tsx             ← 게임 로비 (프로필, 지갑 연동, PLAY)
├── game/
│   └── [id]/
│       └── page.tsx         ← 게임 플레이
├── profile/
│   └── page.tsx             ← 프로필/설정
└── api/
    └── wallet/
        ├── nonce/
        │   └── route.ts     ← Nonce 발급
        └── verify/
            └── route.ts     ← 서명 검증 + 지갑 저장
```

> **핵심**: dapp-kit Provider (`SuiClientProvider`, `WalletProvider`)는 `lobby/layout.tsx`에만 적용. 로그인 페이지에서는 불필요.

### 환경 변수 설정

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...

# 서버 전용 (NEXT_PUBLIC_ 접두사 없음)
SUPABASE_SERVICE_ROLE_KEY=eyJ...

# Sui 설정
NEXT_PUBLIC_SUI_NETWORK=testnet
```

### 필요한 의존성

```json
{
  "dependencies": {
    "@supabase/supabase-js": "^2.x.x",
    "@supabase/ssr": "^0.x.x",
    "@mysten/dapp-kit": "^0.19.11",
    "@mysten/sui": "^2.1.0",
    "@tanstack/react-query": "^5.x.x"
  }
}
```

> **변경점**: `jsonwebtoken` 패키지 불필요 (Supabase Auth가 JWT를 직접 관리). 커스텀 JWT를 직접 발급할 필요 없음.

### 지갑 연동 시 RLS 활용 예시

```sql
-- 지갑 연동 사용자만 접근 가능한 콘텐츠
CREATE POLICY "wallet_required_content"
ON premium_content FOR SELECT TO authenticated
USING (
  EXISTS (
    SELECT 1 FROM wallets
    WHERE user_id = auth.uid()
    AND verified = true
  )
);

-- 또는 JWT app_metadata 기반 (지갑 연동 시 app_metadata에 저장됨)
CREATE POLICY "wallet_required_via_jwt"
ON premium_content FOR SELECT TO authenticated
USING (
  (auth.jwt() -> 'app_metadata' ->> 'wallet_address') IS NOT NULL
);
```

---

## 이전 연구와의 관계

이 연구는 기존 연구 `thoughts/shared/research/2026-02-12-sui-wallet-login-supabase-auth-research.md`의 아키텍처를 **완전히 대체**합니다.

### 유지되는 부분 (이전 연구에서)
- @mysten/dapp-kit 사용법 (Provider, Hooks, ConnectButton)
- @mysten/sui `verifyPersonalMessageSignature` 서명 검증
- 지원 지갑: Sui Wallet, Suiet, Phantom
- Nonce 기반 소유권 증명 패턴

### 제거/변경된 부분
- ~~커스텀 JWT 발급 (jsonwebtoken)~~ → Supabase Auth 네이티브 JWT
- ~~accessToken 옵션으로 Supabase 클라이언트 생성~~ → 쿠키 기반 세션 (@supabase/ssr)
- ~~지갑 로그인이 주 인증~~ → Google OAuth가 주 인증
- ~~localStorage에 토큰 저장~~ → 쿠키에 세션 저장 (SSR 호환)

---

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/shared/research/2026-02-12-sui-wallet-login-supabase-auth-research.md` - 이전 지갑 우선 인증 연구 (본 연구로 대체)
- `thoughts/shared/research/2026-01-22-whoosh-bang-architecture-research.md` - 전체 게임 아키텍처 연구
- `thoughts/shared/plans/2026-01-23-whoosh-bang-mvp-implementation.md` - MVP 구현 계획

---

## 관련 연구

- `thoughts/shared/research/2026-02-12-sui-wallet-login-supabase-auth-research.md` - 지갑 우선 인증 (대체됨)
- `thoughts/shared/research/2026-01-22-whoosh-bang-architecture-research.md` - 전체 게임 아키텍처

---

## 코드 참조

현재 코드베이스에서 인증 관련 파일은 없으며, 다음 위치에 구현 예정:

| 파일 (예정) | 용도 |
|------------|------|
| `apps/web/src/lib/supabase/client.ts` | 브라우저 Supabase 클라이언트 |
| `apps/web/src/lib/supabase/server.ts` | 서버 Supabase 클라이언트 |
| `apps/web/src/middleware.ts` | 토큰 갱신 + 라우트 보호 |
| `apps/web/src/app/auth/callback/route.ts` | OAuth 콜백 (PKCE) |
| `apps/web/src/app/login/page.tsx` | 로그인 페이지 |
| `apps/web/src/app/lobby/page.tsx` | 게임 로비 (지갑 연동 포함) |
| `apps/web/src/app/lobby/layout.tsx` | 로비 레이아웃 (dapp-kit Provider) |
| `apps/web/src/app/providers.tsx` | Sui dapp-kit Provider |
| `apps/web/src/app/api/wallet/nonce/route.ts` | 지갑 Nonce 발급 |
| `apps/web/src/app/api/wallet/verify/route.ts` | 지갑 서명 검증 + 저장 |
| `apps/web/src/components/auth/GoogleLoginButton.tsx` | Google 로그인 버튼 |
| `apps/web/src/components/auth/SessionHandler.tsx` | 세션 감지 |
| `apps/web/src/components/lobby/WalletLinkButton.tsx` | 지갑 연동 버튼 |

---

## 참고 자료

### Supabase Auth
- [Setting up Server-Side Auth for Next.js](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Creating a Supabase client for SSR](https://supabase.com/docs/guides/auth/server-side/creating-a-client)
- [Login with Google](https://supabase.com/docs/guides/auth/social-login/auth-google)
- [PKCE Flow](https://supabase.com/docs/guides/auth/sessions/pkce-flow)
- [Advanced SSR Guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)
- [Migrating to SSR from Auth Helpers](https://supabase.com/docs/guides/auth/server-side/migrating-to-ssr-from-auth-helpers)
- [Custom Access Token Hook](https://supabase.com/docs/guides/auth/auth-hooks/custom-access-token-hook)
- [Identity Linking](https://supabase.com/docs/guides/auth/auth-identity-linking)
- [JWT Claims Reference](https://supabase.com/docs/guides/auth/jwt-fields)
- [Managing User Data](https://supabase.com/docs/guides/auth/managing-user-data)
- [Custom Claims & RBAC](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)

### @mysten/dapp-kit & @mysten/sui
- [dApp Kit 공식 문서](https://sdk.mystenlabs.com/dapp-kit)
- [useSignPersonalMessage](https://sdk.mystenlabs.com/dapp-kit/wallet-hooks/useSignPersonalMessage)
- [Signature Verification](https://sdk.mystenlabs.com/typescript/cryptography/keypairs)
- [Wallet Standard](https://docs.sui.io/standards/wallet-standard)

### Web3 + 소셜 로그인 사례
- [Ronin Wallet + Web3Auth](https://blog.web3auth.io/the-story-of-web3auth-x-ronin-wallet-integration/) - 소셜 로그인 추가 후 전환율 15% → 62%
- [Alchemy Embedded Wallets Guide](https://www.alchemy.com/overviews/the-ultimate-guide-to-embedded-wallets-with-social-login)

---

## 후속 연구: getClaims() vs getUser() 비교

> 2026-02-12 추가 조사

### 개요

`getClaims()`는 2024년 10월 Supabase의 비대칭 JWT 서명 키 이니셔티브의 일환으로 도입되었다. `getUser()`보다 빠른 대안으로 설계되었으며, `@supabase/supabase-js`와 `@supabase/ssr` 모두에서 사용 가능하다.

### 핵심 차이

| 구분 | `getClaims()` | `getUser()` |
|------|---------------|-------------|
| **검증 방식** | JWT 서명 + 만료시간 (로컬 검증) | Auth 서버에 요청 (서버 검증) |
| **네트워크 요청** | 없음 (비대칭 키 사용 시) | 항상 Auth 서버 요청 |
| **로그아웃 감지** | ❌ 다른 기기에서 로그아웃해도 감지 불가 | ✅ 세션 상태 실시간 확인 |
| **성능** | 빠름 — 캐시된 JWKS + WebCrypto API | 느림 — 매번 DB 쿼리 |
| **데이터 최신성** | JWT 갱신 시점의 클레임만 반환 | `auth.users`에서 최신 데이터 반환 |
| **계정 삭제/차단 감지** | ❌ | ✅ |

### JWT 키 타입에 따른 동작

- **비대칭 키 (RS256, ES256)**: `getClaims()`가 `/.well-known/jwks.json`의 공개 키로 **로컬 검증**. 네트워크 요청 없음
- **대칭 키 (HS256, 레거시)**: `getClaims()`가 Auth 서버에 요청하여 검증. `getUser()`와 비슷한 성능

> 2025년 5월 이후 생성된 프로젝트는 기본적으로 RSA 비대칭 키 사용

### 사용 패턴 권장

```typescript
// ✅ MIDDLEWARE — getClaims() 사용 (빠른 라우트 보호)
// middleware.ts
const { data, error } = await supabase.auth.getClaims();
if (error || !data?.sub) {
  return NextResponse.redirect(new URL('/login', request.url));
}

// ✅ SERVER COMPONENT — getUser() 사용 (확실한 인증)
// app/lobby/page.tsx
const { data: { user }, error } = await supabase.auth.getUser();
if (error || !user) {
  redirect('/login');
}

// ✅ 민감한 API ROUTE — getUser() 필수
// app/api/wallet/verify/route.ts
const { data: { user }, error } = await supabase.auth.getUser();
if (error || !user) {
  return Response.json({ error: 'Unauthorized' }, { status: 401 });
}
```

### 우리 아키텍처에서의 적용

| 위치 | 메서드 | 이유 |
|------|--------|------|
| `middleware.ts` | `getClaims()` | 모든 요청에 실행되므로 성능 우선. 라우트 보호용 |
| Server Components (로비 등) | `getUser()` | 사용자 데이터 렌더링에 최신 정보 필요 |
| `/api/wallet/nonce` | `getUser()` | 지갑 연동은 민감한 작업 |
| `/api/wallet/verify` | `getUser()` | 서명 검증은 보안 최우선 |
| `/auth/callback` | `exchangeCodeForSession()` | OAuth 콜백은 코드 교환만 수행 |

### Supabase 공식 입장

> "Always use `supabase.auth.getClaims()` to protect pages and user data."

단, 다음 주의사항 포함:

> "The only way to ensure that a user has logged out or their session has ended is to get the user's details with `getUser()`."

### 참고 자료

- [getClaims() API Reference](https://supabase.com/docs/reference/javascript/auth-getclaims)
- [JWT Signing Keys](https://supabase.com/docs/guides/auth/signing-keys)
- [Advanced Server-Side Auth Guide](https://supabase.com/docs/guides/auth/server-side/advanced-guide)
- [GitHub Issue #40985 — getClaims vs getUser 명확화](https://github.com/supabase/supabase/issues/40985)

---

## 후속 연구: dapp-kit Provider 번들 사이즈 영향

> 2026-02-12 추가 조사

### 패키지 사이즈 (npm unpacked)

| 패키지 | Unpacked Size | 비고 |
|--------|--------------|------|
| `@mysten/dapp-kit` | 1,516 KB | Provider, ConnectButton, hooks |
| `@mysten/sui` | 5,288 KB | Sui SDK (peer dep) |
| `@tanstack/react-query` | 734 KB | 필수 peer dep (~10-12 KB gzipped) |
| `@vanilla-extract/css` | 352 KB | dapp-kit 내부 스타일링 |
| `zustand` | 95 KB | dapp-kit 상태 관리 |
| `@mysten/wallet-standard` | 106 KB | 지갑 표준 인터페이스 |
| **dapp-kit 합계** | **~8,167 KB** | |
| `@supabase/supabase-js` | 384 KB | 인증 클라이언트 |
| `@supabase/ssr` | 243 KB | SSR 지원 |
| **Supabase 합계** | **~627 KB** | |

> Unpacked Size ≠ 브라우저 번들. Tree-shaking + minification + gzip 후 60~80% 감소.

### 결론: 로비 레이아웃에만 배치

Next.js App Router는 route segment별로 자동 코드 스플릿하므로, `/lobby/layout.tsx`에 dapp-kit Provider를 배치하면 `/lobby/*` 경로 접근 시에만 로드된다.

- `/login` 페이지: Supabase만 로드 (~627 KB unpacked)
- `/lobby` 이하: Supabase + dapp-kit 로드 (~8.8 MB unpacked)
- `/game` 페이지: Supabase만 로드

추가 `next/dynamic` 최적화 가능하나, route segment 기반 자동 스플릿만으로 충분.

---

## 후속 연구: Custom Access Token Hook 성능 분석

> 2026-02-12 추가 조사

### Hook 실행 빈도

Custom Access Token Hook은 **모든 API 요청이 아니라 토큰 발급/갱신 시에만** 실행된다:

| 이벤트 | Hook 실행 |
|--------|----------|
| 로그인 (Google OAuth) | ✅ 실행 |
| 토큰 자동 갱신 (refresh) | ✅ 실행 |
| `getClaims()` 호출 | ❌ 미실행 (로컬 JWT 검증만) |
| `getUser()` 호출 | ❌ 미실행 (Auth 서버 조회만) |
| 일반 DB 쿼리 | ❌ 미실행 |

- **기본 JWT 만료 시간**: 1시간 (Supabase 기본값)
- **실제 실행 빈도**: 활성 유저당 **~1회/시간** (토큰 갱신 시)
- **Postgres Hook 타임아웃**: 2초

### 현재 쿼리 성능 분석

현재 Hook의 쿼리:
```sql
SELECT wallet_address FROM public.wallets
WHERE user_id = (event->>'user_id')::uuid AND verified = true;
```

- `user_id`에 `UNIQUE` 인덱스 → **B-tree 인덱스 O(log n) 조회**
- 단일 행 인덱스 조회 소요 시간: **~0.03 ~ 0.06ms**
- PostgreSQL `shared_buffers`가 자주 접근하는 행을 자동 캐시 (LRU)
- 활성 유저의 wallet 레코드는 **메모리에 이미 캐시**될 확률 높음 (캐시 적중률 97~99%)

**결론: 현재 쿼리는 성능 문제가 아니다.** 활성 유저당 시간당 1회, 0.06ms 이하.

### 대안 비교: 3가지 전략

#### 전략 A: 현재 Hook 유지 (권장)

```sql
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
STABLE  -- 트랜잭션 내 결과 캐싱 허용
AS $$
DECLARE
  claims jsonb;
  user_wallet text;
BEGIN
  claims := event->'claims';

  SELECT wallet_address INTO user_wallet
  FROM public.wallets
  WHERE user_id = (event->>'user_id')::uuid
    AND verified = true;

  IF user_wallet IS NOT NULL THEN
    claims := jsonb_set(claims, '{wallet_address}', to_jsonb(user_wallet));
    claims := jsonb_set(claims, '{chain}', '"sui"');
  END IF;

  event := jsonb_set(event, '{claims}', claims);
  RETURN event;
END;
$$;
```

| 항목 | 값 |
|------|-----|
| DB 쿼리/토큰 발급 | 1회 |
| 지연 시간 | ~0.03-0.06ms |
| 데이터 최신성 | 항상 최신 (토큰 갱신 시점) |
| 복잡도 | 낮음 |

#### 전략 B: app_metadata 사용 (Hook 제거)

지갑 연동/해제 시 `auth.users`의 `app_metadata`에 직접 저장:

```typescript
// 지갑 연동 시 (app/api/wallet/verify/route.ts에 추가)
await admin.auth.admin.updateUserById(user.id, {
  app_metadata: { wallet_address, chain: 'sui' }
});

// 지갑 해제 시 (app/api/wallet/unlink/route.ts에 추가)
await admin.auth.admin.updateUserById(user.id, {
  app_metadata: { wallet_address: null, chain: null }
});
```

- `app_metadata`는 JWT의 `app_metadata` 클레임에 **자동 포함**됨
- Custom Access Token Hook **완전 제거** 가능
- RLS에서 접근: `(auth.jwt() -> 'app_metadata' ->> 'wallet_address')`

| 항목 | 값 |
|------|-----|
| DB 쿼리/토큰 발급 | 0회 |
| 지연 시간 | 0ms |
| 데이터 최신성 | 연동/해제 시 즉시 반영 (다음 토큰 갱신부터) |
| 복잡도 | 매우 낮음 |

> **주의**: `app_metadata`는 서버(service_role)에서만 수정 가능 → 클라이언트 조작 불가 → 보안 안전

#### 전략 C: 하이브리드 (app_metadata + Hook 백업)

- `app_metadata`에 기본 저장 + Hook에서 불일치 시 덮어쓰기
- 과도한 복잡성 대비 이점 없음 → **비권장**

### 성능 비교 요약

| 전략 | DB 쿼리 | 지연 | 최신성 | 복잡도 |
|------|---------|------|--------|--------|
| **A: Hook 유지** | 1회/토큰 | ~0.06ms | 항상 최신 | 낮음 |
| **B: app_metadata** | 0회 | 0ms | 연동/해제 시 반영 | 매우 낮음 |
| **C: 하이브리드** | 0~1회 | 0~0.06ms | 항상 최신 | 높음 |

### 최종 권장: 전략 B (app_metadata)

우리 아키텍처에서 **전략 B가 최적**인 이유:

1. **지갑 주소 변경 빈도가 낮음** — 연동/해제는 드문 이벤트
2. **연동/해제 API에서 이미 service_role 사용** — `updateUserById()` 호출 추가 비용 미미
3. **Hook 완전 제거** — 관리 포인트 감소, 2초 타임아웃 우려 제거
4. **`app_metadata`는 클라이언트 조작 불가** — `user_metadata`와 달리 보안 안전
5. **wallets 테이블은 유지** — 상세 지갑 정보의 원본(source of truth)

**구현 변경점:**
- `/api/wallet/verify`: 지갑 저장 후 `admin.auth.admin.updateUserById()` 호출 추가
- `/api/wallet/unlink`: 지갑 삭제 후 `admin.auth.admin.updateUserById()` 호출 추가
- Custom Access Token Hook 함수: **삭제** (Supabase Dashboard에서 등록 해제)
- RLS 정책: `auth.jwt() ->> 'wallet_address'` → `auth.jwt() -> 'app_metadata' ->> 'wallet_address'`

### 참고 자료

- [Custom Access Token Hook](https://supabase.com/docs/guides/auth/auth-hooks/custom-access-token-hook)
- [updateUserById() API](https://supabase.com/docs/reference/javascript/auth-admin-updateuserbyid)
- [User sessions & JWT expiry](https://supabase.com/docs/guides/auth/sessions)
- [GitHub Discussion #30381 — app_metadata vs custom claims](https://github.com/orgs/supabase/discussions/30381)

---

## 미해결 질문

1. ~~**dapp-kit Provider 범위**~~ → **해결됨**: 로비 레이아웃에만 배치. Next.js App Router 자동 코드 스플릿 활용
2. ~~**Supabase getClaims() vs getUser()**~~ → **해결됨**: Middleware에서 `getClaims()`, API Route/Server Component에서 `getUser()` 사용
3. ~~**Custom Access Token Hook 성능**~~ → **해결됨**: Hook 대신 `app_metadata`에 wallet_address 저장. DB 쿼리 0회, Hook 제거. wallets 테이블은 원본 데이터로 유지
4. ~~**지갑 연동 해제 UX**~~ → **해결됨**: 미연동 유저는 "지갑 연동하기" 버튼, 연동 유저는 축약 주소 + "연동 해제" 버튼. DELETE `/api/wallet/unlink`로 처리
5. ~~**다중 지갑 지원**~~ → **해결됨**: 다중 지갑 미지원. 1 유저 = 1 지갑. `user_id UNIQUE` + `wallet_address UNIQUE`로 DB 레벨에서 보장. `is_primary` 컬럼 제거
6. ~~**지갑 중복 연동 방지**~~ → **해결됨**: 연동 시 해당 지갑이 다른 유저에 이미 연동되어 있으면 409 에러 반환. `wallet_address UNIQUE` 제약으로 DB 레벨 보장

**모든 미해결 질문 해결 완료.**

---

## 구현 후 변경사항 (2026-02-13)

구현 과정에서 연구 시점과 달라진 사항을 기록합니다.

### 1. Supabase Publishable Key (Legacy ANON_KEY 교체)

연구 시점에는 `NEXT_PUBLIC_SUPABASE_ANON_KEY` (JWT 기반 anon key)를 사용했으나, 구현 시 Supabase의 최신 **Publishable Key** 형식으로 전환:

```env
# 변경 전 (Legacy)
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJhbGci...

# 변경 후 (Publishable Key)
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_CXPMJCfDn5XATkw3fQSLkg_5mPjiKE6
```

- Publishable Key는 독립적으로 로테이션 가능하여 보안성 향상
- `client.ts`, `server.ts`, `proxy.ts`에서 모두 해당 키 참조

### 2. proxy.ts (Next.js 16 — middleware.ts 불가)

Next.js 16에서 `middleware.ts`가 `proxy.ts`로 변경됨:

| 연구 시점 | 실제 구현 |
|-----------|----------|
| `apps/web/src/middleware.ts` | `apps/web/src/proxy.ts` |

기능은 동일 (토큰 갱신 + 라우트 보호 + `getClaims()` 사용).

### 3. @mysten/dapp-kit v1.0.3 & @mysten/sui v2.4.0

연구 시점의 버전과 달리 최신 버전으로 구현:

| 패키지 | 연구 시점 | 실제 구현 |
|--------|----------|----------|
| `@mysten/dapp-kit` | v0.19.11 | **v1.0.3** |
| `@mysten/sui` | v2.1.0 | **v2.4.0** |

**주요 차이:**
- `getFullnodeUrl()` 제거됨 → 하드코딩 URL + `network` 프로퍼티 사용
- `createNetworkConfig`에 `network` 필드 필수

```typescript
// 실제 구현
const { networkConfig } = createNetworkConfig({
  mainnet: { url: 'https://fullnode.mainnet.sui.io:443', network: 'mainnet' },
  testnet: { url: 'https://fullnode.testnet.sui.io:443', network: 'testnet' },
});
```

### 4. 지갑 연동 UX — 1-step 플로우

연구/계획에서는 **2-step 플로우** (Connect → Link)를 설계했으나, 구현 후 **1-step 플로우**로 개선:

| 연구 시점 (2-step) | 최종 구현 (1-step) |
|-------------------|-------------------|
| 1. ConnectButton으로 지갑 연결 | 1. "Connect Wallet" 버튼 클릭 |
| 2. "지갑 연동하기" 버튼 클릭 → 서명 | 2. 다이얼로그에서 지갑 선택 |
| | 3. 연결 → 즉시 서명 → 검증 → 완료 |

**변경 이유:** UX 간소화 — 사용자가 별도의 "Link" 버튼을 클릭할 필요 없이, 지갑 선택 후 자동으로 서명/검증 진행.

### 5. 커스텀 지갑 선택 다이얼로그

dapp-kit의 `<ConnectButton />`을 사용하지 않고, `useWallets()` + `useConnectWallet()` 훅을 사용하여 게임 테마에 맞는 커스텀 지갑 선택 다이얼로그 구현:

- shadcn Dialog 컴포넌트 활용
- `wallet.icon` (base64 data URI) + `wallet.name`으로 지갑 목록 표시
- 브라우저에 설치된 Sui 지갑 자동 감지

### 6. 로비 UI 리디자인

연구/계획의 단순 레이아웃에서 게임 스타일 풀스크린 로비로 변경:

**새 컴포넌트:**
- `ProfileCard.tsx` — 아바타, 이름, 레벨, XP 프로그레스 바, 전적
- `CharacterDisplay.tsx` — 떠다니는 캐릭터 (CSS float 애니메이션)
- `PlayButton.tsx` — 글로우 + 확대/축소 애니메이션 PLAY 버튼
- `lib/level.ts` — XP 레벨 계산 유틸리티

**테마:**
- 다크 게임 테마 (OKLch 컬러 스페이스)
- Primary 색상: 퍼플 `oklch(0.59 0.20 277)`
- CSS @keyframes 애니메이션 3개: `pulse-glow`, `float`, `scale-pulse`

### 7. Unlink 시 지갑 연결도 해제

`unlinkWallet()`에서 서버 연동 해제 + `disconnectWallet()` 호출을 함께 처리하여, 연동 해제 시 dapp-kit 지갑 연결도 함께 끊김. 연동 실패 시에도 `disconnectWallet()` 호출하여 깔끔한 상태 복원.

### 8. 최종 파일 구조

```
apps/web/src/
├── app/
│   ├── globals.css                    ← 테마 + 애니메이션
│   ├── layout.tsx                     ← SessionHandler
│   ├── page.tsx                       ← 로그인 (Google OAuth)
│   ├── auth/
│   │   ├── callback/route.ts          ← OAuth 콜백 (PKCE)
│   │   └── auth-code-error/page.tsx   ← 인증 오류
│   ├── lobby/
│   │   ├── layout.tsx                 ← SuiProviders
│   │   └── page.tsx                   ← 서버 컴포넌트 (프로필/지갑 fetch)
│   ├── game/
│   │   └── page.tsx                   ← 게임 플레이
│   └── api/wallet/
│       ├── nonce/route.ts             ← Nonce 발급
│       ├── verify/route.ts            ← 서명 검증 + 저장
│       └── unlink/route.ts            ← 연동 해제
├── components/
│   ├── auth/
│   │   ├── GoogleLoginButton.tsx
│   │   └── SessionHandler.tsx
│   ├── lobby/
│   │   ├── LobbyContent.tsx           ← 풀스크린 레이아웃
│   │   ├── ProfileCard.tsx            ← 프로필 카드
│   │   ├── CharacterDisplay.tsx       ← 캐릭터 디스플레이
│   │   ├── PlayButton.tsx             ← PLAY 버튼
│   │   └── WalletLinkButton.tsx       ← 1-step 지갑 연동
│   ├── providers/
│   │   └── SuiProviders.tsx           ← dapp-kit Provider
│   └── game/
│       ├── GameWrapper.tsx
│       └── PhaserGame.tsx
├── lib/
│   ├── level.ts                       ← XP 레벨 유틸리티
│   ├── utils.ts
│   └── supabase/
│       ├── client.ts                  ← 브라우저 클라이언트 (Publishable Key)
│       └── server.ts                  ← 서버 클라이언트
└── proxy.ts                           ← 라우트 보호 (Next.js 16)
```

### 9. 환경 변수 (최종)

```env
NEXT_PUBLIC_SUPABASE_URL=https://fmwjmsfgxhbzhvtnaqta.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_CXPMJCfDn5XATkw3fQSLkg_5mPjiKE6
SUPABASE_SERVICE_ROLE_KEY=eyJ...
NEXT_PUBLIC_SUI_NETWORK=testnet
```
