---
date: 2026-02-18T14:24:00+0900
researcher: arta1069@gmail.com
git_commit: ee3b97cbd70ea41ae63e610c6fd3ee2999669531
branch: main
repository: sdk-js
topic: "sdk-js 2단계 고도화를 위한 현재 상태 분석 및 API 모듈 확장 연구"
tags: [research, codebase, sdk-js, fortem, users-api, collections-api, items-api, phase2]
status: complete
last_updated: 2026-02-18
last_updated_by: arta1069
last_updated_note: "402 토큰 소모 에러 코드 분리 반영 — 토큰 전략 재분석"
---

# 연구: sdk-js 2단계 고도화를 위한 현재 상태 분석 및 API 모듈 확장 연구

**날짜**: 2026-02-18T14:24:00+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: ee3b97cbd70ea41ae63e610c6fd3ee2999669531
**브랜치**: main
**리포지토리**: sdk-js

## 연구 질문

1단계 구현(`createFortemClient()` + `FortemAuth`)이 완료된 상태에서, 2단계 API(Users, Collections, Items)를 추가하기 위한 현재 코드베이스 상태는 어떤가? 새 서브모듈을 추가할 때 기존 패턴(fetch 래퍼, 에러 처리, Bearer 토큰 주입)을 어떻게 활용할 수 있는가?

## 요약

`@fortemlabs/sdk-js` v0.0.1은 1단계 구현이 완료된 상태로, `createFortemClient()` → `FortemClient` → `FortemAuth` 구조가 동작 중이다. 9개 소스 파일(src 6개, 테스트 2개, 테스트 유틸리티 포함)로 구성되며, 17개 단위 테스트가 통과한다. 현재 인증(nonce → access-token) 플로우만 구현되어 있으며, 2단계 API(Users, Collections, Items)는 미구현 상태이다.

**핵심 발견**:
- `Authorization: Bearer` 헤더를 자동 주입하는 fetch 래퍼가 **현재 존재하지 않음** — 2단계에서 구현 필요
- `FortemAuth.getValidToken()`이 이미 토큰 자동 갱신을 지원하므로, Bearer 헤더 주입 래퍼에서 이를 활용 가능
- 새 서브모듈(`FortemUsers`, `FortemCollections`, `FortemItems`)은 기존 `FortemAuth`와 동일한 생성자 패턴(`_apiBaseUrl`, `_fetch`)으로 추가 가능
- 이미지 업로드(`PUT multipart/form-data`)는 기존 `Content-Type: application/json` 기본값과 충돌하므로 별도 처리 필요

## 상세 발견 사항

### 현재 파일 구조

```
sdk-js/
├── src/
│   ├── index.ts          # 팩토리 함수 + re-exports
│   ├── client.ts         # FortemClient 클래스 (Facade)
│   ├── auth.ts           # FortemAuth 서브모듈
│   ├── fetch.ts          # fetch 래퍼 + parseResponse
│   ├── types.ts          # 타입 정의
│   ├── errors.ts         # FortemError, FortemAuthError
│   ├── constants.ts      # 네트워크 설정
│   └── __tests__/
│       ├── auth.test.ts  # FortemAuth 테스트 (12 tests)
│       └── client.test.ts # FortemClient 테스트 (5 tests)
├── package.json          # v0.0.1, tsup dual format
├── tsup.config.ts        # ESM + CJS
├── tsconfig.json
├── vitest.config.ts
├── README.md
└── LICENSE               # BSL 1.1
```

### 현재 클라이언트 구조 (`src/client.ts`)

`FortemClient`는 Facade 패턴으로 서브모듈을 노출한다:

```typescript
export class FortemClient {
  readonly auth: FortemAuth;           // 현재 유일한 서브모듈
  private readonly _fetch: Fetch;      // x-api-key 자동 주입 래퍼
  private readonly _networkConfig: NetworkConfig;

  constructor(options: FortemClientOptions) {
    // apiKey 검증 → 네트워크 설정 → fetch 래퍼 생성 → 서브모듈 초기화
    this.auth = new FortemAuth(this._networkConfig.apiBaseUrl, this._fetch);
  }
}
```

