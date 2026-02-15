---
date: 2026-02-12T18:51:00+09:00
researcher: arta1069@gmail.com
topic: "Sui 지갑 로그인(@mysten/dapp-kit, @mysten/sui) 및 Supabase 회원 관리 통합 연구"
tags: [research, sui, web3, wallet, dapp-kit, supabase, auth, jwt, multiplayer]
status: superseded
last_updated: 2026-02-13
last_updated_by: arta1069@gmail.com
last_updated_note: "이 문서는 완전히 대체됨 — 소셜 로그인 우선 아키텍처로 전환. 아래 문서 참조"
superseded_by: "thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md"
---

# [SUPERSEDED] 연구: Sui 지갑 로그인 및 Supabase 회원 관리 통합

> **이 문서는 대체되었습니다.** 최신 아키텍처는 `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md`를 참조하세요.
> 변경 사유: "지갑 로그인 우선" → "소셜 로그인(Google OAuth) 우선 + 선택적 지갑 연동" 아키텍처로 전환.

**날짜**: 2026-02-12T18:51:00+09:00
**연구자**: arta1069@gmail.com
**리포지토리**: worms-game

## 연구 질문

@mysten/dapp-kit과 @mysten/sui를 활용하여 Sui 기반 Web3 게임에서 지갑 로그인을 구현하고, 추후 멀티플레이어를 위해 Supabase에 회원 관리까지 통합하는 방법 연구

## 요약

### 핵심 결론

1. **현재 코드베이스**: 인증/지갑 관련 구현 없음. 모든 것이 계획 단계
2. **@mysten/dapp-kit (v0.19.11)**: 현재 사용 가능한 유일한 패키지. v2(`dapp-kit-core` + `dapp-kit-react`)는 계획 중이나 **npm 미출시** — 현재 버전으로 구현 후 v2 출시 시 마이그레이션 예정
3. **@mysten/sui v2.1.0**: `verifyPersonalMessageSignature`로 서버 측 서명 검증 가능
4. **지원 지갑**: Sui Wallet, Suiet Wallet, Phantom Wallet (Sui Wallet Standard 자동 감지)
5. **Supabase 통합**: Sui는 Supabase의 네이티브 Web3 인증에 포함되지 않음 → **커스텀 JWT 방식** 필수
6. **권장 플로우**: 지갑 서명 → 서버 검증 → 커스텀 JWT 발급 → Supabase `accessToken` 옵션으로 세션 관리
7. **범위**: 이번 구현은 **지갑 로그인만** 포함. zkLogin(소셜 → Sui 주소)은 별도 단계에서 구현 예정

### 인증 플로우 개요

```
사용자 → 지갑 연결(dapp-kit) → 메시지 서명 → 서버 검증(@mysten/sui/verify)
  → Supabase 사용자 조회/생성 → 커스텀 JWT 발급(SUPABASE_JWT_SECRET)
  → 클라이언트에 JWT 전달 → Supabase Client accessToken 설정
  → RLS 정책으로 데이터 접근 제어
```

---

## 상세 발견 사항

### 1. @mysten/dapp-kit — 현재 API 및 지갑 연결 패턴

#### 패키지 현황 및 v2 마이그레이션 계획

