---
date: "2026-02-15T03:02:00+09:00"
researcher: arta1069@gmail.com
git_commit: N/A (git not initialized)
branch: N/A
repository: whoosh-bang
topic: "매치 통계 집계, 플레이타임 누적, 활성 유저 카운팅 시스템"
tags: [research, match-stats, playtime-tracking, active-user, player-profiles, game-lifecycle]
status: complete
last_updated: 2026-02-15
last_updated_by: arta1069@gmail.com
last_updated_note: "미해결 질문 7개에 대한 설계 논의 결과 추가"
---

# 연구: 매치 통계 집계, 플레이타임 누적, 활성 유저 카운팅 시스템

**날짜**: 2026-02-15T03:02:00+09:00
**연구자**: arta1069@gmail.com
**Git 커밋**: N/A (git 미초기화)
**브랜치**: N/A
**리포지토리**: whoosh-bang

## 연구 질문

1. 유저가 게임 플레이 후 ProfileCard의 Matches/Wins/Losses가 집계되는 구조가 있는가?
2. 플레이 시작~종료까지 소요시간 계산 및 누적 플레이타임 저장 구조가 있는가?
3. 누적 플레이타임 1시간 초과 시 활성 유저 +1 집계 시스템이 있는가?
4. Web3 지갑 로그인 악용 방지를 위한 활성 유저 기준이 반영되어 있는가?

## 요약

**현재 상태**: 매치 통계 집계, 플레이타임 추적, 활성 유저 카운팅 시스템 모두 **구현되지 않음**.

- `player_profiles` 테이블에 `wins`, `losses`, `total_matches`, `total_experience`, `level` 컬럼이 존재하지만 **게임 종료 시 업데이트하는 로직이 없음**
- 게임 종료 시 `GAME_OVER` 이벤트가 `winnerId`만 전달하며, DB에 기록하지 않고 3초 후 로비로 리다이렉트
- 플레이타임 추적 관련 DB 컬럼, API, 로직 전혀 없음
- 활성 유저 카운팅 시스템 전혀 없음
- README PRD에 기획은 문서화되어 있으나, 코드 구현은 0%

---

## 상세 발견 사항

### 1. 현재 게임 매치 라이프사이클

#### 게임 시작 흐름

```
로비 (PlayButton 클릭)
  → /game 페이지 (Server Component)
  → player_profiles에서 selected_character 조회
  → character_inventory에서 소유 검증
  → GameClient → GameWrapper → PhaserGame
  → createGame() → BootScene → GameScene
  → TurnSystem.startGame()
```

- `apps/web/src/components/lobby/PlayButton.tsx` — `router.push("/game")` 으로 단순 라우팅
- `apps/web/src/app/game/page.tsx` — Server Component에서 캐릭터 소유 검증 후 GameClient에 전달
- `packages/game-core/src/scenes/GameScene.ts:72-104` — 지형, 플레이어, 시스템, UI 초기화
- `packages/game-core/src/systems/TurnSystem.ts:47-52` — `startGame()`에서 첫 턴 시작

#### 게임 플레이 루프

턴 기반 상태 머신: `idle` → `playerTurn` → `aiming` → `firing` → `animating` → (반복 또는 `gameOver`)

- `TurnSystem.ts:4-10` — GameState 타입 정의
- 턴 타이머: 30초 (`TurnSystem.ts:98`)
- 플레이어 1: `InputController` (인간), 플레이어 2: `AIController` (AI)
- 풍향 시스템: 매 턴 -5~5 랜덤 (`WindSystem.ts:6-7`)

#### 게임 종료 흐름 (핵심)

```
DamageSystem.checkGameOver() — 생존 캐릭터 수 확인
  → 0~1명 생존 시 EventBus.emit(GAME_OVER, { winnerId })
  → GameScene.onGameOver() — 인게임 승리 오버레이 표시
  → GameWrapper.handleGameOver() — result 상태 설정, 화면 전환
  → 3초 후 router.push('/lobby') — 로비로 리다이렉트
  → ❌ DB 기록 없음, ❌ 통계 업데이트 없음
```

