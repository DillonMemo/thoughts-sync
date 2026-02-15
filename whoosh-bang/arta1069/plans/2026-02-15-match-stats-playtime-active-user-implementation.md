# 매치 통계 집계, 플레이타임 누적, 활성 유저 카운팅 구현 계획

## 개요

게임 종료 시 매치 결과를 DB에 기록하고, 플레이타임을 누적하며, 누적 플레이타임 1시간 초과 시 활성 유저로 판정하는 시스템을 구현한다. 현재 `GAME_OVER` 이벤트 발생 후 DB 기록 없이 로비로 리다이렉트되는 구조를 개선한다.

## 현재 상태 분석

### 존재하는 것
- `player_profiles` 테이블에 `wins`, `losses`, `total_matches`, `total_experience`, `level` 컬럼 존재 (모두 기본값 0/1)
- `GAME_OVER` 이벤트 시스템 (`EventBus.ts` → `GameWrapper.tsx`)
- 레벨/XP 계산 유틸리티 (`level.ts` — 선형 공식 `level * 100`)
- Sonner toast 전역 설정 (`layout.tsx:20`)
- API 라우트 인증 패턴 (`wallet/` 라우트 참조)

### 존재하지 않는 것
- `match_history` 테이블
- `player_profiles.total_playtime_seconds`, `is_active_user` 컬럼
- 매치 결과 기록 로직
- 매치 완료/포기 API 라우트
- 플레이어 액션(무기 발사) 카운팅
- pg_cron 정리 작업

### 핵심 발견
- `DamageSystem.ts:149-158` — `checkGameOver()`가 `{ winnerId: number }` 페이로드로 `GAME_OVER` 이벤트 발생
- `GameWrapper.tsx:33-36` — `handleGameOver` 콜백에서 `setResult(data)` + `setScreen("result")` 후 3초 뒤 로비 이동
- `GameWrapper.tsx:54-56` — Exit 버튼이 `router.push("/lobby")`만 수행 (DB 기록 없음)
- `game/page.tsx:6-40` — Server Component에서 인증 + 캐릭터 검증 후 `GameClient` 렌더링
- `GameClient.tsx:6-18` — `GameWrapper`를 `dynamic import (ssr: false)`로 로드, props 중계
- `TurnSystem.ts:39` — `roundNumber` 프로퍼티 존재 (턴 수 추적 가능)
- `supabase/migrations/` 디렉토리 미존재 — Supabase MCP 도구로 직접 마이그레이션 적용
- Supabase 프로젝트 ID: `fmwjmsfgxhbzhvtnaqta`

## 원하는 최종 상태

1. 게임 시작 시 `match_history`에 `status='playing'` 레코드 생성
2. 정상 종료 시 API 호출로 매치 결과, XP, 플레이타임 원자적 업데이트
3. Exit 버튼 클릭 시 `abandoned` 처리 (losses+1, matches+1, XP 0, playtime 0)
4. 비정상 종료(탭 닫기 등) 시 다음 게임 시작 때 자동 abandoned 처리
5. `player_profiles`의 wins/losses/total_matches/total_experience/level이 실시간 반영
6. 누적 플레이타임 ≥ 3600초 시 `is_active_user = true` 자동 설정
7. ProfileCard에서 실제 통계가 표시됨
8. pg_cron으로 24시간 경과 completed/abandoned 레코드 자동 삭제

### 검증 방법
- 게임 완료 후 로비의 ProfileCard에서 Matches/Wins/Losses/Level 변화 확인
- `SELECT * FROM match_history WHERE user_id = '{id}'` 쿼리로 매치 기록 확인
- `SELECT total_playtime_seconds, is_active_user FROM player_profiles WHERE id = '{id}'` 확인

## 하지 않을 것

- 온라인 멀티플레이어 지원
- 리더보드 UI
- 매치 히스토리 목록 UI (DB 기록만, UI는 별도 작업)
- 매치 리플레이 시스템
- 레벨업 애니메이션/알림 UI
- 플레이타임 UI 표시 (ProfileCard에 추가하지 않음)

## 구현 접근 방식

