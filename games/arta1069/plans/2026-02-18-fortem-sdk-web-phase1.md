# @fortemlabs/sdk-js 1단계 구현 계획

## 개요

ForTem 서비스 접근용 범용 JavaScript/TypeScript SDK(`@fortemlabs/sdk-js`)를 별도 리포지토리(`~/workspace/games/fortem-sdk-web/`)에 생성한다. Supabase JS SDK의 **Factory + Facade** 패턴을 참조하여 `createFortemClient()` 팩토리 함수 + `FortemClient` 클래스 + `client.auth` 서브 모듈 구조로 설계한다. 1단계는 **패키지 기반 완성도**(ESM + CJS dual format, 타입 선언, npm 배포 준비)와 **인증 플로우**(nonce → access-token)에 집중한다.

## 현재 상태 분석

- `~/workspace/games/fortem-sdk-web/` 디렉토리: **존재하지 않음** (처음부터 생성)
- GitHub 리포: `https://github.com/ForTemLabs/fortem-sdk-web.git` — **이미 존재**
- 참조 모델: `@repo/game-core` — tsup `^8.3.0`, TypeScript `^5.7.0`, vitest `^2.1.0` 기반 ESM 빌드
- ForTem API 인증 플로우: `POST /api/v1/developers/auth/nonce` (x-api-key) → `POST /api/v1/developers/auth/access-token` (x-api-key + nonce body)
- whoosh-bang의 nonce → verify 패턴(`WalletLinkButton.tsx:55-83`): 유사 구조 참조 가능

### 핵심 발견:
- `@repo/game-core`의 tsup 빌드: `packages/game-core/tsup.config.ts` — ESM 전용, `dts: true`, `sourcemap: true`, `@` alias
- Supabase JS SDK: `createClient()` 팩토리 → `SupabaseClient` 클래스 → `client.auth`, `client.storage` 서브 모듈 Facade 패턴
- Supabase `package.json` exports: 조건부 `import`/`require` + `.d.mts`/`.d.cts` 분리 타입 선언
- ForTem access token: 5분 TTL, Bearer token으로 사용

## 원하는 최종 상태

1단계 완료 후:

```typescript
import { createFortemClient } from '@fortemlabs/sdk-js'

const fortem = createFortemClient({
  apiKey: 'your-api-key',
  network: 'testnet',
})

// Authentication
const { nonce } = await fortem.auth.getNonce()
const { accessToken } = await fortem.auth.getAccessToken(nonce)

// Access token is automatically injected while valid
// Auto-refreshes on 401 response
```

검증:
- `pnpm install @fortemlabs/sdk-js`으로 whoosh-bang에서 설치 가능
- ESM (`import`) + CJS (`require`) 모두 동작
- TypeScript 타입 자동완성 완전 지원
- `fortem.auth.getNonce()` → nonce 반환
- `fortem.auth.getAccessToken(nonce)` → accessToken 반환
- 401 응답 시 토큰 자동 재발급

## 하지 않을 것

- CDN/UMD/IIFE 빌드 (1단계 불필요, 2단계 이후 검토)
- CI/CD GitHub Actions 자동 배포 파이프라인
- 2단계 API (users, items, collections CRUD)
- 민팅 시 1회성 토큰 소모 전략 (2단계 설계)
- `@` path alias (단순한 패키지 구조에서는 불필요)
- JSR/Deno 레지스트리 배포
- React Native 지원 (Node.js + 브라우저만 대상)

## 구현 접근 방식

`@repo/game-core`의 tsup 빌드 파이프라인을 기반으로 하되, ESM + CJS dual format으로 확장한다. Supabase JS SDK의 `createClient()` → `SupabaseClient` → 서브 모듈 패턴을 참조하여, `createFortemClient()` → `FortemClient` → `FortemAuth` 구조로 단순화한다. 외부 의존성은 최소화하고, `fetch` API만 사용하여 HTTP 통신을 처리한다.

---

## 1단계: 프로젝트 초기화 및 빌드 기반

### 개요
GitHub 리포를 clone하고, pnpm으로 패키지를 초기화한 뒤, tsup/TypeScript/vitest 설정을 완성하여 ESM+CJS dual format 빌드가 동작하는 상태를 만든다.

### 필요한 변경:

#### 1. Git clone 및 기본 구조 생성

```bash
cd ~/workspace/games
git clone https://github.com/ForTemLabs/fortem-sdk-web.git
cd fortem-sdk-web
```