- `packages/game-core/src/systems/DamageSystem.ts:149-158` — `checkGameOver()`
- `packages/game-core/src/EventBus.ts:10` — `GAME_OVER` 이벤트 정의
- `apps/web/src/components/game/GameWrapper.tsx:32-56` — 이벤트 수신 및 결과 화면
- `apps/web/src/components/game/GameWrapper.tsx:44-52` — 3초 자동 리다이렉트

**GAME_OVER 이벤트 페이로드**:
```typescript
{ winnerId: number }  // 0 or 1 (플레이어 인덱스), -1 (무승부)
```

**부족한 정보**: 매치 시작 시간, 종료 시간, 플레이어 ID(auth user ID), 매치 통계 등 전혀 포함되지 않음.

---

### 2. 데이터베이스 스키마 — player_profiles

#### 현재 구현된 테이블

```sql
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
  selected_character TEXT REFERENCES characters(id) DEFAULT 'player',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);
```

**통계 관련 컬럼** (모두 기본값에서 변경되지 않음):
| 컬럼 | 타입 | 기본값 | 현재 업데이트 여부 |
|------|------|--------|-------------------|
| `level` | INTEGER | 1 | ❌ 없음 |
| `total_experience` | BIGINT | 0 | ❌ 없음 |
| `total_matches` | INTEGER | 0 | ❌ 없음 |
| `wins` | INTEGER | 0 | ❌ 없음 |
| `losses` | INTEGER | 0 | ❌ 없음 |

**존재하지 않는 컬럼** (플레이타임/활성유저용):
- `total_playtime_seconds` — 없음
- `is_active_user` — 없음
- `active_user_qualified_at` — 없음

#### ProfileCard가 데이터를 읽는 방식

`apps/web/src/app/lobby/page.tsx:17-21`:
```typescript
const { data: profile } = await supabase
  .from("player_profiles")
  .select("*")
  .eq("id", user.id)
  .single()
```

`apps/web/src/components/lobby/ProfileCard.tsx:21-30`:
```typescript
interface ProfileCardProps {
  displayName: string | null
  username: string
  level: number
  totalExperience: number
  wins: number
  losses: number
  totalMatches: number
  avatarUrl?: string
}
```

현재 모든 유저의 stats가 기본값(0)으로 표시됨 — 게임 결과가 DB에 기록되지 않기 때문.

---

### 3. 레벨/경험치 시스템

`apps/web/src/lib/level.ts:1-22`:

```typescript
// 레벨 N 완료에 필요한 XP: N * 100
export function xpRequiredForLevel(level: number): number {
  return level * 100;
}

export function getLevelProgress(totalExperience: number, level: number) {
  let xpForCompletedLevels = 0;
  for (let i = 1; i < level; i++) {
    xpForCompletedLevels += xpRequiredForLevel(i);
  }
  const currentXp = totalExperience - xpForCompletedLevels;
  const requiredXp = xpRequiredForLevel(level);
  const percentage = Math.min(Math.max((currentXp / requiredXp) * 100, 0), 100);
  return { currentXp, requiredXp, percentage };
}
```

- 레벨 1: 100 XP 필요
- 레벨 2: 200 XP 필요 (누적 300)
- 레벨 3: 300 XP 필요 (누적 600)
- 선형 증가 (`level * 100`)

**XP 부여 로직**: 존재하지 않음. 매치 종료 시 경험치를 부여하는 코드 없음.

---

### 4. 이벤트 시스템 (EventBus)

`packages/game-core/src/EventBus.ts:4-13`:
```typescript
export const EventBus = new Phaser.Events.EventEmitter();

export const GameEvents = {
  TURN_CHANGED: "turn-changed",
  HEALTH_CHANGED: "health-changed",
  GAME_OVER: "game-over",
  WEAPON_SELECTED: "weapon-selected",
  WIND_CHANGED: "wind-changed",
} as const;
```