2단계에서 새 서브모듈 추가 시:
```typescript
readonly users: FortemUsers;
readonly collections: FortemCollections;
// items는 collections의 하위로 접근하거나 별도 서브모듈로 노출
```

### fetch 래퍼 패턴 (`src/fetch.ts`)

**`createFetchWithApiKey()`** — 현재 유일한 fetch 래퍼:
- `x-api-key` 헤더 자동 주입 (`fetch.ts:15`)
- `Content-Type: application/json` 기본값 설정 (`fetch.ts:17-19`)
- `customFetch` 파라미터로 테스트 시 mock 주입 가능

**`parseResponse<T>()`** — 응답 파싱 + 에러 처리:
- `response.json()` → `Record<string, unknown>` 캐스팅 (`fetch.ts:29`)
- 401 → `FortemAuthError`, 기타 에러 → `FortemError` (`fetch.ts:37-41`)
- 성공 → `FortemResponse<T>` (`{ statusCode, data }`) 반환 (`fetch.ts:44`)

**2단계에서 필요한 확장**: `Authorization: Bearer <token>` 헤더 자동 주입 기능
- 현재 `createFetchWithApiKey()`는 `x-api-key`만 주입
- 인증된 API(Users, Collections, Items)는 `Authorization: Bearer` 헤더 필요
- `FortemAuth.getValidToken()`과 연동하여 토큰 자동 주입 + 만료 시 재발급 래퍼 필요

### 인증 모듈 (`src/auth.ts`)

**현재 상태**: nonce → access-token 2단계 플로우 구현 완료
- `getNonce()` — `POST /api/v1/developers/auth/nonce`
- `getAccessToken(nonce)` — `POST /api/v1/developers/auth/access-token`
- `getValidToken()` — 캐시 유효 시 재사용, 만료 시 자동 갱신
- `getToken()` — 캐시된 토큰 반환 (null if expired)
- `clearToken()` — 캐시 초기화

**토큰 캐싱 상수**:
- `TOKEN_TTL_MS = 5 * 60 * 1000` (5분)
- `TOKEN_EXPIRY_MARGIN_MS = 30_000` (30초 조기 만료)

### 타입 시스템 (`src/types.ts`)

**현재 정의된 타입**:
- `FortemNetwork` — `"testnet" | "mainnet"`
- `FortemClientOptions` — `{ apiKey, network?, fetch? }`
- `FortemResponse<T>` — `{ statusCode: number, data: T }`
- `NonceResponse` — `{ nonce: string }`
- `AccessTokenResponse` — `{ accessToken: string }`
- `NetworkConfig` — `{ apiBaseUrl, serviceUrl }`

**2단계에서 추가 필요한 타입** (사용자 제공 API 스펙 기준):

1. **Users API 타입**:
   - `UserResponse` — `{ isUser, nickname, profileImage, walletAddress }`

2. **Collections API 타입**:
   - `Collection` — `{ id, objectId, name, description, tradeVolume, itemCount, createdAt, updatedAt }`
   - `CollectionLink` — `{ website?: string }`
   - `CreateCollectionParams` — `{ name, description, link? }`
   - Collections 목록 응답은 `FortemResponse<Collection[]>` 형태

3. **Items API 타입**:
   - `Item` — `{ id, objectId, name, description, nftNumber, itemImage, quantity, attributes, owner, status, createdAt, updatedAt }`
   - `ItemAttribute` — `{ name, value }`
   - `ItemOwner` — `{ nickname, walletAddress }`
   - `CreateItemParams` — `{ name, quantity, redeemCode, description, recipientAddress, itemImage?, attributes?, redeemUrl? }`
   - `ImageUploadResponse` — `{ itemImage: string }` (CID)
   - `ItemStatus` — `"PROCESSING" | "REDEEMED" | ...`

### 에러 클래스 (`src/errors.ts`)

```
Error
└── FortemError (statusCode, code?)
    └── FortemAuthError (401, "AUTH_ERROR")
```