| 패키지 | 상태 | 비고 |
|--------|------|------|
| `@mysten/dapp-kit` | **현재 사용** (v0.19.11) | npm에 출시된 유일한 패키지. 활발히 유지보수 중 |
| `@mysten/dapp-kit-core` | 계획 중 (npm 미출시) | Framework-agnostic 코어 ([Discussion #221](https://github.com/MystenLabs/ts-sdks/discussions/221)) |
| `@mysten/dapp-kit-react` | 계획 중 (npm 미출시) | React 전용 바인딩. 출시 시점 미정 |

> **중요**: 공식 문서에서 `dapp-kit-react` 사용을 권장하는 문구가 있으나, 2026-02 기준 **npm에 출시되지 않았습니다**. 현재는 `@mysten/dapp-kit`만 사용 가능합니다. v2 출시 시 import 경로만 변경되고 API는 대부분 동일할 것으로 예상되므로, 현재 버전으로 구현 후 마이그레이션이 용이합니다.

**설치**:
```bash
npm install @mysten/dapp-kit @mysten/sui @tanstack/react-query
```

**v2 출시 후 예상 마이그레이션**:
```bash
# v2 출시 시 (미래)
npm uninstall @mysten/dapp-kit
npm install @mysten/dapp-kit-react @mysten/sui
# import 경로만 '@mysten/dapp-kit' → '@mysten/dapp-kit-react'로 변경
```

#### Provider 설정 (Next.js App Router)

```typescript
// app/providers.tsx
'use client';

import { createNetworkConfig, SuiClientProvider, WalletProvider } from '@mysten/dapp-kit';
import { getFullnodeUrl } from '@mysten/sui/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import '@mysten/dapp-kit/dist/index.css'; // 기본 스타일

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

```typescript
// app/layout.tsx
import { SuiProviders } from './providers';

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html>
      <body>
        <SuiProviders>{children}</SuiProviders>
      </body>
    </html>
  );
}
```

**SSR 주의사항**:
- `'use client'` 디렉티브 필수
- dApp Kit의 모든 hooks/components는 클라이언트 전용
- `next/dynamic`으로 SSR 비활성화 불필요 (Provider만 `'use client'`면 충분)

#### 주요 Hooks

| Hook | 용도 |
|------|------|
| `useCurrentAccount` | 현재 선택된 계정(주소) 조회 |
| `useAccounts` | 연결된 모든 계정 목록 |
| `useConnectWallet` | 지갑 연결 (mutation) |
| `useDisconnectWallet` | 지갑 연결 해제 (mutation) |
| `useCurrentWallet` | 현재 연결된 지갑 정보 |
| `useWallets` | 사용 가능한 모든 지갑 목록 |
| `useSignPersonalMessage` | 개인 메시지 서명 (인증용) |
| `useSignTransaction` | 트랜잭션 서명 |
| `useSignAndExecuteTransaction` | 트랜잭션 서명 + 실행 |
| `useAutoConnectWallet` | 자동 지갑 연결 상태 |

#### ConnectButton 컴포넌트

```typescript
import { ConnectButton, useCurrentAccount } from '@mysten/dapp-kit';

function WalletStatus() {
  const account = useCurrentAccount();

  return (
    <div>
      <ConnectButton />
      {account && <p>Connected: {account.address}</p>}
    </div>
  );
}
```

#### 지원 지갑

Sui Wallet Standard를 구현한 모든 지갑이 `WalletProvider`에 의해 자동 검색됨. 최소 지원 대상:

| 지갑 | Sui 지원 | signPersonalMessage | 비고 |
|------|---------|-------------------|------|
| **Sui Wallet** | ✅ 공식 | ✅ | Mysten Labs 개발 |
| **Suiet Wallet** | ✅ | ✅ | Sui 전용 지갑 |
| **Phantom Wallet** | ✅ (2025-05 정식) | ✅ (Wallet Standard 준수) | Solana/ETH/Sui 멀티체인 지갑, ~1500만 사용자 |

`preferredWallets` 설정으로 이 세 지갑을 우선 표시:
```typescript
<WalletProvider
  preferredWallets={['Sui Wallet', 'Suiet', 'Phantom']}
  autoConnect
>
```

> **Phantom Sui 지원 타임라인**: 2024-12 발표 → 2025-01 베타 → 2025-05 정식 출시. Sui Wallet Standard를 완전히 구현하여 dApp Kit에서 자동 감지됨.

---

### 2. @mysten/sui — 서명 검증 및 클라이언트 설정

#### 패키지 정보

- **최신 버전**: 2.1.0 (2026-02-08 기준)
- **이전 이름**: `@mysten/sui.js` → `@mysten/sui` (v1.0부터)
- **프로토콜 지원**: gRPC (권장), GraphQL, JSON RPC (deprecated)

#### verifyPersonalMessageSignature (핵심 API)

```typescript
import { verifyPersonalMessageSignature } from '@mysten/sui/verify';