Server Component에서 매치를 시작하고 서버 타임스탬프 기반으로 duration을 계산하여 클라이언트 시간 조작을 방지한다. 매치 완료/포기는 API Route를 통해 Supabase DB 함수를 호출하여 원자적으로 처리한다.

---

## 1단계: DB 마이그레이션

### 개요
`match_history` 테이블 생성, `player_profiles` 컬럼 추가, DB 함수 생성, RLS 정책, pg_cron 설정

### 필요한 변경:

#### 1-1. pg_cron 확장 활성화
Supabase MCP `apply_migration` 사용:
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron WITH SCHEMA pg_catalog;
GRANT USAGE ON SCHEMA cron TO postgres;
```

#### 1-2. match_history 테이블 생성
```sql
CREATE TABLE match_history (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  ended_at TIMESTAMPTZ,
  duration_seconds INTEGER,
  total_turns INTEGER,
  player_actions INTEGER DEFAULT 0,
  result TEXT,
  status TEXT NOT NULL DEFAULT 'playing',
  xp_earned INTEGER DEFAULT 0
);

CREATE INDEX idx_match_history_user_status ON match_history(user_id, status);
CREATE INDEX idx_match_history_started_at ON match_history(started_at);
```

#### 1-3. player_profiles 컬럼 추가
```sql
ALTER TABLE player_profiles ADD COLUMN total_playtime_seconds BIGINT DEFAULT 0;
ALTER TABLE player_profiles ADD COLUMN is_active_user BOOLEAN DEFAULT false;
```

#### 1-4. RLS 정책 설정
```sql
ALTER TABLE match_history ENABLE ROW LEVEL SECURITY;

-- 자신의 매치만 조회 가능
CREATE POLICY "match_history_select_own"
ON match_history FOR SELECT TO authenticated
USING (user_id = auth.uid());

-- 일반 유저의 직접 INSERT/UPDATE/DELETE 차단 (service_role 또는 DB 함수에서만)
CREATE POLICY "match_history_no_insert"
ON match_history FOR INSERT TO authenticated
WITH CHECK (false);

CREATE POLICY "match_history_no_update"
ON match_history FOR UPDATE TO authenticated
USING (false);

CREATE POLICY "match_history_no_delete"
ON match_history FOR DELETE TO authenticated
USING (false);
```

> **참고**: match_history의 INSERT/UPDATE는 API Route에서 `service_role` 클라이언트(Admin)를 통해 수행하므로 RLS를 우회함. 일반 유저의 직접 조작을 차단.

#### 1-5. complete_match DB 함수
```sql
CREATE OR REPLACE FUNCTION complete_match(
  p_match_id UUID,
  p_user_id UUID,
  p_winner_id INTEGER,
  p_player_actions INTEGER,
  p_total_turns INTEGER
)
RETURNS JSON
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
DECLARE
  v_started_at TIMESTAMPTZ;
  v_duration_seconds INTEGER;
  v_result TEXT;
  v_xp_earned INTEGER := 0;
  v_playtime_credit INTEGER := 0;
  v_current_xp BIGINT;
  v_current_level INTEGER;
  v_new_level INTEGER;
  v_level_xp_sum BIGINT;
