---
date: 2026-02-17T13:21:29Z
researcher: arta1069@gmail.com
git_commit: ca7b89eddd278d8f25e181af54a029d2c459e095
branch: main
repository: Whoosh-Bang
topic: "fortem-sdk-web SDK 라이브러리 생성을 위한 프로젝트 구조 및 패키지 패턴 연구"
tags: [research, codebase, fortem, sdk, npm-package, typescript, tsup, separate-repo]
status: complete
last_updated: 2026-02-17
last_updated_by: arta1069
last_updated_note: "별도 리포지토리 구조로 변경, 1단계 핵심 목표 명확화에 대한 후속 연구 추가"
---

# 연구: fortem-sdk-web SDK 라이브러리 생성을 위한 프로젝트 구조 및 패키지 패턴 연구

**날짜**: 2026-02-17T13:21:29Z
**연구자**: arta1069@gmail.com
**Git 커밋**: ca7b89eddd278d8f25e181af54a029d2c459e095
**브랜치**: main
**리포지토리**: Whoosh-Bang

## 연구 질문

`fortem-sdk-web`이라는 npm install 방식의 별도 SDK 라이브러리를 만들기 위해, 현재 whoosh-bang 모노레포의 패키지 구조, TypeScript 빌드 설정, API 클라이언트 패턴, 환경변수 관리 방식을 파악한다. ForTem은 Sui 블록체인 기반 NFT 마켓플레이스이며, SDK는 whoosh-bang 프로젝트뿐만 아니라 모든 Node/JavaScript 환경에서 공통으로 ForTem 서비스에 접근할 수 있어야 한다.

## 요약

whoosh-bang은 **pnpm 9.0.0 + Turborepo** 기반 모노레포로, `apps/web` (Next.js 16)과 `packages/` (game-core, ui, config) 3개 내부 패키지로 구성된다. `@repo/game-core`가 유일한 빌드 대상 패키지로, **tsup (esbuild 기반)** 으로 ESM 단일 포맷을 출력하며 `.d.mts` 타입 선언을 생성한다. 외부 API 호출은 Supabase 클라이언트와 `fetch()` 기반으로 이루어지며, Wallet nonce/verify 2단계 인증 패턴이 존재한다.

**`fortem-sdk-web`은 whoosh-bang 모노레포 내부 패키지가 아닌, 별도 리포지토리(`https://github.com/ForTemLabs/fortem-sdk-web.git`)로 개발되며, npm 레지스트리를 통해 `pnpm install fortem-sdk-web`으로 소비된다.** `@repo/game-core`의 tsup 기반 빌드 파이프라인을 참조 모델로 삼되, 독립 패키지로서 ESM + CJS dual format과 견고한 패키지 기반을 갖추는 것이 1단계의 핵심이다.

## 프로젝트 배치 구조 (확정)

```
~/workspace/games/
├── whoosh-bang/           # 기존 게임 프로젝트 (소비자)
│   ├── apps/web/          #   → pnpm install fortem-sdk-web 으로 SDK 사용
│   ├── packages/
│   │   ├── game-core/
│   │   ├── ui/
│   │   └── config/
│   └── ...
└── fortem-sdk-web/        # 신규 SDK 프로젝트 (별도 리포지토리)
    ├── .git/              #   → https://github.com/ForTemLabs/fortem-sdk-web.git
    ├── package.json       #   → name: "fortem-sdk-web", npm 공개 배포
    ├── tsconfig.json
    ├── tsup.config.ts
    └── src/
        └── index.ts
```

### 핵심 아키텍처 결정

| 항목 | 결정 |
|------|------|
| **리포지토리** | 별도 (`ForTemLabs/fortem-sdk-web`) — whoosh-bang 모노레포 외부 |
| **GitHub** | `https://github.com/ForTemLabs/fortem-sdk-web.git` |
| **패키지명** | `fortem-sdk-web` |
| **소비 방식** | npm 레지스트리 → `pnpm install fortem-sdk-web` |
| **대상 환경** | Node.js + 브라우저 JavaScript (범용) |
| **빌드 참조 모델** | `@repo/game-core` (tsup 기반) |

### 1단계 핵심 목표

1. **패키지 기반 완성도** — Node/JavaScript 어디서나 `npm install fortem-sdk-web`으로 바로 사용 가능한 견고한 패키지 구조 (ESM + CJS dual format, 타입 선언, 적절한 exports)
2. **인증 플로우** — nonce 발급(Step 1) → access token 획득(Step 2)까지의 2단계 인증

## 상세 발견 사항

### 1. whoosh-bang 모노레포 구조

프로젝트 루트 `package.json`에서 workspace는 `apps/*`와 `packages/*` 두 글로브로 등록된다.

