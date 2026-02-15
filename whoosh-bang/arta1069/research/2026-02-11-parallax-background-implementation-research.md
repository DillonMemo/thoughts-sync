---
date: 2026-02-11T16:57:32+0000
researcher: arta1069@gmail.com
git_commit: N/A (git 미초기화)
branch: N/A
repository: game
topic: "Parallax 배경 에셋 활용 구현을 위한 종합 연구 - 에셋 분석, 현재 시스템, 구현 패턴"
tags: [research, codebase, parallax, background, GameScene, BootScene, Phaser3, assets, freepik]
status: complete
last_updated: 2026-02-11
last_updated_by: arta1069@gmail.com
---

# 연구: Parallax 배경 에셋 활용 구현을 위한 종합 연구

**날짜**: 2026-02-11T16:57:32+0000
**연구자**: arta1069@gmail.com
**Git 커밋**: N/A (git 미초기화)
**브랜치**: N/A
**리포지토리**: game

## 연구 질문

현재 게임 맵의 배경과 구도가 원하는 감성이 아니어서, Freepik 스타일의 Parallax 배경 에셋(`apps/web/public/assets/background/1~8`)을 활용하여 배경 시스템을 재구성하고자 함. 에셋의 시각적 특징, 현재 시스템 구조, 교체에 필요한 기술적 요소를 종합 조사.

## 요약

### 현재 상태
게임은 **모든 배경을 코드(Graphics API)로 절차적 생성**합니다. `GameScene.createBackground()`에서 하늘 그래디언트, 구름 8개, 4개 실루엣 레이어(먼 산, 배경 섬, 중간 숲, 전경 나무), 물을 그립니다. 시각적으로 회청색 톤의 흐린 날씨 스타일이며, 실제 parallax 스크롤은 없고 depth/alpha 차이로만 깊이감을 표현합니다.

### 준비된 에셋
8세트의 parallax 배경 PNG가 `assets/background/1~8/`에 존재합니다. 각 세트는 5~9개 레이어(1920x1080)로 구성되며, 외계행성/심해/해변/독성습지/산맥호수/구름산/밤숲/폭포 등 다양한 테마를 가지고 있습니다. 레이어 번호가 **낮을수록 전경(앞)**, **높을수록 배경(뒤)**입니다.

### 교체 시 핵심 고려사항
- **물 렌더링은 유지 필수**: DamageSystem의 waterLevel(height*0.88=Y633)에 의존하는 익사 메카닉이 있음
- **BootScene에서 동적 로딩**: 8세트 중 1개만 선택하여 레이어 로딩 (메모리 최적화)
- **고정 카메라 게임**: 마우스 기반 미세 parallax 효과로 깊이감 표현 가능
- **기존 배경 코드 제거**: `createBackground()`의 하늘/구름/실루엣 렌더링을 이미지 기반으로 교체

## 상세 발견 사항

### 1. 배경 에셋 시각적 분석

#### 세트별 테마 및 특징

| 세트 | 레이어 수 | 테마 | 주요 색상 | 분위기 |
|------|----------|------|----------|--------|
| 1 | 6 | 외계 행성 용암 | 붉은/주황/보라 | 뜨겁고 적대적, SF |
| 2 | 6 | 심해 생명체 | 청록/파란/보라 | 신비로운 수중 |
| 3 | 6 | 평온한 해변 | 베이지/청록/파란 | 열대 낙원 |
| 4 | 6 | 독성 습지 | 청록/회갈/빨강 | 위험한 환경 |
| 5 | 7 | 산맥 호수 | 하늘색/베이지/갈색 | 청명한 자연 |
| 6 | 9 | 구름과 산 | 하늘색/회색/보라 | 고요한 산악 |
| 7 | 8 | 밤 숲 | 짙은파랑/청록/검정 | 신비로운 야간 |
| 8 | 5 | 보라빛 황혼 폭포 | 보라/청록/파랑 | 판타지 마법 |

#### 공통 레이어 구성 패턴

```
낮은 번호 (전경, 앞)
  1_layer.png: 지면/식물/전경 장식
  2_layer.png: 근경 오브젝트 (또는 투명)
  3_layer.png: 중경 구조물/식물
  ...
  N_layer.png: 하늘 그래디언트/배경
높은 번호 (배경, 뒤)
```

