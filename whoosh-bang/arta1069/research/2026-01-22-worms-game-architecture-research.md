---
date: 2026-01-22T21:30:00+09:00
researcher: dillon
topic: "웜즈 스타일 웹 게임 아키텍처 및 파이프라인 연구"
tags:
  [
    research,
    game-development,
    nextjs,
    phaser,
    architecture,
    pipeline,
    supabase,
    multiplayer,
    capacitor,
    web3,
    sui,
    zklogin,
  ]
status: complete
last_updated: 2026-01-25
last_updated_by: arta1069@gmail.com
last_updated_note: "MVP 구현 현황 반영 - Phase 5 무기 시스템 완료, Pose 기반 캐릭터 애니메이션"
---

# 연구: 웜즈 스타일 웹 게임 아키텍처 및 파이프라인 연구

**날짜**: 2026-01-22
**연구자**: dillon

## 연구 질문

웜즈(Worms) 같은 턴 기반 artillery 게임을 Next.js 기반으로 개발하면서, 추후 앱 빌드까지 고려한 아키텍처와 개발 파이프라인 구축 방법 연구

## 요약

### 핵심 결론

1. **프레임워크 선택**: Next.js 15+ + Phaser 3 (공식 템플릿 존재)
2. **크로스 플랫폼**: PWA 우선 → Capacitor로 앱스토어 배포
3. **게임 아키텍처**: ECS 패턴 + State Machine 기반 턴 관리
4. **개발 파이프라인**: Turborepo 모노레포 + Vitest + Playwright + Vercel

### 기술 스택 추천

| 영역            | 추천 기술                           | 대안         |
| --------------- | ----------------------------------- | ------------ |
| 프레임워크      | Next.js 15 (App Router)             | -            |
| 게임 엔진       | Phaser 3.90.0                       | PixiJS 8.2.6 |
| 물리 엔진       | Matter.js (Phaser 내장)             | 자체 구현    |
| 상태 관리       | Zustand + EventBus                  | Ape-ECS      |
| **백엔드**      | **Supabase (통합 BaaS)**            | Firebase     |
| **실시간 통신** | **Supabase Realtime (Broadcast)**   | Socket.IO    |
| **서버 로직**   | **Server Actions + Edge Functions** | NestJS       |
| **인증 (Web3)** | **SUI zkLogin + @mysten/dapp-kit**  | 지갑 서명    |
| **블록체인**    | **SUI (Mysten Labs)**               | -            |
| 크로스 플랫폼   | Capacitor 8                         | PWA          |
| 모노레포        | Turborepo                           | Nx           |
| 테스트          | Vitest + Playwright                 | Jest         |
| 배포            | Vercel                              | Netlify      |

---

## 상세 발견 사항

### 1. Next.js + 게임 엔진 통합

#### 공식 Phaser 템플릿

```bash
git clone https://github.com/phaserjs/template-nextjs
cd template-nextjs
npm install && npm run dev
```

**핵심 패턴**:

- `'use client'` 디렉티브로 Client Component 선언
- `next/dynamic`의 `ssr: false` 옵션 필수
- EventBus를 통한 React-Phaser 양방향 통신

```typescript
// 동적 import 패턴
const PhaserGame = dynamic(() => import("@/components/PhaserGame"), {
  ssr: false,
})
```

**디렉토리 구조 (추천)**:

```
src/
├── app/                    # Next.js App Router
├── components/
│   ├── game/              # 게임 캔버스
│   └── ui/                # HUD, 메뉴
├── game/
│   ├── scenes/            # Phaser 씬
│   ├── entities/          # 캐릭터, 투사체
│   ├── systems/           # 물리, 턴, 데미지
│   └── EventBus.ts        # React 통신
├── stores/                # Zustand 스토어
└── utils/
```

#### SSR 환경 주의사항

- `window`, `document` 접근 시 `typeof window !== 'undefined'` 체크
- Phaser/PixiJS는 서버 번들에서 제외 (webpack externals)
- React Strict Mode 이중 초기화 방지 필요

---

### 2. 크로스 플랫폼 전략

#### 추천: PWA + Capacitor 이중 전략

**PWA (Progressive Web App)**:

- Next.js 15+ 네이티브 PWA 지원
- 앱스토어 우회, 즉시 배포
- HTML5 Canvas/WebGL 성능 네이티브 수준
- SEO 유지

**Capacitor (앱스토어 배포 필요시)**:

- 100% 코드 공유 (웹↔모바일)
- 네이티브 API 접근 (카메라, 푸시 알림)
- 빠른 개발 속도

**비교표**:

| 항목        | PWA        | Capacitor | Tauri      | Expo        |
| ----------- | ---------- | --------- | ---------- | ----------- |
| 코드 공유   | 100%       | 100%      | 100%       | ~85%        |
| 성능        | 중-높음    | 중간      | 높음       | 매우 높음   |
| 앱스토어    | 불필요     | 필요      | 필요       | 필요        |
| 개발 난이도 | 쉬움       | 쉬움      | 중간       | 어려움      |
| 게임 적합성 | HTML5 게임 | 2D 캐주얼 | 데스크톱만 | 모바일 우선 |

**결론**: 웜즈 스타일 2D 게임에는 PWA로 시작, 앱스토어 필요시 Capacitor 추가

---

### 3. 웜즈 게임 핵심 아키텍처

#### 턴 기반 로직: State Machine

```typescript
// 상태 정의
type GameState =
  | "idle"
  | "playerTurn"
  | "aiming"
  | "firing"
  | "animating"
  | "gameOver"

// FSM 구현
const fsm = StateMachine.create({
  initial: "idle",
  events: [
    { name: "startTurn", from: "idle", to: "playerTurn" },
    { name: "aim", from: "playerTurn", to: "aiming" },
    { name: "fire", from: "aiming", to: "firing" },
    { name: "animate", from: "firing", to: "animating" },
    { name: "endTurn", from: "animating", to: "playerTurn" },
  ],
})
```

#### 물리 기반 투사체: Matter.js