```
whoosh-bang/
├── apps/
│   └── web/          # Next.js 16 (App Router)
├── packages/
│   ├── game-core/    # @repo/game-core — Phaser 게임 코어 (tsup 빌드)
│   ├── ui/           # @repo/ui — React UI (소스 직접 노출, 빌드 없음)
│   └── config/       # @repo/config — ESLint 공유 설정
├── scripts/          # 유틸리티 스크립트
├── turbo.json        # Turborepo 태스크 설정
├── pnpm-workspace.yaml
└── package.json      # packageManager: pnpm@9.0.0
```

Turborepo `build` 태스크는 `dependsOn: ["^build"]`로 의존 패키지를 먼저 빌드하며, 캐시 대상은 `.next/**`와 `dist/**`이다.

### 2. @repo/game-core — 빌드 가능한 패키지의 참조 모델

`fortem-sdk-web` 패키지를 만들 때 빌드 파이프라인의 참조 모델로 삼을 수 있는 기존 패키지이다. 단, `@repo/game-core`는 모노레포 내부 패키지(ESM 전용)이고, `fortem-sdk-web`은 독립 npm 패키지(ESM + CJS dual format)라는 차이가 있다.

**package.json 핵심 설정:**
- `name`: `@repo/game-core`
- `main`: `./dist/index.mjs`
- `types`: `./dist/index.d.mts`
- `exports.".".import`: `./dist/index.mjs`
- `exports.".".types`: `./dist/index.d.mts`
- CJS 진입점 없음 (ESM 전용 — 모노레포 내부 소비 전용이므로)

**빌드 도구: tsup (esbuild 기반)**
- 엔트리: `src/index.ts` 단일 파일
- 출력: ESM (`dist/index.mjs`)
- `dts: true` — 타입 선언 자동 생성
- `sourcemap: true`, `clean: true`
- `esbuildOptions`에서 `@` → `./src` alias 등록

**tsconfig.json 핵심:**
- `target: "ES2020"`, `module: "ESNext"`, `moduleResolution: "bundler"`
- `declaration: true`, `declarationMap: true`
- `paths: { "@/*": ["./src/*"] }`

**테스트: vitest**
- `environment: "jsdom"`, `globals: true`
- `resolve.alias: { "@": "./src" }`

### 3. @repo/ui — 소스 직접 노출 패키지

빌드 없이 `src/index.ts`를 `main`/`types`/`exports`에서 직접 가리킨다. `apps/web`의 `transpilePackages`로 소비자가 직접 트랜스파일한다. 현재 `src/index.ts`는 빈 export만 포함한다.

### 4. @repo/config — 순수 JS 설정 패키지

`private: true`, `exports: { "./eslint/base": "./eslint/base.mjs" }` 서브패스만 존재한다. TypeScript 설정 없음, ESLint 공유 규칙만 제공한다.

### 5. 외부 API 호출 패턴

#### 5-1. Supabase 클라이언트 3종

| 클라이언트 | 패키지 | 용도 | 파일 |
|-----------|--------|------|------|
| `createBrowserClient` | `@supabase/ssr` | 클라이언트 컴포넌트 | `lib/supabase/client.ts` |
| `createServerClient` | `@supabase/ssr` | 서버 컴포넌트/미들웨어 | `lib/supabase/server.ts`, `proxy.ts` |
| `createClient` (admin) | `@supabase/supabase-js` | Route Handler (RLS 우회) | 6곳의 Route Handler |

#### 5-2. fetch() 기반 내부 API 호출

클라이언트 → 내부 API Route 호출은 `fetch()` 직접 사용:
- POST + JSON body: `GameWrapper.tsx`, `WalletLinkButton.tsx`
- DELETE: `WalletLinkButton.tsx`
- 에러 처리: `try/catch` + `toast.error()` 또는 `response.ok` 체크

#### 5-3. 2단계 인증 패턴 (Nonce → Verify)

`WalletLinkButton.tsx`에 ForTem SDK 인증 플로우와 유사한 패턴이 존재:
1. POST `/api/wallet/nonce` → `{ nonce }` 응답
2. POST `/api/wallet/verify` + `{ nonce, signature, ... }` → 검증

이 패턴은 ForTem의 nonce → access-token 2단계 인증과 구조적으로 동일하다.

### 6. 환경변수 관리 방식

`.env.local`은 `apps/web/` 한 곳에만 존재한다. `NEXT_PUBLIC_` 접두사로 클라이언트 노출 여부를 구분한다.

| 변수 | 접두사 | 영역 |
|------|--------|------|
| `NEXT_PUBLIC_SUPABASE_URL` | `NEXT_PUBLIC_` | 클라이언트 + 서버 |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | `NEXT_PUBLIC_` | 클라이언트 + 서버 |
| `SUPABASE_SERVICE_ROLE_KEY` | 없음 | 서버 전용 |
| `NEXT_PUBLIC_SUI_NETWORK` | `NEXT_PUBLIC_` | 클라이언트 |