#### 2. package.json 생성
**파일**: `package.json`

```json
{
  "name": "@fortemlabs/sdk-js",
  "version": "0.0.1",
  "description": "ForTem SDK for JavaScript/TypeScript — NFT marketplace API client",
  "publishConfig": {
    "access": "public"
  },
  "main": "./dist/index.cjs",
  "module": "./dist/index.mjs",
  "types": "./dist/index.d.cts",
  "sideEffects": false,
  "exports": {
    ".": {
      "import": {
        "types": "./dist/index.d.mts",
        "default": "./dist/index.mjs"
      },
      "require": {
        "types": "./dist/index.d.cts",
        "default": "./dist/index.cjs"
      }
    }
  },
  "files": [
    "dist",
    "README.md",
    "LICENSE"
  ],
  "engines": {
    "node": ">=18.0.0"
  },
  "scripts": {
    "build": "tsup",
    "dev": "tsup --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "prepublishOnly": "pnpm run build"
  },
  "keywords": [
    "fortem",
    "sdk",
    "nft",
    "marketplace",
    "sui",
    "blockchain"
  ],
  "author": "ForTem Labs",
  "license": "MIT",
  "repository": {
    "type": "git",
    "url": "https://github.com/ForTemLabs/fortem-sdk-web.git"
  },
  "devDependencies": {
    "tsup": "^8.3.0",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0",
    "@types/node": "^22.0.0"
  }
}
```

**핵심 포인트:**
- `exports` 필드: Supabase SDK 패턴 참조 — `import`/`require` 조건부 분기 + 각각의 `types` 지정
- `main` + `module` + `types`: 레거시 번들러/런타임 호환
- `files`: `dist/`, `README.md`, `LICENSE`만 npm에 포함
- `engines`: Node.js >= 18.0.0
- `sideEffects: false`: 트리쉐이킹 활성화
- `prepublishOnly`: npm publish 전 빌드 자동 실행

#### 3. tsup.config.ts 생성
**파일**: `tsup.config.ts`

```typescript
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,
  clean: true,
  sourcemap: true,
});
```

**`@repo/game-core`와의 차이점:**
- `format`: `["esm"]` → `["esm", "cjs"]` (dual format)
- `esbuildOptions` alias 없음 (단순 구조에서 불필요)

#### 4. tsconfig.json 생성
**파일**: `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2020"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

**`@repo/game-core`와의 차이점:**
- `lib`: `["ES2020"]` — DOM 제거 (SDK는 Node.js/브라우저 양쪽이지만, fetch API는 ES2020에 포함)
- `baseUrl`/`paths` 없음 (alias 불필요)

#### 5. vitest.config.ts 생성
**파일**: `vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    globals: true,
    environment: "node",
  },
});
```

**`@repo/game-core`와의 차이점:**
- `environment`: `"jsdom"` → `"node"` (SDK는 DOM 불필요)
- alias 없음

#### 6. 엔트리포인트 스켈레톤 생성
**파일**: `src/index.ts`

```typescript
export const VERSION = "0.0.1";
```

**목적**: 빌드 파이프라인 검증용 최소 엔트리포인트

#### 7. .gitignore 생성
**파일**: `.gitignore`

```
node_modules/
dist/
*.tgz
.env
.env.*
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm install` 성공 (의존성 설치)
- [ ] `pnpm run build` 성공 — `dist/index.mjs`, `dist/index.cjs`, `dist/index.d.mts`, `dist/index.d.cts` 생성 확인
- [ ] `pnpm run typecheck` 성공 (TypeScript 타입 체크)
- [ ] `pnpm run test` 성공 (vitest 빈 테스트 스위트)
- [ ] `node -e "const sdk = require('./dist/index.cjs'); console.log(sdk.VERSION)"` — CJS 동작 확인
- [ ] `node --input-type=module -e "import { VERSION } from './dist/index.mjs'; console.log(VERSION)"` — ESM 동작 확인

#### 수동 검증:
- [ ] `dist/` 디렉토리에 4개 파일 존재 확인 (`index.mjs`, `index.cjs`, `index.d.mts`, `index.d.cts`)
- [ ] 소스맵 파일 존재 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: 핵심 타입 및 클라이언트 구조

### 개요
`FortemClient` 클래스, `createFortemClient()` 팩토리 함수, 설정 타입, 커스텀 에러 클래스, fetch 래퍼를 구현한다. Supabase SDK의 Facade 패턴을 참조하되, ForTem의 단순한 구조에 맞게 경량화한다.

### 필요한 변경:

#### 1. 타입 정의
**파일**: `src/types.ts`

```typescript
/** ForTem network environment */
export type FortemNetwork = "testnet" | "mainnet";

