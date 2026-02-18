# sdk-js 2단계 API 모듈 구현 계획

## 개요

`@fortemlabs/sdk-js` v0.0.2 — Users, Collections, Items API 서브모듈을 추가하고, Bearer 토큰 자동 주입 + 402 재시도 인프라를 구축한다. 1단계에서 완성된 `createFortemClient()` → `FortemClient` → `FortemAuth` 패턴을 그대로 확장한다.

## 현재 상태 분석

v0.0.1은 인증(nonce → access-token) 플로우만 구현된 상태:
- `FortemClient`는 `auth: FortemAuth` 서브모듈 하나만 노출 (`src/client.ts:7`)
- `createFetchWithApiKey()`는 `x-api-key` + `Content-Type: application/json`만 주입 (`src/fetch.ts:7-23`)
- `parseResponse()`는 401만 `FortemAuthError`로 분기 (`src/fetch.ts:37-39`)
- `getValidToken()`은 TTL 기반 만료만 체크, 토큰 소모 미대응 (`src/auth.ts:94-101`)
- 에러 클래스: `FortemError` → `FortemAuthError` 2개만 존재 (`src/errors.ts`)
- 17개 단위 테스트 통과 (auth 12개 + client 5개)

### 핵심 발견:
- Bearer 토큰 자동 주입 fetch 래퍼가 존재하지 않음 → 2단계에서 구현 필요
- `FortemAuth.getValidToken()`이 이미 자동 갱신 지원 → Bearer 래퍼에서 활용 가능
- 이미지 업로드(`PUT multipart/form-data`)는 기존 `Content-Type: application/json` 기본값과 충돌 → `FormData` 사용 시 `Content-Type` 제거 필요
- 민팅(POST) 시 토큰이 소모되며, 서버는 401(인증 에러)과 402(토큰 소모/만료)를 상태 코드로 구분
- mock fetch 패턴(`createMockFetch`)이 `src/__tests__/auth.test.ts:8-19`에 정의 → 새 테스트에서 재사용 가능

## 원하는 최종 상태

```
createFortemClient(options)
  └── FortemClient
      ├── _fetch: Fetch (x-api-key 자동 주입)
      ├── _networkConfig: { apiBaseUrl, serviceUrl }
      ├── auth: FortemAuth (기존)
      ├── users: FortemUsers ← NEW
      │   └── verify(walletAddress) — GET /users/:walletAddress
      ├── collections: FortemCollections ← NEW
      │   ├── list() — GET /collections
      │   └── create(params) — POST /collections
      └── items: FortemItems ← NEW
          ├── get(collectionId, code) — GET /collections/:id/items/:code
          ├── create(collectionId, params) — POST /collections/:id/items
          └── uploadImage(collectionId, file) — PUT multipart/form-data
```

검증 방법:
- `pnpm test` — 모든 단위 테스트 통과 (기존 17개 + 신규)
- `pnpm run typecheck` — 타입 에러 없음
- `pnpm run build` — ESM + CJS 듀얼 빌드 성공
- package.json version이 `0.0.2`

## 하지 않을 것

- Node.js 파일 시스템 편의 메서드 (이미지 업로드는 `Blob`/`File`만 수용)
- 통합 테스트 (실제 API 호출) — 단위 테스트(mock)만 작성
- README.md 업데이트 — 별도 작업으로 분리
- npm publish — 코드 구현 및 테스트만 수행

## 구현 접근 방식

기존 1단계 패턴(Factory + Facade + 서브모듈)을 그대로 확장한다. 각 서브모듈(`FortemUsers`, `FortemCollections`, `FortemItems`)은 `FortemAuth`와 동일한 생성자 패턴(`_apiBaseUrl`, `_fetch`)을 따르되, Bearer 토큰이 자동 주입되는 authenticated fetch를 받는다.

Bearer 토큰 전략은 이중 안전망:
1. **선제 처리**: 민팅 성공 후 `clearToken()` 자동 호출 → 다음 요청에서 자연스럽게 새 토큰 발급
2. **안전망**: 모든 authenticated 요청에서 402 응답 시 토큰 재발급 + 1회 재시도

---

