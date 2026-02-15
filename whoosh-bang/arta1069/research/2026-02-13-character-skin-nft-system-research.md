---
date: "2026-02-13T16:57:03+0900"
researcher: arta1069@gmail.com
git_commit: N/A (git not initialized)
branch: N/A
repository: game
topic: "캐릭터 스킨 NFT 시스템 - 현재 구현 상태 및 데이터 구조 분석"
tags: [research, codebase, character, skin, nft, lobby, phaser, supabase]
status: complete
last_updated: 2026-02-13
last_updated_by: arta1069@gmail.com
---

# 연구: 캐릭터 스킨 NFT 시스템 - 현재 구현 상태 및 데이터 구조 분석

**날짜**: 2026-02-13T16:57:03+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: N/A (git 미초기화)
**브랜치**: N/A
**리포지토리**: game

## 연구 질문

캐릭터 디스플레이가 정적으로 지정되어 있는 현재 상태에서, 캐릭터 스킨을 NFT 아이템화하여 유저가 스킨을 선택할 수 있게 하기 위한 현재 코드베이스/데이터 구조 분석.

## 요약

현재 코드베이스에는 **5종의 캐릭터 에셋**(Adventurer, Female, Player, Soldier, Zombie)이 동일한 구조(24 Poses + 9 Limbs + 1 Tilesheet)로 준비되어 있으나, **캐릭터 선택 시스템은 미구현**이다. 로비의 `CharacterDisplay`는 `Player` 캐릭터의 idle 포즈만 하드코딩으로 표시하고, Phaser `GameScene`에서는 Player 1 = `player`, Player 2 = `female`로 고정되어 있다. DB(`player_profiles`)에 캐릭터/스킨 관련 컬럼은 없으며, 별도의 캐릭터 소유권 테이블도 존재하지 않는다.

## 상세 발견 사항

### 1. 캐릭터 에셋 구조

**경로**: `apps/web/public/assets/characters/PNG/`

5개 캐릭터가 완전히 동일한 구조로 구성:

| 캐릭터 | 폴더명 | 게임 내 사용 여부 |
|--------|--------|------------------|
| Adventurer | `Adventurer/` | BootScene에서 로드됨 |
| Female | `Female/` | BootScene에서 로드됨, Player 2로 사용 |
| Player | `Player/` | BootScene에서 로드됨, Player 1로 사용, 로비 표시용 |
| Soldier | `Soldier/` | **에셋만 존재, 코드에서 미사용** |
| Zombie | `Zombie/` | **에셋만 존재, 코드에서 미사용** |

각 캐릭터 폴더 구성:
- `{character}_tilesheet.png` — 통합 타일시트 (1개)
- `Limbs/` — 신체 부위 분리 이미지 (9개: arm, body_back, body_front, hand, head, head_back, head_focus, head_hurt, leg)
- `Poses/` — 24개 포즈 이미지 (`{character}_{action}.png` 형식)

**24개 공통 포즈**: action1, action2, back, cheer1, cheer2, climb1, climb2, duck, fall, hang, hold1, hold2, hurt, idle, jump, kick, skid, slide, stand, swim1, swim2, talk, walk1, walk2

**총 에셋**: 5 캐릭터 × 34 파일 = 170개 PNG

### 2. 로비 캐릭터 표시 (CharacterDisplay)

**파일**: `apps/web/src/components/lobby/CharacterDisplay.tsx`

```typescript
const CHARACTER_PATH = "/assets/characters/PNG/Player/Poses/player_idle.png"
```

- Props 없이 `Player`의 `idle` 포즈만 하드코딩 표시
- 150×150px, `animate-float` 애니메이션 + 바닥 그림자
- `pointer-events-none`으로 상호작용 불가
- 동적 캐릭터 전환 미지원

### 3. Phaser 게임 코어 캐릭터 시스템

#### BootScene — 에셋 로딩 (`packages/game-core/src/scenes/BootScene.ts`)