**GAME_OVER 이벤트 발신자**:
- `DamageSystem.ts:154` — 폭발/낙수로 인한 사망 시
- `TurnSystem.ts:86` — `endGame()` 수동 호출 (현재 사용되지 않음)

**GAME_OVER 이벤트 수신자**:
- `GameScene.ts:448` — 인게임 승리 오버레이 UI
- `GameWrapper.tsx:38` — React 결과 화면 및 로비 리다이렉트

---

### 5. 인증 및 지갑 시스템

#### Google OAuth 인증 (구현됨)
- `apps/web/src/components/auth/GoogleLoginButton.tsx` — Google 로그인
- `apps/web/src/app/auth/callback/route.ts` — OAuth 콜백 처리
- DB 트리거가 `auth.users` INSERT 시 자동으로 `player_profiles` 생성

#### SUI 지갑 연동 (구현됨)
- `apps/web/src/components/lobby/WalletLinkButton.tsx` — 지갑 연결 UI
- `apps/web/src/app/api/wallet/nonce/route.ts` — Nonce 발급
- `apps/web/src/app/api/wallet/verify/route.ts` — 서명 검증 및 지갑 저장
- 정책: 1 유저 = 1 지갑 (UNIQUE 제약)

**지갑 관련 테이블**:
```sql
CREATE TABLE wallets (
  user_id UUID NOT NULL UNIQUE REFERENCES auth.users(id),
  wallet_address TEXT NOT NULL UNIQUE,
  chain TEXT NOT NULL DEFAULT 'sui',
  verified BOOLEAN DEFAULT FALSE,
  ...
);
```

---

### 6. 플레이타임 추적 — 존재하지 않음

코드베이스 전체에서 다음 키워드 검색 결과 **0건**:
- `playtime`, `play_time`, `playTime`
- `duration`, `session_time`, `game_time`
- `active_user`, `activeUser`

README PRD에 기획은 문서화되어 있음 (`README.md:32-36`):
> - 게임 플레이를 시작하고 종료까지 총 소요시간을 계산하고 게임이 종료 될 때 유저가 플레이한 시간을 누적한다
> - 누적된 플레이타임이 1시간을 넘으면 게임의 유저가 +1 된다.

**구현 현황**: 0%

---

### 7. 활성 유저 카운팅 — 존재하지 않음

README PRD 기획 (`README.md:35-36`):
> - 이렇게 누적된 게임의 유저는 아이템 유통등 여러 기획에 반영되는 유저수로 계산된다. (이유는 지갑 로그인으로 새로 누적되는 유저 집계를 폭발적으로 증가시키는 악용을 막기위해서)

**목적**: Web3 지갑 로그인의 시빌 공격(Sybil attack) 방지
- 지갑은 무한히 생성 가능 → 단순 가입자 수는 의미 없음
- 실제 플레이 시간 기반으로 "진짜 유저"를 구분
- 이 유저 수가 아이템 유통량 계산의 기준 (예: 총 유저의 3%만 최고 등급 무기 유통)

**구현 현황**: 0%

---

## 코드 참조

### 게임 라이프사이클
- `apps/web/src/components/lobby/PlayButton.tsx` — PLAY 버튼 (router.push("/game"))
- `apps/web/src/app/game/page.tsx` — 게임 페이지 서버 컴포넌트
- `apps/web/src/components/game/GameWrapper.tsx:32-56` — GAME_OVER 이벤트 수신, 결과 화면, 로비 리다이렉트
- `packages/game-core/src/scenes/GameScene.ts:72-104` — 게임 초기화
- `packages/game-core/src/systems/TurnSystem.ts:47-52` — startGame()
- `packages/game-core/src/systems/DamageSystem.ts:149-158` — checkGameOver()
- `packages/game-core/src/EventBus.ts:10` — GAME_OVER 이벤트

### 데이터베이스
- `apps/web/src/app/lobby/page.tsx:17-21` — player_profiles 조회
- `apps/web/src/components/lobby/ProfileCard.tsx:21-30` — ProfileCard props (stats 표시)
- `apps/web/src/lib/level.ts:1-22` — 레벨/XP 계산