## 1단계: 인프라 확장 — 에러, 상수, 타입

### 개요
2단계 API 모듈에 필요한 기반을 먼저 구축한다: 새 에러 클래스, 토큰 소모 상수, API 응답 타입.

### 필요한 변경:

#### 1. 에러 클래스 추가
**파일**: `src/errors.ts`
**변경**: `FortemTokenExpiredError` 추가 (402 전용)

```typescript
/** Token expired/consumed error (402) */
export class FortemTokenExpiredError extends FortemError {
  constructor(message: string = "Access token expired or consumed") {
    super(message, 402, "TOKEN_EXPIRED");
    this.name = "FortemTokenExpiredError";
  }
}
```

#### 2. 상수 추가
**파일**: `src/constants.ts`
**변경**: 토큰 소모 상태 코드 상수 추가

```typescript
/** HTTP status code indicating token expired/consumed (may change to 403 in the future) */
export const TOKEN_EXPIRED_STATUS = 402;
```

#### 3. parseResponse 402 처리 추가
**파일**: `src/fetch.ts`
**변경**: 401 분기 아래에 402 → `FortemTokenExpiredError` 분기 추가

```typescript
import { FortemError, FortemAuthError, FortemTokenExpiredError } from "./errors";
import { TOKEN_EXPIRED_STATUS } from "./constants";

// parseResponse 내부, 기존 401 분기 아래에 추가:
if (response.status === TOKEN_EXPIRED_STATUS) {
  throw new FortemTokenExpiredError(message);
}
```

#### 4. 2단계 API 타입 추가
**파일**: `src/types.ts`
**변경**: Users, Collections, Items API 응답/요청 타입 추가

```typescript
// ── Users API ──

/** User verification response */
export interface UserResponse {
  isUser: boolean;
  nickname: string;
  profileImage: string;
  walletAddress: string;
}

// ── Collections API ──

/** Collection object */
export interface Collection {
  id: number;
  objectId: string;
  name: string;
  description: string;
  tradeVolume: number;
  itemCount: number;
  createdAt: string;
  updatedAt: string;
}

/** Collection link */
export interface CollectionLink {
  website?: string;
}

/** Parameters for creating a collection */
export interface CreateCollectionParams {
  name: string;
  description: string;
  link?: CollectionLink;
}

// ── Items API ──

/** Item attribute */
export interface ItemAttribute {
  name: string;
  value: string;
}

/** Item owner */
export interface ItemOwner {
  nickname: string;
  walletAddress: string;
}

/** Item status */
export type ItemStatus = "PROCESSING" | "REDEEMED";

/** Item object */
export interface Item {
  id: number;
  objectId: string;
  name: string;
  description: string;
  nftNumber: number;
  itemImage: string;
  quantity: number;
  attributes: ItemAttribute[];
  owner: ItemOwner;
  status: ItemStatus;
  createdAt: string;
  updatedAt: string;
}

/** Parameters for creating an item */
export interface CreateItemParams {
  name: string;
  quantity: number;
  redeemCode: string;
  description: string;
  recipientAddress: string;
  itemImage?: string;
  attributes?: ItemAttribute[];
  redeemUrl?: string;
}

/** Image upload response */
export interface ImageUploadResponse {
  itemImage: string;
}
```

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] 기존 17개 테스트 통과: `pnpm test`
- [ ] 빌드 성공: `pnpm run build`

---

## 2단계: Authenticated Fetch 래퍼

### 개요
`FortemClient`에서 Bearer 토큰을 자동 주입하고, 402 응답 시 토큰 재발급 + 1회 재시도하는 래퍼를 생성한다. 이 래퍼를 2단계 서브모듈들에 전달한다.

### 필요한 변경:

#### 1. AuthenticatedFetch 타입 및 생성 로직
**파일**: `src/client.ts`
**변경**: `_createAuthenticatedFetch()` private 메서드 추가