```typescript
// 투사체 생성
const projectile = Matter.Bodies.circle(x, y, radius, {
  restitution: 0.8,
  friction: 0.01,
})

// 발사 (각도 + 파워)
const angle = (45 * Math.PI) / 180
const power = 20
Matter.Body.setVelocity(projectile, {
  x: power * Math.cos(angle),
  y: -power * Math.sin(angle),
})

// 바람 영향
function updateProjectile(windForce: number, dt: number) {
  projectile.vx += windForce * dt
  projectile.vy += gravity * dt
}
```

#### 파괴 가능 지형: 픽셀 기반

```typescript
// 1D 배열로 지형 저장
const terrain = new Array(width * height).fill(false)

// 폭발로 지형 파괴
function explode(x: number, y: number, radius: number) {
  for (let dy = -radius; dy <= radius; dy++) {
    for (let dx = -radius; dx <= radius; dx++) {
      if (Math.sqrt(dx * dx + dy * dy) <= radius) {
        terrain[(y + dy) * width + (x + dx)] = false
      }
    }
  }
}

// 충돌 감지: Bresenham's Line Algorithm
function checkCollision(x0, y0, x1, y1) {
  // 이전 위치에서 현재 위치까지 선을 따라 검사
}
```

#### 게임 상태 관리: ECS 패턴

```typescript
// Entity
class Entity {
  id: string
  components: Map<string, Component>
}

// Components (데이터만)
class Position {
  x: number
  y: number
}
class Velocity {
  vx: number
  vy: number
}
class Health {
  max: number
  current: number
}
class Explosive {
  radius: number
  damage: number
}

// Systems (로직)
class PhysicsSystem extends System {
  update(dt: number) {
    const entities = this.world.query(["Position", "Velocity"])
    entities.forEach((e) => {
      const pos = e.getComponent(Position)
      const vel = e.getComponent(Velocity)
      vel.vy += gravity * dt
      pos.x += vel.vx * dt
      pos.y += vel.vy * dt
    })
  }
}
```

**추천 라이브러리**: Ape-ECS (고성능, 풍부한 쿼리, 직렬화 지원)

---

### 4. 개발 파이프라인

#### 모노레포 구조 (Turborepo)

```
game-monorepo/
├── apps/
│   ├── web/                 # Next.js 웹 앱
│   └── game-server/         # (선택) 멀티플레이어
├── packages/
│   ├── game-core/           # Phaser 게임 로직
│   ├── ui/                  # 공유 UI
│   ├── types/               # TypeScript 타입
│   └── config/              # ESLint, TS 설정
├── turbo.json
└── pnpm-workspace.yaml
```

#### Asset Pipeline

**스프라이트시트**: Free-Tex-Packer (무료, CLI 지원)

```bash
free-tex-packer-cli --project ./sprites --output ./public/assets
```

**이미지 최적화**: Sharp (4-5배 빠름)

```javascript
await sharp(input)
  .resize(2048, 2048, { fit: "inside" })
  .webp({ quality: 85 })
  .toFile(output)
```

**오디오**: Howler.js + Audiosprite

```bash
audiosprite --format howler2 --output sfx src/audio/*.wav
```

#### 테스트 전략

```
        E2E Tests (Playwright)
       /  3-5 시나리오  \
      /                  \
    Integration Tests (RTL)
   /      >=80% 커버리지    \
  /                        \
Unit Tests (Vitest)
   50-60% 커버리지
```

**게임 로직 단위 테스트**:

```typescript
// collision.test.ts
describe("Collision System", () => {
  it("should detect collision", () => {
    const rect1 = { x: 0, y: 0, width: 50, height: 50 }
    const rect2 = { x: 25, y: 25, width: 50, height: 50 }
    expect(checkCollision(rect1, rect2)).toBe(true)
  })
})
```

#### CI/CD: GitHub Actions + Vercel

```yaml
# .github/workflows/deploy.yml
jobs:
  test:
    - npm ci
    - npm run build:assets
    - npm run test:coverage
    - npx playwright test

  deploy:
    - vercel deploy --prod
```

---

## 오픈소스 참고 프로젝트

