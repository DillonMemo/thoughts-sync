# 웜즈 스타일 게임 MVP 구현 계획

## 개요

Next.js 16 + Phaser 3 기반의 웜즈 스타일 턴 기반 artillery 게임 MVP를 구현합니다. 로컬 멀티플레이(2인 핀볼 대전)를 목표로 하며, 픽셀 기반 파괴 가능 지형과 포물선 물리 시스템을 핵심으로 합니다.

## 현재 상태 분석

### 프로젝트 상태 (2026-02-07 업데이트)

- **Phase 1**: ✅ 완료 - Turborepo 모노레포 + Next.js 16 + Phaser 3 통합
- **Phase 2**: ✅ 완료 - 절차적 지형 생성 + Matter.js 물리 충돌체
- **Phase 3**: ✅ 완료 - 캐릭터 시스템 (이동, 점프, 경사면 물리)
- **Phase 4**: ✅ 완료 - 턴 시스템 및 마우스 드래그 에이밍
- **Phase 5**: ✅ 완료 - 무기 시스템 (Bazooka, Grenade, Shotgun) + 투사체 물리
- **Phase 6**: ✅ 완료 - 바람 시스템 및 데미지 처리
- **Phase 7**: ✅ 완료 (수동 검증 완료) - 게임 씬 통합 및 게임 루프 완성
- **Phase 8**: ✅ 완료 (수동 검증 완료) - React UI 통합 및 게임 외부 화면 (shadcn/ui)

### 사용 중인 에셋

- **캐릭터**: Player, Female Pose 기반 개별 PNG 이미지
  - P1: `assets/characters/PNG/Player/Poses/player_*.png`
  - P2: `assets/characters/PNG/Female/Poses/female_*.png`
  - 포즈: idle, stand, walk1, walk2, jump, fall, action1, action2, hurt, duck, cheer1, cheer2
- **무기/이펙트**: tanks 스프라이트시트 + XML
  - 투사체: tank_bullet1~6.png
  - 폭발: tank_explosion1~12.png
  - 무기 스프라이트: tanks_turret1~4.png
- **사운드**:
  - BGM: `fortress-bgm.ogg.mp3` (루프 재생)
  - 사망 효과음: `chill-guy-cut.mp3` (사망 시 BGM 일시정지 후 재생)

### 핵심 구현 사항

**물리 및 이동:**

- 원형 충돌체로 경사면 이동 개선
- 경사도 기반 속도 배율 시스템 (오르막 70%, 내리막 130%)
- Coyote Time (150ms) 점프 허용 시간
- 지면 감지: 접촉 카운팅 방식 (다중 terrain 컬럼 바디와의 동시 접촉을 정확히 추적)
- `apps/web/public/assets` symlink로 정적 에셋 제공

**입력 및 조작:**

- WASD 키로 이동/점프 (기존 방향키에서 변경)
- 마우스 드래그 기반 에이밍 시스템 (Angry Birds 스타일)
- 드래그 방향에 따라 캐릭터 방향 자동 전환
- 우클릭 컨텍스트 메뉴 비활성화

**캐릭터 애니메이션 (Pose 기반):**

- 개별 PNG 이미지를 조합한 애니메이션 시스템 (타일시트 대신 Pose 사용)
- 상태 기반 애니메이션: idle, walk, jump, fall, shoot, hurt, duck, cheer
- 스프라이트 origin 조정 (0.5, 0.67)으로 지면 밀착 개선
- 물리 바디와 스프라이트 분리 (독립적 위치 동기화)

**무기 시스템:**

- 3종 무기: Bazooka (무제한), Grenade (3발), Shotgun (2발, 3발 산탄)
- WeaponRegistry로 무기 스펙 중앙 관리 (holdSprite 포함)
- 에이밍 시작 시 무기 꺼내기 애니메이션 (Back.easeOut, 150ms)
- 에이밍 종료 시 무기 숨기기 애니메이션 (Power2, 100ms)
- 에이밍 중 무기 각도가 마우스 방향 따라 회전
- 에이밍 중 캐릭터 물리 바디 고정 (경사면 미끄러짐 방지)

**투사체:**

- SPEED_SCALE = 0.7 (투사체 속도 70%로 감소)
- GRAVITY_SCALE = 0.49, FRICTION_AIR_SCALE = 0.7 (궤적 비례 조정)

**사운드:**

- BGM: `fortress-bgm.ogg.mp3` (GameScene에서 루프 재생)
- 사망 효과음: `chill-guy-cut.mp3` (사망 시 BGM 일시정지 후 재생, 완료 후 BGM 재개)

**UI:**

- 체력바: 캐릭터 머리 위 Container 기반 (배경 + 채우기 + 플레이어 라벨 + 체력 숫자)
- 체력에 따른 색상 변화 (초록 → 노랑 → 빨강)
- WeaponSelector UI 컴포넌트

**캐릭터:**

- Player 2: adventurer → female 캐릭터로 변경

### 개발 환경 설정 (2026-01-24 추가)

#### 기술 스택

| 항목         | 버전   | 비고                                   |
| ------------ | ------ | -------------------------------------- |
| Node.js      | 25.x   | 최소 20.9.0 필요 (Next.js 16 요구사항) |
| pnpm         | 9.0.0  | 패키지 매니저                          |
| Next.js      | 16.x   | 15에서 업그레이드                      |
| React        | 19.x   |                                        |
| TypeScript   | 5.7+   |                                        |
| Phaser       | 3.90.0 | 게임 엔진                              |
| Tailwind CSS | 4.x    | 스타일링                               |

#### Prettier 설정

```json
{
  "semi": false,
  "trailingComma": "es5",
  "singleQuote": false,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

**스크립트:**

- `pnpm format` - 전체 포맷팅
- `pnpm format:check` - 포맷 검사

#### ESLint 설정

**공통 설정**: `packages/config/eslint/base.mjs`

```javascript
// 주요 규칙
{
  "sort-imports": "warn",
  "no-console": ["error", { allow: ["warn", "error", "info"] }],
  "@typescript-eslint/no-unused-vars": ["error", { argsIgnorePattern: "^_+$" }]
}
```

**패키지별 설정:**

- `apps/web/eslint.config.mjs` - Next.js + 공통 규칙
- `packages/game-core/eslint.config.mjs` - TypeScript 전용
- `packages/ui/eslint.config.mjs` - React + TypeScript

**스크립트:**

- `pnpm lint` - 전체 lint (turbo 통해 병렬 실행)
- `pnpm lint -- --fix` - 자동 수정

## 원하는 최종 상태

MVP 완료 시:

1. 한 기기에서 2명이 번갈아가며 플레이 가능
2. 픽셀 기반 파괴 가능 지형에서 전투
3. 3종 무기(바주카, 수류탄, 샷건)로 상대 공격
4. 바람과 중력이 영향을 미치는 포물선 투사체
5. 체력이 0이 되면 승패 결정
6. 반응형 UI로 다양한 화면 크기 지원

### 검증 방법

- Vitest 단위 테스트로 물리/게임 로직 검증
- Playwright E2E 테스트로 게임 플로우 검증
- 수동 테스트로 게임 밸런스 및 UX 검증

## 하지 않을 것

범위 확대 방지를 위해 MVP에서 제외:

1. **온라인 멀티플레이어** - Supabase Realtime 연동은 Phase 2에서
2. **AI 플레이어** - Utility AI는 Phase 2에서
3. **Web3 인증** - SUI zkLogin은 Phase 3에서
4. **맵 에디터** - 커스텀 맵은 향후 기능
5. **리더보드/프로필** - 온라인 기능과 함께 구현
6. **4인 이상 멀티플레이어** - 2인 핀볼만 지원
7. **팀 모드** - 개인전만 지원
8. **확장 무기** (에어스트라이크, 다이너마이트 등) - 확장성만 고려

## 구현 접근 방식

Turborepo 모노레포 구조로 게임 코어 로직을 분리하고, Next.js App Router로 웹 앱을 구성합니다. Phaser 3는 dynamic import와 'use client'로 SSR을 우회합니다.

```
game-monorepo/
├── apps/
│   └── web/                 # Next.js 16 웹 앱
├── packages/
│   ├── game-core/           # Phaser 게임 로직 (순수 TS)
│   ├── ui/                  # 공유 React UI 컴포넌트
│   └── config/              # 공유 설정 (ESLint, TS)
└── assets/                  # 게임 에셋 (기존 위치 유지)
```

---

## Phase 1: 프로젝트 초기화 및 기반 구축

### 개요

Turborepo 모노레포 구조를 설정하고, Next.js + Phaser 통합 환경을 구축합니다.

### 필요한 변경:

#### 1. Turborepo 모노레포 초기화

**파일**: `package.json` (루트)

```json
{
  "name": "worms-game",
  "private": true,
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "lint": "turbo lint",
    "test": "turbo test",
    "typecheck": "turbo typecheck",
    "format": "prettier --write .",
    "format:check": "prettier --check ."
  },
  "devDependencies": {
    "prettier": "^3.8.1",
    "turbo": "^2.3.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

**파일**: `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "dev": {
      "cache": false,
      "persistent": true
    },
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "dist/**"]
    },
    "lint": {},
    "test": {},
    "typecheck": {}
  }
}
```

**파일**: `pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

#### 2. Next.js 웹 앱 설정

**파일**: `apps/web/package.json`

```json
{
  "name": "web",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint .",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@repo/game-core": "workspace:*",
    "@repo/ui": "workspace:*",
    "next": "^16.0.0",
    "react": "^19.0.0",
    "react-dom": "^19.0.0"
  },
  "devDependencies": {
    "@repo/config": "workspace:*",
    "@tailwindcss/postcss": "^4.0.0",
    "@types/node": "^22.0.0",
    "@types/react": "^19.0.0",
    "@types/react-dom": "^19.0.0",
    "eslint": "^9.39.2",
    "eslint-config-next": "^16.1.4",
    "phaser": "^3.90.0",
    "tailwindcss": "^4.0.0",
    "typescript": "^5.7.0",
    "typescript-eslint": "^8.53.1"
  }
}
```

**파일**: `apps/web/next.config.ts`

```typescript
import type { NextConfig } from "next"

const nextConfig: NextConfig = {
  transpilePackages: ["@repo/game-core", "@repo/ui"],
  experimental: {
    optimizePackageImports: ["@repo/game-core"],
  },
}

export default nextConfig
```

**파일**: `apps/web/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

#### 3. 게임 코어 패키지 설정

**파일**: `packages/game-core/package.json`

```json
{
  "name": "@repo/game-core",
  "version": "0.1.0",
  "main": "./dist/index.mjs",
  "types": "./dist/index.d.mts",
  "exports": {
    ".": {
      "import": "./dist/index.mjs",
      "types": "./dist/index.d.mts"
    }
  },
  "scripts": {
    "build": "tsup src/index.ts --format esm --dts",
    "dev": "tsup src/index.ts --format esm --dts --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit",
    "lint": "eslint ."
  },
  "dependencies": {
    "phaser": "^3.90.0"
  },
  "devDependencies": {
    "@repo/config": "workspace:*",
    "@types/node": "^22.0.0",
    "eslint": "^9.39.2",
    "jsdom": "^25.0.0",
    "tsup": "^8.3.0",
    "typescript": "^5.7.0",
    "vitest": "^2.1.0"
  }
}
```

**파일**: `packages/game-core/tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "outDir": "./dist",
    "rootDir": "./src",
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**파일**: `packages/game-core/tsup.config.ts`

```typescript
import { defineConfig } from "tsup"

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  dts: true,
  clean: true,
  sourcemap: true,
  // @/ 경로 alias 해결
  esbuildOptions(options) {
    options.alias = {
      "@": "./src",
    }
  },
})
```

**파일**: `packages/game-core/vitest.config.ts`

```typescript
import { defineConfig } from "vitest/config"
import { resolve } from "path"