#### 이미지 사양
- **해상도**: 모든 레이어 1920x1080 PNG
- **비율**: 16:9 (게임 1280x720과 동일)
- **스케일링**: `setDisplaySize(1280, 720)`으로 축소 가능
- **합성 미리보기**: 각 세트에 `{setId}_game_background.png` 존재

### 2. 현재 배경 렌더링 시스템 (교체 대상)

#### 렌더링 레이어 구조 (depth 순)

| depth | 요소 | 렌더링 방식 | 교체 여부 |
|-------|------|------------|----------|
| -25 | 하늘 그래디언트 | Graphics (수평선 반복) | **교체** → 최상위 레이어 이미지 |
| -22 | 구름 | Graphics (원 조합) | **교체** → 에셋 레이어에 포함 |
| -20 | 먼 산 실루엣 | Graphics (사인파 윤곽) | **교체** → 에셋 레이어 |
| -17 | 배경 섬 실루엣 | Graphics (다각형+나무) | **교체** → 에셋 레이어 |
| -14 | 중간 숲 실루엣 | Graphics (사인파 윤곽) | **교체** → 에셋 레이어 |
| -11 | 전경 나무 실루엣 | Graphics (삼각형 조합) | **교체** → 에셋 레이어 |
| -5 | 물 | Graphics (그래디언트+반짝임) | **유지** (게임 메카닉) |
| 0 | 지형 스프라이트 | Canvas 텍스처→Image | **유지** |
| 5 | 데코레이션 | Graphics (나무/바위/풀) | **유지** |

#### 교체 대상 코드 범위

- `GameScene.ts:315-380` - `createBackground()`: 하늘 그래디언트 + 물 (물은 유지)
- `GameScene.ts:382-404` - `drawClouds()`: 구름 8개 그리기
- `GameScene.ts:406-432` - `drawDistantMountains()`: 먼 산 실루엣
- `GameScene.ts:434-525` - `drawBackgroundIslands()` + `drawIslandShape()`: 배경 섬
- `GameScene.ts:527-556` - `drawBackgroundForest()`: 배경 숲
- `GameScene.ts:558-608` - `drawForegroundTrees()`: 전경 나무

### 3. waterLevel 의존성 (보존 필수)

#### waterLevel 계산
- **공식**: `Math.floor(height * 0.88)` = Y 633 (1280x720 기준)
- **물 영역**: Y 633~720 (87픽셀 높이)

#### 사용 위치

| 위치 | 용도 | 의존도 |
|------|------|--------|
| `GameScene.ts:316` | 물 렌더링 시작점 | 시각적 |
| `DamageSystem.ts:29` | 익사 기준선 | **게임 메카닉** |
| `DamageSystem.ts:48` | `pos.y > waterLevel+30`이면 즉사 | **게임 메카닉** |
| `TerrainData.ts:59` | 섬 바닥 기준선 (baseY) | 지형 생성 |

#### DamageSystem 익사 로직
```typescript
// DamageSystem.ts:41-68
private checkWaterDeath(): void {
  this.characters.forEach((character) => {
    const pos = character.getPosition()
    if (pos.y > this.waterLevel + 30) {  // Y > 663이면 익사
      character.takeDamage(character.getHealth())  // 즉사
    }
  })
}
```

### 4. 에셋 로딩 파이프라인 (BootScene)

#### 현재 로딩 구조 (BootScene.ts)
- 캐릭터 PNG: `/assets/characters/PNG/{type}/Poses/{type}_{pose}.png`
- 탱크 아틀라스: `/assets/resource/Spritesheet/tanks_spritesheetDefault.{png,xml}`
- 사운드: `/assets/sound/{file}`
- **배경 이미지 로딩 없음** (현재 모든 배경은 코드 생성)

#### 에셋 경로 구조
- 실제 위치: `/Users/dillon/workspace/game/assets/background/{1~8}/{N}_layer.png`
- 심볼릭 링크: `apps/web/public/assets` → `../../../assets`
- 브라우저 접근: `/assets/background/{setId}/{N}_layer.png`