BEGIN
  -- 1. 매치 조회 및 검증
  SELECT started_at INTO v_started_at
  FROM public.match_history
  WHERE id = p_match_id AND user_id = p_user_id AND status = 'playing';

  IF NOT FOUND THEN
    RETURN json_build_object('error', 'Match not found or already completed');
  END IF;

  -- 2. 소요 시간 계산 (서버 타임스탬프)
  v_duration_seconds := EXTRACT(EPOCH FROM (NOW() - v_started_at))::INTEGER;

  -- 3. 결과 판정 (winnerId: 0 = 플레이어 승리, 1 = AI 승리, -1 = 무승부)
  IF p_winner_id = 0 THEN
    v_result := 'win';
  ELSIF p_winner_id = -1 THEN
    v_result := 'draw';
  ELSE
    v_result := 'loss';
  END IF;

  -- 4. XP 계산 (player_actions >= 3인 경우만)
  IF p_player_actions >= 3 THEN
    -- 결과별 기본 XP
    IF v_result = 'win' THEN v_xp_earned := 50;
    ELSIF v_result = 'draw' THEN v_xp_earned := 30;
    ELSE v_xp_earned := 20;
    END IF;

    -- 플레이 시간 보너스
    IF v_duration_seconds >= 1200 THEN
      v_xp_earned := v_xp_earned + 25;
    ELSIF v_duration_seconds >= 600 THEN
      v_xp_earned := v_xp_earned + 15;
    ELSIF v_duration_seconds >= 300 THEN
      v_xp_earned := v_xp_earned + 5;
    END IF;

    -- 플레이타임도 player_actions >= 3인 경우만 누적
    v_playtime_credit := v_duration_seconds;
  END IF;

  -- 5. match_history 업데이트
  UPDATE public.match_history
  SET
    ended_at = NOW(),
    duration_seconds = v_duration_seconds,
    player_actions = p_player_actions,
    total_turns = p_total_turns,
    result = v_result,
    status = 'completed',
    xp_earned = v_xp_earned
  WHERE id = p_match_id;

  -- 6. player_profiles 현재 stats 조회
  SELECT total_experience, level
  INTO v_current_xp, v_current_level
  FROM public.player_profiles
  WHERE id = p_user_id;

  -- 7. 레벨업 계산 (100 * 2^(level-1) 공식)
  v_new_level := v_current_level;
  v_level_xp_sum := 0;
  FOR i IN 1..v_new_level - 1 LOOP
    v_level_xp_sum := v_level_xp_sum + (100 * POW(2, i - 1))::BIGINT;
  END LOOP;

  WHILE (v_current_xp + v_xp_earned) >= (v_level_xp_sum + (100 * POW(2, v_new_level - 1))::BIGINT) LOOP
    v_level_xp_sum := v_level_xp_sum + (100 * POW(2, v_new_level - 1))::BIGINT;
    v_new_level := v_new_level + 1;
  END LOOP;

  -- 8. player_profiles 업데이트
  UPDATE public.player_profiles
  SET
    total_matches = total_matches + 1,
    wins = wins + CASE WHEN v_result = 'win' THEN 1 ELSE 0 END,
    losses = losses + CASE WHEN v_result IN ('loss', 'draw') THEN 1 ELSE 0 END,
    total_experience = total_experience + v_xp_earned,
    level = v_new_level,
    total_playtime_seconds = total_playtime_seconds + v_playtime_credit,
    is_active_user = CASE
      WHEN (total_playtime_seconds + v_playtime_credit) >= 3600 THEN true
      ELSE is_active_user
    END,
    updated_at = NOW()
  WHERE id = p_user_id;

  -- 9. 결과 반환
  RETURN json_build_object(
    'xp_earned', v_xp_earned,
    'new_level', v_new_level,
    'duration_seconds', v_duration_seconds,
    'result', v_result,
    'is_active_user', COALESCE(
      (SELECT is_active_user FROM public.player_profiles WHERE id = p_user_id),
      false
    )
  );
END;
$$;
```

#### 1-6. record_abandoned_match DB 함수
```sql
CREATE OR REPLACE FUNCTION record_abandoned_match(
  p_match_id UUID,
  p_user_id UUID
)
RETURNS JSON
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = ''
AS $$
BEGIN
  -- 매치 상태 검증
  IF NOT EXISTS (
    SELECT 1 FROM public.match_history
    WHERE id = p_match_id AND user_id = p_user_id AND status = 'playing'
  ) THEN
    RETURN json_build_object('error', 'Match not found or already processed');
  END IF;

  -- match_history abandoned 처리
  UPDATE public.match_history
  SET
    ended_at = NOW(),
    duration_seconds = EXTRACT(EPOCH FROM (NOW() - started_at))::INTEGER,
    result = 'abandoned',
    status = 'abandoned',
    xp_earned = 0
  WHERE id = p_match_id;

  -- player_profiles 업데이트 (matches+1, losses+1, XP 0, playtime 0)
  UPDATE public.player_profiles
  SET
    total_matches = total_matches + 1,
    losses = losses + 1,
    updated_at = NOW()
  WHERE id = p_user_id;

  RETURN json_build_object('success', true);