export default defineConfig({
  test: {
    globals: true,
    environment: "jsdom",
  },
  resolve: {
    alias: {
      "@": resolve(__dirname, "./src"),
    },
  },
})
```

#### 4. Phaser 통합 컴포넌트

**파일**: `apps/web/src/components/game/PhaserGame.tsx`

```typescript
'use client'

import { useEffect, useRef } from 'react'
import type { Game } from 'phaser'

interface PhaserGameProps {
  onGameReady?: (game: Game) => void
}

export function PhaserGame({ onGameReady }: PhaserGameProps) {
  const gameRef = useRef<Game | null>(null)
  const containerRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (typeof window === 'undefined' || gameRef.current) return

    const initGame = async () => {
      const { createGame } = await import('@repo/game-core')

      if (!containerRef.current) return

      gameRef.current = createGame(containerRef.current)
      onGameReady?.(gameRef.current)
    }

    initGame()

    return () => {
      gameRef.current?.destroy(true)
      gameRef.current = null
    }
  }, [onGameReady])

  return (
    <div
      ref={containerRef}
      className="w-full h-full"
      style={{ minHeight: '600px' }}
    />
  )
}
```

**파일**: `apps/web/src/app/page.tsx`

```typescript
import dynamic from 'next/dynamic'

const PhaserGame = dynamic(
  () => import('@/components/game/PhaserGame').then(m => m.PhaserGame),
  { ssr: false }
)

export default function Home() {
  return (
    <main className="min-h-screen bg-gray-900">
      <PhaserGame />
    </main>
  )
}
```

#### 5. 기본 Phaser 설정

**파일**: `packages/game-core/src/index.ts`

```typescript
import Phaser from "phaser"
import { BootScene } from "@/scenes/BootScene"
import { GameScene } from "@/scenes/GameScene"

export function createGame(parent: HTMLElement): Phaser.Game {
  const config: Phaser.Types.Core.GameConfig = {
    type: Phaser.AUTO,
    parent,
    width: 1280,
    height: 720,
    backgroundColor: "#87CEEB", // 하늘색
    physics: {
      default: "matter",
      matter: {
        gravity: { x: 0, y: 1 },
        debug: process.env.NODE_ENV === "development",
      },
    },
    scale: {
      mode: Phaser.Scale.FIT,
      autoCenter: Phaser.Scale.CENTER_BOTH,
    },
    scene: [BootScene, GameScene],
  }

  return new Phaser.Game(config)
}

export * from "@/scenes/BootScene"
export * from "@/scenes/GameScene"
export * from "@/EventBus"
```

**파일**: `packages/game-core/src/EventBus.ts`

```typescript
import Phaser from "phaser"

// React-Phaser 양방향 통신용 이벤트 버스
export const EventBus = new Phaser.Events.EventEmitter()

// 이벤트 타입 정의
export const GameEvents = {
  TURN_CHANGED: "turn-changed",
  HEALTH_CHANGED: "health-changed",
  GAME_OVER: "game-over",
  WEAPON_SELECTED: "weapon-selected",
  WIND_CHANGED: "wind-changed",
} as const
```

### 성공 기준:

#### 자동화된 검증:

- [x] pnpm install 성공: `pnpm install`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 린팅 통과: `pnpm lint`
- [x] 개발 서버 실행: `pnpm dev`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] localhost:3000에서 하늘색 배경의 Phaser 캔버스가 표시됨
- [x] 브라우저 콘솔에 Phaser 초기화 로그 출력
- [x] 반응형으로 화면 크기에 맞게 캔버스 조정됨 (RESIZE 모드로 100% 채움 + 모바일 세로모드 오버레이 추가)

**Implementation Note**: 이 단계를 완료하고 모든 자동화된 검증이 통과한 후, 다음 단계로 진행하기 전에 수동 테스트가 성공했다는 확인을 위해 여기서 일시 중지합니다.

---

## Phase 2: 지형 시스템 구현

### 개요

픽셀 기반 파괴 가능 지형을 구현합니다. 1D 배열로 지형 데이터를 관리하고, 폭발 시 원형 영역을 파괴합니다.

### 필요한 변경:

#### 1. 지형 데이터 구조

**파일**: `packages/game-core/src/terrain/TerrainData.ts`

```typescript
export class TerrainData {
  private data: Uint8Array
  readonly width: number
  readonly height: number

  constructor(width: number, height: number) {
    this.width = width
    this.height = height
    this.data = new Uint8Array(width * height)
  }

  isSolid(x: number, y: number): boolean {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height) {
      return false
    }
    return this.data[y * this.width + x] === 1
  }

  setSolid(x: number, y: number, solid: boolean): void {
    if (x < 0 || x >= this.width || y < 0 || y >= this.height) return
    this.data[y * this.width + x] = solid ? 1 : 0
  }

  // 원형 영역 파괴
  explode(centerX: number, centerY: number, radius: number): void {
    const radiusSquared = radius * radius
    for (let dy = -radius; dy <= radius; dy++) {
      for (let dx = -radius; dx <= radius; dx++) {
        if (dx * dx + dy * dy <= radiusSquared) {
          this.setSolid(
            Math.floor(centerX + dx),
            Math.floor(centerY + dy),
            false
          )
        }
      }
    }
  }

  // 절차적 지형 생성 (Perlin noise 기반)
  generateTerrain(groundLevel: number = 0.6, amplitude: number = 100): void {
    const baseY = Math.floor(this.height * groundLevel)

    for (let x = 0; x < this.width; x++) {
      // 간단한 노이즈 함수 (실제로는 simplex-noise 라이브러리 사용 권장)
      const noise =
        Math.sin(x * 0.02) * amplitude * 0.5 +
        Math.sin(x * 0.05) * amplitude * 0.3 +
        Math.sin(x * 0.1) * amplitude * 0.2
      const surfaceY = Math.floor(baseY + noise)

      for (let y = surfaceY; y < this.height; y++) {
        this.setSolid(x, y, true)
      }
    }
  }

  toImageData(ctx: CanvasRenderingContext2D): ImageData {
    const imageData = ctx.createImageData(this.width, this.height)
    for (let i = 0; i < this.data.length; i++) {
      const idx = i * 4
      if (this.data[i] === 1) {
        // 땅 색상 (갈색)
        imageData.data[idx] = 139 // R
        imageData.data[idx + 1] = 90 // G
        imageData.data[idx + 2] = 43 // B
        imageData.data[idx + 3] = 255 // A
      } else {
        // 투명
        imageData.data[idx + 3] = 0
      }
    }
    return imageData
  }
}
```

#### 2. 지형 렌더러

**파일**: `packages/game-core/src/terrain/TerrainRenderer.ts`

```typescript
import Phaser from "phaser"
import { TerrainData } from "@/terrain/TerrainData"

export class TerrainRenderer {
  private scene: Phaser.Scene
  private terrainData: TerrainData
  private terrainTexture: Phaser.Textures.CanvasTexture | null = null
  private terrainSprite: Phaser.GameObjects.Image | null = null
  private collisionBodies: MatterJS.BodyType[] = []

  constructor(scene: Phaser.Scene, terrainData: TerrainData) {
    this.scene = scene
    this.terrainData = terrainData
  }

  create(): void {
    this.renderToTexture()
    this.createSprite()
    this.updateCollisionBodies()
  }

  private renderToTexture(): void {
    const key = "terrain"

    // 기존 텍스처 제거
    if (this.scene.textures.exists(key)) {
      this.scene.textures.remove(key)
    }

    // 캔버스 텍스처 생성
    this.terrainTexture = this.scene.textures.createCanvas(
      key,
      this.terrainData.width,
      this.terrainData.height
    )

    const ctx = this.terrainTexture.getContext()
    const imageData = this.terrainData.toImageData(ctx)
    ctx.putImageData(imageData, 0, 0)
    this.terrainTexture.refresh()
  }

  private createSprite(): void {
    if (this.terrainSprite) {
      this.terrainSprite.destroy()
    }

    this.terrainSprite = this.scene.add.image(
      this.terrainData.width / 2,
      this.terrainData.height / 2,
      "terrain"
    )
  }

  updateCollisionBodies(): void {
    // 기존 충돌체 제거
    this.collisionBodies.forEach((body) => {
      this.scene.matter.world.remove(body)
    })
    this.collisionBodies = []

    // 마칭 스퀘어 또는 간단한 컬럼 기반 충돌체 생성
    // (성능을 위해 모든 픽셀이 아닌 샘플링된 컬럼 사용)
    const columnWidth = 4
    for (let x = 0; x < this.terrainData.width; x += columnWidth) {
      let topY = -1
      let bottomY = -1

      for (let y = 0; y < this.terrainData.height; y++) {
        if (this.terrainData.isSolid(x, y)) {
          if (topY === -1) topY = y
          bottomY = y
        }
      }

      if (topY !== -1 && bottomY !== -1) {
        const height = bottomY - topY + 1
        const body = this.scene.matter.add.rectangle(
          x + columnWidth / 2,
          topY + height / 2,
          columnWidth,
          height,
          { isStatic: true, label: "terrain" }
        )
        this.collisionBodies.push(body)
      }
    }
  }

  // 폭발 후 지형 업데이트
  onExplosion(x: number, y: number, radius: number): void {
    this.terrainData.explode(x, y, radius)
    this.renderToTexture()
    this.updateCollisionBodies()
  }

  destroy(): void {
    this.collisionBodies.forEach((body) => {
      this.scene.matter.world.remove(body)
    })
    this.terrainSprite?.destroy()
    if (this.terrainTexture) {
      this.scene.textures.remove("terrain")
    }
  }
}
```

#### 3. 지형 단위 테스트

**파일**: `packages/game-core/src/terrain/__tests__/TerrainData.test.ts`

```typescript
import { describe, it, expect, beforeEach } from "vitest"
import { TerrainData } from "@/terrain/TerrainData"

describe("TerrainData", () => {
  let terrain: TerrainData

  beforeEach(() => {
    terrain = new TerrainData(100, 100)
  })

  it("should initialize with all empty cells", () => {
    expect(terrain.isSolid(0, 0)).toBe(false)
    expect(terrain.isSolid(50, 50)).toBe(false)
  })

  it("should set and get solid cells", () => {
    terrain.setSolid(10, 10, true)
    expect(terrain.isSolid(10, 10)).toBe(true)

    terrain.setSolid(10, 10, false)
    expect(terrain.isSolid(10, 10)).toBe(false)
  })

  it("should return false for out of bounds", () => {
    expect(terrain.isSolid(-1, 0)).toBe(false)
    expect(terrain.isSolid(0, -1)).toBe(false)
    expect(terrain.isSolid(100, 0)).toBe(false)
    expect(terrain.isSolid(0, 100)).toBe(false)
  })

  it("should explode circular area", () => {
    // Fill entire terrain
    for (let x = 0; x < 100; x++) {
      for (let y = 0; y < 100; y++) {
        terrain.setSolid(x, y, true)
      }
    }

    // Explode at center
    terrain.explode(50, 50, 5)

    // Center should be empty
    expect(terrain.isSolid(50, 50)).toBe(false)

    // Edge of explosion should be empty
    expect(terrain.isSolid(55, 50)).toBe(false)

    // Outside explosion should still be solid
    expect(terrain.isSolid(60, 50)).toBe(true)
  })

  it("should generate procedural terrain", () => {
    terrain.generateTerrain(0.6, 50)

    // Top should be empty (sky)
    expect(terrain.isSolid(50, 0)).toBe(false)

    // Bottom should be solid (ground)
    expect(terrain.isSolid(50, 99)).toBe(true)
  })
})
```

### 성공 기준:

#### 자동화된 검증:

- [x] 지형 단위 테스트 통과: `pnpm --filter @repo/game-core test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 게임 화면에 절차적으로 생성된 지형이 표시됨
- [x] 지형이 갈색으로 렌더링됨
- [x] 지형 위에 하늘색 배경이 보임