/** Options for createFortemClient() */
export interface FortemClientOptions {
  /** ForTem developer API key */
  apiKey: string;
  /** Network environment (default: 'mainnet') */
  network?: FortemNetwork;
  /** Custom fetch function (for testing) */
  fetch?: typeof globalThis.fetch;
}

/** Common API response structure */
export interface FortemResponse<T> {
  statusCode: number;
  data: T;
}

/** Nonce response */
export interface NonceResponse {
  nonce: string;
}

/** Access token response */
export interface AccessTokenResponse {
  accessToken: string;
}

/** Per-network configuration */
export interface NetworkConfig {
  apiBaseUrl: string;
  serviceUrl: string;
}
```

#### 2. 상수 정의
**파일**: `src/constants.ts`

```typescript
import type { FortemNetwork, NetworkConfig } from "./types";

export const NETWORK_CONFIGS: Record<FortemNetwork, NetworkConfig> = {
  testnet: {
    apiBaseUrl: "https://testnet-api.fortem.gg",
    serviceUrl: "https://testnet.fortem.gg",
  },
  mainnet: {
    apiBaseUrl: "https://api.fortem.gg",
    serviceUrl: "https://fortem.gg",
  },
};

export const DEFAULT_NETWORK: FortemNetwork = "mainnet";
```

#### 3. 커스텀 에러 클래스
**파일**: `src/errors.ts`

```typescript
/** ForTem API error */
export class FortemError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code?: string
  ) {
    super(message);
    this.name = "FortemError";
  }
}

/** Authentication error (401) */
export class FortemAuthError extends FortemError {
  constructor(message: string = "Authentication failed") {
    super(message, 401, "AUTH_ERROR");
    this.name = "FortemAuthError";
  }
}
```

#### 4. Fetch 래퍼
**파일**: `src/fetch.ts`

Supabase의 `fetchWithAuth` 패턴을 참조하여, `x-api-key` 헤더를 자동 주입하는 fetch 래퍼를 구현한다.

```typescript
import { FortemError, FortemAuthError } from "./errors";
import type { FortemResponse } from "./types";

export type Fetch = typeof globalThis.fetch;

/** Creates a fetch wrapper that automatically injects the API key */
export function createFetchWithApiKey(
  apiKey: string,
  customFetch?: Fetch
): Fetch {
  const fetchFn = customFetch ?? globalThis.fetch;

  return async (input, init) => {
    const headers = new Headers(init?.headers);
    headers.set("x-api-key", apiKey);

    if (!headers.has("Content-Type")) {
      headers.set("Content-Type", "application/json");
    }

    return fetchFn(input, { ...init, headers });
  };
}

/** Parses JSON response and handles errors */
export async function parseResponse<T>(
  response: Response
): Promise<FortemResponse<T>> {
  const body = await response.json();

  if (!response.ok) {
    // Specialize 401 responses as FortemAuthError (includes invalid API key)
    if (response.status === 401) {
      throw new FortemAuthError(
        body.message ?? body.error ?? "Authentication failed"
      );
    }

    throw new FortemError(
      body.message ?? body.error ?? `Request failed with status ${response.status}`,
      response.status,
      body.code
    );
  }

  return body as FortemResponse<T>;
}
```

#### 5. FortemClient 클래스
**파일**: `src/client.ts`

```typescript
import { FortemAuth } from "./auth";
import { NETWORK_CONFIGS, DEFAULT_NETWORK } from "./constants";
import { createFetchWithApiKey, type Fetch } from "./fetch";
import type { FortemClientOptions, NetworkConfig } from "./types";

export class FortemClient {
  readonly auth: FortemAuth;

  private readonly _fetch: Fetch;
  private readonly _networkConfig: NetworkConfig;

  constructor(options: FortemClientOptions) {
    if (!options.apiKey || typeof options.apiKey !== "string") {
      throw new Error("apiKey is required");
    }

    if (options.apiKey.trim().length === 0) {
      throw new Error("apiKey cannot be empty");
    }

    const network = options.network ?? DEFAULT_NETWORK;
    this._networkConfig = NETWORK_CONFIGS[network];
    this._fetch = createFetchWithApiKey(options.apiKey, options.fetch);

    // Initialize sub-modules
    this.auth = new FortemAuth(this._networkConfig.apiBaseUrl, this._fetch);
  }