#### 동적 로딩 패턴 (구현 필요)
```typescript
// BootScene.preload()에서:
// 1. 랜덤 세트 선택
const setId = Phaser.Math.Between(1, 8)

// 2. 세트별 레이어 수 매핑
const layerCounts: Record<number, number> = {
  1: 6, 2: 6, 3: 6, 4: 6, 5: 7, 6: 9, 7: 8, 8: 5
}

// 3. 선택된 세트의 레이어만 로딩
for (let i = 1; i <= layerCounts[setId]; i++) {
  this.load.image(`bg_layer_${i}`, `/assets/background/${setId}/${i}_layer.png`)
}
```

### 5. 게임 설정

| 항목 | 값 | 위치 |
|------|-----|------|
| 해상도 | 1280 x 720 | `index.ts:10, 24` |
| 스케일 모드 | Phaser.Scale.FIT | `index.ts:20` |
| 자동 정렬 | CENTER_BOTH | `index.ts:22` |
| 물리 엔진 | Matter.js | `index.ts:12` |
| Phaser 버전 | ^3.90.0 | `package.json` |
| 씬 순서 | [BootScene, GameScene] | `index.ts:26` |

### 6. Phaser 3 Parallax 구현 패턴

#### 방법 A: 고정 카메라 + 마우스 기반 미세 이동 (권장)

현재 게임은 카메라가 고정(스크롤 없음)이므로, 마우스 위치에 따른 미세한 레이어 이동으로 깊이감을 표현:

```typescript
// 마우스 위치에 따라 각 레이어를 다른 강도로 이동
this.input.on('pointermove', (pointer) => {
  const offsetX = (pointer.x - width/2) / width  // -0.5 ~ 0.5
  const offsetY = (pointer.y - height/2) / height

  // 뒤쪽 레이어: 거의 안 움직임, 앞쪽 레이어: 더 많이 움직임
  skyLayer.x = width/2 - offsetX * 2
  farLayer.x = width/2 - offsetX * 5
  midLayer.x = width/2 - offsetX * 10
  nearLayer.x = width/2 - offsetX * 15
})
```

#### 방법 B: setScrollFactor() (카메라 스크롤 시)
```typescript
this.add.image(width/2, height/2, 'sky').setScrollFactor(0)
this.add.image(width/2, height/2, 'far').setScrollFactor(0.25)
this.add.image(width/2, height/2, 'mid').setScrollFactor(0.5)
```

#### 방법 C: TileSprite + 자동 스크롤
```typescript
this.bg = this.add.tileSprite(0, 0, width, height, 'bg')
// update에서: this.bg.tilePosition.x -= 0.05
```

#### Worms/Artillery 장르 참고
- Worms 시리즈: "복잡한 애니메이션 parallax 스크롤링 배경" 사용 (Worms 5)
- Director's Cut: 9 레벨 parallax 스크롤링
- 고정 화면에서도 레이어 깊이와 미세 움직임으로 몰입감 제공

### 7. 성능 최적화 고려사항

- **선택적 로딩**: 8세트 중 1개만 로딩 (나머지 7세트의 메모리 낭비 방지)
- **PNG 최적화**: 1920x1080 PNG × 5~9장 ≈ 30~60MB 비압축 → WebP 변환 시 70% 감소 가능
- **TileSprite POT 제한**: WebGL에서 TileSprite 텍스처는 2의 거듭제곱 크기 필요 (Image 사용이 더 안전)
- **텍스처 관리**: 게임 재시작 시 이전 세트 텍스처 제거 후 새 세트 로딩

## 코드 참조

- `packages/game-core/src/scenes/GameScene.ts:315-380` - `createBackground()`: 교체 대상 메인 함수
- `packages/game-core/src/scenes/GameScene.ts:382-404` - `drawClouds()`: 교체 대상
- `packages/game-core/src/scenes/GameScene.ts:406-608` - 4개 실루엣 레이어 함수들: 교체 대상
- `packages/game-core/src/scenes/GameScene.ts:356-379` - 물 렌더링: **유지 대상**
- `packages/game-core/src/scenes/BootScene.ts:31-61` - `preload()`: 배경 로딩 추가 필요
- `packages/game-core/src/systems/DamageSystem.ts:17-68` - waterLevel 및 익사 체크: **보존 필수**
- `packages/game-core/src/terrain/TerrainData.ts:58-76` - 섬 생성 (waterLevel 기준): **보존 필수**
- `packages/game-core/src/index.ts:10-26` - Phaser Config (1280x720, FIT)