| 프로젝트                                                                                     | 기술 스택             | 특징                    |
| -------------------------------------------------------------------------------------------- | --------------------- | ----------------------- |
| [Worms-Armageddon-HTML5-Clone](https://github.com/CiaranMcCann/Worms-Armageddon-HTML5-Clone) | TypeScript, Socket.io | 308 stars, 멀티플레이어 |
| [worms-js](https://github.com/Petally/worms-js)                                              | ES Modules            | 픽셀 지형 파괴          |
| [Kobold Kombat](https://half-shot.uk/blog/kobold-kombat-intro/)                              | PixiJS                | 완전 오픈소스 에셋      |

---

## 현재 프로젝트 Assets 분석

```
assets/
├── characters/           # 18개 캐릭터 (Char 1-18)
│   └── Char X/
│       ├── Character X.png
│       └── Preview Character X.gif
├── resource/             # 탱크 에셋 (Kenney.nl, CC0)
│   ├── PNG/Default size/
│   ├── PNG/Retina/
│   ├── Spritesheet/      # 스프라이트시트 준비됨
│   └── Vector/
└── sound/
    ├── chill-guy.mp3
    └── fortress-bgm.ogg.mp3
```

**다음 단계**:

1. 캐릭터 스프라이트를 웜즈 스타일 애니메이션으로 변환 필요
2. 지형 타일셋 추가 필요
3. 무기 이펙트/폭발 스프라이트 추가 필요

---

## 추천 시작 순서

### Phase 1: 프로젝트 셋업

1. Turborepo 모노레포 초기화
2. Next.js + Phaser 공식 템플릿 통합
3. 기본 개발 환경 구성 (ESLint, TypeScript, Vitest)

### Phase 2: 코어 게임 메카닉

1. 지형 렌더링 (픽셀 기반)
2. 캐릭터 이동/점프
3. 투사체 물리 (포물선, 중력)
4. 지형 파괴

### Phase 3: 턴 시스템

1. State Machine 구현
2. 턴 타이머
3. 에이밍 UI
4. 바람 시스템

### Phase 4: 게임 완성

1. 무기 종류 추가
2. 데미지 시스템
3. 승리 조건
4. UI/UX 폴리싱

### Phase 5: 배포

1. PWA 설정
2. Vercel 배포
3. (선택) Capacitor 앱 빌드

---

## 미해결 질문

1. **멀티플레이어**: 로컬 멀티만? 온라인도?
2. **맵 에디터**: 커스텀 맵 지원 여부
3. **AI**: 컴퓨터 플레이어 구현 여부
4. **저장/로드**: 게임 진행 상태 저장 필요 여부

---

## 참고 자료

### 공식 문서

- [Phaser 3 Documentation](https://docs.phaser.io/)
- [Next.js Documentation](https://nextjs.org/docs)
- [Matter.js Documentation](https://brm.io/matter-js/docs/)
- [Capacitor Documentation](https://capacitorjs.com/docs)
- [Turborepo Documentation](https://turbo.build/repo/docs)

### 템플릿

- [Phaser Next.js Template](https://github.com/phaserjs/template-nextjs)
- [Turborepo Starter](https://turbo.build/repo/docs)

### 튜토리얼

- [Destructible Pixel Terrain](https://code.tutsplus.com/coding-destructible-pixel-terrain-how-to-make-everything-explode--gamedev-45t)
- [ECS Pattern for Games](https://jsforgames.com/ecs/)
- [Matter.js Physics Tutorial](https://codersblock.com/blog/javascript-physics-with-matter-js/)

---

## 후속 연구 (2026-01-22)

### 연구 질문

기존 연구의 "미해결 질문"에 대한 구체적인 요구사항이 제시됨:

1. 한 게임에 최소 4명 ~ 최대 6명의 플레이어
2. 개인전, 2:2, 3:3과 같은 멀티플레이어 모드 지원
3. 빈 슬롯은 AI 플레이어로 자동 대체
4. 게임 중 유저 이탈 시 AI로 대체
5. **서버 인프라: Next.js + Supabase 단순화된 아키텍처**

---

### 1. 멀티플레이어 아키텍처 (Next.js + Supabase)

#### 단순화된 아키텍처 선택 이유

**웜즈 스타일 턴 기반 게임 특성**:

- 턴 주기: 30-60초 간격 (실시간 액션 게임 대비 저빈도)
- 동시 활성 플레이어: 턴당 1명
- Supabase Realtime 성능 (224,000 msg/s)이 충분

**단순화의 이점**:

- 인프라 복잡도 감소 (단일 배포 대상)
- 개발/유지보수 비용 절감
- Vercel 원클릭 배포
- Capacitor 앱 빌드 시 동일한 코드베이스 사용

**전체 아키텍처**:

```
┌─────────────────────────────────────────────────────────────────┐
│              Next.js (App Router + Server Actions)              │
│  ┌────────────┐  ┌────────────┐  ┌─────────────────────────┐   │
│  │ React UI   │  │ Phaser 3   │  │ Supabase Client         │   │
│  │ (HUD/Menu) │  │ (Canvas)   │  │ (Auth, Realtime)        │   │
│  └────────────┘  └────────────┘  └─────────────────────────┘   │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Server Actions (서버 측 로직)                             │  │
│  │ - 턴 검증 및 게임 상태 업데이트                           │  │
│  │ - 매칭/로비 관리                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────┘
                              │
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        Supabase                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ Auth        │  │ PostgreSQL  │  │ Realtime                │ │
│  │ - JWT 인증  │  │ - 프로필    │  │ - Broadcast (게임 상태) │ │
│  │ - OAuth     │  │ - 매치 기록 │  │ - Presence (온라인)     │ │
│  └─────────────┘  │ - 리더보드  │  └─────────────────────────┘ │
│                   └─────────────┘                               │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Edge Functions                                            │  │
│  │ - AI 턴 처리 (백그라운드)                                 │  │
│  │ - 보상/점수 계산                                          │  │
│  │ - 매치 종료 처리                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

#### 책임 분담

**Next.js Server Actions 담당**:

- ✅ 턴 검증 (서버 권위)
- ✅ 게임 상태 업데이트 및 Broadcast 전송
- ✅ 로비 생성/참가 처리
- ✅ 매칭 로직

**Supabase 담당**:

- ✅ 사용자 인증 (Auth)
- ✅ 게임 상태 실시간 동기화 (Realtime Broadcast)
- ✅ 플레이어 온라인 상태 (Presence)
- ✅ 영구 데이터 저장 (PostgreSQL)

**Supabase Edge Functions 담당**:

- ✅ AI 플레이어 턴 처리
- ✅ 매치 종료 후 보상 계산
- ✅ 비동기 백그라운드 작업

#### Server Actions 기반 게임 로직

```typescript
// app/actions/game.ts
"use server"

import { createClient } from "@/utils/supabase/server"
import { revalidatePath } from "next/cache"

// 게임 모드별 설정
const gameModes = {
  ffa: { teams: 1, playersPerTeam: 6 },
  "2v2": { teams: 2, playersPerTeam: 2 },
  "3v3": { teams: 2, playersPerTeam: 3 },
}

export async function makeMove(roomId: string, action: GameAction) {
  const supabase = await createClient()
  const {
    data: { user },
  } = await supabase.auth.getUser()

  if (!user) throw new Error("Unauthorized")

  // 1. 현재 게임 상태 조회
  const { data: game } = await supabase
    .from("active_games")
    .select("*")
    .eq("id", roomId)
    .single()

  // 2. 서버 권위: 턴 검증
  if (game.current_player_id !== user.id) {
    throw new Error("Not your turn")
  }

  // 3. 액션 유효성 검증
  const validatedAction = validateAction(game.state, action)
  if (!validatedAction.valid) {
    throw new Error(validatedAction.error)
  }

  // 4. 게임 상태 업데이트
  const newState = applyAction(game.state, action)
  const nextPlayer = determineNextPlayer(newState)

  await supabase
    .from("active_games")
    .update({
      state: newState,
      current_player_id: nextPlayer.id,
      updated_at: new Date().toISOString(),
    })
    .eq("id", roomId)

  // 5. Realtime Broadcast로 모든 플레이어에게 전송
  const channel = supabase.channel(`game:${roomId}`)
  await channel.send({
    type: "broadcast",
    event: "game_update",
    payload: {
      state: newState,
      lastAction: action,
      nextPlayer,
    },
  })

  // 6. 다음 플레이어가 AI인 경우 Edge Function 호출
  if (nextPlayer.isAI) {
    await supabase.functions.invoke("ai-turn", {
      body: { roomId, playerId: nextPlayer.id },
    })
  }

  return { success: true, newState }
}
```

#### 클라이언트 실시간 연결

```typescript
// hooks/useGameRoom.ts
"use client"

import { useEffect, useState } from "react"
import { createClient } from "@/utils/supabase/client"

export function useGameRoom(roomId: string) {
  const [gameState, setGameState] = useState<GameState | null>(null)
  const [players, setPlayers] = useState<Player[]>([])
  const supabase = createClient()

  useEffect(() => {
    const channel = supabase.channel(`game:${roomId}`, {
      config: { presence: { key: "players" } },
    })

    channel
      .on("broadcast", { event: "game_update" }, ({ payload }) => {
        setGameState(payload.state)
      })
      .on("presence", { event: "sync" }, () => {
        const state = channel.presenceState()
        setPlayers(Object.values(state).flat())
      })
      .subscribe()

    return () => {
      supabase.removeChannel(channel)
    }
  }, [roomId])

  return { gameState, players }
}
```

#### 로비 및 매칭 시스템

```typescript
// app/actions/lobby.ts
"use server"

import { createClient } from "@/utils/supabase/server"

export async function findOrCreateMatch(mode: string) {
  const supabase = await createClient()
  const {
    data: { user },
  } = await supabase.auth.getUser()
  const config = gameModes[mode]

  // 1. 대기 중인 로비 찾기
  const { data: waitingLobbies } = await supabase
    .from("game_lobbies")
    .select("*, lobby_players(count)")
    .eq("status", "waiting")
    .eq("game_mode", mode)
    .order("created_at", { ascending: true })
    .limit(10)

  for (const lobby of waitingLobbies || []) {
    const playerCount = lobby.lobby_players[0]?.count || 0
    if (playerCount < config.playersPerTeam * (config.teams || 1)) {
      await supabase
        .from("lobby_players")
        .insert({ lobby_id: lobby.id, player_id: user.id })
      return { lobbyId: lobby.id, isNew: false }
    }
  }

  // 2. 새 로비 생성
  const { data: newLobby } = await supabase
    .from("game_lobbies")
    .insert({
      game_mode: mode,
      creator_id: user.id,
      max_players: config.playersPerTeam * (config.teams || 1),
      status: "waiting",
    })
    .select()
    .single()

  await supabase
    .from("lobby_players")
    .insert({ lobby_id: newLobby.id, player_id: user.id })

  return { lobbyId: newLobby.id, isNew: true }
}
```

---

### 2. AI 플레이어 시스템

#### 의사결정 구조: Utility AI (권장)

**웜즈 스타일 게임에 적합한 이유**:

- 웨폰 선택, 타겟 선택, 포지셔닝 등 다양한 요소를 점수화 가능
- 상황에 따라 동적으로 우선순위 변경
- 예측 가능하지만 자연스러운 AI 행동

```typescript
class UtilityAI {
  // 각 행동의 효용성 계산
  calculateUtility(action: Action, gameState: GameState): number {
    let utility = 1.0

    utility *= this.healthConsideration(gameState)
    utility *= this.damageConsideration(action, gameState)
    utility *= this.positionConsideration(action, gameState)
    utility *= this.windConsideration(action, gameState)

    return utility
  }

  // 타겟 선택
  selectTarget(
    enemies: Worm[],
    myPosition: Vector2,
    difficulty: Difficulty
  ): Worm {
    const scores = enemies.map((enemy) => ({
      enemy,
      score: this.calculateTargetScore(enemy, myPosition),
    }))

    // 난이도별 다른 선택 전략
    if (difficulty === "HARD") {
      return scores.reduce((best, curr) =>
        curr.score > best.score ? curr : best
      ).enemy
    } else if (difficulty === "MEDIUM") {
      const topHalf = scores
        .sort((a, b) => b.score - a.score)
        .slice(0, Math.ceil(scores.length / 2))
      return topHalf[Math.floor(Math.random() * topHalf.length)].enemy
    } else {
      return enemies[Math.floor(Math.random() * enemies.length)]
    }
  }
}
```

#### 난이도별 설정

```typescript
interface DifficultySettings {
  aimAccuracy: number // 0-1
  thinkingTime: number // ms
  missChance: number // 0-1
  useAdvancedWeapons: boolean
  considerWind: boolean
}

const difficulties: Record<string, DifficultySettings> = {
  EASY: {
    aimAccuracy: 0.5,
    thinkingTime: 2000,
    missChance: 0.4,
    useAdvancedWeapons: false,
    considerWind: false,
  },
  MEDIUM: {
    aimAccuracy: 0.75,
    thinkingTime: 3000,
    missChance: 0.2,
    useAdvancedWeapons: true,
    considerWind: true,
  },
  HARD: {
    aimAccuracy: 0.95,
    thinkingTime: 4000,
    missChance: 0.05,
    useAdvancedWeapons: true,
    considerWind: true,
  },
}
```

#### 탄도 계산 알고리즘

```typescript
class BallisticCalculator {
  // 목표 지점을 맞추기 위한 각도 계산
  calculateAngleToHit(
    start: Vector2,
    target: Vector2,
    power: number,
    gravity: number
  ): { low: number; high: number } | null {
    const dx = target.x - start.x
    const dy = target.y - start.y
    const v2 = power * power

    const discriminant = v2 * v2 - gravity * (gravity * dx * dx + 2 * v2 * dy)

    if (discriminant < 0) return null // 도달 불가능

    const sqrtD = Math.sqrt(discriminant)
    return {
      low: Math.atan((v2 - sqrtD) / (gravity * dx)),
      high: Math.atan((v2 + sqrtD) / (gravity * dx)),
    }
  }

  // 바람을 고려한 시뮬레이션 기반 조준
  findBestShot(start: Vector2, target: Vector2, wind: number): Shot {
    const testShots: Shot[] = []

    for (let angle = 20; angle <= 70; angle += 5) {
      for (let power = 50; power <= 100; power += 10) {
        const impact = this.simulateShot(start, angle, power, wind)
        const distance = this.distance(impact, target)

        testShots.push({ angle, power, distance })
      }
    }

    return testShots.reduce((best, curr) =>
      curr.distance < best.distance ? curr : best
    )
  }
}
```

---

### 3. 플레이어 슬롯 관리 및 AI 대체 시스템

#### 슬롯 기반 플레이어 관리

```typescript
interface PlayerSlot {
  index: number
  player: Player | null
  state: "empty" | "connected" | "disconnected" | "ai"
  reconnectToken: string | null
  disconnectTime: number | null
  teamId: number
}

class GameRoom {
  slots: PlayerSlot[]
  reconnectTimeout = 60000 // 60초

  constructor(maxPlayers: number) {
    this.slots = Array(maxPlayers)
      .fill(null)
      .map((_, index) => ({
        index,
        player: null,
        state: "empty",
        reconnectToken: null,
        disconnectTime: null,
        teamId: -1,
      }))
  }

  // 빈 슬롯에 AI 채우기
  fillEmptySlotsWithAI() {
    this.slots.forEach((slot, index) => {
      if (slot.state === "empty") {
        slot.state = "ai"
        slot.player = {
          id: `ai_${index}`,
          name: `AI Player ${index + 1}`,
          isAI: true,
        }
      }
    })
  }

  // 플레이어 연결 해제 처리
  handleDisconnect(socketId: string) {
    const slot = this.slots.find((s) => s.player?.socketId === socketId)
    if (!slot) return

    slot.state = "disconnected"
    slot.disconnectTime = Date.now()
    slot.reconnectToken = this.generateToken()

    // 재연결 타임아웃 설정
    setTimeout(() => {
      if (slot.state === "disconnected") {
        this.convertToAI(slot)
      }
    }, this.reconnectTimeout)
  }

  // AI로 변환
  convertToAI(slot: PlayerSlot) {
    const originalName = slot.player?.name || `Player ${slot.index + 1}`
    slot.state = "ai"
    slot.player = {
      id: `ai_${slot.index}`,
      name: `${originalName} (AI)`,
      isAI: true,
      // 기존 게임 상태 유지 (체력, 위치 등)
      ...slot.player,
    }

    // 현재 턴이면 AI 행동 트리거
    if (this.isCurrentTurn(slot)) {
      this.executeAITurn(slot)
    }
  }

  // 재연결 처리
  handleReconnect(reconnectToken: string, socket: any): boolean {
    const slot = this.slots.find((s) => s.reconnectToken === reconnectToken)
    if (!slot || slot.state !== "disconnected") return false

    slot.player.socketId = socket.id
    slot.state = "connected"
    slot.disconnectTime = null

    // 현재 게임 상태 전송
    this.sendGameState(socket)
    return true
  }
}
```

#### Colyseus의 재연결 메커니즘

```typescript
async onLeave(client: Client, consented: boolean) {
  const player = this.state.players.get(client.sessionId);
  player.connected = false;

  try {
    if (consented) {
      throw new Error("consented leave");
    }

    // 60초 동안 재연결 허용
    const reconnection = this.allowReconnection(client, 60);

    // 2턴 이상 놓치면 AI로 전환
    const startTurn = this.state.currentTurn;
    const interval = setInterval(() => {
      if (this.state.currentTurn - startTurn > 2) {
        reconnection.reject();
        clearInterval(interval);
      }
    }, 1000);

    await reconnection;

    // 재연결 성공
    player.connected = true;
    clearInterval(interval);

  } catch (e) {
    // 재연결 실패 - AI로 대체
    this.convertPlayerToAI(client.sessionId);
  }
}
```

---

### 4. 팀 기반 게임 모드

#### 팀 할당 알고리즘

```typescript
// 간단한 팀 밸런싱 (스킬 레이팅 기반)
function balanceTeams(players: Player[], teamCount: number): Team[] {
  // 스킬 레이팅으로 정렬
  const sorted = [...players].sort((a, b) => b.skillRating - a.skillRating)

  const teams: Team[] = Array(teamCount)
    .fill(null)
    .map(() => ({ players: [], totalRating: 0 }))

  // 스네이크 드래프트 방식
  sorted.forEach((player, index) => {
    const round = Math.floor(index / teamCount)
    const teamIndex =
      round % 2 === 0 ? index % teamCount : teamCount - 1 - (index % teamCount)

    teams[teamIndex].players.push(player)
    teams[teamIndex].totalRating += player.skillRating
  })

  return teams
}

// 2v2, 3v3 모드 지원
function assignTeams(players: Player[], mode: "2v2" | "3v3") {
  const playersPerTeam = mode === "2v2" ? 2 : 3
  const teamCount = 2

  return balanceTeams(players, teamCount)
}
```

#### 팀 기반 턴 순서

```typescript
// 턴 순서 전략
type TurnOrder = "alternating" | "team-sequential" | "speed-based"

class TurnManager {
  // 번갈아가며 턴 (Team A Player 1 → Team B Player 1 → Team A Player 2 → ...)
  getAlternatingOrder(teams: Team[]): Player[] {
    const maxSize = Math.max(...teams.map((t) => t.players.length))
    const order: Player[] = []

    for (let i = 0; i < maxSize; i++) {
      teams.forEach((team) => {
        if (team.players[i]) {
          order.push(team.players[i])
        }
      })
    }

    return order
  }

  // 팀 순차 턴 (Team A 전원 → Team B 전원)
  getTeamSequentialOrder(teams: Team[]): Player[] {
    return teams.flatMap((team) => team.players)
  }

  // 속도 기반 턴 순서
  getSpeedBasedOrder(allPlayers: Player[]): Player[] {
    return [...allPlayers].sort((a, b) => b.speed - a.speed)
  }
}
```

#### 팀 승리 조건

```typescript
interface VictoryCondition {
  type: "elimination" | "points" | "objectives"
  checkVictory(gameState: GameState): Team | null
}

// 섬멸전 (기본 웜즈 스타일)
const eliminationVictory: VictoryCondition = {
  type: "elimination",
  checkVictory(gameState: GameState): Team | null {
    const teamsWithAliveWorms = gameState.teams.filter((team) =>
      team.players.some((player) =>
        player.worms.some((worm) => worm.health > 0)
      )
    )

    if (teamsWithAliveWorms.length === 1) {
      return teamsWithAliveWorms[0]
    }

    if (teamsWithAliveWorms.length === 0) {
      // 무승부
      return null
    }

    return null // 게임 계속
  },
}

// 점수 기반
const pointsVictory: VictoryCondition = {
  type: "points",
  checkVictory(gameState: GameState): Team | null {
    const targetPoints = 1000
    return gameState.teams.find((team) => team.score >= targetPoints) || null
  },
}
```

#### 팀 채팅 시스템 (Supabase Broadcast)

```typescript
// hooks/useTeamChat.ts
"use client"

import { useEffect, useState } from "react"
import { createClient } from "@/utils/supabase/client"

export function useTeamChat(roomId: string, teamId: string) {
  const [messages, setMessages] = useState<ChatMessage[]>([])
  const supabase = createClient()

  useEffect(() => {
    // 팀 전용 채널
    const teamChannel = supabase.channel(`team:${roomId}:${teamId}`)
    // 전체 게임 채널
    const gameChannel = supabase.channel(`game:${roomId}:chat`)

    teamChannel
      .on("broadcast", { event: "team_message" }, ({ payload }) => {
        setMessages((prev) => [...prev, { ...payload, type: "team" }])
      })
      .subscribe()

    gameChannel
      .on("broadcast", { event: "game_message" }, ({ payload }) => {
        setMessages((prev) => [...prev, { ...payload, type: "all" }])
      })
      .subscribe()

    return () => {
      supabase.removeChannel(teamChannel)
      supabase.removeChannel(gameChannel)
    }
  }, [roomId, teamId])

  const sendTeamMessage = async (message: string) => {
    const channel = supabase.channel(`team:${roomId}:${teamId}`)
    await channel.send({
      type: "broadcast",
      event: "team_message",
      payload: { message, timestamp: Date.now() },
    })
  }

  const sendGameMessage = async (message: string) => {
    const channel = supabase.channel(`game:${roomId}:chat`)
    await channel.send({
      type: "broadcast",
      event: "game_message",
      payload: { message, timestamp: Date.now() },
    })
  }

  return { messages, sendTeamMessage, sendGameMessage }
}
```

---

### 5. Supabase 데이터베이스 스키마 설계

#### 핵심 테이블 구조

```sql
-- 플레이어 프로필 (auth.users와 연동)
CREATE TABLE player_profiles (
  id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
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

-- 게임 로비/룸
CREATE TABLE game_lobbies (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  lobby_name TEXT NOT NULL,
  creator_id UUID REFERENCES player_profiles(id),
  game_mode TEXT NOT NULL, -- 'ffa', '2v2', '3v3'
  max_players INTEGER DEFAULT 6,
  status TEXT DEFAULT 'waiting', -- 'waiting', 'in_progress', 'completed'
  settings JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  started_at TIMESTAMPTZ,
  ended_at TIMESTAMPTZ
);

-- 매치 히스토리
CREATE TABLE match_history (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  lobby_id UUID REFERENCES game_lobbies(id),
  game_mode TEXT NOT NULL,
  duration_seconds INTEGER,
  winner_team INTEGER,
  match_data JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 플레이어별 매치 결과
CREATE TABLE player_match_results (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  match_id UUID REFERENCES match_history(id) ON DELETE CASCADE,
  player_id UUID REFERENCES player_profiles(id) ON DELETE CASCADE,
  team_number INTEGER,
  score INTEGER DEFAULT 0,
  kills INTEGER DEFAULT 0,
  deaths INTEGER DEFAULT 0,
  rank INTEGER,
  rewards JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- 리더보드
CREATE TABLE leaderboard_entries (
  id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  player_id UUID REFERENCES player_profiles(id) ON DELETE CASCADE,
  leaderboard_type TEXT NOT NULL, -- 'global', 'weekly', 'season_1'
  score BIGINT NOT NULL,
  rank INTEGER,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(player_id, leaderboard_type)
);

-- 인덱스
CREATE INDEX idx_leaderboard_rank ON leaderboard_entries(leaderboard_type, score DESC);
CREATE INDEX idx_match_results_player ON player_match_results(player_id);
CREATE INDEX idx_lobbies_status ON game_lobbies(status, game_mode);
```

#### RLS 보안 정책

```sql
-- 플레이어는 자신의 프로필만 수정 가능
CREATE POLICY "players_update_own_profile"
ON player_profiles FOR UPDATE TO authenticated
USING ((SELECT auth.uid()) = id)
WITH CHECK ((SELECT auth.uid()) = id);

-- 모든 프로필 조회 가능 (공개 정보)
CREATE POLICY "profiles_publicly_readable"
ON player_profiles FOR SELECT TO authenticated
USING (true);

-- 리더보드는 모든 인증된 사용자가 조회 가능
CREATE POLICY "leaderboard_readable"
ON leaderboard_entries FOR SELECT TO authenticated
USING (true);
```

---

### 6. SUI 블록체인 지갑 인증 (Web3)

#### 인증 방식 비교

| 방식               | 사용자 경험   | 구현 복잡도 | 게임 온보딩 |
| ------------------ | ------------- | ----------- | ----------- |
| **zkLogin (추천)** | 소셜 로그인만 | 높음        | 마찰 최소   |
| **지갑 연결**      | 지갑 앱 필요  | 중간        | 마찰 높음   |

**추천**: zkLogin을 기본으로, 고급 사용자를 위한 지갑 연결 옵션 제공

#### 설치 패키지

```bash
npm install @mysten/dapp-kit @mysten/sui @tanstack/react-query
```

#### Provider 설정 (Next.js App Router)

```typescript
// app/providers.tsx
'use client';

import { createNetworkConfig, SuiClientProvider, WalletProvider } from '@mysten/dapp-kit';
import { getFullnodeUrl } from '@mysten/sui/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const { networkConfig } = createNetworkConfig({
  mainnet: { url: getFullnodeUrl('mainnet') },
  testnet: { url: getFullnodeUrl('testnet') },
});

const queryClient = new QueryClient();

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <SuiClientProvider networks={networkConfig} defaultNetwork="mainnet">
        <WalletProvider>
          {children}
        </WalletProvider>
      </SuiClientProvider>
    </QueryClientProvider>
  );
}
```

#### zkLogin 인증 플로우

```
1. 임시 키 쌍 생성 (Ephemeral Key Pair)
2. OAuth 로그인 (Google/Apple/Discord 등)
3. Salt Service에서 사용자별 고유 salt 수신
4. Zero-Knowledge Proof 생성
5. zkLogin 주소 도출 → Supabase 세션 생성
```

**zkLogin 지원 OAuth 제공자**: Google, Facebook, Twitch, Apple, Kakao

#### 지갑 서명 기반 인증 구현

```typescript
// hooks/useWalletAuth.ts
"use client"

import { useCurrentAccount, useSignPersonalMessage } from "@mysten/dapp-kit"

export function useWalletAuth() {
  const account = useCurrentAccount()
  const { mutateAsync: signMessage } = useSignPersonalMessage()

  const authenticate = async () => {
    if (!account) throw new Error("지갑이 연결되지 않았습니다")

    // 1. Nonce 요청
    const { nonce } = await fetch("/api/auth/nonce", {
      method: "POST",
      body: JSON.stringify({ address: account.address }),
    }).then((r) => r.json())

    // 2. 메시지 서명
    const message = `게임 로그인\n주소: ${account.address}\nNonce: ${nonce}`
    const { signature, bytes } = await signMessage({
      message: new TextEncoder().encode(message),
    })

    // 3. 서명 검증 및 JWT 수신
    const { token } = await fetch("/api/auth/verify", {
      method: "POST",
      body: JSON.stringify({
        address: account.address,
        signature,
        message: bytes,
        nonce,
      }),
    }).then((r) => r.json())

    return token
  }

  return { authenticate, account }
}
```

#### 서버 측 서명 검증

```typescript
// app/api/auth/verify/route.ts
import { verifyPersonalMessageSignature } from "@mysten/sui/verify"
import jwt from "jsonwebtoken"

export async function POST(req: Request) {
  const { address, signature, message, nonce } = await req.json()

  // 1. Nonce 검증 (DB에서 확인 후 삭제)
  // ...

  // 2. SUI 서명 검증
  await verifyPersonalMessageSignature(message, signature, { address })

  // 3. 사용자 생성/조회 후 JWT 발급
  const token = jwt.sign(
    { walletAddress: address, role: "authenticated" },
    process.env.SUPABASE_JWT_SECRET!,
    { expiresIn: "7d" }
  )

  return Response.json({ token })
}
```

#### 인증 UI 컴포넌트

```typescript
// components/AuthModal.tsx
'use client';

import { ConnectButton } from '@mysten/dapp-kit';
import { useWalletAuth } from '@/hooks/useWalletAuth';

export function AuthModal() {
  const { authenticate, account } = useWalletAuth();

  return (
    <div className="auth-modal">
      {/* zkLogin (추천) */}
      <button onClick={() => startZkLogin('google')}>
        Google로 시작하기
      </button>

      {/* 지갑 연결 (고급) */}
      <ConnectButton />
      {account && (
        <button onClick={authenticate}>
          지갑으로 로그인
        </button>
      )}
    </div>
  );
}
```

#### 데이터베이스 스키마 수정

```sql
-- player_profiles 테이블에 지갑 주소 추가
ALTER TABLE player_profiles
ADD COLUMN wallet_address TEXT UNIQUE,
ADD COLUMN zklogin_address TEXT UNIQUE,
ADD COLUMN auth_provider TEXT; -- 'wallet', 'google', 'apple' 등

-- Nonce 테이블 (일회용 서명 검증)
CREATE TABLE auth_nonces (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_address TEXT NOT NULL,
  nonce TEXT NOT NULL,
  expires_at TIMESTAMPTZ NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_nonces_wallet ON auth_nonces(wallet_address, nonce);
```

---

### 8. Supabase Realtime 활용 패턴

#### Broadcast vs Presence 사용 구분

| 기능          | Broadcast          | Presence      | Postgres Changes    |
| ------------- | ------------------ | ------------- | ------------------- |
| **용도**      | 일시적 이벤트      | 온라인 상태   | DB 변경 알림        |
| **성능**      | 224,000 msg/s, 6ms | 제한적        | 10,000 msg/s        |
| **게임 활용** | 채팅, 이모티콘     | 플레이어 목록 | 결과 동기화         |
| **권장도**    | ⭐⭐⭐⭐⭐         | ⭐⭐⭐⭐      | ⭐⭐ (프로토타입만) |

#### Presence를 통한 플레이어 온라인 상태 추적

```typescript
// 클라이언트 측
const lobbyChannel = supabase.channel("lobby", {
  config: { presence: { key: userId } },
})

lobbyChannel
  .on("presence", { event: "sync" }, () => {
    const onlinePlayers = lobbyChannel.presenceState()
    updatePlayerList(onlinePlayers)
  })
  .on("presence", { event: "join" }, ({ key }) => {
    console.log("Player joined lobby:", key)
  })
  .on("presence", { event: "leave" }, ({ key }) => {
    console.log("Player left lobby:", key)
  })
  .subscribe(async (status) => {
    if (status === "SUBSCRIBED") {
      await lobbyChannel.track({
        username: currentUser.username,
        status: "online",
      })
    }
  })
```

---

### 9. 배포 전략

#### 프로덕션 아키텍처

```
                    ┌─────────────────┐
                    │   Cloudflare    │
                    │   (CDN/WAF)     │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
    ┌─────────▼───────────┐       ┌─────────▼─────────┐
    │   Next.js (Vercel)  │       │     Supabase      │
    │   - App Router      │       │   - Auth          │
    │   - Server Actions  │◄─────►│   - PostgreSQL    │
    │   - Phaser 3        │       │   - Realtime      │
    │                     │       │   - Edge Functions│
    └─────────────────────┘       └───────────────────┘
```

#### 배포 단순화의 이점

| 측면            | 기존 (NestJS + Supabase) | 단순화 (Next.js + Supabase) |
| --------------- | ------------------------ | --------------------------- |
| **배포 대상**   | Vercel + Fly.io + Redis  | Vercel만                    |
| **월 비용**     | $50~100+                 | $0~20                       |
| **배포 복잡도** | CI/CD 멀티 타겟          | Vercel 자동 배포            |
| **스케일링**    | 수동 설정 필요           | Vercel/Supabase 자동        |

#### Vercel 배포 설정

```json
// vercel.json
{
  "framework": "nextjs",
  "regions": ["icn1"],
  "env": {
    "NEXT_PUBLIC_SUPABASE_URL": "@supabase_url",
    "NEXT_PUBLIC_SUPABASE_ANON_KEY": "@supabase_anon_key",
    "SUPABASE_SERVICE_ROLE_KEY": "@supabase_service_key"
  }
}
```

#### Capacitor 앱 빌드

```bash
# 웹 빌드 후 Capacitor 동기화
npm run build
npx cap sync

# iOS/Android 빌드
npx cap open ios
npx cap open android
```

동일한 Next.js 코드베이스로 웹과 네이티브 앱을 모두 지원합니다.

---

### 추가 참고 자료

#### Next.js + Supabase

- [Next.js with Supabase](https://supabase.com/docs/guides/getting-started/quickstarts/nextjs)
- [Server Actions with Supabase](https://supabase.com/docs/guides/auth/server-side/nextjs)
- [Supabase Auth Helpers for Next.js](https://supabase.com/docs/guides/auth/auth-helpers/nextjs)

#### Supabase

- [Supabase Realtime Documentation](https://supabase.com/docs/guides/realtime)
- [Supabase Realtime Benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)
- [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security)
- [Exploring Supabase Realtime By Building a Game](https://www.aleksandra.codes/supabase-game)

#### SUI 블록체인 (Web3)

- [Sui dApp Kit 문서](https://sdk.mystenlabs.com/dapp-kit)
- [zkLogin 개념](https://docs.sui.io/concepts/cryptography/zklogin)
- [zkLogin 통합 가이드](https://docs.sui.io/guides/developer/cryptography/zklogin-integration)
- [@mysten/dapp-kit - npm](https://www.npmjs.com/package/@mysten/dapp-kit)
- [@mysten/sui - npm](https://www.npmjs.com/package/@mysten/sui)
- [Sui Gaming 문서](https://docs.sui.io/concepts/gaming)

#### Capacitor

- [Capacitor Official Docs](https://capacitorjs.com/docs)
- [Next.js + Capacitor Integration](https://capacitorjs.com/docs/getting-started/with-nextjs)
- [Supabase in Capacitor Apps](https://supabase.com/docs/guides/getting-started/tutorials/with-ionic-react)

#### 게임 아키텍처

- [Client-Server Game Architecture - Gabriel Gambetta](https://www.gabrielgambetta.com/client-server-game-architecture.html)
- [Phaser 3 Official Examples](https://phaser.io/examples)

#### AI 시스템

- [GameAIPro - Utility AI](http://www.gameaipro.com/GameAIPro/GameAIPro_Chapter09_An_Introduction_to_Utility_Theory.pdf)
- [BehaviorTree.js](https://github.com/Calamari/BehaviorTree.js)
- [Solving Ballistic Trajectories](https://www.forrestthewoods.com/blog/solving_ballistic_trajectories/)

#### 오픈소스 예제

- [Worms-Armageddon-HTML5-Clone](https://github.com/CiaranMcCann/Worms-Armageddon-HTML5-Clone)
- [MultiTurn Framework](https://github.com/nomoid/MultiTurn)

---

### 업데이트된 미해결 질문

이전 미해결 질문에 대한 답변:

| 질문                                     | 상태    | 답변 요약                                   |
| ---------------------------------------- | ------- | ------------------------------------------- |
| 멀티플레이어: 로컬 멀티만? 온라인도?     | ✅ 해결 | 온라인 멀티플레이어, Supabase Realtime 기반 |
| 서버 인프라: 어떤 기술 스택 사용?        | ✅ 해결 | Next.js + Supabase 단순화된 아키텍처        |
| **인증: 로그인 방식?**                   | ✅ 해결 | **SUI zkLogin (소셜) + 지갑 연결 (Web3)**   |
| 맵 에디터: 커스텀 맵 지원 여부           | ❓ 미정 | 추후 결정 필요                              |
| AI: 컴퓨터 플레이어 구현 여부            | ✅ 해결 | Utility AI 기반, Edge Functions에서 실행    |
| 저장/로드: 게임 진행 상태 저장 필요 여부 | ❓ 미정 | 추후 결정 필요                              |

새로운 미해결 질문:

1. **AI 난이도 밸런싱**: 실제 플레이테스트를 통한 AI 난이도 조정 필요
2. **Supabase 요금제**: 프로덕션 환경에서의 Supabase 요금제 선택 (Free vs Pro)
3. **음성 채팅**: 팀 게임에서 음성 채팅 지원 여부
4. **NFT 통합**: 게임 내 아이템/스킨을 NFT로 발행할지 여부