// 함수 시그니처
verifyPersonalMessageSignature(
  message: Uint8Array,
  signature: string,
  options?: {
    address?: string;           // 특정 주소와 매칭 검증
  }
): Promise<PublicKey>
```

**서버 측 검증 예시 (Next.js API Route)**:
```typescript
// app/api/auth/verify/route.ts
import { verifyPersonalMessageSignature } from '@mysten/sui/verify';

export async function POST(req: Request) {
  const { address, message, signature } = await req.json();

  try {
    const messageBytes = new TextEncoder().encode(message);

    // 서명 검증 + 주소 확인
    await verifyPersonalMessageSignature(messageBytes, signature, {
      address,
    });

    return Response.json({ verified: true });
  } catch (error) {
    return Response.json({ verified: false }, { status: 401 });
  }
}
```

**Edge Runtime 호환**: @mysten/sui SDK는 Vercel Edge, Cloudflare Workers 등과 호환. 순수 암호화 연산이므로 서버리스 환경에서 문제없음.

#### SuiClient 설정

```typescript
import { SuiClient, getFullnodeUrl } from '@mysten/sui/client';

const client = new SuiClient({ url: getFullnodeUrl('mainnet') });

// 네트워크 엔드포인트
// localnet:  http://127.0.0.1:9000
// devnet:    https://fullnode.devnet.sui.io:443
// testnet:   https://fullnode.testnet.sui.io:443
// mainnet:   https://fullnode.mainnet.sui.io:443
```

---

### 3. Sui 지갑 인증 + Supabase Auth 통합

#### Supabase Web3 인증 현황

| 체인 | Supabase 네이티브 지원 | 구현 방법 |
|------|----------------------|-----------|
| Ethereum | ✅ EIP-4361 | `supabase.auth.signInWithWeb3()` |
| Solana | ✅ | `supabase.auth.signInWithWeb3()` |
| **Sui** | ❌ 미지원 | **커스텀 JWT 필요** |

> Sui는 Supabase의 네이티브 Web3 인증에 포함되지 않으므로, 커스텀 JWT 기반의 인증을 구현해야 합니다.

#### 커스텀 JWT 통합 전략

**전체 인증 플로우**:

```
┌──────────────┐    ┌──────────────────┐    ┌───────────────────┐
│  Frontend    │    │ Backend (API)    │    │    Supabase       │
│  (dapp-kit)  │    │ (Server Actions) │    │                   │
└──────┬───────┘    └────────┬─────────┘    └─────────┬─────────┘
       │                     │                        │
  1. 지갑 연결               │                        │
  2. 메시지 서명 요청         │                        │
       │                     │                        │
  3. {address, msg, sig} ──→ │                        │
       │                     │                        │
       │     4. verifyPersonalMessageSignature         │
       │                     │                        │
       │     5. 사용자 조회 ───────────────────────→  │
       │                     │                        │
       │     6. 없으면 생성 ──────────────────────→   │
       │                     │                        │
       │     7. JWT 생성 (SUPABASE_JWT_SECRET)         │
       │                     │                        │
  8. ←── {token, user}       │                        │
       │                     │                        │
  9. Supabase Client에                                │
     accessToken 설정        │                        │
       │                     │                        │
  10. RLS 보호 데이터 접근 ─────────────────────────→ │
       │                     │                        │
```

#### Supabase 클라이언트 설정 (커스텀 JWT)

```typescript
import { createClient } from '@supabase/supabase-js';

// 방법 1: accessToken 함수 사용 (권장)
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    accessToken: async () => {
      const token = localStorage.getItem('sui_auth_token');
      return token || '';
    },
  }
);