```typescript
// 라인 55-59: 현재 등록된 캐릭터 (5종 중 3종만)
const CHARACTERS = [
  { key: "player", path: "Player", prefix: "player" },
  { key: "adventurer", path: "Adventurer", prefix: "adventurer" },
  { key: "female", path: "Female", prefix: "female" },
]

// 라인 39-52: 로드하는 포즈 (24개 중 12개만)
// idle, stand, walk1, walk2, jump, fall, duck, hurt, action1, action2, cheer1, cheer2
```

- 에셋 로딩 경로 패턴: `/assets/characters/PNG/${char.path}/Poses/${char.prefix}_${pose}.png`
- 캐릭터별 8가지 애니메이션 생성: idle, walk, jump, fall, shoot, hurt, duck, cheer

#### GameScene — 캐릭터 생성 (`packages/game-core/src/scenes/GameScene.ts`)

```typescript
// 라인 258-260: Player 1 하드코딩
characterType: "player"

// 라인 269-271: Player 2 하드코딩
characterType: "female"
```

#### Character 엔티티 (`packages/game-core/src/entities/Character.ts`)

```typescript
// 라인 5-13: CharacterConfig 인터페이스
export interface CharacterConfig {
  scene: Phaser.Scene
  x: number
  y: number
  playerId: number
  characterType: string  // "player" or "adventurer" 등
  terrainData: TerrainData
}
```

- `characterType`은 이미 string 타입으로 설계되어 있어 새 캐릭터 타입을 유연하게 받을 수 있음
- 애니메이션 키 생성: `${this.characterType}_${state}` (라인 281)

### 4. 데이터베이스 현재 상태

**Supabase 프로젝트**: `fmwjmsfgxhbzhvtnaqta` (Whoosh Bang, ap-northeast-1)

#### player_profiles 테이블

| 컬럼 | 타입 | 설명 |
|------|------|------|
| id | UUID (PK) | auth.users 참조 |
| username | TEXT (UNIQUE) | 고유 사용자명 |
| display_name | TEXT | 표시 이름 |
| avatar_url | TEXT | Google 아바타 URL |
| level | INTEGER (기본 1) | 레벨 |
| total_experience | BIGINT (기본 0) | 총 경험치 |
| total_matches | INTEGER (기본 0) | 총 게임 수 |
| wins | INTEGER (기본 0) | 승리 |
| losses | INTEGER (기본 0) | 패배 |
| created_at | TIMESTAMPTZ | 생성일 |
| updated_at | TIMESTAMPTZ | 수정일 |

**캐릭터/스킨 관련 컬럼**: 없음
**캐릭터 소유권 테이블**: 없음
**NFT/아이템 테이블**: 없음

#### 기존 마이그레이션

| 마이그레이션 | 내용 |
|-------------|------|
| `create_core_tables` | player_profiles, wallets, wallet_nonces 생성 |
| `create_user_trigger` | 자동 프로필 생성 트리거 |
| `setup_rls_policies` | RLS 정책 |

### 5. 로비 컴포넌트 구조

**`LobbyContent`** (루트) → 4개 서브 컴포넌트 조합:
- **`ProfileCard`** — 아바타, 이름, Lv, XP 바, 전적 (profile 데이터 사용)
- **`CharacterDisplay`** — 정적 캐릭터 표시 (props 없음)
- **`WalletLinkButton`** — Sui 지갑 연동/해제
- **`PlayButton`** — `/game`으로 이동

데이터 흐름:
1. `apps/web/src/app/lobby/page.tsx`에서 서버 사이드로 `player_profiles` + `wallets` 조회
2. `LobbyContent`에 props로 전달
3. `CharacterDisplay`는 아무 데이터도 받지 않음 (독립적)

### 6. 에셋 참조 방식

- `apps/web/public/assets` → `../../../assets` 심볼릭 링크
- Next.js 앱에서 `/assets/` 경로로 정적 파일 접근
- 경로 패턴: `/assets/characters/PNG/{CharacterName}/Poses/{character_name}_{pose}.png`
  - 폴더명: PascalCase (`Player`, `Adventurer`)
  - 파일 접두사: lowercase (`player`, `adventurer`)

## 코드 참조