```typescript
import { FortemTokenExpiredError } from "./errors";

// FortemClient 클래스 내부:

/**
 * Creates a fetch wrapper that:
 * 1. Injects Authorization: Bearer <token> header via auth.getValidToken()
 * 2. On 402 response: clears token, re-fetches new token, retries once
 */
private _createAuthenticatedFetch(): Fetch {
  return async (input, init) => {
    const token = await this.auth.getValidToken();
    const headers = new Headers(init?.headers);
    headers.set("Authorization", `Bearer ${token}`);

    const response = await this._fetch(input, { ...init, headers });

    // 402: token expired/consumed → retry once with fresh token
    if (response.status === TOKEN_EXPIRED_STATUS) {
      this.auth.clearToken();
      const newToken = await this.auth.getValidToken();
      const retryHeaders = new Headers(init?.headers);
      retryHeaders.set("Authorization", `Bearer ${newToken}`);
      return this._fetch(input, { ...init, headers: retryHeaders });
    }

    return response;
  };
}
```

주의: 이 래퍼는 `this._fetch`(x-api-key 주입 완료) 위에 쌓이므로, 최종 요청에는 `x-api-key` + `Authorization: Bearer` 두 헤더가 모두 포함된다.

#### 2. FortemClient 생성자에서 래퍼 생성
**파일**: `src/client.ts`
**변경**: `_authenticatedFetch` 필드 추가 및 생성자에서 초기화

```typescript
private readonly _authenticatedFetch: Fetch;

// 생성자 내:
this._authenticatedFetch = this._createAuthenticatedFetch();
```

#### 3. Authenticated Fetch 테스트
**파일**: `src/__tests__/client.test.ts`
**변경**: authenticated fetch의 Bearer 주입 및 402 재시도 동작을 검증하는 테스트 추가.

테스트 항목:
- authenticated fetch가 Bearer 토큰 헤더를 주입하는지
- 402 응답 시 토큰 재발급 후 1회 재시도하는지
- 재시도 후에도 402면 그대로 반환하는지 (무한 재시도 방지)
- 401 응답은 재시도 없이 그대로 반환하는지

> 참고: `_authenticatedFetch`는 private이므로, 서브모듈(`users`, `collections`, `items`)을 통해 간접 테스트하거나, 서브모듈 구현 전에는 `FortemClient`를 상속한 테스트용 클래스로 접근할 수 있다. 3~5단계에서 서브모듈 테스트를 통해 자연스럽게 검증된다.

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] 기존 + 신규 테스트 통과: `pnpm test`

---

## 3단계: FortemUsers 서브모듈

### 개요
유저 검증 API를 래핑하는 `FortemUsers` 클래스를 구현한다.

### 필요한 변경:

#### 1. FortemUsers 클래스
**파일**: `src/users.ts` (신규)

```typescript
import { parseResponse, type Fetch } from "./fetch";
import type { UserResponse } from "./types";
import type { FortemResponse } from "./types";

export class FortemUsers {
  constructor(
    private readonly _apiBaseUrl: string,
    private readonly _fetch: Fetch
  ) {}

  /**
   * Verify if a wallet address is a registered ForTem user.
   *
   * @param walletAddress - The wallet address to verify
   * @returns User information if found
   */
  async verify(walletAddress: string): Promise<FortemResponse<UserResponse>> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/users/${walletAddress}`,
      { method: "GET" }
    );
    return parseResponse<UserResponse>(response);
  }
}
```

#### 2. FortemClient에 users 서브모듈 등록
**파일**: `src/client.ts`
**변경**: `users: FortemUsers` 필드 추가, 생성자에서 초기화

```typescript
import { FortemUsers } from "./users";

// 클래스 필드:
readonly users: FortemUsers;

// 생성자 내 (auth 초기화 후):
this.users = new FortemUsers(this._networkConfig.apiBaseUrl, this._authenticatedFetch);
```

#### 3. 테스트
**파일**: `src/__tests__/users.test.ts` (신규)

테스트 항목:
- `verify(walletAddress)` 성공 시 `UserResponse` 반환
- 올바른 엔드포인트(`/api/v1/developers/users/:addr`)로 GET 요청
- Authorization Bearer 헤더 포함 검증
- 404 응답 시 `FortemError` throw
- 402 응답 시 토큰 재발급 후 재시도 (authenticated fetch 통합 검증)

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] 신규 users 테스트 전부 통과: `pnpm test`

---

## 4단계: FortemCollections 서브모듈

### 개요
컬렉션 목록 조회 및 생성 API를 래핑하는 `FortemCollections` 클래스를 구현한다. `create()` 성공 시 토큰 캐시를 선제적으로 무효화한다.

### 필요한 변경:

#### 1. FortemCollections 클래스
**파일**: `src/collections.ts` (신규)

```typescript
import { parseResponse, type Fetch } from "./fetch";
import type { Collection, CreateCollectionParams, FortemResponse } from "./types";
import type { FortemAuth } from "./auth";