현재 2개 에러 클래스가 존재한다. 2단계 API에서 발생할 수 있는 에러(404 Not Found, 400 Bad Request 등)는 기존 `FortemError`로 충분히 처리 가능하다.

### 테스트 패턴 (`src/__tests__/`)

**Mock Fetch 생성 패턴** (`auth.test.ts:8-19`):
```typescript
function createMockFetch(responses: Array<{ status: number; body: unknown }>) {
  let callIndex = 0;
  return vi.fn(async () => {
    const res = responses[callIndex++];
    return { ok, status, json, headers } as Response;
  });
}
```

- 순차적 응답 배열로 여러 API 호출 시나리오 테스트
- `createFortemClient({ ..., fetch: mockFetch })`로 mock 주입
- `mockFetch.mock.calls`로 호출 인자 검증

### whoosh-bang 내 관련 패턴

**nonce → verify 패턴** (`WalletLinkButton.tsx:55-84`):
- ForTem 인증 플로우와 구조적으로 동일한 2단계 패턴
- `response.ok`로 성공/실패 분기
- 에러 시 `toast.error(data.error)` 표시

**2단계 API 직접 호출**: whoosh-bang에서 `developers/users`, `developers/collections`, `developers/items` 엔드포인트를 직접 호출하는 코드는 현재 존재하지 않음. 이 엔드포인트들은 sdk-js를 통해 사용될 예정.

### 네트워크 설정 (`src/constants.ts`)

| 네트워크 | apiBaseUrl | serviceUrl |
|---------|-----------|------------|
| testnet | `https://testnet-api.fortem.gg` | `https://testnet.fortem.gg` |
| mainnet | `https://api.fortem.gg` | `https://fortem.gg` |

모든 엔드포인트는 `${apiBaseUrl}/api/v1/developers/...` 패턴으로 조합된다.

## 2단계 API 스펙 요약

사용자가 제공한 API 스펙:

| API | Method | Endpoint | Auth |
|-----|--------|----------|------|
| 유저 검증 | `GET` | `/api/v1/developers/users/:walletAddress` | Bearer |
| 컬렉션 목록 | `GET` | `/api/v1/developers/collections` | Bearer |
| 컬렉션 생성 | `POST` | `/api/v1/developers/collections` | Bearer |
| 아이템 조회 (리딤코드) | `GET` | `/api/v1/developers/collections/:collectionId/items/:code` | Bearer |
| 아이템 생성 | `POST` | `/api/v1/developers/collections/:collectionId/items` | Bearer |
| 이미지 업로드 | `PUT` | `/api/v1/developers/collections/:collectionId/items/image-upload` | Bearer |

**공통 특징**:
- 모든 API는 `Authorization: Bearer <access_token>` 필요
- 모든 응답은 `{ statusCode, data }` 래퍼 구조
- JSON 기반 (이미지 업로드만 `multipart/form-data`)

## 코드 참조

- `src/client.ts:6-38` — FortemClient 클래스 (서브모듈 확장 지점)
- `src/fetch.ts:7-23` — createFetchWithApiKey (Bearer 래퍼 확장 필요)
- `src/fetch.ts:26-45` — parseResponse (재사용 가능)
- `src/auth.ts:94-101` — getValidToken (Bearer 토큰 공급원)
- `src/types.ts` — 타입 정의 (확장 필요)
- `src/index.ts` — 엔트리포인트 re-exports (확장 필요)
- `src/__tests__/auth.test.ts:8-19` — mock fetch 패턴 (재사용 가능)

## 아키텍처 문서화

### 현재 패턴

```
createFortemClient(options)
  └── FortemClient
      ├── _fetch: Fetch (x-api-key 자동 주입)
      ├── _networkConfig: { apiBaseUrl, serviceUrl }
      └── auth: FortemAuth
          ├── getNonce() — POST /auth/nonce
          ├── getAccessToken(nonce) — POST /auth/access-token
          ├── getValidToken() — 자동 갱신
          ├── getToken() — 캐시 반환
          └── clearToken() — 캐시 초기화
```

### 2단계 확장 시 예상 구조