## 아키텍처 문서화

### 배경 교체 시 영향 범위

```
BootScene.preload()
  + 배경 세트 랜덤 선택 로직 추가
  + 선택된 세트의 레이어 PNG 로딩 추가
  + 세트 ID를 GameScene에 전달하는 메커니즘 필요

GameScene.createBackground()
  - 하늘 그래디언트 Graphics 코드 제거
  - drawClouds() 호출 제거
  - 4개 실루엣 레이어 Graphics 코드 제거
  + 로딩된 PNG 레이어를 Image로 배치
  + (선택) 마우스 기반 미세 parallax 효과
  ※ 물 렌더링 코드는 유지

GameScene.drawClouds() - 제거 가능
GameScene.drawDistantMountains() - 제거 가능
GameScene.drawBackgroundIslands() - 제거 가능
GameScene.drawIslandShape() - 제거 가능
GameScene.drawBackgroundForest() - 제거 가능
GameScene.drawForegroundTrees() - 제거 가능
```

### 세트 ID 전달 패턴

```typescript
// 방법 1: Scene data로 전달
this.scene.start("GameScene", { backgroundSet: selectedSetId })

// 방법 2: Registry 사용
this.registry.set("backgroundSet", selectedSetId)

// 방법 3: 전역 상태
// GameScene.create()에서 직접 선택
```

## 히스토리 컨텍스트 (thoughts/에서)

- `thoughts/shared/research/2026-02-09-parallax-background-map-system.md` - 이전 연구: 현재 배경 시스템 분석 완료, 8세트 에셋 구조 문서화 완료, 구현 방향 제안
- `thoughts/shared/research/2026-01-22-worms-game-architecture-research.md` - 초기 아키텍처 연구: Turborepo 모노레포, Phaser 3 + Next.js 통합
- `thoughts/shared/plans/2026-01-23-worms-game-mvp-implementation.md` - MVP 구현 계획: Phase 1-8 모두 완료 상태

## 관련 연구

- `thoughts/shared/research/2026-02-09-parallax-background-map-system.md` - 이전 parallax 배경 연구 (본 연구의 전 단계)

## 웹 연구 소스

- [Ourcade - Add Pizazz with Parallax Scrolling in Phaser 3](https://blog.ourcade.co/posts/2020/add-pizazz-parallax-scrolling-phaser-3/)
- [Josh Morony - How to Create a Parallax Background in Phaser](https://www.joshmorony.com/how-to-create-a-parallax-background-in-phaser/)
- [Phaser - How I optimized my Phaser 3 action game in 2025](https://phaser.io/news/2025/03/how-i-optimized-my-phaser-3-action-game-in-2025)
- [GameMaker - Creating Depth & Immersion: Parallax](https://gamemaker.io/en/blog/creating-depth-and-immersion-parallax)
- [Phaser Discourse - TileSprite or Image?](https://phaser.discourse.group/t/tilesprite-or-image/7169)
- [CraftPix - Free Cartoon Parallax 2D Backgrounds](https://craftpix.net/freebies/free-cartoon-parallax-2d-backgrounds/)

## 미해결 질문

1. **전경 레이어 처리**: 에셋의 전경 레이어(1_layer.png)가 지형/캐릭터를 가릴 수 있음 → 전경 레이어를 사용할지, 생략할지 결정 필요
2. **물 영역과 에셋 하단 조화**: 에셋 하단부와 게임의 물 렌더링(Y633~720)이 시각적으로 자연스럽게 연결되는지 확인 필요
3. **데코레이션 유지 여부**: 기존 코드 생성 나무/바위/풀이 새 배경 스타일과 어울리는지
4. **재시작 시 배경**: 같은 배경 유지 vs 새로 랜덤 선택
5. **마우스 parallax 강도**: 고정 카메라에서 미세 이동 효과의 적절한 강도 결정