// 방법 2: 글로벌 헤더 설정
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    global: {
      headers: {
        Authorization: `Bearer ${customJWT}`,
      },
    },
  }
);
```

#### JWT 필수 클레임 (Supabase 호환)

커스텀 JWT가 Supabase RLS와 호환되려면 다음 클레임이 필수:

```typescript
{
  sub: "user-uuid",              // 사용자 ID (UUID)
  aud: "authenticated",          // 필수: "authenticated"
  role: "authenticated",         // 필수: "authenticated"
  iss: "https://xxx.supabase.co/auth/v1",  // Supabase URL + /auth/v1
  iat: 1234567890,               // 발급 시간
  exp: 1234567890,               // 만료 시간
  // 커스텀 클레임
  wallet_address: "0x...",
  chain: "sui",
}
```

---

### 4. 구현 코드 예시

#### 4.1 클라이언트: 지갑 인증 Hook

```typescript
// hooks/useWalletAuth.ts
'use client';

import { useCurrentAccount, useSignPersonalMessage } from '@mysten/dapp-kit';
import { useState, useCallback } from 'react';

interface AuthState {
  token: string | null;
  user: { id: string; wallet_address: string } | null;
  isLoading: boolean;
  error: string | null;
}

export function useWalletAuth() {
  const account = useCurrentAccount();
  const { mutateAsync: signMessage } = useSignPersonalMessage();
  const [authState, setAuthState] = useState<AuthState>({
    token: null,
    user: null,
    isLoading: false,
    error: null,
  });

  const authenticate = useCallback(async () => {
    if (!account) throw new Error('지갑이 연결되지 않았습니다');

    setAuthState(prev => ({ ...prev, isLoading: true, error: null }));

    try {
      // 1. Nonce 요청
      const nonceRes = await fetch('/api/auth/nonce', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ address: account.address }),
      });
      const { nonce } = await nonceRes.json();

      // 2. 메시지 서명
      const message = `Sign in to Worms Game\nAddress: ${account.address}\nNonce: ${nonce}\nTimestamp: ${Date.now()}`;
      const { signature, bytes } = await signMessage({
        message: new TextEncoder().encode(message),
      });

      // 3. 서버에 검증 요청
      const verifyRes = await fetch('/api/auth/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          address: account.address,
          signature,
          message,
          nonce,
        }),
      });

      if (!verifyRes.ok) throw new Error('서명 검증 실패');

      const { token, user } = await verifyRes.json();

      // 4. 토큰 저장
      localStorage.setItem('sui_auth_token', token);
      setAuthState({ token, user, isLoading: false, error: null });

      return { token, user };
    } catch (error) {
      const msg = error instanceof Error ? error.message : '인증 실패';
      setAuthState(prev => ({ ...prev, isLoading: false, error: msg }));
      throw error;
    }
  }, [account, signMessage]);

  const logout = useCallback(() => {
    localStorage.removeItem('sui_auth_token');
    setAuthState({ token: null, user: null, isLoading: false, error: null });
  }, []);

  return {
    authenticate,
    logout,
    account,
    ...authState,
  };
}
```

#### 4.2 서버: Nonce 발급 API

```typescript
// app/api/auth/nonce/route.ts
import { createClient } from '@supabase/supabase-js';
import { randomBytes } from 'crypto';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(req: Request) {
  const { address } = await req.json();

  if (!address) {
    return Response.json({ error: 'Address required' }, { status: 400 });
  }

  const nonce = randomBytes(32).toString('hex');
  const expiresAt = new Date(Date.now() + 5 * 60 * 1000); // 5분 후 만료

  // Supabase에 nonce 저장
  await supabase.from('auth_nonces').insert({
    wallet_address: address,
    nonce,
    expires_at: expiresAt.toISOString(),
  });

  return Response.json({ nonce });
}
```

#### 4.3 서버: 서명 검증 및 JWT 발급 API

```typescript
// app/api/auth/verify/route.ts
import { verifyPersonalMessageSignature } from '@mysten/sui/verify';
import { createClient } from '@supabase/supabase-js';
import jwt from 'jsonwebtoken';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