```
createFortemClient(options)
  └── FortemClient
      ├── _fetch: Fetch (x-api-key 자동 주입)
      ├── _authFetch: Fetch (Bearer token 자동 주입) ← NEW
      ├── _networkConfig: { apiBaseUrl, serviceUrl }
      ├── auth: FortemAuth (기존)
      ├── users: FortemUsers ← NEW
      │   └── verify(walletAddress) — GET /users/:walletAddress
      ├── collections: FortemCollections ← NEW (플랫 패턴)
      │   ├── list() — GET /collections
      │   └── create(params) — POST /collections
      └── items: FortemItems ← NEW (플랫 패턴)
          ├── get(collectionId, code) — GET /collections/:id/items/:code
          ├── create(collectionId, params) — POST /collections/:id/items
          └── uploadImage(collectionId, file) — PUT multipart/form-data (Blob/File만 수용)
```

### Bearer 토큰 주입 전략

`FortemAuth.getValidToken()`을 활용하여 인증된 fetch 래퍼를 생성해야 한다:
- 요청 전 `auth.getValidToken()`으로 유효한 토큰 확보
- `Authorization: Bearer <token>` 헤더 자동 주입
- 401 응답 시 토큰 갱신 후 재시도 (optional)

### 이미지 업로드 특수 처리