END;
$$;
```

#### 1-7. pg_cron 정리 작업
```sql
-- 매 시간 실행: completed/abandoned 상태 + 24시간 경과 레코드 삭제
-- playing 상태 + 24시간 경과는 보존 (악성 유저 추적용)
SELECT cron.schedule(
  'cleanup-old-matches',
  '0 * * * *',
  $$DELETE FROM public.match_history
    WHERE status IN ('completed', 'abandoned')
    AND started_at < NOW() - INTERVAL '24 hours'$$
);
```

### 성공 기준:

#### 자동화된 검증:
- [x] 마이그레이션 적용 완료: Supabase MCP `apply_migration` 성공
- [x] `match_history` 테이블 존재: `SELECT * FROM match_history LIMIT 0`
- [x] `player_profiles` 컬럼 존재: `SELECT total_playtime_seconds, is_active_user FROM player_profiles LIMIT 0`
- [x] DB 함수 존재: `SELECT proname FROM pg_proc WHERE proname IN ('complete_match', 'record_abandoned_match')`
- [x] RLS 활성화: `SELECT rowsecurity FROM pg_tables WHERE tablename = 'match_history'`
- [x] pg_cron 작업 등록: `SELECT * FROM cron.job WHERE jobname = 'cleanup-old-matches'`

#### 수동 검증:
- [x] Supabase 대시보드에서 match_history 테이블 구조 확인
- [x] DB 함수 테스트: `SELECT complete_match(...)` 직접 호출 후 결과 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 사람의 확인을 위해 여기서 일시 중지합니다.

---

## 2단계: 레벨 공식 변경

### 개요
`level.ts`의 레벨 공식을 선형(`level * 100`)에서 지수(`100 * 2^(level-1)`)로 변경

### 필요한 변경:

#### 2-1. level.ts 수정
**파일**: `apps/web/src/lib/level.ts`

```typescript
// 레벨 N 완료에 필요한 XP: 100 * 2^(N-1)
export function xpRequiredForLevel(level: number): number {
  return 100 * Math.pow(2, level - 1);
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

변경 사항: `xpRequiredForLevel` 함수의 `return level * 100` → `return 100 * Math.pow(2, level - 1)`

### 성공 기준:

#### 자동화된 검증:
- [x] 타입 체크 통과: `pnpm --filter web typecheck`
- [x] `xpRequiredForLevel(1)` = 100, `xpRequiredForLevel(2)` = 200, `xpRequiredForLevel(3)` = 400

#### 수동 검증:
- [x] ProfileCard의 XP 바가 올바르게 표시되는지 확인

**Implementation Note**: 현재 모든 유저의 XP가 0이므로 데이터 마이그레이션 불필요.

---

## 3단계: 이벤트 시스템 확장

### 개요
플레이어의 무기 발사 횟수를 추적하기 위해 `WEAPON_FIRED` 이벤트 추가

### 필요한 변경:

#### 3-1. EventBus.ts에 이벤트 추가
**파일**: `packages/game-core/src/EventBus.ts`

`GameEvents` 객체에 추가:
```typescript
export const GameEvents = {
  TURN_CHANGED: "turn-changed",
  HEALTH_CHANGED: "health-changed",
  GAME_OVER: "game-over",
  WEAPON_SELECTED: "weapon-selected",
  WIND_CHANGED: "wind-changed",
  WEAPON_FIRED: "weapon-fired",  // 추가
} as const;
```

#### 3-2. GameScene.ts에서 발사 이벤트 발생
**파일**: `packages/game-core/src/scenes/GameScene.ts`

무기 발사 처리 함수에서, 인간 플레이어(playerId === 0)가 발사할 때 이벤트 발생:

```typescript
// handleFire() 또는 무기 발사 처리 로직 내부
// 인간 플레이어만 카운트 (AI 제외)
if (character.playerId === 0) {
  EventBus.emit(GameEvents.WEAPON_FIRED);
}
```

정확한 삽입 위치는 무기 발사 처리 함수의 투사체 생성 직후.

#### 3-3. game-core index.ts export 확인
**파일**: `packages/game-core/src/index.ts`

`GameEvents`는 이미 `EventBus`와 함께 export되고 있으므로 추가 변경 불필요.

### 성공 기준:

#### 자동화된 검증:
- [x] game-core 빌드 성공: `pnpm --filter @repo/game-core build`
- [x] 타입 체크 통과: `pnpm --filter @repo/game-core typecheck`

#### 수동 검증:
- [x] 게임에서 무기 발사 시 콘솔에서 이벤트 발생 확인 (개발 중 임시 로그)

**Implementation Note**: 이 단계 완료 후 game-core를 빌드해야 web 앱에서 새 이벤트를 사용할 수 있습니다.

---

## 4단계: 게임 시작 시 매치 기록

### 개요
`game/page.tsx` Server Component에서 매치 시작 기록 + 비정상 종료 매치 자동 처리 + `matchId`를 클라이언트로 전달

### 필요한 변경:

#### 4-1. game/page.tsx 수정
**파일**: `apps/web/src/app/game/page.tsx`

캐릭터 검증 완료 후, GameClient 렌더링 전에 매치 시작 로직 추가:

```typescript
import { createClient as createAdminClient } from "@supabase/supabase-js"

export default async function GamePage() {
  const supabase = await createClient()
  const { data: { user } } = await supabase.auth.getUser()

  if (!user) {
    redirect("/login")
  }

  // ... 기존 프로필 조회 및 캐릭터 검증 코드 ...

  // --- 매치 시작 로직 추가 ---
  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  // 1. 기존 playing 상태 매치 abandoned 처리
  const { data: existingMatches } = await admin
    .from("match_history")
    .select("id")
    .eq("user_id", user.id)
    .eq("status", "playing")

  if (existingMatches && existingMatches.length > 0) {
    for (const match of existingMatches) {
      await admin.rpc("record_abandoned_match", {
        p_match_id: match.id,
        p_user_id: user.id,
      })
    }
  }

  // 2. 새 매치 시작 기록
  const { data: newMatch } = await admin
    .from("match_history")
    .insert({
      user_id: user.id,
      status: "playing",
    })
    .select("id")
    .single()

  const matchId = newMatch?.id ?? null

  return (
    <GameClient
      characterType={verifiedCharacter}
      playerName={playerName}
      matchId={matchId}
    />
  )
}
```

#### 4-2. GameClient.tsx props 확장
**파일**: `apps/web/src/app/game/GameClient.tsx`

```typescript
interface GameClientProps {
  characterType: CharacterType
  playerName: string
  matchId: string | null  // 추가
}

export default function GameClient({ characterType, playerName, matchId }: GameClientProps) {
  return (
    <GameWrapper
      characterType={characterType}
      playerName={playerName}
      matchId={matchId}
    />
  )
}
```

#### 4-3. GameWrapper props 확장 (타입만)
**파일**: `apps/web/src/components/game/GameWrapper.tsx`

```typescript
interface GameWrapperProps {
  characterType: CharacterType
  playerName: string
  matchId: string | null  // 추가
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] 타입 체크 통과: `pnpm --filter web typecheck`
- [x] 빌드 성공: `pnpm --filter web build`

#### 수동 검증:
- [x] 게임 시작 후 `SELECT * FROM match_history WHERE status = 'playing'`에서 레코드 확인
- [x] 게임을 시작하지 않고 탭을 닫은 후 다시 게임 시작 시, 이전 매치가 abandoned 처리되었는지 확인

**Implementation Note**: 이 단계에서 GameWrapper의 GAME_OVER 핸들러는 아직 수정하지 않습니다. matchId prop만 추가합니다.

---

## 5단계: 매치 완료/포기 API

### 개요
매치 정상 완료와 Exit 버튼 포기를 처리하는 API 라우트 생성

### 필요한 변경:

#### 5-1. POST /api/match/complete
**파일**: `apps/web/src/app/api/match/complete/route.ts` (신규 생성)

```typescript
import { createClient } from "@/lib/supabase/server"
import { createClient as createAdminClient } from "@supabase/supabase-js"

export async function POST(request: Request) {
  const supabase = await createClient()
  const {
    data: { user },
    error,
  } = await supabase.auth.getUser()

  if (error || !user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 })
  }

  const { matchId, winnerId, playerActions, totalTurns } = await request.json()

  if (!matchId) {
    return Response.json({ error: "Missing matchId" }, { status: 400 })
  }

  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  const { data, error: rpcError } = await admin.rpc("complete_match", {
    p_match_id: matchId,
    p_user_id: user.id,
    p_winner_id: winnerId,
    p_player_actions: playerActions ?? 0,
    p_total_turns: totalTurns ?? 0,
  })

  if (rpcError) {
    console.error("Match completion error:", rpcError)
    return Response.json({ error: rpcError.message }, { status: 500 })
  }

  // DB 함수가 에러 객체를 반환한 경우
  if (data?.error) {
    return Response.json({ error: data.error }, { status: 400 })
  }

  return Response.json({ success: true, ...data })
}
```

#### 5-2. POST /api/match/abandon
**파일**: `apps/web/src/app/api/match/abandon/route.ts` (신규 생성)

```typescript
import { createClient } from "@/lib/supabase/server"
import { createClient as createAdminClient } from "@supabase/supabase-js"