export async function POST(req: Request) {
  const { address, signature, message, nonce } = await req.json();

  try {
    // 1. Nonce 검증
    const { data: nonceRecord, error: nonceError } = await supabase
      .from('auth_nonces')
      .select('*')
      .eq('wallet_address', address)
      .eq('nonce', nonce)
      .gt('expires_at', new Date().toISOString())
      .single();

    if (nonceError || !nonceRecord) {
      return Response.json({ error: 'Invalid or expired nonce' }, { status: 401 });
    }

    // Nonce 삭제 (일회용)
    await supabase.from('auth_nonces').delete().eq('id', nonceRecord.id);

    // 2. Sui 서명 검증
    const messageBytes = new TextEncoder().encode(message);
    await verifyPersonalMessageSignature(messageBytes, signature, {
      address,
    });

    // 3. 사용자 조회 또는 생성
    let { data: user } = await supabase
      .from('player_profiles')
      .select('*')
      .eq('wallet_address', address)
      .single();

    if (!user) {
      const { data: newUser } = await supabase
        .from('player_profiles')
        .insert({
          wallet_address: address,
          auth_provider: 'sui_wallet',
          username: `player_${address.slice(0, 8)}`,
        })
        .select()
        .single();
      user = newUser;
    }

    // 4. Supabase 호환 JWT 발급
    const token = jwt.sign(
      {
        wallet_address: address,
        chain: 'sui',
        aud: 'authenticated',
        role: 'authenticated',
        iss: `${process.env.NEXT_PUBLIC_SUPABASE_URL}/auth/v1`,
      },
      process.env.SUPABASE_JWT_SECRET!,
      {
        subject: user.id,
        expiresIn: '7d',
      }
    );

    return Response.json({ token, user });
  } catch (error) {
    console.error('Auth verification failed:', error);
    return Response.json({ error: 'Verification failed' }, { status: 401 });
  }
}
```

#### 4.4 인증 UI 컴포넌트

```typescript
// components/auth/WalletLoginButton.tsx
'use client';

import { ConnectButton, useCurrentAccount } from '@mysten/dapp-kit';
import { useWalletAuth } from '@/hooks/useWalletAuth';

export function WalletLoginButton() {
  const account = useCurrentAccount();
  const { authenticate, logout, user, isLoading, error } = useWalletAuth();

  // 지갑 미연결 상태
  if (!account) {
    return <ConnectButton />;
  }

  // 인증 완료 상태
  if (user) {
    return (
      <div>
        <span>{user.username}</span>
        <span>{account.address.slice(0, 6)}...{account.address.slice(-4)}</span>
        <button onClick={logout}>로그아웃</button>
      </div>
    );
  }

  // 지갑 연결됨, 인증 필요
  return (
    <div>
      <button onClick={authenticate} disabled={isLoading}>
        {isLoading ? '서명 중...' : '지갑으로 로그인'}
      </button>
      {error && <p>{error}</p>}
    </div>
  );
}
```

---

### 5. Supabase 데이터베이스 스키마 (회원 관리)

#### 테이블 구조

```sql
-- 인증 Nonce 테이블 (일회용 서명 검증)
CREATE TABLE auth_nonces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_address TEXT NOT NULL,
  nonce TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_nonces_wallet ON auth_nonces(wallet_address, nonce);

-- 자동 만료 처리 (pg_cron 또는 Supabase Edge Function으로)
-- DELETE FROM auth_nonces WHERE expires_at < NOW();