- `PUT /collections/:id/items/image-upload` — `Content-Type: multipart/form-data`
- 기존 fetch 래퍼의 `Content-Type: application/json` 기본값과 충돌
- `FormData` 사용 시 브라우저가 자동으로 `Content-Type`에 boundary를 설정하므로, `Content-Type` 헤더를 명시적으로 제거해야 함

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md` — 1단계 연구 문서. 프로젝트 배치 구조(별도 리포 결정), 빌드 파이프라인(tsup dual format), Supabase SDK Factory + Facade 패턴 참조 결정
- `thoughts/arta1069/plans/2026-02-18-fortem-sdk-web-phase1.md` — 1단계 구현 계획. 모든 성공 기준이 [x]로 체크되어 1단계 완료 확인. line 53에 "2단계 API (users, items, collections CRUD)"가 "하지 않을 것" 항목으로 명시되어 있음

## 관련 연구

- `thoughts/arta1069/research/2026-02-17-fortem-sdk-web-monorepo-structure-research.md` — 1단계 코드베이스 연구

## 후속 연구 2026-02-18T14:40:00+0900

### 민팅 시 1회성 토큰 소모 전략 분석

1단계 연구(`2026-02-17-fortem-sdk-web-monorepo-structure-research.md:288`)에서 확인된 access token 특성:
- **유효시간**: 5분 TTL
- **1회성**: 민팅(생성) API 호출 시 토큰이 소모되며 재사용 불가

#### 실제 사용 흐름 분석

```
유저 조회 (GET)      → 토큰 A ✓  (소모 안 됨)
콜렉션 조회 (GET)    → 토큰 A ✓  (소모 안 됨)
아이템 조회 (GET)    → 토큰 A ✓  (소모 안 됨)
아이템 민팅 (POST)   → 토큰 A ✓  (소모됨 → 토큰 A 무효화)
─────────────────────────────────────────────
민팅 아이템 조회 (GET) → 토큰 A ✗ (401) → 토큰 B 재발급 필요
```

#### 현재 `getValidToken()` 한계

현재 구현(`src/auth.ts:94-101`)은 **TTL 기반 만료만 체크**한다:
```typescript
async getValidToken(): Promise<string> {
  const cached = this.getToken();  // TTL 체크만 수행
  if (cached) return cached;       // TTL 남아있으면 소모된 토큰도 반환
  // ...
}
```

민팅 후 TTL이 아직 남아있으면 **이미 소모된 토큰을 유효하다고 판단**하는 문제가 있다.

#### API별 토큰 소모 분류

| API | Method | 토큰 소모 |
|-----|--------|----------|
| 유저 검증 | `GET /users/:addr` | 소모 안 됨 (읽기) |
| 컬렉션 목록 | `GET /collections` | 소모 안 됨 (읽기) |
| 컬렉션 생성 | `POST /collections` | **소모됨 (민팅)** |
| 아이템 조회 | `GET /collections/:id/items/:code` | 소모 안 됨 (읽기) |
| 아이템 생성 | `POST /collections/:id/items` | **소모됨 (민팅)** |
| 이미지 업로드 | `PUT /collections/:id/items/image-upload` | 소모 안 됨 |

#### HTTP 상태 코드 분리에 따른 에러 분류

ForTem API는 인증 관련 에러를 상태 코드로 명확히 구분한다:

| 상태 코드 | 의미 | 대응 |
|-----------|------|------|
| **401** | 일반 인증 에러 (API Key 문제 등) | 재시도 의미 없음 → 즉시 throw |
| **402** | access token 소모/만료 | 토큰 재발급 후 1회 재시도 |

> **참고**: 현재 402로 확정. 추후 403으로 변경될 수 있으므로, SDK에서 토큰 소모 상태 코드를 상수(`TOKEN_EXPIRED_STATUS = 402`)로 관리하여 변경 시 한 곳만 수정하면 되도록 설계해야 한다.

#### 현재 `parseResponse()` 한계

현재 구현(`src/fetch.ts:37-39`)은 401만 `FortemAuthError`로 분기한다:
```typescript
if (response.status === 401) {
  throw new FortemAuthError(message);
}
```

2단계에서는 402 응답도 별도 처리가 필요하다.

#### Bearer 래퍼 설계 시 고려사항 (재분석)

**이전 분석의 옵션 A 단점이었던 "잘못된 토큰 vs 소모된 토큰 구분 어려움"이 상태 코드 분리(401 vs 402)로 해소되었다.**

~~**옵션 A 단독**~~ → 402 재시도만으로도 동작하지만, 민팅 후 다음 요청에서 불필요한 402 왕복 1회 발생
~~**옵션 B 단독**~~ → 캐시 무효화만으로는 예상치 못한 토큰 소모에 대응 불가
~~**옵션 C (기존)**~~ → 401 vs 402 구분이 없던 시점의 설계. 이제 불필요한 복잡도

**채택: 민팅 후 캐시 무효화 (B) + 402 자동 재시도 안전망 (A 개선)**

1. **선제 처리 (최적화)**: `POST /collections`, `POST /items` 성공 시 `clearToken()` 자동 실행
   - 다음 `getValidToken()` 호출 시 자연스럽게 새 토큰 발급
   - 불필요한 402 왕복을 방지

2. **안전망 (범용)**: 모든 Bearer API 호출에서 402 응답 시 자동 재발급 + 1회 재시도
   - 401은 즉시 `FortemAuthError` throw (재시도 안 함)
   - 무한 재시도 방지: 재시도 후에도 402면 throw
   - SDK가 예상치 못한 토큰 소모 상황에도 대응 가능

3. **상태 코드 상수화**: `TOKEN_EXPIRED_STATUS = 402`
   - 추후 402 → 403 변경 시 상수 한 곳만 수정하면 전체 반영

## 결정 사항

1. ~~**Items 서브모듈의 접근 패턴**~~ → **해결**: 플랫 패턴 채택. `fortem.items.create(collectionId, params)`, `fortem.collections.create(params)` 각각 독립 모듈로 접근.
2. ~~**Bearer 토큰 재시도 전략**~~ → **해결**: 402 상태 코드 분리로 전략 확정. 민팅 후 캐시 무효화(선제) + 402 자동 재시도(안전망). 상태 코드 상수화(`TOKEN_EXPIRED_STATUS = 402`)로 추후 403 변경 대응.
3. ~~**이미지 업로드의 Node.js 호환성**~~ → **해결**: 파일 시스템 편의 메서드 제공하지 않음. `Blob`/`File` 객체만 수용. 대신 타입 정의, JSDoc, 사용 예시를 명확히 제공하여 DX 확보.
4. ~~**이미지 업로드의 토큰 소모 여부**~~ → **해결**: 소모하지 않음.

## 미해결 질문

없음. 모든 설계 결정 완료.