export class FortemCollections {
  constructor(
    private readonly _apiBaseUrl: string,
    private readonly _fetch: Fetch,
    private readonly _auth: FortemAuth
  ) {}

  /**
   * List all collections.
   */
  async list(): Promise<FortemResponse<Collection[]>> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/collections`,
      { method: "GET" }
    );
    return parseResponse<Collection[]>(response);
  }

  /**
   * Create a new collection.
   * Note: This consumes the current access token (minting operation).
   *
   * @param params - Collection creation parameters
   */
  async create(params: CreateCollectionParams): Promise<FortemResponse<Collection>> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/collections`,
      {
        method: "POST",
        body: JSON.stringify(params),
      }
    );
    const result = await parseResponse<Collection>(response);

    // Proactively invalidate cached token after minting
    this._auth.clearToken();

    return result;
  }
}
```

주의: `FortemCollections`는 `_auth` 참조를 받아 민팅 후 `clearToken()`을 호출한다. `FortemUsers`는 읽기 전용이므로 `_auth` 불필요.

#### 2. FortemClient에 collections 서브모듈 등록
**파일**: `src/client.ts`

```typescript
import { FortemCollections } from "./collections";

readonly collections: FortemCollections;

// 생성자 내:
this.collections = new FortemCollections(
  this._networkConfig.apiBaseUrl,
  this._authenticatedFetch,
  this.auth
);
```

#### 3. 테스트
**파일**: `src/__tests__/collections.test.ts` (신규)

테스트 항목:
- `list()` 성공 시 `Collection[]` 반환
- `create(params)` 성공 시 `Collection` 반환
- `create()` 성공 후 토큰 캐시가 무효화됨 (clearToken 호출 확인)
- 올바른 엔드포인트 + HTTP 메서드 사용
- Authorization Bearer 헤더 포함 검증
- `create()` 시 body에 params가 JSON으로 전달됨

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] 신규 collections 테스트 전부 통과: `pnpm test`

---

## 5단계: FortemItems 서브모듈

### 개요
아이템 조회, 생성, 이미지 업로드 API를 래핑하는 `FortemItems` 클래스를 구현한다. `create()` 성공 시 토큰 캐시 무효화. `uploadImage()`는 `multipart/form-data`를 사용하며 `Content-Type` 헤더를 명시적으로 제거한다.

### 필요한 변경:

#### 1. FortemItems 클래스
**파일**: `src/items.ts` (신규)

```typescript
import { parseResponse, type Fetch } from "./fetch";
import type {
  Item,
  CreateItemParams,
  ImageUploadResponse,
  FortemResponse,
} from "./types";
import type { FortemAuth } from "./auth";

export class FortemItems {
  constructor(
    private readonly _apiBaseUrl: string,
    private readonly _fetch: Fetch,
    private readonly _auth: FortemAuth
  ) {}