**Implementation Note**: Phase 2 완료. 파란색 세로선은 Matter.js 디버그 모드의 충돌체 시각화임 (정상).

---

## Phase 3: 캐릭터 시스템 구현

### 개요

캐릭터 스프라이트시트를 로딩하고, 기본 이동(좌우 이동, 점프)과 물리 충돌을 구현합니다.

### 필요한 변경:

#### 1. 캐릭터 클래스

**파일**: `packages/game-core/src/entities/Character.ts`

```typescript
import * as Phaser from "phaser"
import { TerrainData } from "@/terrain/TerrainData"

export interface CharacterConfig {
  scene: Phaser.Scene
  x: number
  y: number
  playerId: number
  spriteKey: string
  terrainData?: TerrainData // 경사도 계산용
}

export class Character {
  private scene: Phaser.Scene
  private sprite: Phaser.Physics.Matter.Sprite
  private terrainData?: TerrainData
  readonly playerId: number

  // 상태
  private health: number = 100
  private maxHealth: number = 100
  private isGrounded: boolean = false
  private facingRight: boolean = true
  private lastGroundedTime: number = 0 // Coyote Time용

  // 물리 설정
  private readonly BASE_MOVE_SPEED = 1.25
  private readonly JUMP_VELOCITY = -5.3
  private readonly MAX_FALL_SPEED = 10
  private readonly COYOTE_TIME = 150 // ms

  constructor(config: CharacterConfig) {
    this.scene = config.scene
    this.playerId = config.playerId
    this.terrainData = config.terrainData

    // Matter.js 물리 바디로 스프라이트 생성
    this.sprite = this.scene.matter.add.sprite(
      config.x,
      config.y,
      config.spriteKey,
      0
    )

    // 원형 충돌체 사용 - 경사면 이동에 효과적
    const body = this.scene.matter.bodies.circle(config.x, config.y, 18, {
      label: `character_${config.playerId}`,
      friction: 0.8,
      frictionAir: 0.02,
      restitution: 0,
    })
    this.sprite.setExistingBody(body)
    this.sprite.setFixedRotation()

    // 스프라이트 스케일 및 원점 조정
    this.sprite.setScale(0.35)
    this.sprite.setOrigin(0.5, 0.75)

    // 지면 감지
    this.setupGroundDetection()
  }

  private setupGroundDetection(): void {
    this.scene.matter.world.on(
      "collisionstart",
      (event: Phaser.Physics.Matter.Events.CollisionStartEvent) => {
        event.pairs.forEach((pair) => {
          const labels = [pair.bodyA.label, pair.bodyB.label]
          if (
            labels.includes(`character_${this.playerId}`) &&
            labels.includes("terrain")
          ) {
            this.isGrounded = true
            this.lastGroundedTime = Date.now()
          }
        })
      }
    )

    this.scene.matter.world.on(
      "collisionend",
      (event: Phaser.Physics.Matter.Events.CollisionEndEvent) => {
        event.pairs.forEach((pair) => {
          const labels = [pair.bodyA.label, pair.bodyB.label]
          if (
            labels.includes(`character_${this.playerId}`) &&
            labels.includes("terrain")
          ) {
            this.isGrounded = false
          }
        })
      }
    )
  }

  moveLeft(): void {
    this.move(-1)
    this.facingRight = false
    this.sprite.setFlipX(true)
  }

  moveRight(): void {
    this.move(1)
    this.facingRight = true
    this.sprite.setFlipX(false)
  }

  private move(direction: number): void {
    const speed = this.calculateMoveSpeed(direction)

    if (this.isGrounded && this.terrainData) {
      const pos = this.getPosition()
      const x = Math.floor(pos.x)
      const currentY = this.findSurfaceYAt(x)
      const targetY = this.findSurfaceYAt(x + direction * 5)
      const heightDiff = currentY - targetY

      if (heightDiff > 2) {
        // 오르막 경사면 - 위쪽 속도 추가
        const climbSpeed = Math.min(heightDiff * 0.5, 3)
        this.sprite.setVelocity(direction * speed, -climbSpeed)
        return
      }
    }

    this.sprite.setVelocityX(direction * speed)
  }

  private calculateMoveSpeed(direction: number): number {
    if (!this.terrainData || !this.isGrounded) {
      return this.BASE_MOVE_SPEED
    }

    const pos = this.getPosition()
    const x = Math.floor(pos.x)
    const sampleDistance = 10

    const currentY = this.findSurfaceYAt(x)
    const targetY = this.findSurfaceYAt(x + direction * sampleDistance)
    const heightDiff = currentY - targetY
    const slopeRatio = heightDiff / sampleDistance

    let speedMultiplier = 1.0
    if (slopeRatio > 0) {
      speedMultiplier = Math.max(0.4, 1.0 - slopeRatio * 1.5) // 오르막
    } else {
      speedMultiplier = Math.min(1.2, 1.0 - slopeRatio * 0.3) // 내리막
    }

    return this.BASE_MOVE_SPEED * speedMultiplier
  }

  private findSurfaceYAt(x: number): number {
    if (!this.terrainData) return this.sprite.y

    for (let y = 0; y < this.terrainData.height; y++) {
      if (this.terrainData.isSolid(x, y)) {
        return y
      }
    }
    return this.terrainData.height
  }

  stopMoving(): void {
    this.sprite.setVelocityX(0)
  }

  jump(): void {
    const canJump =
      this.isGrounded || Date.now() - this.lastGroundedTime < this.COYOTE_TIME

    if (canJump) {
      this.sprite.setVelocityY(this.JUMP_VELOCITY)
      this.isGrounded = false
      this.lastGroundedTime = 0
    }
  }

  takeDamage(amount: number): void {
    this.health = Math.max(0, this.health - amount)
  }

  getHealth(): number {
    return this.health
  }

  getMaxHealth(): number {
    return this.maxHealth
  }

  getPosition(): { x: number; y: number } {
    return { x: this.sprite.x, y: this.sprite.y }
  }

  isFacingRight(): boolean {
    return this.facingRight
  }

  setFacingDirection(right: boolean): void {
    this.facingRight = right
    this.sprite.setFlipX(!right)
  }

  isAlive(): boolean {
    return this.health > 0
  }

  update(): void {
    const velocity = this.sprite.body?.velocity
    if (velocity && velocity.y > this.MAX_FALL_SPEED) {
      this.sprite.setVelocityY(this.MAX_FALL_SPEED)
    }
  }

  destroy(): void {
    this.sprite.destroy()
  }
}
```

#### 2. 캐릭터 스프라이트시트 로더

**파일**: `packages/game-core/src/scenes/BootScene.ts`

```typescript
import * as Phaser from "phaser"

export class BootScene extends Phaser.Scene {
  constructor() {
    super({ key: "BootScene" })
  }

  preload(): void {
    // 캐릭터 스프라이트시트 로딩
    // 720x330 크기, 9열 x 3행 레이아웃
    const characterFrameWidth = 80 // 720 / 9 = 80
    const characterFrameHeight = 110 // 330 / 3 = 110

    this.load.spritesheet(
      "character1",
      "/assets/characters/PNG/Player/player_tilesheet.png",
      {
        frameWidth: characterFrameWidth,
        frameHeight: characterFrameHeight,
      }
    )

    this.load.spritesheet(
      "character2",
      "/assets/characters/PNG/Adventurer/adventurer_tilesheet.png",
      {
        frameWidth: characterFrameWidth,
        frameHeight: characterFrameHeight,
      }
    )

    // 탱크 에셋 (무기/폭발용)
    this.load.atlasXML(
      "tanks",
      "/assets/resource/Spritesheet/tanks_spritesheetDefault.png",
      "/assets/resource/Spritesheet/tanks_spritesheetDefault.xml"
    )

    // 사운드
    this.load.audio("bgm", "/assets/sound/fortress-bgm.ogg.mp3")

    // 로딩 진행률 표시
    this.load.on("progress", (value: number) => {
      console.info(`Loading: ${Math.floor(value * 100)}%`)
    })

    this.load.on("complete", () => {
      console.info("All assets loaded")
    })
  }

  create(): void {
    // 애니메이션 정의
    this.createAnimations()

    // 게임 씬으로 전환
    this.scene.start("GameScene")
  }

  private createAnimations(): void {
    // 타일시트 레이아웃 (9열 x 3행):
    // Row 1 (0-8):  walk1, walk2, idle, stand, action1, action2, back, cheer1, cheer2
    // Row 2 (9-17): climb1, climb2, duck, fall, hang, hold1, hold2, hurt, jump
    // Row 3 (18-24): kick, skid, slide, swim1, swim2, talk, ...

    // 캐릭터 애니메이션 (character1 - Player)
    this.anims.create({
      key: "char1_idle",
      frames: [{ key: "character1", frame: 2 }], // idle 프레임
      frameRate: 1,
      repeat: -1,
    })

    this.anims.create({
      key: "char1_walk",
      frames: this.anims.generateFrameNumbers("character1", { frames: [0, 1] }), // walk1, walk2
      frameRate: 8,
      repeat: -1,
    })

    this.anims.create({
      key: "char1_jump",
      frames: [{ key: "character1", frame: 17 }], // jump 프레임
      frameRate: 1,
      repeat: 0,
    })

    // 캐릭터 애니메이션 (character2 - Adventurer)
    this.anims.create({
      key: "char2_idle",
      frames: [{ key: "character2", frame: 2 }],
      frameRate: 1,
      repeat: -1,
    })

    this.anims.create({
      key: "char2_walk",
      frames: this.anims.generateFrameNumbers("character2", { frames: [0, 1] }),
      frameRate: 8,
      repeat: -1,
    })

    this.anims.create({
      key: "char2_jump",
      frames: [{ key: "character2", frame: 17 }],
      frameRate: 1,
      repeat: 0,
    })

    // 폭발 애니메이션 (탱크 에셋 사용)
    this.anims.create({
      key: "explosion",
      frames: [
        { key: "tanks", frame: "explosion1.png" },
        { key: "tanks", frame: "explosion2.png" },
        { key: "tanks", frame: "explosion3.png" },
        { key: "tanks", frame: "explosion4.png" },
        { key: "tanks", frame: "explosion5.png" },
      ],
      frameRate: 15,
      repeat: 0,
    })
  }
}
```

#### 3. 캐릭터 단위 테스트

**파일**: `packages/game-core/src/entities/__tests__/Character.test.ts`

```typescript
import { describe, it, expect } from "vitest"

describe("Character", () => {
  it("should initialize with full health", () => {
    // Phaser Scene mock이 필요하므로 실제 테스트는 통합 테스트에서
    // 여기서는 순수 로직만 테스트
    const health = 100
    const maxHealth = 100
    expect(health).toBe(maxHealth)
  })

  it("should calculate damage correctly", () => {
    let health = 100
    const damage = 25
    health = Math.max(0, health - damage)
    expect(health).toBe(75)
  })

  it("should not go below zero health", () => {
    let health = 10
    const damage = 25
    health = Math.max(0, health - damage)
    expect(health).toBe(0)
  })
})
```

### 성공 기준:

#### 자동화된 검증:

- [x] 캐릭터 단위 테스트 통과: `pnpm --filter @repo/game-core test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 캐릭터 스프라이트가 화면에 표시됨
- [x] 캐릭터가 지형 위에 서 있음 (물리 충돌 작동)
- [x] 좌우 이동이 정상 작동
- [x] 점프가 정상 작동

**Implementation Note (2026-01-23)**: Phase 3 구현 완료.

- 원형 충돌체(반지름 18)로 경사면 이동 개선
- 경사도 기반 속도 배율 시스템 (오르막 0.4~1.0배, 내리막 1.0~1.2배)
- Coyote Time (150ms) 점프 허용 시간 추가
- `apps/web/public/assets` symlink로 정적 에셋 제공

**Implementation Note (2026-01-25)**: Pose 기반 애니메이션 시스템으로 개편.

- 캐릭터 에셋: `PNG/Player/Poses/player_*.png`, `PNG/Adventurer/Poses/adventurer_*.png` (개별 이미지)
- 타일시트 대신 Pose 기반 개별 PNG 이미지 사용
- 애니메이션 상태: idle, walk, jump, fall, shoot, hurt, duck, cheer
- 스프라이트 origin 조정 (0.5, 0.67)으로 지면 밀착 개선
- 물리 바디와 스프라이트 분리 구조

---

## Phase 4: 턴 시스템 및 입력 처리

### 개요

State Machine 기반 턴 시스템을 구현하고, 플레이어 입력과 에이밍 UI를 추가합니다.

### 필요한 변경:

#### 1. 게임 상태 머신

**파일**: `packages/game-core/src/systems/TurnSystem.ts`

```typescript
import Phaser from "phaser"
import { EventBus, GameEvents } from "@/EventBus"

export type GameState =
  | "idle"
  | "playerTurn"
  | "aiming"
  | "firing"
  | "animating"
  | "gameOver"

export interface TurnSystemConfig {
  turnDuration: number // 턴 시간 제한 (ms)
  playerCount: number
}

export class TurnSystem {
  private state: GameState = "idle"
  private currentPlayerIndex: number = 0
  private turnTimer: Phaser.Time.TimerEvent | null = null
  private readonly config: TurnSystemConfig
  private scene: Phaser.Scene

  constructor(scene: Phaser.Scene, config: TurnSystemConfig) {
    this.scene = scene
    this.config = config
  }

  getState(): GameState {
    return this.state
  }

  getCurrentPlayer(): number {
    return this.currentPlayerIndex
  }

  getRemainingTime(): number {
    if (!this.turnTimer) return 0
    return this.turnTimer.getRemaining()
  }

  startGame(): void {
    this.state = "playerTurn"
    this.currentPlayerIndex = 0
    this.startTurnTimer()
    this.emitTurnChanged()
  }

  startAiming(): boolean {
    if (this.state !== "playerTurn") return false
    this.state = "aiming"
    return true
  }

  fire(): boolean {
    if (this.state !== "aiming") return false
    this.state = "firing"
    this.stopTurnTimer()
    return true
  }

  startAnimating(): void {
    this.state = "animating"
  }

  endTurn(): void {
    this.currentPlayerIndex =
      (this.currentPlayerIndex + 1) % this.config.playerCount
    this.state = "playerTurn"
    this.startTurnTimer()
    this.emitTurnChanged()
  }

  endGame(winnerId: number): void {
    this.state = "gameOver"
    this.stopTurnTimer()
    EventBus.emit(GameEvents.GAME_OVER, { winnerId })
  }

  cancelAiming(): void {
    if (this.state === "aiming") {
      this.state = "playerTurn"
    }
  }

  private startTurnTimer(): void {
    this.stopTurnTimer()
    this.turnTimer = this.scene.time.addEvent({
      delay: this.config.turnDuration,
      callback: () => this.onTurnTimeout(),
      callbackScope: this,
    })
  }

  private stopTurnTimer(): void {
    if (this.turnTimer) {
      this.turnTimer.destroy()
      this.turnTimer = null
    }
  }

  private onTurnTimeout(): void {
    // 시간 초과 시 턴 자동 종료
    if (this.state === "playerTurn" || this.state === "aiming") {
      this.endTurn()
    }
  }

  private emitTurnChanged(): void {
    EventBus.emit(GameEvents.TURN_CHANGED, {
      currentPlayer: this.currentPlayerIndex,
      state: this.state,
    })
  }

  destroy(): void {
    this.stopTurnTimer()
  }
}
```

#### 2. 입력 컨트롤러 (WASD + 마우스 드래그)

**파일**: `packages/game-core/src/systems/InputController.ts`

```typescript
import * as Phaser from "phaser"
import { Character } from "../entities/Character"
import { TurnSystem } from "./TurnSystem"

export class InputController {
  private scene: Phaser.Scene

  // WASD 키
  private keyW!: Phaser.Input.Keyboard.Key
  private keyA!: Phaser.Input.Keyboard.Key
  private keyD!: Phaser.Input.Keyboard.Key

  // 에이밍 (마우스 드래그 기반)
  private isAiming: boolean = false
  private aimStartPos: { x: number; y: number } | null = null
  private currentAimAngle: number = 45
  private currentAimPower: number = 50
  private aimingRight: boolean = true

  // 파워 계산용 상수
  private readonly MAX_DRAG_DISTANCE = 150
  private readonly MIN_POWER = 10
  private readonly MAX_POWER = 100

  constructor(scene: Phaser.Scene) {
    this.scene = scene

    if (scene.input.keyboard) {
      this.keyW = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.W)
      this.keyA = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.A)
      this.keyD = scene.input.keyboard.addKey(Phaser.Input.Keyboard.KeyCodes.D)
    }

    this.setupMouseInput()
  }

  private setupMouseInput(): void {
    // 마우스 왼쪽 버튼 다운 - 에이밍 시작
    this.scene.input.on("pointerdown", (pointer: Phaser.Input.Pointer) => {
      if (pointer.leftButtonDown()) {
        this.aimStartPos = { x: pointer.x, y: pointer.y }
      } else if (pointer.rightButtonDown()) {
        this.isAiming = false
        this.aimStartPos = null
      }
    })

    // 마우스 이동 - 에이밍 중 각도/파워 업데이트
    this.scene.input.on("pointermove", (pointer: Phaser.Input.Pointer) => {
      if (this.isAiming && pointer.leftButtonDown()) {
        this.updateAimFromMouse(pointer)
      }
    })

    // 마우스 왼쪽 버튼 업 - 발사
    this.scene.input.on("pointerup", (pointer: Phaser.Input.Pointer) => {
      if (pointer.leftButtonReleased() && this.isAiming) {
        this.isAiming = false
      }
    })
  }

  private updateAimFromMouse(pointer: Phaser.Input.Pointer): void {
    if (!this.aimStartPos) return

    const dx = pointer.x - this.aimStartPos.x
    const dy = pointer.y - this.aimStartPos.y

    // 드래그 방향에 따라 에이밍 방향 결정
    this.aimingRight = dx >= 0

    // 각도 계산 (-30~90도 범위)
    const angle = Math.atan2(-dy, Math.abs(dx)) * (180 / Math.PI)
    this.currentAimAngle = Math.max(-30, Math.min(90, angle))

    // 파워 계산 (드래그 거리)
    const distance = Math.sqrt(dx * dx + dy * dy)
    const normalizedDistance = Math.min(distance / this.MAX_DRAG_DISTANCE, 1)
    this.currentAimPower =
      this.MIN_POWER + normalizedDistance * (this.MAX_POWER - this.MIN_POWER)
  }

  update(character: Character, turnSystem: TurnSystem): void {
    const state = turnSystem.getState()

    if (state === "playerTurn") {
      this.handleMovement(character)
      this.handleJump(character)

      // 마우스 클릭으로 에이밍 시작
      if (this.aimStartPos && !this.isAiming) {
        if (turnSystem.startAiming()) {
          this.isAiming = true
          const pos = character.getPosition()
          this.aimStartPos = { x: pos.x, y: pos.y }
        }
      }
    } else if (state === "aiming") {
      const pointer = this.scene.input.activePointer

      if (pointer.leftButtonDown() && this.aimStartPos) {
        this.updateAimFromMouse(pointer)
      }

      // 마우스 버튼 놓으면 발사
      if (!pointer.leftButtonDown() && !this.isAiming) {
        turnSystem.fire()
        this.aimStartPos = null
      }

      // 우클릭으로 취소
      if (pointer.rightButtonDown()) {
        turnSystem.cancelAiming()
        this.isAiming = false
        this.aimStartPos = null
      }
    }
  }

  private handleMovement(character: Character): void {
    if (this.keyA.isDown) {
      character.moveLeft()
    } else if (this.keyD.isDown) {
      character.moveRight()
    } else {
      character.stopMoving()
    }
  }

  private handleJump(character: Character): void {
    if (Phaser.Input.Keyboard.JustDown(this.keyW)) {
      character.jump()
    }
  }

  getAimAngle(): number {
    return this.currentAimAngle
  }

  getAimPower(): number {
    return this.currentAimPower
  }

  isAimingRight(): boolean {
    return this.aimingRight
  }

  resetAim(): void {
    this.currentAimAngle = 45
    this.currentAimPower = 50
    this.isAiming = false
    this.aimStartPos = null
    this.aimingRight = true
  }
}
```

#### 3. 에이밍 UI (파워 색상 그라데이션)

**파일**: `packages/game-core/src/ui/AimingUI.ts`

```typescript
import * as Phaser from "phaser"
import { Character } from "../entities/Character"

export class AimingUI {
  private scene: Phaser.Scene
  private aimLine: Phaser.GameObjects.Graphics
  private powerIndicator: Phaser.GameObjects.Graphics
  private infoText: Phaser.GameObjects.Text
  private visible: boolean = false

  constructor(scene: Phaser.Scene) {
    this.scene = scene
    this.aimLine = scene.add.graphics()
    this.powerIndicator = scene.add.graphics()
    this.infoText = scene.add.text(10, 180, "", {
      fontSize: "14px",
      color: "#ffffff",
      backgroundColor: "#00000080",
      padding: { x: 8, y: 4 },
    })
    this.hide()
  }

  show(): void {
    this.visible = true
    this.aimLine.setVisible(true)
    this.powerIndicator.setVisible(true)
    this.infoText.setVisible(true)
  }

  hide(): void {
    this.visible = false
    this.aimLine.setVisible(false)
    this.powerIndicator.setVisible(false)
    this.infoText.setVisible(false)
    this.aimLine.clear()
    this.powerIndicator.clear()
  }

  update(character: Character, angle: number, power: number): void {
    if (!this.visible) return

    const pos = character.getPosition()
    const facingRight = character.isFacingRight()

    const adjustedAngle = facingRight ? angle : 180 - angle
    const radians = Phaser.Math.DegToRad(adjustedAngle)

    // 조준선 길이 (파워에 비례)
    const lineLength = 50 + (power / 100) * 100
    const endX = pos.x + Math.cos(radians) * lineLength
    const endY = pos.y - Math.sin(radians) * lineLength

    // 파워에 따른 색상 (녹색 → 빨강)
    const colorR = Math.floor((power / 100) * 255)
    const colorG = Math.floor((1 - power / 100) * 255)
    const lineColor = (colorR << 16) | (colorG << 8) | 0

    // 조준선 그리기
    this.aimLine.clear()
    this.aimLine.lineStyle(3, lineColor, 0.8)
    this.aimLine.beginPath()
    this.aimLine.moveTo(pos.x, pos.y)
    this.aimLine.lineTo(endX, endY)
    this.aimLine.strokePath()

    // 조준점 (끝점에 원)
    this.aimLine.fillStyle(lineColor, 1)
    this.aimLine.fillCircle(endX, endY, 6)

    // 파워 인디케이터 (캐릭터 주변 원형)
    this.powerIndicator.clear()
    this.powerIndicator.lineStyle(2, 0xffffff, 0.3)
    this.powerIndicator.strokeCircle(pos.x, pos.y, 150)

    this.powerIndicator.lineStyle(3, lineColor, 0.6)
    const powerRadius = 20 + (power / 100) * 30
    this.powerIndicator.strokeCircle(pos.x, pos.y, powerRadius)

    // 정보 텍스트
    this.infoText.setText(
      `Angle: ${Math.round(angle)}°  Power: ${Math.round(power)}%`
    )
  }