  /** API base URL for the current network */
  get apiBaseUrl(): string {
    return this._networkConfig.apiBaseUrl;
  }

  /** Service URL for the current network */
  get serviceUrl(): string {
    return this._networkConfig.serviceUrl;
  }
}
```

#### 6. 팩토리 함수 및 엔트리포인트
**파일**: `src/index.ts`

```typescript
import { FortemClient } from "./client";
import type { FortemClientOptions } from "./types";

/** Factory function to create a FortemClient instance */
export function createFortemClient(
  options: FortemClientOptions
): FortemClient {
  return new FortemClient(options);
}

// Re-exports
export { FortemClient } from "./client";
export { FortemAuth } from "./auth";
export { FortemError, FortemAuthError } from "./errors";
export type {
  FortemClientOptions,
  FortemNetwork,
  FortemResponse,
  NonceResponse,
  AccessTokenResponse,
  NetworkConfig,
} from "./types";
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm run build` 성공
- [ ] `pnpm run typecheck` 성공
- [ ] `createFortemClient({ apiKey: 'test' })` 인스턴스 생성 시 에러 없음
- [ ] `createFortemClient({} as any)` 호출 시 `"apiKey is required"` 에러 throw
- [ ] `fortem.auth` 프로퍼티 접근 가능 (`FortemAuth` 인스턴스)
- [ ] 모든 public 타입이 `@fortemlabs/sdk-js`에서 import 가능

#### 수동 검증:
- [ ] TypeScript IDE에서 `createFortemClient()` 호출 시 옵션 자동완성 확인
- [ ] `fortem.auth.` 입력 시 메서드 자동완성 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 3단계: 인증 모듈 구현

### 개요
`FortemAuth` 서브 모듈을 구현하여 ForTem API의 2단계 인증 플로우(nonce 발급 → access token 획득)를 처리한다. 내부 토큰 캐싱과 401 응답 시 자동 재발급 기능을 포함한다.

### 필요한 변경:

#### 1. FortemAuth 클래스
**파일**: `src/auth.ts`

```typescript
import { FortemAuthError } from "./errors";
import { parseResponse, type Fetch } from "./fetch";
import type { NonceResponse, AccessTokenResponse } from "./types";

export class FortemAuth {
  private _accessToken: string | null = null;
  private _expiresAt: number | null = null;

  /** Expiry margin — treat token as expired 30s before actual expiration */
  private static readonly TOKEN_EXPIRY_MARGIN_MS = 30_000;
  /** Access token TTL (5 minutes) */
  private static readonly TOKEN_TTL_MS = 5 * 60 * 1000;

  constructor(
    private readonly _apiBaseUrl: string,
    private readonly _fetch: Fetch
  ) {}