- `apps/web/src/components/lobby/CharacterDisplay.tsx:5` — 하드코딩된 캐릭터 경로
- `packages/game-core/src/scenes/BootScene.ts:55-59` — CHARACTERS 상수 (3종 등록)
- `packages/game-core/src/scenes/BootScene.ts:39-52` — CHARACTER_POSES 상수 (12개 포즈)
- `packages/game-core/src/scenes/BootScene.ts:109-115` — 캐릭터 이미지 로딩 루프
- `packages/game-core/src/scenes/BootScene.ts:149-218` — 캐릭터 애니메이션 생성
- `packages/game-core/src/scenes/GameScene.ts:254-272` — Player 1/2 캐릭터 타입 하드코딩
- `packages/game-core/src/entities/Character.ts:5-13` — CharacterConfig 인터페이스
- `packages/game-core/src/entities/Character.ts:281` — 애니메이션 키 생성 `${characterType}_${state}`
- `apps/web/src/app/lobby/page.tsx:17-28` — 서버 사이드 프로필/지갑 조회
- `apps/web/src/components/lobby/LobbyContent.tsx:14-27` — LobbyContentProps 인터페이스

## 아키텍처 문서화

### 현재 캐릭터 데이터 흐름

```
[에셋 폴더] → [BootScene 로딩] → [GameScene 하드코딩 할당] → [Character 엔티티 렌더링]

[에셋 폴더] → [CharacterDisplay 하드코딩 경로] → [로비 이미지 표시]
```

- 두 흐름 모두 DB를 통하지 않고 코드에서 직접 참조
- 유저 선택이나 소유권 개념 없음

### 캐릭터 식별 체계

| 위치 | 식별자 | 예시 |
|------|--------|------|
| 에셋 폴더명 | PascalCase | `Player`, `Adventurer`, `Female`, `Soldier`, `Zombie` |
| BootScene key | lowercase | `player`, `adventurer`, `female` |
| BootScene prefix | lowercase | `player`, `adventurer`, `female` |
| CharacterConfig.characterType | string | `"player"`, `"female"` |
| 포즈 파일명 | `{prefix}_{pose}` | `player_idle`, `adventurer_jump` |

### Wallet/NFT 인프라 현황

- Sui 지갑 연동 완료 (`wallets` 테이블 + nonce 기반 서명 검증)
- `app_metadata`에 `wallet_address` 저장됨
- NFT 조회/민팅 로직은 미구현
- 계획 문서에서 "NFT/토큰 트랜잭션 로직 (추후 구현)"으로 명시

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` — 전체 아키텍처 연구, Asset Pipeline 및 캐릭터 에셋 분석 포함
- `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md` — MVP 구현 계획, Phase 3에서 캐릭터 시스템 구현 완료 기록, "NFT 스킨/아이템 (미래)" 언급
- `thoughts/shared/plans/2026-02-13-social-login-wallet-linking-implementation.md` — 소셜 로그인 + 지갑 연동 구현 계획, 로비 UI 리디자인 기록, "NFT/토큰 트랜잭션 로직 (추후 구현)" 명시
- `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md` — Supabase + Sui 지갑 연동 아키텍처 연구

## 관련 연구

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md`
- `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md`

## 결정 사항 (2026-02-13)

1. **캐릭터 소유권 모델**: **DB만으로 관리** (온체인 NFT 조회 불필요)
2. **기본 캐릭터**: 모든 신규 유저에게 **Player만** 기본 제공
3. **Soldier/Zombie**: 추후 확장 예정 (현재 미활성화 유지)
4. **NFT 컨트랙트**: 불필요 — NFT화는 외부 플랫폼 **Fortem**에서 컨트롤
5. **에셋 확장**: 향후 새 캐릭터가 동일한 에셋 구조를 따를 보장 없음 — 에셋 구조를 하드코딩하지 않는 유연한 설계 필요

### 추가 변경 사항

- GameScene에서 Player 2의 `characterType`을 `"female"` → `"player"`로 변경 (모든 캐릭터를 기본 캐릭터로 통일)