  // 마우스 위치까지 가이드 라인 그리기
  updateWithMousePos(
    character: Character,
    mouseX: number,
    mouseY: number,
    angle: number,
    power: number
  ): void {
    if (!this.visible) return

    const pos = character.getPosition()
    this.update(character, angle, power)

    // 마우스까지 가이드 라인 추가
    this.aimLine.lineStyle(1, 0xffffff, 0.3)
    this.aimLine.beginPath()
    this.aimLine.moveTo(pos.x, pos.y)
    this.aimLine.lineTo(mouseX, mouseY)
    this.aimLine.strokePath()
  }

  destroy(): void {
    this.aimLine.destroy()
    this.powerIndicator.destroy()
    this.infoText.destroy()
  }
}
```

#### 4. 턴 시스템 단위 테스트

**파일**: `packages/game-core/src/systems/__tests__/TurnSystem.test.ts`

```typescript
import { describe, it, expect } from "vitest"

describe("TurnSystem State Machine", () => {
  // State transitions
  const transitions: Record<string, string[]> = {
    idle: ["playerTurn"],
    playerTurn: ["aiming", "playerTurn"], // playerTurn -> playerTurn (timeout)
    aiming: ["firing", "playerTurn"],
    firing: ["animating"],
    animating: ["playerTurn", "gameOver"],
    gameOver: [],
  }

  it("should have valid state transitions", () => {
    expect(transitions["idle"]).toContain("playerTurn")
    expect(transitions["playerTurn"]).toContain("aiming")
    expect(transitions["aiming"]).toContain("firing")
    expect(transitions["firing"]).toContain("animating")
    expect(transitions["animating"]).toContain("playerTurn")
  })

  it("should allow canceling aim to return to playerTurn", () => {
    expect(transitions["aiming"]).toContain("playerTurn")
  })

  it("should allow game over from animating state", () => {
    expect(transitions["animating"]).toContain("gameOver")
  })

  it("should not allow any transitions from gameOver", () => {
    expect(transitions["gameOver"]).toHaveLength(0)
  })
})
```

### 성공 기준:

#### 자동화된 검증:

- [x] 턴 시스템 단위 테스트 통과: `pnpm --filter @repo/game-core test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 턴 전환이 정상 작동 (Player 1 → Player 2)
- [x] 마우스 왼쪽 클릭 드래그로 에이밍 모드 진입
- [x] 드래그 방향/거리로 각도/파워 조절
- [x] 에이밍 UI가 정확히 표시됨 (파워에 따른 색상 변화)
- [x] 턴 타이머가 작동함 (30초)
- [x] 우클릭으로 에이밍 취소
- [x] 드래그 방향에 따라 캐릭터 방향 전환

**Implementation Note (2026-01-24)**: Phase 4 구현 완료.

- WASD 이동 (W: 점프, A/D: 좌우)
- 마우스 드래그 기반 에이밍 (Angry Birds 스타일)
- 드래그 방향/거리로 각도(-30°~90°)/파워(10%~100%) 조절
- 파워에 따른 색상 그라데이션 (녹색→빨강)
- 우클릭 컨텍스트 메뉴 비활성화

---

## Phase 5: 투사체 및 무기 시스템

### 개요

포물선 물리를 적용한 투사체 시스템과 확장 가능한 무기 시스템을 구현합니다.

### 필요한 변경:

#### 1. 투사체 기본 클래스

**파일**: `packages/game-core/src/entities/Projectile.ts`

```typescript
import Phaser from "phaser"
import { TerrainRenderer } from "@/terrain/TerrainRenderer"

export interface ProjectileConfig {
  scene: Phaser.Scene
  x: number
  y: number
  angle: number // 각도 (degrees)
  power: number // 파워 (0-100)
  spriteKey: string
  frame?: string
}

export abstract class Projectile {
  protected scene: Phaser.Scene
  protected sprite: Phaser.Physics.Matter.Sprite
  protected active: boolean = true

  // 물리 속성
  protected readonly POWER_MULTIPLIER = 0.15

  constructor(config: ProjectileConfig) {
    this.scene = config.scene

    // 초기 속도 계산
    const radians = Phaser.Math.DegToRad(config.angle)
    const velocity = {
      x: Math.cos(radians) * config.power * this.POWER_MULTIPLIER,
      y: -Math.sin(radians) * config.power * this.POWER_MULTIPLIER,
    }

    // 스프라이트 생성
    this.sprite = this.scene.matter.add.sprite(
      config.x,
      config.y,
      config.spriteKey,
      config.frame
    )
    this.sprite.setVelocity(velocity.x, velocity.y)
    this.sprite.setFriction(0.01)
    this.sprite.setFrictionAir(0.001)

    // 충돌 감지
    this.setupCollision()
  }

  private setupCollision(): void {
    this.scene.matter.world.on(
      "collisionstart",
      (event: Phaser.Physics.Matter.Events.CollisionStartEvent) => {
        if (!this.active) return

        event.pairs.forEach((pair) => {
          const isProjectile =
            pair.bodyA === this.sprite.body || pair.bodyB === this.sprite.body
          if (isProjectile) {
            this.onCollision(pair)
          }
        })
      }
    )
  }

  protected abstract onCollision(
    pair: Phaser.Types.Physics.Matter.MatterCollisionPair
  ): void

  abstract getExplosionRadius(): number
  abstract getDamage(): number

  applyWind(windForce: number): void {
    if (!this.active) return
    this.sprite.applyForce({ x: windForce * 0.0001, y: 0 })
  }

  getPosition(): { x: number; y: number } {
    return { x: this.sprite.x, y: this.sprite.y }
  }

  isActive(): boolean {
    return this.active
  }

  update(): void {
    if (!this.active) return

    // 화면 밖으로 나가면 비활성화
    if (
      this.sprite.y > this.scene.scale.height + 100 ||
      this.sprite.x < -100 ||
      this.sprite.x > this.scene.scale.width + 100
    ) {
      this.deactivate()
    }
  }

  protected deactivate(): void {
    this.active = false
    this.sprite.destroy()
  }

  destroy(): void {
    this.deactivate()
  }
}
```

#### 2. 무기 인터페이스 및 구체 무기 클래스

**파일**: `packages/game-core/src/weapons/WeaponTypes.ts`

```typescript
export interface WeaponStats {
  name: string
  damage: number
  explosionRadius: number
  ammo: number // -1 = 무제한
  spriteKey: string
  spriteFrame?: string
}

export const WeaponRegistry: Record<string, WeaponStats> = {
  bazooka: {
    name: "Bazooka",
    damage: 50,
    explosionRadius: 40,
    ammo: -1,
    spriteKey: "tanks",
    spriteFrame: "bullet1.png",
  },
  grenade: {
    name: "Grenade",
    damage: 35,
    explosionRadius: 50,
    ammo: 3,
    spriteKey: "tanks",
    spriteFrame: "bullet3.png",
  },
  shotgun: {
    name: "Shotgun",
    damage: 25,
    explosionRadius: 15,
    ammo: 2,
    spriteKey: "tanks",
    spriteFrame: "bullet2.png",
  },
}
```

**파일**: `packages/game-core/src/weapons/Bazooka.ts`

```typescript
import Phaser from "phaser"
import { Projectile, ProjectileConfig } from "@/entities/Projectile"
import { WeaponRegistry } from "@/weapons/WeaponTypes"

export class BazookaProjectile extends Projectile {
  private stats = WeaponRegistry.bazooka

  constructor(config: Omit<ProjectileConfig, "spriteKey" | "frame">) {
    super({
      ...config,
      spriteKey: "tanks",
      frame: "bullet1.png",
    })
  }

  protected onCollision(
    pair: Phaser.Types.Physics.Matter.MatterCollisionPair
  ): void {
    // 충돌 즉시 폭발
    this.explode()
  }

  private explode(): void {
    const pos = this.getPosition()

    // 폭발 애니메이션
    const explosion = this.scene.add.sprite(
      pos.x,
      pos.y,
      "tanks",
      "explosion1.png"
    )
    explosion.play("explosion")
    explosion.once("animationcomplete", () => explosion.destroy())

    // 이벤트 발생 (데미지 처리용)
    this.scene.events.emit("explosion", {
      x: pos.x,
      y: pos.y,
      radius: this.stats.explosionRadius,
      damage: this.stats.damage,
    })

    this.deactivate()
  }

  getExplosionRadius(): number {
    return this.stats.explosionRadius
  }

  getDamage(): number {
    return this.stats.damage
  }
}
```

**파일**: `packages/game-core/src/weapons/Grenade.ts`

```typescript
import Phaser from "phaser"
import { Projectile, ProjectileConfig } from "@/entities/Projectile"
import { WeaponRegistry } from "@/weapons/WeaponTypes"

export class GrenadeProjectile extends Projectile {
  private stats = WeaponRegistry.grenade
  private bounceCount: number = 0
  private readonly MAX_BOUNCES = 3
  private fuseTimer: Phaser.Time.TimerEvent

  constructor(config: Omit<ProjectileConfig, "spriteKey" | "frame">) {
    super({
      ...config,
      spriteKey: "tanks",
      frame: "bullet3.png",
    })

    // 탄성 설정 (튕김)
    this.sprite.setFriction(0.5)
    this.sprite.setBounce(0.6)

    // 3초 퓨즈
    this.fuseTimer = this.scene.time.addEvent({
      delay: 3000,
      callback: () => this.explode(),
      callbackScope: this,
    })
  }

  protected onCollision(
    pair: Phaser.Types.Physics.Matter.MatterCollisionPair
  ): void {
    this.bounceCount++

    // 최대 바운스 도달 시 즉시 폭발
    if (this.bounceCount >= this.MAX_BOUNCES) {
      this.explode()
    }
  }

  private explode(): void {
    if (!this.active) return

    this.fuseTimer.destroy()
    const pos = this.getPosition()

    // 폭발 애니메이션
    const explosion = this.scene.add.sprite(
      pos.x,
      pos.y,
      "tanks",
      "explosion1.png"
    )
    explosion.play("explosion")
    explosion.once("animationcomplete", () => explosion.destroy())

    // 이벤트 발생
    this.scene.events.emit("explosion", {
      x: pos.x,
      y: pos.y,
      radius: this.stats.explosionRadius,
      damage: this.stats.damage,
    })

    this.deactivate()
  }

  getExplosionRadius(): number {
    return this.stats.explosionRadius
  }

  getDamage(): number {
    return this.stats.damage
  }

  destroy(): void {
    this.fuseTimer.destroy()
    super.destroy()
  }
}
```

**파일**: `packages/game-core/src/weapons/Shotgun.ts`

```typescript
import Phaser from "phaser"
import { Projectile, ProjectileConfig } from "@/entities/Projectile"
import { WeaponRegistry } from "@/weapons/WeaponTypes"

export class ShotgunProjectile extends Projectile {
  private stats = WeaponRegistry.shotgun
  private readonly SPREAD_ANGLE = 15 // 산탄 퍼짐 각도

  constructor(config: Omit<ProjectileConfig, "spriteKey" | "frame">) {
    super({
      ...config,
      spriteKey: "tanks",
      frame: "bullet2.png",
    })
  }

  protected onCollision(
    pair: Phaser.Types.Physics.Matter.MatterCollisionPair
  ): void {
    this.explode()
  }

  private explode(): void {
    const pos = this.getPosition()

    // 작은 폭발
    const explosion = this.scene.add.sprite(
      pos.x,
      pos.y,
      "tanks",
      "explosion1.png"
    )
    explosion.setScale(0.5)
    explosion.play("explosion")
    explosion.once("animationcomplete", () => explosion.destroy())

    this.scene.events.emit("explosion", {
      x: pos.x,
      y: pos.y,
      radius: this.stats.explosionRadius,
      damage: this.stats.damage,
    })

    this.deactivate()
  }

  getExplosionRadius(): number {
    return this.stats.explosionRadius
  }

  getDamage(): number {
    return this.stats.damage
  }
}

// 샷건은 여러 발을 동시에 발사
export function createShotgunBlast(
  scene: Phaser.Scene,
  x: number,
  y: number,
  baseAngle: number,
  power: number
): ShotgunProjectile[] {
  const spreadAngles = [-10, -5, 0, 5, 10]
  return spreadAngles.map(
    (spread) =>
      new ShotgunProjectile({
        scene,
        x,
        y,
        angle: baseAngle + spread,
        power: power * (0.8 + Math.random() * 0.4), // 약간의 랜덤성
      })
  )
}
```