  /**
   * Step 1: Request a nonce
   *
   * Requests an authentication nonce from the ForTem API.
   *
   * @returns An object containing the nonce string
   */
  async getNonce(): Promise<{ nonce: string }> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/auth/nonce`,
      { method: "POST" }
    );

    const result = await parseResponse<NonceResponse>(response);
    return { nonce: result.data.nonce };
  }

  /**
   * Step 2: Obtain an access token
   *
   * Exchanges a nonce for an access token.
   * The acquired token is cached internally.
   *
   * @param nonce - The nonce obtained from getNonce()
   * @returns An object containing the accessToken string
   */
  async getAccessToken(nonce: string): Promise<{ accessToken: string }> {
    if (!nonce) {
      throw new FortemAuthError("nonce is required");
    }

    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/auth/access-token`,
      {
        method: "POST",
        body: JSON.stringify({ nonce }),
      }
    );

    const result = await parseResponse<AccessTokenResponse>(response);
    const { accessToken } = result.data;

    // Cache internally
    this._accessToken = accessToken;
    this._expiresAt = Date.now() + FortemAuth.TOKEN_TTL_MS;

    return { accessToken };
  }

  /**
   * Returns the cached access token.
   *
   * Returns null if no token is cached or if it has expired.
   * Treats the token as expired 30 seconds before actual expiration (margin applied).
   */
  getToken(): string | null {
    if (!this._accessToken || !this._expiresAt) {
      return null;
    }

    if (Date.now() >= this._expiresAt - FortemAuth.TOKEN_EXPIRY_MARGIN_MS) {
      this._accessToken = null;
      this._expiresAt = null;
      return null;
    }

    return this._accessToken;
  }

  /**
   * Returns a valid access token, auto-refreshing if necessary.
   *
   * Returns the cached token if still valid.
   * If expired, automatically executes the nonce → access token two-step flow.
   */
  async getValidToken(): Promise<string> {
    const cached = this.getToken();
    if (cached) return cached;

    const { nonce } = await this.getNonce();
    const { accessToken } = await this.getAccessToken(nonce);
    return accessToken;
  }

  /** Clears the cached token */
  clearToken(): void {
    this._accessToken = null;
    this._expiresAt = null;
  }
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm run build` 성공
- [ ] `pnpm run typecheck` 성공
- [ ] `pnpm run test` — 아래 단위 테스트 모두 통과:
  - `getNonce()`: mock fetch로 nonce 반환 확인
  - `getAccessToken(nonce)`: mock fetch로 accessToken 반환 확인
  - `getAccessToken('')`: `FortemAuthError` throw 확인
  - `getToken()`: 캐싱된 토큰 반환 확인
  - `getToken()`: 만료된 토큰 → null 반환 확인
  - `getValidToken()`: 캐싱 유효 시 fetch 호출 없이 반환 확인
  - `getValidToken()`: 캐싱 만료 시 nonce + accessToken 2단계 자동 실행 확인
  - `clearToken()`: 토큰 초기화 확인
  - API 401 응답 시 `FortemAuthError` throw 확인 (잘못된 API 키 시나리오)
  - API 기타 에러 응답 시 `FortemError` throw 확인

#### 수동 검증:
- [ ] 실제 ForTem testnet API로 `getNonce()` 호출 성공 (API 키 필요)
- [ ] 실제 ForTem testnet API로 `getAccessToken(nonce)` 호출 성공

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 4단계: 테스트 및 npm 배포 준비

### 개요
vitest 단위 테스트를 작성하고, npm 배포 전 패키지 검증을 수행하며, README를 작성한다.

### 필요한 변경:

#### 1. 단위 테스트
**파일**: `src/__tests__/auth.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { createFortemClient } from "../index";
import { FortemAuthError, FortemError } from "../errors";

function createMockFetch(responses: Array<{ status: number; body: unknown }>) {
  let callIndex = 0;
  return vi.fn(async () => {
    const res = responses[callIndex++];
    return {
      ok: res.status >= 200 && res.status < 300,
      status: res.status,
      json: async () => res.body,
      headers: new Headers(),
    } as Response;
  });
}