  /**
   * Get an item by its redeem code.
   *
   * @param collectionId - The collection ID
   * @param code - The redeem code
   */
  async get(collectionId: number, code: string): Promise<FortemResponse<Item>> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/collections/${collectionId}/items/${code}`,
      { method: "GET" }
    );
    return parseResponse<Item>(response);
  }

  /**
   * Create a new item in a collection.
   * Note: This consumes the current access token (minting operation).
   *
   * @param collectionId - The collection ID
   * @param params - Item creation parameters
   */
  async create(
    collectionId: number,
    params: CreateItemParams
  ): Promise<FortemResponse<Item>> {
    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/collections/${collectionId}/items`,
      {
        method: "POST",
        body: JSON.stringify(params),
      }
    );
    const result = await parseResponse<Item>(response);

    // Proactively invalidate cached token after minting
    this._auth.clearToken();

    return result;
  }

  /**
   * Upload an image for an item.
   *
   * @param collectionId - The collection ID
   * @param file - Image file (Blob or File)
   */
  async uploadImage(
    collectionId: number,
    file: Blob
  ): Promise<FortemResponse<ImageUploadResponse>> {
    const formData = new FormData();
    formData.append("file", file);

    const response = await this._fetch(
      `${this._apiBaseUrl}/api/v1/developers/collections/${collectionId}/items/image-upload`,
      {
        method: "PUT",
        body: formData,
        headers: { "Content-Type": "" },
        // Note: Content-Type will be deleted so the browser/runtime
        // sets the correct multipart boundary automatically.
      }
    );
    return parseResponse<ImageUploadResponse>(response);
  }
}
```

**이미지 업로드 Content-Type 처리**: `createFetchWithApiKey`에서 `Content-Type`이 이미 존재하면 덮어쓰지 않는다(`!headers.has("Content-Type")` 체크, `src/fetch.ts:17`). 빈 문자열로 설정 후 실제 전송 전에 삭제하거나, 아니면 `FormData` 전달 시 `Content-Type` 헤더를 아예 설정하지 않는 별도 처리가 필요하다.

**구현 시 결정 필요**: `createFetchWithApiKey`의 `Content-Type` 기본값 로직을 개선하여, `body`가 `FormData` 인스턴스일 때 `Content-Type`을 설정하지 않도록 수정하는 것이 가장 깔끔하다:

```typescript
// src/fetch.ts — createFetchWithApiKey 내부 수정:
if (!headers.has("Content-Type") && !(init?.body instanceof FormData)) {
  headers.set("Content-Type", "application/json");
}
```

이렇게 하면 `uploadImage`에서 별도 헤더 조작 없이 `FormData`를 body로 전달하기만 하면 된다.

#### 2. FortemClient에 items 서브모듈 등록
**파일**: `src/client.ts`

```typescript
import { FortemItems } from "./items";

readonly items: FortemItems;

// 생성자 내:
this.items = new FortemItems(
  this._networkConfig.apiBaseUrl,
  this._authenticatedFetch,
  this.auth
);
```

#### 3. 테스트
**파일**: `src/__tests__/items.test.ts` (신규)

테스트 항목:
- `get(collectionId, code)` 성공 시 `Item` 반환
- `create(collectionId, params)` 성공 시 `Item` 반환
- `create()` 성공 후 토큰 캐시 무효화 확인
- `uploadImage(collectionId, file)` 성공 시 `ImageUploadResponse` 반환
- `uploadImage()` 시 `Content-Type: application/json`이 설정되지 않음 확인
- `uploadImage()` 시 body가 `FormData` 인스턴스임을 확인
- 올바른 엔드포인트 + HTTP 메서드 사용
- Authorization Bearer 헤더 포함 검증

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] 신규 items 테스트 전부 통과: `pnpm test`

---

## 6단계: 통합 및 마무리

### 개요
모든 서브모듈을 `index.ts`에서 re-export하고, `FortemClient` 테스트를 업데이트하며, 버전을 v0.0.2로 범프한다.

### 필요한 변경:

#### 1. index.ts re-exports 업데이트
**파일**: `src/index.ts`

```typescript
// 기존 re-exports에 추가:
export { FortemUsers } from "./users";
export { FortemCollections } from "./collections";
export { FortemItems } from "./items";
export { FortemTokenExpiredError } from "./errors";
export type {
  // 기존 타입들...
  UserResponse,
  Collection,
  CollectionLink,
  CreateCollectionParams,
  Item,
  ItemAttribute,
  ItemOwner,
  ItemStatus,
  CreateItemParams,
  ImageUploadResponse,
} from "./types";
```

#### 2. FortemClient 테스트 업데이트
**파일**: `src/__tests__/client.test.ts`
**변경**: 새 서브모듈 존재 확인 테스트 추가

```typescript
it("should have users sub-module", () => {
  const client = createFortemClient({ apiKey: TEST_API_KEY, network: TEST_NETWORK });
  expect(client.users).toBeDefined();
});