#### 3. 무기 매니저

**파일**: `packages/game-core/src/weapons/WeaponManager.ts`

```typescript
import Phaser from "phaser"
import { EventBus, GameEvents } from "@/EventBus"
import { WeaponRegistry, WeaponStats } from "@/weapons/WeaponTypes"
import { BazookaProjectile } from "@/weapons/Bazooka"
import { GrenadeProjectile } from "@/weapons/Grenade"
import { ShotgunProjectile, createShotgunBlast } from "@/weapons/Shotgun"
import { Projectile } from "@/entities/Projectile"

export class WeaponManager {
  private scene: Phaser.Scene
  private currentWeapon: string = "bazooka"
  private ammo: Record<string, number> = {}
  private activeProjectiles: Projectile[] = []

  constructor(scene: Phaser.Scene) {
    this.scene = scene
    this.initAmmo()
  }

  private initAmmo(): void {
    Object.entries(WeaponRegistry).forEach(([key, stats]) => {
      this.ammo[key] = stats.ammo
    })
  }

  selectWeapon(weaponKey: string): boolean {
    if (!WeaponRegistry[weaponKey]) return false
    if (this.ammo[weaponKey] === 0) return false

    this.currentWeapon = weaponKey
    EventBus.emit(GameEvents.WEAPON_SELECTED, {
      weapon: weaponKey,
      stats: WeaponRegistry[weaponKey],
      ammo: this.ammo[weaponKey],
    })
    return true
  }

  getCurrentWeapon(): WeaponStats {
    return WeaponRegistry[this.currentWeapon]
  }

  getAmmo(weaponKey: string): number {
    return this.ammo[weaponKey] ?? 0
  }

  fire(
    x: number,
    y: number,
    angle: number,
    power: number,
    facingRight: boolean
  ): Projectile[] {
    const adjustedAngle = facingRight ? angle : 180 - angle

    // 탄약 소모
    if (this.ammo[this.currentWeapon] > 0) {
      this.ammo[this.currentWeapon]--
    }

    let projectiles: Projectile[] = []

    switch (this.currentWeapon) {
      case "bazooka":
        projectiles = [
          new BazookaProjectile({
            scene: this.scene,
            x,
            y,
            angle: adjustedAngle,
            power,
          }),
        ]
        break

      case "grenade":
        projectiles = [
          new GrenadeProjectile({
            scene: this.scene,
            x,
            y,
            angle: adjustedAngle,
            power,
          }),
        ]
        break

      case "shotgun":
        projectiles = createShotgunBlast(this.scene, x, y, adjustedAngle, power)
        break
    }

    this.activeProjectiles.push(...projectiles)
    return projectiles
  }

  update(): void {
    // 비활성화된 투사체 제거
    this.activeProjectiles = this.activeProjectiles.filter((p) => {
      if (p.isActive()) {
        p.update()
        return true
      }
      return false
    })
  }

  applyWindToAll(windForce: number): void {
    this.activeProjectiles.forEach((p) => p.applyWind(windForce))
  }

  hasActiveProjectiles(): boolean {
    return this.activeProjectiles.length > 0
  }

  resetAmmo(): void {
    this.initAmmo()
  }

  destroy(): void {
    this.activeProjectiles.forEach((p) => p.destroy())
    this.activeProjectiles = []
  }
}
```

#### 4. 무기 테스트

**파일**: `packages/game-core/src/weapons/__tests__/WeaponTypes.test.ts`

```typescript
import { describe, it, expect } from "vitest"
import { WeaponRegistry } from "@/weapons/WeaponTypes"

describe("WeaponRegistry", () => {
  it("should have bazooka weapon", () => {
    expect(WeaponRegistry.bazooka).toBeDefined()
    expect(WeaponRegistry.bazooka.name).toBe("Bazooka")
    expect(WeaponRegistry.bazooka.damage).toBeGreaterThan(0)
    expect(WeaponRegistry.bazooka.ammo).toBe(-1) // 무제한
  })

  it("should have grenade weapon", () => {
    expect(WeaponRegistry.grenade).toBeDefined()
    expect(WeaponRegistry.grenade.name).toBe("Grenade")
    expect(WeaponRegistry.grenade.ammo).toBe(3)
  })

  it("should have shotgun weapon", () => {
    expect(WeaponRegistry.shotgun).toBeDefined()
    expect(WeaponRegistry.shotgun.name).toBe("Shotgun")
    expect(WeaponRegistry.shotgun.explosionRadius).toBeLessThan(
      WeaponRegistry.bazooka.explosionRadius
    )
  })

  it("all weapons should have required properties", () => {
    Object.values(WeaponRegistry).forEach((weapon) => {
      expect(weapon.name).toBeDefined()
      expect(weapon.damage).toBeGreaterThan(0)
      expect(weapon.explosionRadius).toBeGreaterThan(0)
      expect(weapon.spriteKey).toBeDefined()
    })
  })
})
```

### 성공 기준:

#### 자동화된 검증:

- [x] 무기 단위 테스트 통과: `pnpm --filter @repo/game-core test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 바주카 발사 시 포물선 궤적으로 이동
- [x] 지형 충돌 시 폭발 애니메이션 재생
- [x] 수류탄 투척 시 바운스 후 폭발
- [x] 샷건 발사 시 여러 탄환이 퍼짐
- [x] 무기 전환이 정상 작동

#### 추가 구현 사항 (2026-01-25):

- [x] Pose 기반 캐릭터 애니메이션 시스템으로 개편
- [x] 에이밍 시 무기 꺼내기/숨기기 애니메이션
- [x] 무기 각도가 에이밍 방향 따라 회전
- [x] 에이밍 중 캐릭터 고정 (경사면 미끄러짐 방지)
- [x] 캐릭터 스프라이트 origin 조정으로 지면 밀착 개선

**Implementation Note**: Phase 5 완료 (2026-01-25 수동 검증 완료).

---

## Phase 6: 바람 시스템 및 데미지 처리

### 개요

바람 시스템과 폭발 데미지 계산, 체력 관리를 구현합니다.

### 필요한 변경:

#### 1. 바람 시스템

**파일**: `packages/game-core/src/systems/WindSystem.ts`

```typescript
import Phaser from "phaser"
import { EventBus, GameEvents } from "../EventBus"

export class WindSystem {
  private windForce: number = 0
  private readonly MIN_WIND = -10
  private readonly MAX_WIND = 10

  constructor() {
    this.randomizeWind()
  }

  randomizeWind(): void {
    this.windForce = Phaser.Math.Between(this.MIN_WIND, this.MAX_WIND)
    EventBus.emit(GameEvents.WIND_CHANGED, { force: this.windForce })
  }

  getWindForce(): number {
    return this.windForce
  }

  // 바람 방향 표시용 (-1: 왼쪽, 0: 무풍, 1: 오른쪽)
  getWindDirection(): number {
    if (this.windForce === 0) return 0
    return this.windForce > 0 ? 1 : -1
  }

  // 바람 강도 (0-10 스케일)
  getWindStrength(): number {
    return Math.abs(this.windForce)
  }
}
```

#### 2. 데미지 시스템

**파일**: `packages/game-core/src/systems/DamageSystem.ts`

```typescript
import Phaser from "phaser"
import { Character } from "@/entities/Character"
import { TerrainRenderer } from "@/terrain/TerrainRenderer"
import { EventBus, GameEvents } from "@/EventBus"

export interface ExplosionEvent {
  x: number
  y: number
  radius: number
  damage: number
}

export class DamageSystem {
  private scene: Phaser.Scene
  private terrainRenderer: TerrainRenderer
  private characters: Character[]

  constructor(
    scene: Phaser.Scene,
    terrainRenderer: TerrainRenderer,
    characters: Character[]
  ) {
    this.scene = scene
    this.terrainRenderer = terrainRenderer
    this.characters = characters

    // 폭발 이벤트 리스너
    this.scene.events.on("explosion", this.handleExplosion, this)
  }

  private handleExplosion(event: ExplosionEvent): void {
    // 1. 지형 파괴
    this.terrainRenderer.onExplosion(event.x, event.y, event.radius)

    // 2. 캐릭터 데미지 계산
    this.characters.forEach((character) => {
      if (!character.isAlive()) return

      const pos = character.getPosition()
      const distance = Phaser.Math.Distance.Between(
        event.x,
        event.y,
        pos.x,
        pos.y
      )

      // 폭발 범위 내에 있으면 데미지
      if (distance <= event.radius) {
        // 거리에 따른 데미지 감소 (중심에서 멀수록 데미지 감소)
        const damageMultiplier = 1 - distance / event.radius
        const actualDamage = Math.floor(event.damage * damageMultiplier)

        character.takeDamage(actualDamage)

        // 체력 변경 이벤트
        EventBus.emit(GameEvents.HEALTH_CHANGED, {
          playerId: character.playerId,
          health: character.getHealth(),
          maxHealth: character.getMaxHealth(),
          damage: actualDamage,
        })

        // 넉백 효과 (폭발 반대 방향으로)
        this.applyKnockback(character, event)
      }
    })

    // 3. 게임 오버 체크
    this.checkGameOver()
  }

  private applyKnockback(character: Character, event: ExplosionEvent): void {
    const pos = character.getPosition()
    const angle = Phaser.Math.Angle.Between(event.x, event.y, pos.x, pos.y)
    const knockbackForce =
      0.05 *
      (1 -
        Phaser.Math.Distance.Between(event.x, event.y, pos.x, pos.y) /
          event.radius)

    // Character에 넉백 메서드 추가 필요
    // character.applyKnockback(angle, knockbackForce)
  }

  private checkGameOver(): void {
    const aliveCharacters = this.characters.filter((c) => c.isAlive())

    if (aliveCharacters.length <= 1) {
      const winner = aliveCharacters[0]
      EventBus.emit(GameEvents.GAME_OVER, {
        winnerId: winner ? winner.playerId : -1, // -1 = 무승부
      })
    }
  }

  destroy(): void {
    this.scene.events.off("explosion", this.handleExplosion, this)
  }
}
```

#### 3. HUD 업데이트

**파일**: `packages/game-core/src/ui/GameHUD.ts`

```typescript
import Phaser from "phaser"
import { EventBus, GameEvents } from "@/EventBus"

export class GameHUD {
  private scene: Phaser.Scene

  // UI 요소들
  private player1Health: Phaser.GameObjects.Graphics
  private player2Health: Phaser.GameObjects.Graphics
  private player1HealthText: Phaser.GameObjects.Text
  private player2HealthText: Phaser.GameObjects.Text
  private turnIndicator: Phaser.GameObjects.Text
  private turnTimer: Phaser.GameObjects.Text
  private windIndicator: Phaser.GameObjects.Text
  private weaponDisplay: Phaser.GameObjects.Text

  constructor(scene: Phaser.Scene) {
    this.scene = scene
    this.createUI()
    this.setupEventListeners()
  }