### 인증/지갑
- `apps/web/src/components/auth/GoogleLoginButton.tsx` — Google 로그인
- `apps/web/src/components/lobby/WalletLinkButton.tsx` — 지갑 연결
- `apps/web/src/app/api/wallet/verify/route.ts` — 지갑 검증

---

## 아키텍처 문서화

### 현재 데이터 흐름 (매치 결과)

```
게임 종료
  → DamageSystem.checkGameOver()
  → EventBus.emit(GAME_OVER, { winnerId })
  → GameWrapper: result 상태 설정
  → 3초 후 /lobby 리다이렉트
  → ❌ 데이터 유실 (DB 기록 없음)
```

### 누락된 데이터 흐름

```
게임 종료 시 필요한 정보:
  - 승자/패자 auth user ID
  - 매치 시작 시간, 종료 시간 (소요 시간 계산)
  - 매치 결과 (승/패/무승부)
  → player_profiles.total_matches++
  → player_profiles.wins++ (또는 losses++)
  → player_profiles.total_experience += XP
  → player_profiles.total_playtime_seconds += duration (❌ 컬럼 미존재)
  → 활성 유저 자격 확인 (total_playtime >= 3600) (❌ 시스템 미존재)
```

### 현재 2-플레이어 구조

- 항상 1v1 vs AI (멀티플레이어 없음)
- 플레이어 1: 인간 (auth.users), 플레이어 2: 컴퓨터 (AI)
- 하드코딩된 players 배열: `GameWrapper.tsx:24-27`
  ```typescript
  const players = [
    { name: playerName, isAI: false },
    { name: "Computer", isAI: true },
  ]
  ```

### 현재 게임 결과 타입

```typescript
// GameWrapper.tsx:9-11
interface GameResult {
  winnerId: number  // -1 = 무승부, 0 = 플레이어 1, 1 = 플레이어 2(AI)
}
```

---

## 히스토리 컨텍스트 (thoughts/에서)

### 아키텍처 연구 (2026-01-22)
- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md`
  - `match_history`, `player_match_results`, `game_lobbies` 테이블 설계됨 (미구현)
  - Supabase Realtime Presence 기반 온라인 유저 추적 설계됨 (미구현)

### 소셜 로그인/지갑 연동 (2026-02-12~13)
- `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md`
  - "Social Login First + Optional Wallet Linking" 아키텍처 결정
- `thoughts/shared/plans/2026-02-13-social-login-wallet-linking-implementation.md`
  - `player_profiles`, `wallets`, `wallet_nonces` 테이블 구현 완료

### 캐릭터 스킨 시스템 (2026-02-13)
- `thoughts/shared/plans/2026-02-13-character-skin-selection-system.md`
  - `characters`, `character_inventory` 테이블 구현 완료
  - `handle_new_user()` 트리거로 프로필 자동 생성

### MVP 구현 (2026-01-23)
- `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md`
  - Phase 1-8 완료 (로컬 2인 게임)
  - 온라인 멀티플레이어, 매치 히스토리, 통계 업데이트 → Phase 2로 연기

---

## 관련 연구

- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` — 전체 게임 아키텍처 설계
- `thoughts/shared/research/2026-02-12-social-login-wallet-linking-architecture-research.md` — 인증 아키텍처
- `thoughts/shared/research/2026-02-13-character-skin-nft-system-research.md` — NFT 캐릭터 시스템 (아이템 유통 관련)

---

## 미해결 질문 → 해결됨 (2026-02-15 설계 논의)

모든 미해결 질문이 설계 논의를 통해 해결되었습니다. 상세 결정 사항은 아래 "후속 연구" 섹션 참조.

---

## 후속 연구 2026-02-15T04:30:00+09:00

### 설계 논의 요약

미해결 질문 7개에 대한 심도 있는 설계 논의가 진행되어, 매치 통계 집계·플레이타임 누적·활성 유저 카운팅 시스템의 전체 설계가 확정되었습니다.

---