### 7. 패키지 간 참조 방식

- `workspace:*` 프로토콜로 pnpm workspace 내 로컬 패키지 연결
- TypeScript Project References (`references`) 미사용
- `apps/web`의 `transpilePackages: ["@repo/game-core", "@repo/ui"]`로 소비자 측 트랜스파일
- 패키지의 `exports` 필드와 빌드된 `.d.ts` 파일을 통한 타입 연결

**fortem-sdk-web의 경우:** whoosh-bang에서 `workspace:*`가 아닌 npm 레지스트리 버전으로 의존하게 된다. 즉, whoosh-bang의 `apps/web/package.json`에 `"fortem-sdk-web": "^x.x.x"`로 선언된다.

### 8. TypeScript 공통 설정

모든 패키지에서 공통:
- `target: "ES2020"`, `module: "ESNext"`, `moduleResolution: "bundler"`
- `strict: true`, `esModuleInterop: true`, `skipLibCheck: true`

`game-core`에서 추가로: `declaration: true`, `declarationMap: true`, `outDir: "./dist"`, `rootDir: "./src"`, `baseUrl: "."`, `paths: { "@/*": ["./src/*"] }`

## 코드 참조

- `package.json` — 모노레포 workspace 설정, `packageManager: "pnpm@9.0.0"`
- `turbo.json` — Turborepo 태스크 파이프라인, `build.dependsOn: ["^build"]`
- `packages/game-core/package.json` — tsup 빌드 대상 패키지, ESM exports
- `packages/game-core/tsup.config.ts` — tsup 빌드 설정 (esm, dts, sourcemap)
- `packages/game-core/tsconfig.json` — TypeScript 설정 (ES2020, bundler)
- `packages/game-core/vitest.config.ts` — vitest 테스트 설정
- `packages/game-core/src/index.ts` — 엔트리포인트, createGame + re-exports
- `packages/ui/package.json` — 빌드 없는 소스 직접 노출 패키지
- `packages/config/package.json` — ESLint 공유 설정 패키지
- `apps/web/next.config.ts:4` — transpilePackages 설정
- `apps/web/src/lib/supabase/client.ts` — 브라우저 Supabase 클라이언트
- `apps/web/src/lib/supabase/server.ts` — 서버 Supabase 클라이언트
- `apps/web/src/components/lobby/WalletLinkButton.tsx:55-83` — nonce → verify 2단계 인증 패턴
- `apps/web/.env.local` — 환경변수 정의

## 아키텍처 문서화

### whoosh-bang 빌드 파이프라인

```
[Turborepo] turbo build
    ├── packages/game-core  →  tsup src/index.ts --format esm --dts  →  dist/index.mjs + dist/index.d.mts
    ├── packages/ui         →  (빌드 없음, 소스 직접 노출)
    ├── packages/config     →  (빌드 없음, ESLint 설정만)
    └── apps/web            →  next build (transpilePackages로 game-core, ui 포함)
```

### whoosh-bang 패키지 의존 관계

```
@repo/config (ESLint 설정)
    ↑ devDependency
@repo/game-core (Phaser 게임 코어)
@repo/ui        (React UI)
    ↑ dependency (workspace:*)
apps/web        (Next.js)
    ↑ dependency (npm registry)
fortem-sdk-web  (ForTem SDK — 별도 리포지토리, npm 배포)
```

### fortem-sdk-web 예상 구조 (별도 리포지토리)

```
~/workspace/games/fortem-sdk-web/     # 별도 리포지토리
├── .git/                              # → https://github.com/ForTemLabs/fortem-sdk-web.git
├── package.json                       # name: "fortem-sdk-web", private: false
├── tsconfig.json
├── tsup.config.ts                     # ESM + CJS dual format 출력
├── vitest.config.ts
├── .npmignore 또는 "files" 필드
├── README.md
├── LICENSE
└── src/
    ├── index.ts                       # 엔트리포인트 (public API)
    ├── client.ts                      # FortemClient 클래스
    ├── auth.ts                        # nonce → access-token 2단계 인증
    ├── types.ts                       # 타입 정의
    ├── errors.ts                      # 커스텀 에러 클래스
    └── __tests__/                     # 테스트
```

### fortem-sdk-web과 whoosh-bang의 관계

```
[fortem-sdk-web]                    [whoosh-bang]
(별도 Git 리포)                     (기존 모노레포)

  npm publish
      │
      ▼
  npm registry ─── pnpm install ──→ apps/web/package.json
  "fortem-sdk-web"                    "fortem-sdk-web": "^x.x.x"
```

## ForTem API 인증 플로우 (1단계 구현 범위)

### Step 1: Nonce 발급