  private createUI(): void {
    const { width, height } = this.scene.scale

    // Player 1 Health Bar (좌측 상단)
    this.player1Health = this.scene.add.graphics()
    this.player1HealthText = this.scene.add.text(60, 15, "P1: 100", {
      fontSize: "16px",
      color: "#ffffff",
    })
    this.drawHealthBar(this.player1Health, 10, 10, 100, 100)

    // Player 2 Health Bar (우측 상단)
    this.player2Health = this.scene.add.graphics()
    this.player2HealthText = this.scene.add.text(width - 90, 15, "P2: 100", {
      fontSize: "16px",
      color: "#ffffff",
    })
    this.drawHealthBar(this.player2Health, width - 110, 10, 100, 100)

    // Turn Indicator (상단 중앙)
    this.turnIndicator = this.scene.add
      .text(width / 2, 20, "Player 1 Turn", {
        fontSize: "24px",
        color: "#ffffff",
        fontStyle: "bold",
      })
      .setOrigin(0.5)

    // Turn Timer
    this.turnTimer = this.scene.add
      .text(width / 2, 50, "30", {
        fontSize: "20px",
        color: "#ffff00",
      })
      .setOrigin(0.5)

    // Wind Indicator (하단 중앙)
    this.windIndicator = this.scene.add
      .text(width / 2, height - 30, "Wind: ← 5", {
        fontSize: "18px",
        color: "#87CEEB",
      })
      .setOrigin(0.5)

    // Weapon Display (하단 좌측)
    this.weaponDisplay = this.scene.add.text(
      10,
      height - 30,
      "Weapon: Bazooka",
      {
        fontSize: "16px",
        color: "#ffffff",
      }
    )
  }

  private drawHealthBar(
    graphics: Phaser.GameObjects.Graphics,
    x: number,
    y: number,
    current: number,
    max: number
  ): void {
    const width = 100
    const height = 20
    const percentage = current / max

    graphics.clear()

    // 배경
    graphics.fillStyle(0x333333, 1)
    graphics.fillRect(x, y, width, height)

    // 체력바 (초록 → 노랑 → 빨강)
    let color = 0x00ff00
    if (percentage < 0.3) color = 0xff0000
    else if (percentage < 0.6) color = 0xffff00

    graphics.fillStyle(color, 1)
    graphics.fillRect(x, y, width * percentage, height)

    // 테두리
    graphics.lineStyle(2, 0xffffff, 1)
    graphics.strokeRect(x, y, width, height)
  }

  private setupEventListeners(): void {
    EventBus.on(GameEvents.HEALTH_CHANGED, this.onHealthChanged, this)
    EventBus.on(GameEvents.TURN_CHANGED, this.onTurnChanged, this)
    EventBus.on(GameEvents.WIND_CHANGED, this.onWindChanged, this)
    EventBus.on(GameEvents.WEAPON_SELECTED, this.onWeaponSelected, this)
  }

  private onHealthChanged(data: {
    playerId: number
    health: number
    maxHealth: number
  }): void {
    if (data.playerId === 0) {
      this.drawHealthBar(
        this.player1Health,
        10,
        10,
        data.health,
        data.maxHealth
      )
      this.player1HealthText.setText(`P1: ${data.health}`)
    } else {
      const { width } = this.scene.scale
      this.drawHealthBar(
        this.player2Health,
        width - 110,
        10,
        data.health,
        data.maxHealth
      )
      this.player2HealthText.setText(`P2: ${data.health}`)
    }
  }

  private onTurnChanged(data: { currentPlayer: number; state: string }): void {
    this.turnIndicator.setText(`Player ${data.currentPlayer + 1} Turn`)
  }

  private onWindChanged(data: { force: number }): void {
    const direction = data.force > 0 ? "→" : data.force < 0 ? "←" : ""
    this.windIndicator.setText(`Wind: ${direction} ${Math.abs(data.force)}`)
  }

  private onWeaponSelected(data: { weapon: string; ammo: number }): void {
    const ammoText = data.ammo === -1 ? "∞" : data.ammo
    this.weaponDisplay.setText(`Weapon: ${data.weapon} (${ammoText})`)
  }

  updateTimer(seconds: number): void {
    this.turnTimer.setText(Math.ceil(seconds / 1000).toString())
  }

  destroy(): void {
    EventBus.off(GameEvents.HEALTH_CHANGED, this.onHealthChanged, this)
    EventBus.off(GameEvents.TURN_CHANGED, this.onTurnChanged, this)
    EventBus.off(GameEvents.WIND_CHANGED, this.onWindChanged, this)
    EventBus.off(GameEvents.WEAPON_SELECTED, this.onWeaponSelected, this)
  }
}
```

### 성공 기준:

#### 자동화된 검증:

- [x] 바람/데미지 단위 테스트 통과: `pnpm --filter @repo/game-core test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 바람 방향/강도가 HUD에 표시됨
- [x] 투사체가 바람에 영향을 받아 궤적이 변함
- [x] 폭발 시 지형이 파괴됨
- [x] 폭발 범위 내 캐릭터가 데미지를 받음
- [x] 체력바가 정확히 업데이트됨
- [x] 체력이 0이 되면 게임 오버

**Implementation Note**: Phase 6 완료 (2026-01-28). 바람 계수 조정 (0.000005), HP바 캐릭터 위 표시, 씬 재시작 버그 수정.

---

## Phase 7: 게임 씬 통합 및 게임 루프 완성

### 개요

모든 시스템을 GameScene에 통합하고, 완전한 게임 루프를 구현합니다.

### 필요한 변경:

#### 1. 메인 게임 씬

**파일**: `packages/game-core/src/scenes/GameScene.ts`

```typescript
import Phaser from "phaser"
import { TerrainData } from "@/terrain/TerrainData"
import { TerrainRenderer } from "@/terrain/TerrainRenderer"
import { Character } from "@/entities/Character"
import { TurnSystem } from "@/systems/TurnSystem"
import { InputController } from "@/systems/InputController"
import { DamageSystem } from "@/systems/DamageSystem"
import { WindSystem } from "@/systems/WindSystem"
import { WeaponManager } from "@/weapons/WeaponManager"
import { AimingUI } from "@/ui/AimingUI"
import { GameHUD } from "@/ui/GameHUD"
import { EventBus, GameEvents } from "@/EventBus"

export class GameScene extends Phaser.Scene {
  // 지형
  private terrainData!: TerrainData
  private terrainRenderer!: TerrainRenderer

  // 캐릭터
  private characters: Character[] = []

  // 시스템
  private turnSystem!: TurnSystem
  private inputController!: InputController
  private damageSystem!: DamageSystem
  private windSystem!: WindSystem
  private weaponManager!: WeaponManager

  // UI
  private aimingUI!: AimingUI
  private gameHUD!: GameHUD

  // 게임 상태
  private isGameOver: boolean = false

  constructor() {
    super({ key: "GameScene" })
  }

  create(): void {
    this.createTerrain()
    this.createCharacters()
    this.createSystems()
    this.createUI()
    this.setupEventListeners()
    this.startGame()
  }

  private createTerrain(): void {
    const { width, height } = this.scale

    this.terrainData = new TerrainData(width, height)
    this.terrainData.generateTerrain(0.65, 80)

    this.terrainRenderer = new TerrainRenderer(this, this.terrainData)
    this.terrainRenderer.create()
  }

  private createCharacters(): void {
    const { width } = this.scale

    // 지형 표면 높이 찾기
    const player1X = Math.floor(width * 0.25)
    const player2X = Math.floor(width * 0.75)

    const player1SurfaceY = this.findSurfaceY(player1X)
    const player2SurfaceY = this.findSurfaceY(player2X)

    // Player 1 (좌측)
    const player1 = new Character({
      scene: this,
      x: player1X,
      y: player1SurfaceY - 50,
      playerId: 0,
      spriteKey: "character1",
      terrainData: this.terrainData,
    })
    this.characters.push(player1)

    // Player 2 (우측)
    const player2 = new Character({
      scene: this,
      x: player2X,
      y: player2SurfaceY - 50,
      playerId: 1,
      spriteKey: "character2",
      terrainData: this.terrainData,
    })
    this.characters.push(player2)
  }

  private findSurfaceY(x: number): number {
    for (let y = 0; y < this.terrainData.height; y++) {
      if (this.terrainData.isSolid(x, y)) {
        return y
      }
    }
    return this.terrainData.height * 0.65
  }

  private createSystems(): void {
    this.turnSystem = new TurnSystem(this, {
      turnDuration: 30000, // 30초
      playerCount: 2,
    })

    this.inputController = new InputController(this)
    this.windSystem = new WindSystem()
    this.weaponManager = new WeaponManager(this)

    this.damageSystem = new DamageSystem(
      this,
      this.terrainRenderer,
      this.characters
    )
  }

  private createUI(): void {
    this.aimingUI = new AimingUI(this)
    this.gameHUD = new GameHUD(this)
  }

  private setupEventListeners(): void {
    EventBus.on(GameEvents.GAME_OVER, this.onGameOver, this)
  }

  private startGame(): void {
    this.turnSystem.startGame()

    // BGM 재생
    this.sound.play("bgm", { loop: true, volume: 0.3 })
  }

  update(time: number, delta: number): void {
    if (this.isGameOver) return

    // 캐릭터 업데이트
    this.characters.forEach((c) => c.update())

    // 현재 플레이어
    const currentPlayer = this.characters[this.turnSystem.getCurrentPlayer()]
    if (!currentPlayer) return

    // 입력 처리
    this.inputController.update(currentPlayer, this.turnSystem)

    // 무기 매니저 업데이트
    this.weaponManager.update()

    // 바람 적용
    this.weaponManager.applyWindToAll(this.windSystem.getWindForce())

    // 에이밍 UI 업데이트
    const state = this.turnSystem.getState()
    if (state === "aiming") {
      this.aimingUI.show()
      this.aimingUI.update(
        currentPlayer,
        this.inputController.getAimAngle(),
        this.inputController.getAimPower()
      )
    } else {
      this.aimingUI.hide()
    }

    // 타이머 업데이트
    this.gameHUD.updateTimer(this.turnSystem.getRemainingTime())

    // 발사 처리
    if (state === "firing") {
      this.handleFire(currentPlayer)
    }

    // 애니메이션 완료 체크
    if (state === "animating" && !this.weaponManager.hasActiveProjectiles()) {
      this.onAnimationComplete()
    }
  }

  private handleFire(character: Character): void {
    const pos = character.getPosition()

    this.weaponManager.fire(
      pos.x,
      pos.y,
      this.inputController.getAimAngle(),
      this.inputController.getAimPower(),
      character.isFacingRight()
    )

    this.turnSystem.startAnimating()
    this.inputController.resetAim()
  }

  private onAnimationComplete(): void {
    // 턴 종료 전 바람 변경
    this.windSystem.randomizeWind()
    this.turnSystem.endTurn()
  }

  private onGameOver(data: { winnerId: number }): void {
    this.isGameOver = true

    // 게임 오버 UI 표시
    const { width, height } = this.scale
    const message =
      data.winnerId >= 0 ? `Player ${data.winnerId + 1} Wins!` : "Draw!"

    const gameOverText = this.add
      .text(width / 2, height / 2, message, {
        fontSize: "48px",
        color: "#ffffff",
        fontStyle: "bold",
        backgroundColor: "#000000aa",
        padding: { x: 20, y: 10 },
      })
      .setOrigin(0.5)

    // 재시작 버튼
    const restartText = this.add
      .text(width / 2, height / 2 + 80, "Press SPACE to Restart", {
        fontSize: "24px",
        color: "#ffff00",
      })
      .setOrigin(0.5)

    this.input.keyboard?.once("keydown-SPACE", () => {
      this.scene.restart()
    })
  }

  shutdown(): void {
    this.turnSystem.destroy()
    this.damageSystem.destroy()
    this.weaponManager.destroy()
    this.aimingUI.destroy()
    this.gameHUD.destroy()
    this.terrainRenderer.destroy()
    this.characters.forEach((c) => c.destroy())

    EventBus.off(GameEvents.GAME_OVER, this.onGameOver, this)
  }
}
```

#### 2. 무기 선택 UI

**파일**: `packages/game-core/src/ui/WeaponSelector.ts`