it("should have collections sub-module", () => {
  const client = createFortemClient({ apiKey: TEST_API_KEY, network: TEST_NETWORK });
  expect(client.collections).toBeDefined();
});

it("should have items sub-module", () => {
  const client = createFortemClient({ apiKey: TEST_API_KEY, network: TEST_NETWORK });
  expect(client.items).toBeDefined();
});
```

#### 3. 버전 범프
**파일**: `package.json`
**변경**: `"version": "0.0.1"` → `"version": "0.0.2"`

#### 4. createMockFetch 공유 유틸리티 추출
**파일**: `src/__tests__/helpers.ts` (신규)
**변경**: `auth.test.ts`의 `createMockFetch`를 공유 유틸리티로 추출하여 모든 테스트에서 import

```typescript
import { vi } from "vitest";

export function createMockFetch(responses: Array<{ status: number; body: unknown }>) {
  let callIndex = 0;
  return vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) => {
    const res = responses[callIndex++];
    return {
      ok: res.status >= 200 && res.status < 300,
      status: res.status,
      json: async () => res.body,
      headers: new Headers(),
    } as Response;
  });
}
```

기존 `auth.test.ts`에서 로컬 `createMockFetch` 삭제 후 `import { createMockFetch } from "./helpers"` 사용.

### 성공 기준:

#### 자동화된 검증:
- [ ] 타입 체크 통과: `pnpm run typecheck`
- [ ] **모든** 테스트 통과: `pnpm test`
- [ ] 빌드 성공: `pnpm run build`
- [ ] package.json version이 `0.0.2`

---

## 테스트 전략

### 단위 테스트:
- **users.test.ts**: verify() 성공/실패, Bearer 헤더 주입, 402 재시도
- **collections.test.ts**: list() 성공, create() 성공 + 토큰 무효화, Bearer 헤더
- **items.test.ts**: get() 성공, create() 성공 + 토큰 무효화, uploadImage() FormData 처리
- **client.test.ts**: 서브모듈 존재 확인 (users, collections, items)
- **auth.test.ts**: 기존 12개 테스트 유지 (변경 없음)

### 핵심 엣지 케이스:
- 402 재시도 후에도 실패 → `FortemTokenExpiredError` throw (무한 재시도 방지)
- `uploadImage()`에서 `Content-Type`이 `application/json`으로 설정되지 않음
- 민팅 후 다음 요청에서 `getValidToken()`이 새 토큰을 발급하는지

### 공유 테스트 유틸리티:
- `createMockFetch()` — `src/__tests__/helpers.ts`로 추출

## 최종 파일 구조

```
sdk-js/src/
├── index.ts          # 팩토리 + re-exports (확장)
├── client.ts         # FortemClient (서브모듈 추가 + authenticated fetch)
├── auth.ts           # FortemAuth (변경 없음)
├── users.ts          # FortemUsers (신규)
├── collections.ts    # FortemCollections (신규)
├── items.ts          # FortemItems (신규)
├── fetch.ts          # fetch 래퍼 + parseResponse (402 처리 + FormData 지원)
├── types.ts          # 타입 정의 (확장)
├── errors.ts         # 에러 클래스 (FortemTokenExpiredError 추가)
├── constants.ts      # 네트워크 설정 + TOKEN_EXPIRED_STATUS
└── __tests__/
    ├── helpers.ts         # 공유 테스트 유틸리티 (신규)
    ├── auth.test.ts       # FortemAuth 테스트 (기존, import 수정)
    ├── client.test.ts     # FortemClient 테스트 (확장)
    ├── users.test.ts      # FortemUsers 테스트 (신규)
    ├── collections.test.ts # FortemCollections 테스트 (신규)
    └── items.test.ts      # FortemItems 테스트 (신규)
```

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-18-sdk-js-phase2-api-modules-research.md`
- 1단계 연구: `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md`
- 1단계 계획: `thoughts/arta1069/plans/2026-02-18-fortem-sdk-web-phase1.md`