export async function POST(request: Request) {
  const supabase = await createClient()
  const {
    data: { user },
    error,
  } = await supabase.auth.getUser()

  if (error || !user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 })
  }

  const { matchId } = await request.json()

  if (!matchId) {
    return Response.json({ error: "Missing matchId" }, { status: 400 })
  }

  const admin = createAdminClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )

  const { data, error: rpcError } = await admin.rpc("record_abandoned_match", {
    p_match_id: matchId,
    p_user_id: user.id,
  })

  if (rpcError) {
    console.error("Match abandon error:", rpcError)
    return Response.json({ error: rpcError.message }, { status: 500 })
  }

  if (data?.error) {
    return Response.json({ error: data.error }, { status: 400 })
  }

  return Response.json({ success: true })
}
```

### 성공 기준:

#### 자동화된 검증:
- [x] 타입 체크 통과: `pnpm --filter web typecheck`
- [x] 빌드 성공: `pnpm --filter web build`

#### 수동 검증:
- [x] curl로 API 직접 테스트 (인증 토큰 필요)
- [x] 존재하지 않는 matchId로 요청 시 400 에러 반환 확인
- [x] 이미 completed된 매치에 대해 재요청 시 에러 반환 확인

**Implementation Note**: API만 생성합니다. GameWrapper에서의 호출은 6단계에서 구현합니다.

---

## 6단계: GameWrapper 통합

### 개요
GameWrapper에서 `playerActions` 카운팅, `GAME_OVER` 시 complete API 호출, Exit 버튼 abandon API 호출, Sonner toast 처리

### 필요한 변경:

#### 6-1. GameWrapper.tsx 전체 수정
**파일**: `apps/web/src/components/game/GameWrapper.tsx`

주요 변경 사항:
1. `matchId` prop 수신
2. `playerActions` 상태 추가 + `WEAPON_FIRED` 이벤트 구독
3. `handleGameOver` 콜백에서 `/api/match/complete` 호출
4. Exit 버튼 클릭 시 `/api/match/abandon` 호출 후 로비 이동
5. API 실패 시 Sonner toast 표시
6. 3초 자동 리다이렉트를 API 완료 후로 변경

```typescript
"use client"