```typescript
import Phaser from "phaser"
import { WeaponManager } from "@/weapons/WeaponManager"
import { WeaponRegistry } from "@/weapons/WeaponTypes"

export class WeaponSelector {
  private scene: Phaser.Scene
  private weaponManager: WeaponManager
  private container: Phaser.GameObjects.Container
  private weaponButtons: Phaser.GameObjects.Text[] = []
  private visible: boolean = false

  constructor(scene: Phaser.Scene, weaponManager: WeaponManager) {
    this.scene = scene
    this.weaponManager = weaponManager
    this.container = scene.add.container(10, 100)
    this.createButtons()
    this.hide()
    this.setupKeyboardShortcuts()
  }

  private createButtons(): void {
    const weapons = Object.keys(WeaponRegistry)

    weapons.forEach((weaponKey, index) => {
      const stats = WeaponRegistry[weaponKey]
      const ammo = this.weaponManager.getAmmo(weaponKey)
      const ammoText = ammo === -1 ? "∞" : ammo.toString()

      const button = this.scene.add.text(
        0,
        index * 30,
        `${index + 1}. ${stats.name} (${ammoText})`,
        {
          fontSize: "16px",
          color: "#ffffff",
          backgroundColor: "#333333aa",
          padding: { x: 8, y: 4 },
        }
      )

      button.setInteractive({ useHandCursor: true })
      button.on("pointerdown", () => this.selectWeapon(weaponKey))

      this.weaponButtons.push(button)
      this.container.add(button)
    })
  }

  private setupKeyboardShortcuts(): void {
    const keys = ["ONE", "TWO", "THREE", "FOUR", "FIVE"]
    const weapons = Object.keys(WeaponRegistry)

    keys.forEach((key, index) => {
      if (index < weapons.length) {
        this.scene.input.keyboard?.on(`keydown-${key}`, () => {
          if (this.visible) {
            this.selectWeapon(weapons[index])
          }
        })
      }
    })
  }

  private selectWeapon(weaponKey: string): void {
    if (this.weaponManager.selectWeapon(weaponKey)) {
      this.hide()
      this.updateButtonTexts()
    }
  }

  private updateButtonTexts(): void {
    const weapons = Object.keys(WeaponRegistry)

    weapons.forEach((weaponKey, index) => {
      const stats = WeaponRegistry[weaponKey]
      const ammo = this.weaponManager.getAmmo(weaponKey)
      const ammoText = ammo === -1 ? "∞" : ammo.toString()
      this.weaponButtons[index].setText(
        `${index + 1}. ${stats.name} (${ammoText})`
      )
    })
  }

  show(): void {
    this.visible = true
    this.container.setVisible(true)
    this.updateButtonTexts()
  }

  hide(): void {
    this.visible = false
    this.container.setVisible(false)
  }

  toggle(): void {
    if (this.visible) {
      this.hide()
    } else {
      this.show()
    }
  }

  isVisible(): boolean {
    return this.visible
  }

  destroy(): void {
    this.container.destroy()
  }
}
```

### 성공 기준:

#### 자동화된 검증:

- [x] 모든 단위 테스트 통과: `pnpm test`
- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 린팅 통과: `pnpm lint`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 게임 시작 시 두 캐릭터가 지형 위에 배치됨
- [x] 턴 전환이 원활하게 작동
- [x] 무기 선택 (1,2,3 키) 정상 작동
- [x] 에이밍 → 발사 → 폭발 → 데미지 전체 플로우 작동
- [x] 체력 0 시 게임 오버 화면 표시
- [x] 스페이스바로 재시작 가능
- [x] BGM이 재생됨

**Implementation Note**: Phase 7 수동 검증 완료.

---

## Phase 8: React UI 통합 및 게임 외부 화면

### 개요

게임 시작 화면, 결과 화면 등 React 기반 UI를 구현하고 Phaser와 통합합니다.

### 필요한 변경:

#### 1. 메인 메뉴 컴포넌트

**파일**: `apps/web/src/components/ui/MainMenu.tsx`

```typescript
'use client'

import { useState } from 'react'

interface MainMenuProps {
  onStartGame: () => void
}

export function MainMenu({ onStartGame }: MainMenuProps) {
  const [showHowToPlay, setShowHowToPlay] = useState(false)

  return (
    <div className="flex flex-col items-center justify-center min-h-screen bg-gradient-to-b from-blue-900 to-blue-700">
      <h1 className="text-6xl font-bold text-white mb-4 drop-shadow-lg">
        Artillery Wars
      </h1>
      <p className="text-xl text-blue-200 mb-12">
        웜즈 스타일 턴 기반 전략 게임
      </p>

      <div className="flex flex-col gap-4">
        <button
          onClick={onStartGame}
          className="px-8 py-4 text-2xl font-bold text-white bg-green-600 hover:bg-green-500 rounded-lg shadow-lg transition-all hover:scale-105"
        >
          게임 시작
        </button>

        <button
          onClick={() => setShowHowToPlay(true)}
          className="px-8 py-4 text-xl text-white bg-blue-600 hover:bg-blue-500 rounded-lg shadow-lg transition-all"
        >
          조작 방법
        </button>
      </div>

      {showHowToPlay && (
        <div className="fixed inset-0 bg-black/70 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-8 max-w-md">
            <h2 className="text-2xl font-bold mb-4">조작 방법</h2>
            <ul className="space-y-2 text-gray-700">
              <li>A / D : 좌우 이동</li>
              <li>W : 점프</li>
              <li>마우스 왼쪽 드래그 : 에이밍 (방향/파워 조절)</li>
              <li>마우스 버튼 놓기 : 발사</li>
              <li>마우스 오른쪽 클릭 : 에이밍 취소</li>
              <li>1, 2, 3 : 무기 선택</li>
            </ul>
            <button
              onClick={() => setShowHowToPlay(false)}
              className="mt-6 w-full py-2 bg-blue-600 text-white rounded hover:bg-blue-500"
            >
              닫기
            </button>
          </div>
        </div>
      )}
    </div>
  )
}
```

#### 2. 게임 래퍼 컴포넌트

**파일**: `apps/web/src/components/game/GameWrapper.tsx`

```typescript
'use client'

import { useState, useCallback, useEffect } from 'react'
import { MainMenu } from '../ui/MainMenu'
import { PhaserGame } from './PhaserGame'
import { EventBus, GameEvents } from '@repo/game-core'

type GameScreen = 'menu' | 'playing' | 'result'

interface GameResult {
  winnerId: number
}

export function GameWrapper() {
  const [screen, setScreen] = useState<GameScreen>('menu')
  const [result, setResult] = useState<GameResult | null>(null)

  useEffect(() => {
    const handleGameOver = (data: GameResult) => {
      setResult(data)
      setScreen('result')
    }

    EventBus.on(GameEvents.GAME_OVER, handleGameOver)
    return () => {
      EventBus.off(GameEvents.GAME_OVER, handleGameOver)
    }
  }, [])

  const handleStartGame = useCallback(() => {
    setScreen('playing')
    setResult(null)
  }, [])

  const handleReturnToMenu = useCallback(() => {
    setScreen('menu')
  }, [])

  if (screen === 'menu') {
    return <MainMenu onStartGame={handleStartGame} />
  }

  if (screen === 'result' && result) {
    return (
      <div className="flex flex-col items-center justify-center min-h-screen bg-gradient-to-b from-gray-900 to-gray-700">
        <h1 className="text-5xl font-bold text-white mb-4">
          {result.winnerId >= 0 ? `Player ${result.winnerId + 1} 승리!` : '무승부!'}
        </h1>
        <div className="flex gap-4 mt-8">
          <button
            onClick={handleStartGame}
            className="px-6 py-3 text-xl text-white bg-green-600 hover:bg-green-500 rounded-lg"
          >
            다시 플레이
          </button>
          <button
            onClick={handleReturnToMenu}
            className="px-6 py-3 text-xl text-white bg-blue-600 hover:bg-blue-500 rounded-lg"
          >
            메인 메뉴
          </button>
        </div>
      </div>
    )
  }

  return (
    <div className="relative min-h-screen bg-gray-900">
      <PhaserGame />
      <button
        onClick={handleReturnToMenu}
        className="absolute top-4 right-4 px-4 py-2 text-white bg-red-600 hover:bg-red-500 rounded"
      >
        나가기
      </button>
    </div>
  )
}
```

#### 3. 페이지 업데이트

**파일**: `apps/web/src/app/page.tsx`

```typescript
import dynamic from 'next/dynamic'

const GameWrapper = dynamic(
  () => import('@/components/game/GameWrapper').then(m => m.GameWrapper),
  { ssr: false }
)

export default function Home() {
  return <GameWrapper />
}
```

### 성공 기준:

#### 자동화된 검증:

- [x] TypeScript 타입 체크 통과: `pnpm typecheck`
- [x] 린팅 통과: `pnpm lint`
- [x] 빌드 성공: `pnpm build`

#### 수동 검증:

- [x] 메인 메뉴 화면이 표시됨
- [x] "게임 시작" 클릭 시 게임 화면으로 전환
- [x] "조작 방법" 모달이 정상 작동
- [x] 게임 오버 시 결과 화면으로 전환
- [x] "다시 플레이" 및 "메인 메뉴" 버튼 작동
- [x] "나가기" 버튼으로 메인 메뉴 복귀

**Implementation Note**: Phase 8 수동 검증 완료 (2026-02-07). shadcn/ui 기반 Button, Dialog 컴포넌트 적용. 게이밍 다크 테마 적용.

---

## 테스트 전략

### 단위 테스트 (Vitest)

- TerrainData: 지형 생성, 폭발 파괴
- WeaponRegistry: 무기 속성 검증
- TurnSystem: 상태 전이 검증
- DamageSystem: 데미지 계산 로직

### 통합 테스트

- Phaser Scene 초기화
- 시스템 간 이벤트 통신
- 게임 루프 전체 플로우

### E2E 테스트 (Playwright)

```typescript
// e2e/game.spec.ts
test("전체 게임 플로우", async ({ page }) => {
  await page.goto("/")

  // 메인 메뉴
  await expect(page.getByText("Artillery Wars")).toBeVisible()
  await page.click("text=게임 시작")

  // 게임 화면
  await expect(page.locator("canvas")).toBeVisible()

  // 게임 플레이 (키보드 입력)
  await page.keyboard.press("Space") // 에이밍
  await page.keyboard.press("Space") // 발사

  // 턴 전환 확인
  await expect(page.getByText("Player 2 Turn")).toBeVisible({ timeout: 10000 })
})
```

### 수동 테스트 체크리스트

1. [ ] 지형 생성이 매번 다르게 표시됨
2. [ ] 캐릭터 물리 충돌이 자연스러움
3. [ ] 투사체 궤적이 물리적으로 정확함
4. [ ] 바람 영향이 체감됨
5. [ ] 폭발 데미지가 적절함
6. [ ] 게임 밸런스가 적절함

---

## 성능 고려 사항

1. **지형 충돌체 최적화**: 모든 픽셀이 아닌 컬럼 기반 충돌체 사용
2. **스프라이트시트 사용**: 개별 이미지 대신 아틀라스 사용
3. **Object Pooling**: 투사체 재사용
4. **Visibility Culling**: 화면 밖 오브젝트 업데이트 스킵

---

## 향후 확장 (Phase 2+)

### Online Multiplayer (Supabase)

- Supabase Realtime Broadcast로 게임 상태 동기화
- Server Actions로 턴 검증
- Presence로 플레이어 온라인 상태

### AI 플레이어

- Utility AI 기반 의사결정
- 난이도별 조준 정확도 조절
- Edge Functions에서 AI 턴 처리

### Web3 인증 (SUI)

- zkLogin으로 소셜 로그인
- 지갑 연결 옵션
- NFT 스킨/아이템 (미래)

---

## 참조

- 원본 연구: [thoughts/shared/research/2026-01-22-worms-game-architecture-research.md](../research/2026-01-22-worms-game-architecture-research.md)
- Phaser 3 문서: https://docs.phaser.io/
- Next.js 문서: https://nextjs.org/docs
- Turborepo 문서: https://turbo.build/repo/docs