```http
POST {baseUrl}/api/v1/developers/auth/nonce
Header: x-api-key: <YOUR_API_KEY>
```

Response: `{ "statusCode": 200, "data": { "nonce": "f09d58d9..." } }`

### Step 2: Access Token 발급

```http
POST {baseUrl}/api/v1/developers/auth/access-token
Header: x-api-key: <YOUR_API_KEY>
Content-Type: application/json
Body: { "nonce": "<nonce_from_step_1>" }
```

Response: `{ "statusCode": 200, "data": { "accessToken": "eyJhbGci..." } }`

### API 엔드포인트 환경

| 환경 | API Base URL | Service URL |
|------|-------------|-------------|
| Testnet | `https://testnet-api.fortem.gg` | `https://testnet.fortem.gg` |
| Mainnet | `https://api.fortem.gg` | `https://fortem.gg` |

### Access Token 특성

- **유효시간**: 5분
- **1회성**: 민팅(생성) API 호출 시 토큰이 소모되며 재사용 불가 (2단계에서 적용)
- **Bearer Token**: 이후 API 호출에서 `Authorization: Bearer <accessToken>` 헤더로 사용

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` — 무기/캐릭터 아이템화 연구, ForTem export/import 플로우 언급
- `thoughts/arta1069/plans/2026-02-17-weapon-itemization-system.md` — 무기 아이템화 시스템 구현 계획, ForTem 연동 계획 포함
- `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md` — 캐릭터 스킨 선택 시스템, ForTem NFT 연동 계획
- `thoughts/arta1069/research/2026-02-13-character-skin-nft-system-research.md` — 캐릭터 스킨 NFT 시스템 연구
- `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md` — 초기 아키텍처 연구, SDK/블록체인/NFT 내용 포함

## 관련 연구

- `thoughts/arta1069/research/2026-02-12-social-login-wallet-linking-architecture-research.md` — Sui 지갑 연동 인증 아키텍처 (2단계 인증 패턴 참고)
- `thoughts/arta1069/research/2026-02-17-weapon-character-itemization-research.md` — ForTem 관련 아이템화 연구

## 미해결 질문

1. ~~**패키지 배포 범위**: `fortem-sdk-web`은 npm 공개 배포인가, 아니면 모노레포 내부 전용인가?~~ → **해결**: 별도 리포지토리(`ForTemLabs/fortem-sdk-web`)에서 npm 공개 배포
2. **CJS 지원**: `@repo/game-core`는 ESM 전용이지만, `fortem-sdk-web`은 범용 npm 패키지이므로 ESM + CJS dual format이 필요할 가능성이 높다.
3. **ForTem API 전체 스펙**: 2단계(유저 조회, 콜렉션 CRUD, 아이템 CRUD)의 상세 API 스펙이 추가로 필요하다.
4. **Access Token 관리 전략**: 5분 TTL + 민팅 시 1회성이라는 특성을 고려할 때, SDK 레벨에서 토큰 캐싱/자동 갱신 전략을 어떻게 설계할 것인가?

## 후속 연구 2026-02-17T13:35:00Z

### 프로젝트 구조 변경: 모노레포 내부 → 별도 리포지토리

사용자 요구에 따라 `fortem-sdk-web`의 배치가 변경되었다:

| 항목 | 초기 가정 | 확정 방향 |
|------|----------|-----------|
| **위치** | `whoosh-bang/packages/fortem-sdk-web/` | `~/workspace/games/fortem-sdk-web/` (별도 리포) |
| **GitHub** | whoosh-bang 모노레포 내부 | `https://github.com/ForTemLabs/fortem-sdk-web.git` |
| **소비 방식** | `workspace:*` 프로토콜 | **npm 레지스트리** → `pnpm install fortem-sdk-web` |
| **패키지명** | `@repo/fortem-sdk-web` 등 | `fortem-sdk-web` (스코프 없음) |

### 1단계 핵심 재정의

사용자가 강조한 1단계의 핵심은 **인증 플로우 구현보다 패키지 기반의 완성도**에 있다:

> "Node와 javascript 환경에서 누구나 공통으로 라이브러리를 install 잘 활용할 수 있게 패키지 기반을 잘 다듬는게 핵심"

따라서 1단계에서 집중해야 할 영역:
1. **ESM + CJS dual format** 빌드 출력 (범용 Node.js 호환)
2. **TypeScript 타입 선언** 완전 지원 (`.d.ts` / `.d.mts` / `.d.cts`)
3. **package.json exports 필드** 올바른 조건부 export 설정
4. **npm 배포 설정** — `files` 필드, `.npmignore`, `prepublishOnly` 스크립트
5. **README, LICENSE** 등 공개 패키지 표준 파일
6. **테스트 기반** — vitest로 인증 플로우 단위 테스트
7. 그 위에 **인증 플로우**(nonce → access-token) 구현