### 결정 1: 매치 결과 기록 방식

**기록 구간**: `TurnSystem.startGame()` (실제 플레이 시작) ~ `GAME_OVER` 이벤트 발생

**기록 방식**: `match_history` 테이블 사용 (서버 타임스탬프 기반)
- 게임 시작 시: `game/page.tsx` Server Component에서 `match_history` INSERT (`status='playing'`, `started_at=NOW()`)
- 게임 종료 시: API 호출로 `match_history` UPDATE (`ended_at=NOW()`, duration 서버 계산)

**종료 후 흐름**: `GAME_OVER` → 결과 화면 표시 → API 호출 (매치 기록 저장) → 로비 이동
- API 성공: 정상 로비 이동
- API 실패: 로비 이동은 하되, **Sonner toast**로 실패 메시지 안내 (게임 흐름을 막지 않음)

---

### 결정 2: XP 부여 공식

**레벨 공식 변경**: 기존 `level * 100` (선형) → `100 * 2^(level-1)` (지수)

| 레벨 | 필요 XP | 누적 XP |
|------|---------|---------|
| 1 | 100 | 100 |
| 2 | 200 | 300 |
| 3 | 400 | 700 |
| 4 | 800 | 1,500 |
| 5 | 1,600 | 3,100 |
| 10 | 51,200 | 102,300 |

**결과별 기본 XP**:
| 결과 | XP |
|------|-----|
| 승리 | 50 |
| 무승부 | 30 |
| 패배 | 20 |

**플레이 시간 보너스 XP** (구간별):
| 플레이 시간 | 보너스 XP |
|------------|-----------|
| 5분 미만 | 0 |
| 5분 ~ 10분 | +5 |
| 10분 ~ 20분 | +15 |
| 20분 이상 | +25 |

**매치당 XP 범위**: 최소 20 XP (패배 + 5분 미만) ~ 최대 75 XP (승리 + 20분 이상)

---

### 결정 3: 레벨업 로직 위치

**Supabase DB 함수** (서버 사이드)에서 처리
- 보안상 클라이언트 계산 배제
- 매치 완료 시 DB 함수가 XP 부여 + 레벨업 조건 체크 + 레벨 업데이트를 원자적으로 처리

---

### 결정 4: 플레이타임 저장 방식

**`match_history` 테이블 + `player_profiles` 컬럼 추가** 방식 채택

초기 논의에서는 `player_profiles`에 단순 컬럼 추가만으로 해결하려 했으나, 서버 타임스탬프 기반 조작 방지·동시 세션 관리·비정상 종료 처리 등의 요구사항이 복합적으로 연결되어 `match_history` 테이블이 필수로 판단됨.

**match_history 테이블 스키마**:
```sql
CREATE TABLE match_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id),
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ended_at TIMESTAMPTZ,
  duration_seconds INTEGER,
  total_turns INTEGER,
  player_actions INTEGER,        -- 무기 발사 횟수
  result TEXT,                    -- 'win' | 'loss' | 'draw' | 'abandoned'
  status TEXT NOT NULL DEFAULT 'playing',  -- 'playing' | 'completed' | 'abandoned'
  xp_earned INTEGER DEFAULT 0
);
```

**player_profiles 추가 컬럼**:
```sql
ALTER TABLE player_profiles ADD COLUMN total_playtime_seconds BIGINT DEFAULT 0;
ALTER TABLE player_profiles ADD COLUMN is_active_user BOOLEAN DEFAULT false;
```

**데이터 정리 정책** (`pg_cron` 매 시간 실행):
- `status IN ('completed', 'abandoned')` AND `started_at < NOW() - INTERVAL '24 hours'` → **삭제** (처리 완료, stats 이미 반영됨)
- `status = 'playing'` AND `started_at < NOW() - INTERVAL '24 hours'` → **보존** (비정상 — 악성 유저 추적/감사용)

---

### 결정 5: 활성 유저 판정 시점