import { useCallback, useEffect, useRef, useState } from "react"
import { useRouter } from "next/navigation"
import { type CharacterType, EventBus, GameEvents } from "@repo/game-core"
import { PhaserGame } from "./PhaserGame"
import { Button } from "@/components/ui/shadcn/button"
import { toast } from "sonner"

interface GameResult {
  winnerId: number
}

interface PlayerInfo {
  name: string
  isAI: boolean
}

interface GameWrapperProps {
  characterType: CharacterType
  playerName: string
  matchId: string | null
}

export function GameWrapper({ characterType, playerName, matchId }: GameWrapperProps) {
  const players: PlayerInfo[] = [
    { name: playerName, isAI: false },
    { name: "Computer", isAI: true },
  ]
  const router = useRouter()
  const [screen, setScreen] = useState<"playing" | "result">("playing")
  const [result, setResult] = useState<GameResult | null>(null)
  const playerActionsRef = useRef(0)
  const roundNumberRef = useRef(0) // TurnSystem.roundNumber 직접 참조
  const isCompletingRef = useRef(false) // GAME_OVER 중복 호출 방어

  // 플레이어 액션(무기 발사) 카운팅 + 라운드 추적
  useEffect(() => {
    const handleWeaponFired = () => {
      playerActionsRef.current += 1
    }
    const handleTurnChanged = (data: { round: number }) => {
      roundNumberRef.current = data.round
    }

    EventBus.on(GameEvents.WEAPON_FIRED, handleWeaponFired)
    EventBus.on(GameEvents.TURN_CHANGED, handleTurnChanged)
    return () => {
      EventBus.off(GameEvents.WEAPON_FIRED, handleWeaponFired)
      EventBus.off(GameEvents.TURN_CHANGED, handleTurnChanged)
    }
  }, [])

  // 게임 오버 처리
  useEffect(() => {
    const handleGameOver = async (data: GameResult) => {
      // GAME_OVER 중복 호출 방어 (DamageSystem에서 동일 프레임 다중 발생 가능)
      if (isCompletingRef.current) return
      isCompletingRef.current = true

      setResult(data)
      setScreen("result")

      // 매치 완료 API 호출
      if (matchId) {
        try {
          const response = await fetch("/api/match/complete", {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify({
              matchId,
              winnerId: data.winnerId,
              playerActions: playerActionsRef.current,
              totalTurns: roundNumberRef.current,
            }),
          })

          if (!response.ok) {
            throw new Error("Failed to save match result")
          }

          const result = await response.json()
          console.info("[Match Complete]", result)
        } catch {
          toast.error("Failed to save match result. Your stats may not be updated.")
        }
      }

      // API 완료 후 3초 뒤 로비 이동
      setTimeout(() => {
        router.push("/lobby")
      }, 3000)
    }

    EventBus.on(GameEvents.GAME_OVER, handleGameOver)
    return () => {
      EventBus.off(GameEvents.GAME_OVER, handleGameOver)
    }
  }, [matchId, router])

  // Exit 버튼 클릭 → abandoned 처리
  const handleReturnToLobby = useCallback(async () => {
    if (matchId) {
      try {
        await fetch("/api/match/abandon", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ matchId }),
        })
      } catch {
        toast.error("Failed to save match result.")
      }
    }
    router.push("/lobby")
  }, [matchId, router])

  if (screen === "result" && result) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen bg-linear-to-b from-gray-900 to-gray-700">
        {result.winnerId >= 0 ? (
          <div className="flex items-end justify-center gap-1.5">
            <h1 className="text-3xl font-bold text-foreground">
              {players[result.winnerId].name}
            </h1>
            <span>Wins!</span>
          </div>
        ) : (
          <h1 className="text-3xl font-bold text-foreground">Draw!</h1>
        )}
        <div className="flex flex-col items-center gap-3">
          <div className="coffee_cup" />
        </div>
      </div>
    )
  }

  return (
    <div className="relative min-h-screen bg-gray-900">
      <PhaserGame characterType={characterType} />
      <Button
        onClick={handleReturnToLobby}
        variant="destructive"
        size="sm"
        className="absolute top-4 right-4 z-10"
      >
        Exit
      </Button>
    </div>
  )
}
```

핵심 변경 사항:
- `playerActionsRef`, `roundNumberRef`: `useRef`로 관리 (리렌더링 방지)
- `roundNumberRef`: `TURN_CHANGED` 페이로드의 `round` 값을 직접 참조 (`TurnSystem.roundNumber`와 동일)
- `isCompletingRef`: `GAME_OVER` 중복 호출 방어 (`DamageSystem.checkGameOver()`가 동일 프레임에 여러 번 호출될 수 있음)
- `WEAPON_FIRED` 이벤트 구독으로 발사 횟수 카운트
- `TURN_CHANGED` 이벤트 구독으로 라운드 번호 추적 (카운팅 아닌 직접 참조)
- `handleGameOver`: API 호출 → 응답 데이터 `console.info` 출력 → 3초 대기 → 로비 이동
- `handleReturnToLobby`: abandon API 호출 → 로비 이동
- 기존 3초 자동 리다이렉트 useEffect 제거 (handleGameOver 내부에서 setTimeout으로 대체)
- API 응답 데이터 (`xp_earned`, `new_level`, `duration_seconds`, `result`) console 출력 (UI 표시는 추후 별도 작업)

### 성공 기준:

#### 자동화된 검증:
- [x] 타입 체크 통과: `pnpm --filter web typecheck`
- [x] 빌드 성공: `pnpm --filter web build`

#### 수동 검증:
- [x] 게임 완료 후 ProfileCard에서 Matches/Wins/Losses 변화 확인
- [x] 게임 완료 후 XP 바 진행률 변화 확인
- [x] Exit 버튼 클릭 후 로비로 이동 + match_history에 abandoned 레코드 확인
- [x] 게임 중 탭 닫기 후 재접속 → 게임 시작 시 이전 매치 abandoned 처리 확인
- [x] API 실패 시 (네트워크 끊기 시뮬레이션) Sonner toast 표시 확인
- [x] player_actions < 3인 경우 XP/playtime 미지급 확인
- [x] 총 플레이타임 1시간 초과 시 is_active_user = true 확인

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 수동 테스트를 통해 전체 흐름을 검증합니다.

---

## 테스트 전략

### 통합 테스트 시나리오:

1. **정상 게임 완료 (승리)**
   - 게임 시작 → 플레이 → 승리 → 로비 복귀
   - 확인: total_matches +1, wins +1, XP +50~75, playtime 누적

2. **정상 게임 완료 (패배)**
   - 게임 시작 → 플레이 → 패배 → 로비 복귀
   - 확인: total_matches +1, losses +1, XP +20~45, playtime 누적

3. **무승부**
   - 게임 시작 → 무승부 → 로비 복귀
   - 확인: total_matches +1, losses +1, XP +30~55, playtime 누적

4. **Exit 버튼 포기**
   - 게임 시작 → Exit 클릭 → 로비 복귀
   - 확인: total_matches +1, losses +1, XP 0, playtime 0

5. **비정상 종료 (탭 닫기)**
   - 게임 시작 → 탭 닫기 → 새 탭에서 게임 시작
   - 확인: 이전 매치 abandoned, 새 매치 정상 시작

6. **최소 액션 미달**
   - 게임 시작 → 2회 이하 발사 → 게임 종료
   - 확인: total_matches +1, result 기록, XP 0, playtime 0

7. **레벨업**
   - 여러 게임 플레이 후 XP 누적으로 레벨업
   - 확인: level 증가, ProfileCard 반영

8. **API 실패 복원력**
   - 네트워크 차단 상태에서 게임 완료
   - 확인: Sonner toast 표시, 로비 이동은 정상 수행

### 수동 테스트 단계:
1. 로비 → ProfileCard stats 확인 (기본값)
2. PLAY 클릭 → 게임 진입 → match_history에 playing 레코드 확인
3. 무기 3회 이상 발사 → 적 처치 → 승리
4. 결과 화면 표시 → 3초 후 로비 이동
5. ProfileCard stats 변화 확인 (Matches +1, Wins +1, XP 증가)
6. 다시 게임 시작 → Exit 클릭 → 로비
7. ProfileCard stats 변화 확인 (Matches +1, Losses +1)

## 성능 고려 사항

- `match_history` 인덱스: `(user_id, status)` 복합 인덱스로 playing 매치 조회 최적화
- `started_at` 인덱스: pg_cron 정리 작업 성능 최적화
- DB 함수 `SECURITY DEFINER`: RLS 우회하여 성능 향상
- `useRef` 사용: playerActions/totalTurns 업데이트 시 불필요한 리렌더링 방지
- API 호출은 게임 흐름을 차단하지 않음 (fire-and-forget + toast)

## 마이그레이션 참고 사항

- 모든 유저의 현재 stats가 기본값(0)이므로 기존 데이터 마이그레이션 불필요
- 레벨 공식 변경도 기존 유저에게 영향 없음 (모두 level 1, XP 0)
- `match_history` 테이블은 새로 생성되므로 데이터 충돌 없음

## 참조

- 원본 연구: `thoughts/arta1069/research/2026-02-15-match-stats-playtime-active-user-research.md`
- 아키텍처 연구: `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md`
- 소셜 로그인/지갑 구현: `thoughts/arta1069/plans/2026-02-13-social-login-wallet-linking-implementation.md`
- 캐릭터 시스템: `thoughts/arta1069/plans/2026-02-13-character-skin-selection-system.md`