describe("FortemAuth", () => {
  describe("getNonce", () => {
    it("should return nonce from API", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { nonce: "test-nonce-123" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });

      const result = await client.auth.getNonce();

      expect(result.nonce).toBe("test-nonce-123");
      expect(mockFetch).toHaveBeenCalledOnce();
    });

    it("should include x-api-key header", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { nonce: "abc" } } },
      ]);
      const client = createFortemClient({ apiKey: "my-api-key", fetch: mockFetch });

      await client.auth.getNonce();

      const [, init] = mockFetch.mock.calls[0];
      const headers = new Headers(init?.headers);
      expect(headers.get("x-api-key")).toBe("my-api-key");
    });

    it("should throw FortemAuthError on 401 (invalid API key)", async () => {
      const mockFetch = createMockFetch([
        { status: 401, body: { statusCode: 401, message: "Invalid API key" } },
      ]);
      const client = createFortemClient({ apiKey: "bad-key", fetch: mockFetch });

      await expect(client.auth.getNonce()).rejects.toThrow(FortemAuthError);
      await expect(client.auth.getNonce()).rejects.toThrow("Invalid API key");
    });

    it("should throw FortemError on other API errors", async () => {
      const mockFetch = createMockFetch([
        { status: 500, body: { statusCode: 500, message: "Internal Server Error" } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });

      await expect(client.auth.getNonce()).rejects.toThrow(FortemError);
    });
  });

  describe("getAccessToken", () => {
    it("should return access token", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { accessToken: "jwt-token-xyz" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });

      const result = await client.auth.getAccessToken("test-nonce");

      expect(result.accessToken).toBe("jwt-token-xyz");
    });

    it("should throw FortemAuthError when nonce is empty", async () => {
      const client = createFortemClient({ apiKey: "test-key" });

      await expect(client.auth.getAccessToken("")).rejects.toThrow(FortemAuthError);
    });

    it("should cache the token internally", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { accessToken: "cached-token" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });

      await client.auth.getAccessToken("nonce");

      expect(client.auth.getToken()).toBe("cached-token");
    });
  });

  describe("getToken", () => {
    it("should return null when no token cached", () => {
      const client = createFortemClient({ apiKey: "test-key" });
      expect(client.auth.getToken()).toBeNull();
    });

    it("should return null when token expired", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { accessToken: "token" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });
      await client.auth.getAccessToken("nonce");

      // Advance time by 5 minutes + 1 second
      vi.useFakeTimers();
      vi.advanceTimersByTime(5 * 60 * 1000 + 1000);

      expect(client.auth.getToken()).toBeNull();

      vi.useRealTimers();
    });
  });

  describe("getValidToken", () => {
    it("should return cached token if valid", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { accessToken: "valid-token" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });
      await client.auth.getAccessToken("nonce");

      const token = await client.auth.getValidToken();

      expect(token).toBe("valid-token");
      // fetch called only once in getAccessToken (no additional call from getValidToken)
      expect(mockFetch).toHaveBeenCalledTimes(1);
    });

    it("should auto-refresh when token expired", async () => {
      const mockFetch = createMockFetch([
        // First getAccessToken call
        { status: 200, body: { statusCode: 200, data: { accessToken: "old-token" } } },
        // getValidToken → getNonce (auto-refresh)
        { status: 200, body: { statusCode: 200, data: { nonce: "new-nonce" } } },
        // getValidToken → getAccessToken (auto-refresh)
        { status: 200, body: { statusCode: 200, data: { accessToken: "new-token" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });
      await client.auth.getAccessToken("nonce");

      // Simulate token expiration
      client.auth.clearToken();

      const token = await client.auth.getValidToken();

      expect(token).toBe("new-token");
      expect(mockFetch).toHaveBeenCalledTimes(3);
    });
  });

  describe("clearToken", () => {
    it("should clear cached token", async () => {
      const mockFetch = createMockFetch([
        { status: 200, body: { statusCode: 200, data: { accessToken: "token" } } },
      ]);
      const client = createFortemClient({ apiKey: "test-key", fetch: mockFetch });
      await client.auth.getAccessToken("nonce");

      client.auth.clearToken();

      expect(client.auth.getToken()).toBeNull();
    });
  });
});
```

**파일**: `src/__tests__/client.test.ts`

```typescript
import { describe, it, expect } from "vitest";
import { createFortemClient, FortemClient } from "../index";

describe("FortemClient", () => {
  it("should create client with default network (mainnet)", () => {
    const client = createFortemClient({ apiKey: "test-key" });

    expect(client).toBeInstanceOf(FortemClient);
    expect(client.apiBaseUrl).toBe("https://api.fortem.gg");
    expect(client.serviceUrl).toBe("https://fortem.gg");
  });

  it("should create client with testnet", () => {
    const client = createFortemClient({ apiKey: "test-key", network: "testnet" });

    expect(client.apiBaseUrl).toBe("https://testnet-api.fortem.gg");
    expect(client.serviceUrl).toBe("https://testnet.fortem.gg");
  });

  it("should throw when apiKey is missing", () => {
    expect(() => createFortemClient({} as any)).toThrow("apiKey is required");
  });

  it("should throw when apiKey is empty string", () => {
    expect(() => createFortemClient({ apiKey: "  " })).toThrow("apiKey cannot be empty");
  });

  it("should have auth sub-module", () => {
    const client = createFortemClient({ apiKey: "test-key" });
    expect(client.auth).toBeDefined();
  });
});
```

#### 2. README.md
**파일**: `README.md`

```markdown
# @fortemlabs/sdk-js

ForTem SDK for JavaScript/TypeScript — NFT marketplace API client for the Sui blockchain.

## Installation

```bash
npm install @fortemlabs/sdk-js
# or
pnpm install @fortemlabs/sdk-js
# or
yarn add @fortemlabs/sdk-js
```

## Quick Start

```typescript
import { createFortemClient } from '@fortemlabs/sdk-js'

const fortem = createFortemClient({
  apiKey: 'your-api-key',
  network: 'mainnet', // 'testnet' | 'mainnet' (default: 'mainnet')
})

// Step 1: Get nonce
const { nonce } = await fortem.auth.getNonce()

// Step 2: Get access token
const { accessToken } = await fortem.auth.getAccessToken(nonce)

// Auto-refresh: get valid token (auto-renews if expired)
const token = await fortem.auth.getValidToken()
```