매치 기록 저장하는 **DB 함수 내에서 즉시 반영**:
- `total_playtime_seconds >= 3600` 체크 → `is_active_user = true` 설정
- 별도 배치 처리 불필요

---

### 결정 6: 활성 유저 수 조회

단순 COUNT 쿼리:
```sql
SELECT COUNT(*) FROM player_profiles WHERE is_active_user = true;
```
- 캐싱/뷰 불필요 — 현재는 단순 활성유저 측정을 위한 검증/확인 수단

---

### 결정 7: 조작 방지

`match_history` 테이블의 서버 타임스탬프 기반으로 대부분의 조작 방지가 자연스럽게 해결됨.

**서버 타임스탬프 기반 검증**:
- `started_at`, `ended_at` 모두 서버 시간 (`NOW()`) → 클라이언트 시간 조작 불가
- `duration_seconds = ended_at - started_at` 서버에서 계산

**플레이어 행동 검증**:
- `player_actions` = 플레이어가 무기를 발사한 턴 수
- 서버 검증: `player_actions >= 3` (최소 3회 발사) → XP/플레이타임 지급 자격
- 이 조건으로 최소 턴 수 제한도 함께 해소 (별도 최소 턴 수 불필요)

**동시 세션 방지**:
- `status = 'playing'` 인 매치가 이미 존재하면, 해당 매치를 **abandoned 처리** 후 새 게임 시작
- 유저 잠금 없음, 대기 없음 — 새 게임은 항상 시작 가능

**API 중복 호출 방지**:
- `match_history.id` (서버 생성 UUID) + `status` 상태 추적
- 이미 `completed`/`abandoned` 된 매치에 대한 재요청 거부

**비정상 종료 처리**:
- 브라우저 크래시, 탭 닫기, 네트워크 끊김 등으로 매치 완료 API 미호출 시
- `match_history`에 `status = 'playing'` 레코드가 잔류
- **다음 접속 시** (로비 페이지 또는 게임 페이지 진입) 자동 감지 및 처리:
  - `result = 'abandoned'`, `status = 'abandoned'`
  - `matches +1`, `losses +1`, `XP 0`, `playtime 0`
- 24시간 이상 `playing` 상태로 남아있는 레코드는 **악성 유저로 간주, 삭제하지 않고 보존**

---

### 전체 흐름 다이어그램

```
[게임 시작] — game/page.tsx Server Component
  1. status='playing' 인 매치 존재? → abandoned 처리 (패배, matches+1, XP 0, playtime 0)
  2. match_history INSERT (status='playing', started_at=NOW())
  3. match_id를 클라이언트에 전달
  4. GameWrapper → PhaserGame → TurnSystem.startGame()

[게임 플레이 중]
  - 클라이언트에서 playerActions(무기 발사 횟수) 카운트
  - 턴 기반 상태 머신 진행

[정상 종료] — POST /api/match/complete
  1. match_id로 매치 조회, status='playing' 확인
  2. ended_at = NOW(), duration_seconds = ended_at - started_at
  3. player_actions >= 3 검증 (미달 시 XP/playtime 미지급, matches/result는 기록)
  4. XP 계산: 결과 XP + 시간 보너스
  5. DB 함수에서 원자적 처리:
     - match_history UPDATE (status='completed', result, xp_earned 등)
     - player_profiles UPDATE (total_matches++, wins/losses++, total_experience+=, total_playtime_seconds+=)
     - 레벨업 체크 및 반영
     - is_active_user 체크 (total_playtime_seconds >= 3600)
  6. 결과 반환 → 성공 시 로비 이동, 실패 시 Sonner toast 후 로비 이동

[비정상 종료] — 다음 접속 시 자동 감지
  1. 로비/게임 페이지 진입 시 status='playing' 매치 발견
  2. result='abandoned', status='abandoned'
  3. matches +1, losses +1, XP 0, playtime 0
  4. 정상 흐름으로 복귀

[데이터 정리] — pg_cron 매 시간
  - completed/abandoned + 24시간 경과 → 삭제
  - playing + 24시간 경과 → 보존 (악성 유저 추적용)
```