-- player_profiles 테이블 (기존 연구 스키마 확장)
CREATE TABLE player_profiles (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  wallet_address TEXT UNIQUE,
  auth_provider TEXT NOT NULL DEFAULT 'sui_wallet', -- 'sui_wallet' (추후 확장 가능)
  username TEXT UNIQUE NOT NULL,
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

-- 사용자 지갑 매핑 (다중 지갑 지원)
CREATE TABLE user_wallets (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id UUID REFERENCES player_profiles(id) ON DELETE CASCADE,
  chain TEXT NOT NULL DEFAULT 'sui',
  wallet_address TEXT NOT NULL,
  is_primary BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(wallet_address, chain)
);

CREATE INDEX idx_user_wallets_address ON user_wallets(wallet_address);
CREATE INDEX idx_user_wallets_user ON user_wallets(user_id);
```

#### RLS 정책 (커스텀 JWT 호환)

```sql
-- RLS 활성화
ALTER TABLE player_profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE user_wallets ENABLE ROW LEVEL SECURITY;

-- 프로필 조회: 모든 인증된 사용자 가능
CREATE POLICY "profiles_select"
ON player_profiles FOR SELECT TO authenticated
USING (true);

-- 프로필 수정: 자신만 가능 (JWT의 sub 클레임으로 확인)
CREATE POLICY "profiles_update_own"
ON player_profiles FOR UPDATE TO authenticated
USING (id = (auth.jwt() ->> 'sub')::uuid)
WITH CHECK (id = (auth.jwt() ->> 'sub')::uuid);

-- 지갑 주소로 자신의 데이터 접근
CREATE POLICY "profiles_select_by_wallet"
ON player_profiles FOR SELECT TO authenticated
USING (wallet_address = (auth.jwt() ->> 'wallet_address'));

-- user_wallets: 자신의 지갑만 관리
CREATE POLICY "wallets_select_own"
ON user_wallets FOR SELECT TO authenticated
USING (user_id = (auth.jwt() ->> 'sub')::uuid);

CREATE POLICY "wallets_insert_own"
ON user_wallets FOR INSERT TO authenticated
WITH CHECK (user_id = (auth.jwt() ->> 'sub')::uuid);
```

#### Custom Access Token Hook (추후 소셜 로그인 병행 시)

추후 Supabase Auth 네이티브 소셜 로그인을 병행할 경우, Custom Access Token Hook으로 JWT에 지갑 정보 추가 가능:

```sql
CREATE OR REPLACE FUNCTION public.custom_access_token_hook(event jsonb)
RETURNS jsonb
LANGUAGE plpgsql
STABLE
AS $$
DECLARE
  claims jsonb;
  wallet_addr text;
BEGIN
  SELECT wallet_address INTO wallet_addr
  FROM public.player_profiles
  WHERE id = (event->>'user_id')::uuid;

  claims := event->'claims';

  IF wallet_addr IS NOT NULL THEN
    claims := jsonb_set(claims, '{wallet_address}', to_jsonb(wallet_addr));
    claims := jsonb_set(claims, '{chain}', '"sui"');
  END IF;

  event := jsonb_set(event, '{claims}', claims);
  RETURN event;
END;
$$;

GRANT EXECUTE ON FUNCTION public.custom_access_token_hook TO supabase_auth_admin;
REVOKE EXECUTE ON FUNCTION public.custom_access_token_hook FROM authenticated, anon, public;
```

---

### 6. 현재 구현 범위 및 추후 확장

#### 현재 구현: 지갑 로그인만

| 시나리오 | 상태 | 비고 |
|----------|------|------|
| **Sui 지갑 로그인** | ✅ 이번 구현 | 커스텀 JWT 기반 |
| 소셜(Google/Kakao) 로그인 | 🔮 추후 | Supabase Auth 네이티브 |
| zkLogin (소셜 → Sui 주소) | 🔮 추후 | Salt Service, ZK Proof 등 별도 인프라 필요 |
| 소셜 로그인 후 지갑 연결 | 🔮 추후 | 계정 링킹 API |

> **참고**: zkLogin 및 소셜 로그인 통합은 별도 연구에서 다룰 예정. 이번 구현은 지갑 서명 기반 인증에만 집중합니다.

---

### 7. 환경 변수 설정

```env
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...
SUPABASE_SERVICE_ROLE_KEY=eyJ...
SUPABASE_JWT_SECRET=your-supabase-jwt-secret

# Sui 설정
NEXT_PUBLIC_SUI_NETWORK=testnet  # 또는 mainnet
```

**SUPABASE_JWT_SECRET 확인 방법**:
Supabase Dashboard → Project Settings → API → JWT Secret

---

### 8. 필요한 의존성 목록

```json
{
  "dependencies": {
    "@mysten/dapp-kit": "^0.19.11",
    "@mysten/sui": "^2.1.0",
    "@tanstack/react-query": "^5.x.x",
    "@supabase/supabase-js": "^2.x.x",
    "@supabase/ssr": "^0.x.x",
    "jsonwebtoken": "^9.x.x"
  },
  "devDependencies": {
    "@types/jsonwebtoken": "^9.x.x"
  }
}
```

---

## 아키텍처 문서화

### 인증 아키텍처 다이어그램

```
┌─────────────────────────────────────────────────────────────────┐
│                     Next.js App (Client)                        │
│                                                                 │
│  ┌───────────────┐  ┌────────────────┐  ┌────────────────────┐ │
│  │ @mysten/      │  │ Supabase       │  │ React UI           │ │
│  │ dapp-kit      │  │ Client         │  │ (Game + Auth)      │ │
│  │               │  │                │  │                    │ │
│  │ - ConnectBtn  │  │ - accessToken  │  │ - WalletLogin      │ │
│  │ - useSign...  │  │ - RLS queries  │  │ - ProfileMenu      │ │
│  │ - useAccount  │  │ - Realtime     │  │ - GameLobby        │ │
│  └───────┬───────┘  └───────┬────────┘  └────────────────────┘ │
│          │                  │                                    │
└──────────┼──────────────────┼────────────────────────────────────┘
           │                  │
           ▼                  ▼
┌──────────────────┐  ┌─────────────────────────────────────────┐
│ Sui Blockchain   │  │              Supabase                   │
│                  │  │                                         │
│ - Wallet Sign    │  │  ┌──────────┐  ┌──────────────────────┐│
│ - Verify Sig     │  │  │ Auth     │  │ PostgreSQL           ││
│ - Transactions   │  │  │ (JWT)    │  │ - player_profiles    ││
│                  │  │  └──────────┘  │ - user_wallets       ││
│                  │  │                │ - auth_nonces         ││
│                  │  │  ┌──────────┐  │ - game_lobbies       ││
│                  │  │  │ Realtime │  │ - match_history      ││
│                  │  │  │ (Games)  │  └──────────────────────┘│
│                  │  │  └──────────┘                           │
│                  │  │                                         │
│                  │  │  ┌──────────────────────────────────┐  │
│                  │  │  │ Edge Functions                    │  │
│                  │  │  │ - AI 턴 처리                      │  │
│                  │  │  │ - 매치 종료 처리                   │  │
│                  │  │  └──────────────────────────────────┘  │
└──────────────────┘  └─────────────────────────────────────────┘
```

### Server Actions vs API Routes

| 방식 | 장점 | 단점 | 용도 |
|------|------|------|------|
| **API Routes** | 외부 접근 가능, 표준 REST | 보일러플레이트 | 지갑 인증 (외부 서명 검증) |
| **Server Actions** | 간결, 자동 보안 | 내부 전용 | 게임 로직, 매칭, 턴 처리 |

**권장**: 인증 관련은 API Routes, 게임 로직은 Server Actions 사용

---

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` - 전체 아키텍처 연구
- `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md` - MVP 구현 계획

본 연구는 **지갑 로그인만** 다루며, zkLogin 및 소셜 인증은 별도 연구에서 진행 예정.

---

## 관련 연구

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` - 전체 게임 아키텍처
- `thoughts/shared/research/2026-02-11-parallax-background-implementation-research.md` - 배경 구현

---

## 코드 참조

현재 코드베이스에서 인증 관련 파일은 없으며, 다음 위치에 구현 예정:

| 파일 (예정) | 용도 |
|------------|------|
| `apps/web/src/app/providers.tsx` | Sui + Supabase Provider 설정 |
| `apps/web/src/hooks/useWalletAuth.ts` | 지갑 인증 Hook |
| `apps/web/src/app/api/auth/nonce/route.ts` | Nonce 발급 API |
| `apps/web/src/app/api/auth/verify/route.ts` | 서명 검증 + JWT 발급 API |
| `apps/web/src/components/auth/WalletLoginButton.tsx` | 지갑 로그인 UI |
| `apps/web/src/lib/supabase/client.ts` | Supabase 클라이언트 (커스텀 JWT) |
| `apps/web/src/lib/supabase/server.ts` | Supabase 서버 클라이언트 |

---

## 참고 자료

### @mysten/dapp-kit
- [공식 문서](https://sdk.mystenlabs.com/dapp-kit)
- [npm](https://www.npmjs.com/package/@mysten/dapp-kit)
- [WalletProvider 문서](https://sdk.mystenlabs.com/dapp-kit/wallet-provider)
- [ConnectButton 문서](https://sdk.mystenlabs.com/dapp-kit/wallet-components/ConnectButton)
- [useSignPersonalMessage](https://sdk.mystenlabs.com/dapp-kit/wallet-hooks/useSignPersonalMessage)
- [GitHub - MystenLabs/ts-sdks](https://github.com/MystenLabs/ts-sdks)
- [Build React Apps on Sui](https://blog.sui.io/react-apps-dapp-kit/)

### @mysten/sui
- [공식 문서](https://sdk.mystenlabs.com/typescript)
- [npm](https://www.npmjs.com/package/@mysten/sui)
- [Key Pairs & Signature Verification](https://sdk.mystenlabs.com/typescript/cryptography/keypairs)
- [Migration to v1.0](https://sdk.mystenlabs.com/typescript/migrations/sui-1.0)

### Supabase
- [Web3 Authentication](https://supabase.com/docs/guides/auth/auth-web3)
- [Custom JWT](https://supabase.com/docs/guides/auth/jwts)
- [JWT Claims Reference](https://supabase.com/docs/guides/auth/jwt-fields)
- [Custom Access Token Hook](https://supabase.com/docs/guides/auth/auth-hooks/custom-access-token-hook)
- [Custom Claims & RBAC](https://supabase.com/docs/guides/database/postgres/custom-claims-and-role-based-access-control-rbac)
- [Third-Party Auth](https://supabase.com/docs/guides/auth/third-party/overview)
- [Securing Edge Functions](https://supabase.com/docs/guides/functions/auth)
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)

### Web3 + Supabase 통합 예시 (다른 체인)
- [Solana + Supabase 가이드](https://solana.com/developers/guides/getstarted/supabase-auth-guide)
- [Moralis + Supabase](https://docs.moralis.com/authentication-api/evm/integrations/supabase-nodejs)
- [Picket + Supabase](https://docs.picketapi.com/picket-docs/reference/integrations/supabase)

### Sui 블록체인
- [Wallet Standard](https://docs.sui.io/standards/wallet-standard)
- [Transaction Authentication](https://docs.sui.io/concepts/cryptography/transaction-auth/signatures)

### 지갑
- [Phantom Wallet Sui 지원 발표](https://phantom.com/learn/blog/introducing-sui-on-phantom)
- [Phantom Sui 정식 출시](https://blog.sui.io/phantom-wallet-live/)
- [Phantom Sui 시작 가이드](https://help.phantom.com/hc/en-us/articles/37534047697043-Get-started-with-Sui-in-Phantom)
- [dApp Kit v2 Discussion](https://github.com/MystenLabs/ts-sdks/discussions/221)

---

## 미해결 질문

1. **토큰 갱신 전략**: 커스텀 JWT의 만료 시 갱신 메커니즘 (refresh token 패턴 필요 여부)
2. **Supabase Third-Party Auth 통합**: 커스텀 JWT 대신 Supabase의 Third-Party Auth 기능(Firebase Auth 패턴)을 활용할 수 있는지 추가 검토
3. **Phantom signPersonalMessage**: Phantom이 Sui Wallet Standard의 선택적 기능인 `signPersonalMessage`를 지원하는 것으로 예상되나, 실제 테스트로 확인 필요
4. **dapp-kit v2 출시 시점**: `@mysten/dapp-kit-react` npm 출시 후 마이그레이션 타이밍