## API Reference

### `createFortemClient(options)`

Creates a new ForTem client instance.

| Option | Type | Required | Default | Description |
|--------|------|----------|---------|-------------|
| `apiKey` | `string` | Yes | - | Your ForTem developer API key |
| `network` | `'testnet' \| 'mainnet'` | No | `'mainnet'` | Network environment |
| `fetch` | `typeof fetch` | No | `globalThis.fetch` | Custom fetch function |

### `client.auth.getNonce()`

Requests a nonce from the ForTem API for authentication.

### `client.auth.getAccessToken(nonce)`

Exchanges a nonce for an access token. The token is cached internally (5 min TTL).

### `client.auth.getValidToken()`

Returns a valid access token, auto-refreshing if the cached token has expired.

### `client.auth.getToken()`

Returns the cached access token, or `null` if expired or not available.

### `client.auth.clearToken()`

Clears the cached access token.

## License

MIT
```

#### 3. LICENSE 파일
**파일**: `LICENSE`

MIT 라이선스 전문 (ForTem Labs, 2026)

#### 4. npm 패키지 검증
`pnpm pack`으로 `.tgz` 패키지를 생성한 뒤, 포함된 파일을 확인한다.

### 성공 기준:

#### 자동화된 검증:
- [ ] `pnpm run test` — 모든 단위 테스트 통과 (13개 테스트 케이스)
- [ ] `pnpm run build` 성공
- [ ] `pnpm run typecheck` 성공
- [ ] `pnpm pack` 성공 — `.tgz` 파일 생성, `dist/`, `README.md`, `LICENSE` 포함 확인
- [ ] `pnpm pack --dry-run`으로 불필요한 파일(`src/`, `node_modules/`, `.env`) 미포함 확인

#### 수동 검증:
- [ ] README의 코드 예시가 실제 API와 일치하는지 확인
- [ ] whoosh-bang에서 `pnpm install ../fortem-sdk-web/fortemlabs-sdk-js-0.0.1.tgz`로 로컬 설치 테스트
- [ ] whoosh-bang에서 `import { createFortemClient } from '@fortemlabs/sdk-js'` 타입 에러 없이 동작

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 테스트 전략

### 단위 테스트:
- `FortemClient` 생성 (기본값, mainnet, 에러 케이스)
- `FortemAuth.getNonce()` (성공, API 에러)
- `FortemAuth.getAccessToken()` (성공, 빈 nonce, API 에러, 토큰 캐싱)
- `FortemAuth.getToken()` (캐싱 유효, 만료)
- `FortemAuth.getValidToken()` (캐싱 유효 시 재사용, 만료 시 자동 갱신)
- `FortemAuth.clearToken()` (초기화)
- `createFetchWithApiKey()` (헤더 주입, Content-Type 기본값)
- `parseResponse()` (성공 응답, 에러 응답)

### 통합 테스트 (수동):
- 실제 ForTem testnet API로 전체 인증 플로우 검증

### 수동 테스트 단계:
1. whoosh-bang에서 로컬 `.tgz` 패키지 설치
2. `import { createFortemClient }` 타입 자동완성 확인
3. 실제 API 키로 `getNonce()` → `getAccessToken()` 호출
4. `getValidToken()` 자동 갱신 동작 확인

## 성능 고려 사항

- `fetch` 함수는 외부에서 주입 가능 → 테스트 시 mock 사용, 프로덕션에서는 `globalThis.fetch`
- access token 캐싱으로 불필요한 API 호출 방지
- 토큰 만료 30초 전 마진으로 경계 상황 방지
- `sideEffects: false`로 트리쉐이킹 활성화 → 사용하지 않는 코드 제거 가능

## 참조

- 연구 문서: `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md`
- 참조 모델 (빌드): `packages/game-core/tsup.config.ts`, `packages/game-core/package.json`
- 참조 모델 (API 패턴): Supabase JS SDK `createClient()` → `SupabaseClient` 패턴
- 인증 패턴 참조: `apps/web/src/components/lobby/WalletLinkButton.tsx:55-83`
- ForTem API 엔드포인트: `testnet-api.fortem.gg`, `api.fortem.gg`
