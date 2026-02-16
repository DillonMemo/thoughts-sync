---
date: 2026-02-16T17:19:42+0900
researcher: arta1069@gmail.com
git_commit: 726fe8601543e0ed40363062f06ef43b8a712b9c
branch: main
repository: Whoosh-Bang
topic: "모바일 반응형 UI 및 화면 회전(orientation) 전략 연구"
tags: [research, codebase, mobile, responsive, orientation, lobby, game, phaser]
status: complete
last_updated: 2026-02-16
last_updated_by: arta1069
---

# 연구: 모바일 반응형 UI 및 화면 회전(orientation) 전략

**날짜**: 2026-02-16T17:19:42+0900
**연구자**: arta1069@gmail.com
**Git 커밋**: 726fe8601543e0ed40363062f06ef43b8a712b9c
**브랜치**: main
**리포지토리**: Whoosh-Bang

## 연구 질문

1. 로비 화면이 모바일, PC를 고려하여 반응형으로 처리 필요
2. 게임 화면도 가로모드에 최적화 하고 플레이어 이동 조작 UI 편의도 제공 필요
3. (중요) 유저가 가로모드로 디바이스를 전환하게 유도 하는것이 아닌 프로젝트 화면 자체가 디스플레이 회전 되어 시작 하는게 가능한지 검토

## 요약

### 현재 상태
- **로비 화면**: 반응형 처리가 거의 없음. `md:max-w-86` 1개만 존재. PC 레이아웃 기반으로 모바일에서 UI 요소가 겹치거나 사용하기 어려움
- **게임 화면**: 1280x720 고정 해상도 + `Phaser.Scale.FIT` 모드. 모바일 세로모드에서 "가로로 회전해주세요" 오버레이 표시. 별도 터치 조작 UI 없음 (마우스 드래그 에이밍 방식)
- **화면 회전**: Screen Orientation API `lock()` 사용 없음. 현재는 사용자에게 수동 회전을 유도하는 오버레이만 존재

### 화면 강제 회전 검토 결과

| 전략 | Android | iOS iPhone | iOS iPad | 실용성 |
|------|---------|-----------|----------|--------|
| Screen Orientation API `lock()` | ✅ Fullscreen 필요 | ❌ 미지원 | ❌ lock() 비활성 | 중간 |
| Fullscreen + Orientation Lock | ✅ 가능 | ❌ Fullscreen 미지원 | ⚠️ 부분적 | 중간 |
| CSS transform rotate(90deg) | ✅ 가능 | ✅ 가능 | ✅ 가능 | 낮음 (터치 좌표 역전) |
| PWA manifest orientation | ✅ 홈화면 추가 시 | ❌ 무시됨 | ❌ 무시됨 | 낮음 |
| viewport meta | ❌ orientation 제어 불가 | ❌ 불가 | ❌ 불가 | 없음 |

**결론**: iOS iPhone에서 화면을 강제로 가로로 회전시키는 것은 **기술적으로 불가능**. Android에서는 Fullscreen API + Orientation Lock 조합으로 가능하나, iOS 사용자 비율을 고려하면 **하이브리드 전략**이 필요함.

## 상세 발견 사항

### 1. 로비 화면 현재 상태

#### 레이아웃 구조 (`apps/web/src/components/lobby/LobbyContent.tsx:70-119`)
- 풀스크린 컨테이너: `h-screen w-screen overflow-hidden`
- 배경: Next.js `<Image>` + `object-cover` + 어두운 오버레이 (`bg-black/30`)
- **절대 위치 기반 배치**:
  - 좌상단: ProfileCard + WalletLinkButton (z-10)
  - 중앙: CharacterDisplay
  - 하단 중앙: CharacterSelector
  - 우하단: PlayButton (`absolute bottom-12 right-12`)

#### 반응형 처리 현황
- **`md:max-w-86`** (`LobbyContent.tsx:85`) - 유일한 md breakpoint 사용
- **`sm:max-w-sm`** (`WalletLinkButton.tsx:161`) - 다이얼로그 너비 제한
- Dialog 컴포넌트에 `sm:text-left`, `sm:flex-row` 등 기본 반응형 존재
- **모바일 전용 레이아웃이나 breakpoint 분기 없음**

#### 모바일에서의 문제점
- PlayButton이 `absolute bottom-12 right-12`로 고정되어 모바일에서 캐릭터 셀렉터와 겹칠 수 있음
- ProfileCard가 좌상단에 고정되어 모바일 화면을 많이 차지함
- CharacterSelector가 `absolute bottom-6 left-1/2`로 하단 고정되어 PlayButton과 겹침
- 터치 영역이 작은 UI 요소들이 있음 (프로필 카드 내 버튼 등)

### 2. 게임 화면 현재 상태

#### Phaser 게임 설정 (`packages/game-core/src/index.ts:11-41`)
- 고정 해상도: `1280 x 720` (16:9 가로 비율)
- 스케일 모드: `Phaser.Scale.FIT` + `Phaser.Scale.CENTER_BOTH`
- Matter.js 물리 엔진 (중력: `{ x: 0, y: 1 }`)

#### 입력 처리 (`packages/game-core/src/systems/InputController.ts`)
- **WASD 키보드**: W(점프), A(좌), D(우)
- **마우스 드래그 에이밍**: pointerdown → pointermove → pointerup 패턴
  - 드래그 방향/거리로 각도(-30~90°)/파워(10~100%) 계산
  - 최대 드래그 거리: 150px
- **우클릭**: 에이밍 취소
- **1, 2, 3 키**: 무기 선택
- **별도 모바일 터치 조작 UI 없음** - Phaser가 touch→pointer 자동 변환

#### 게임 내 UI (`packages/game-core/src/ui/`)
- **GameHUD**: 라운드 표시, 타이머, 바람 정보, 이동 게이지
- **AimingUI**: 조준선, 파워 인디케이터, 각도/파워 텍스트
- **WeaponSelector**: 좌하단 3개 무기 박스 (50x48px, 클릭 가능)
- **WindVisualizer**: 바람 파티클 효과
- 모든 UI는 고정 픽셀 크기 (1280x720 기준)

#### 모바일 감지 및 세로모드 오버레이 (`apps/web/src/components/game/PhaserGame.tsx:23-115`)
- User Agent 정규식 + `window.innerWidth <= 768`으로 모바일 감지
- `window.innerHeight > window.innerWidth`로 세로모드 판단
- 세로모드 시 전체화면 오버레이: "화면을 가로로 회전해주세요" 메시지

### 3. 화면 강제 회전 기술 검토

#### 3-1. Screen Orientation API
```javascript
await screen.orientation.lock('landscape');
```
- **Android Chrome/Samsung Internet**: Fullscreen 모드에서 작동
- **iOS Safari**: `lock()` 메서드 미지원 (iOS 16.4에서 `orientation.type`, `angle`, `onchange`만 추가)
- **선행 조건**: 반드시 fullscreen 모드가 활성화되어야 함
- **사용자 인터랙션 필요**: 자동 실행 불가

#### 3-2. CSS Transform Rotate(90deg) 전략
```css
@media (orientation: portrait) {
  body {
    transform: rotate(90deg);
    transform-origin: left top;
    width: 100vh;
    height: 100vw;
  }
}
```
- **장점**: 모든 브라우저에서 시각적으로 동작
- **치명적 단점**:
  - 터치 이벤트 좌표가 원본(세로) 좌표계 기준으로 전달됨 → 좌표 역전
  - Phaser 내부 입력 시스템과 충돌 (Phaser는 DOM 좌표 → 게임 좌표 변환을 자체 처리)
  - 스크롤 방향 역전
  - `vw`, `vh` 단위가 원본 방향 기준

#### 3-3. Fullscreen API + Orientation Lock 조합
```javascript
await document.documentElement.requestFullscreen();
await screen.orientation.lock('landscape');
```
- **Android**: 완전 작동 (사용자 클릭 이벤트 내에서 호출 필요)
- **iOS iPhone**: `requestFullscreen()` 자체가 미지원 → 불가능
- **iOS iPad**: Fullscreen 지원하지만 `lock()` 미지원

#### 3-4. PWA Manifest Orientation
```json
{ "orientation": "landscape", "display": "standalone" }
```
- **Android**: 홈 화면에 추가한 경우에만 작동
- **iOS**: manifest orientation 설정 무시

### 4. 권장 하이브리드 전략

iOS에서 강제 회전이 불가능하므로, **플랫폼별 최선의 경험**을 제공하는 하이브리드 접근이 필요:

#### 전략 A: 조건부 Orientation Lock (권장)
1. **Android**: 게임 진입 시 Fullscreen + Orientation Lock 시도
   - "전체화면으로 전환" 버튼 → 클릭 시 fullscreen + lock
   - 성공하면 자동으로 가로모드 전환
2. **iOS**: CSS transform rotate(90deg) + 터치 좌표 보정
   - 세로모드에서 화면 자체를 90도 회전
   - Phaser 입력과의 호환성 위해 좌표 변환 레이어 필요
   - **복잡도가 매우 높고 버그 위험**
3. **모든 플랫폼 공통**: 세로모드 진입 시 가로모드 유도 오버레이 유지 (현재와 동일)

#### 전략 B: CSS 회전 + 게임 엔진 대응 (실험적)
1. 세로모드 감지 시 `#game-container`에 CSS `transform: rotate(90deg)` 적용
2. Phaser의 `input.scaleManager`를 커스텀하여 좌표 변환
3. **위험성**: Phaser 내부 좌표 시스템과 충돌, 디버깅 난이도 극도로 높음

#### 전략 C: 유도 오버레이 강화 (가장 안전, 현실적)
1. 현재 "가로로 회전해주세요" 오버레이를 유지
2. Android에서는 "전체화면 전환" 버튼 추가 → 전체화면 + orientation lock
3. iOS에서는 애니메이션이 포함된 회전 유도 오버레이 표시
4. **장점**: 구현 단순, 모든 플랫폼에서 안정적, Phaser와 충돌 없음

## 코드 참조

### 로비 화면
- `apps/web/src/app/lobby/page.tsx` - 로비 서버 컴포넌트 (인증, 데이터 조회)
- `apps/web/src/components/lobby/LobbyContent.tsx:70-119` - 로비 메인 레이아웃
- `apps/web/src/components/lobby/ProfileCard.tsx:99-211` - 프로필 카드 UI
- `apps/web/src/components/lobby/CharacterSelector.tsx:15-80` - 캐릭터 선택 패널
- `apps/web/src/components/lobby/CharacterDisplay.tsx:10-34` - 캐릭터 표시
- `apps/web/src/components/lobby/PlayButton.tsx:11-35` - 플레이 버튼
- `apps/web/src/components/lobby/WalletLinkButton.tsx:25-201` - 지갑 연동 UI

### 게임 화면
- `apps/web/src/components/game/PhaserGame.tsx:12-118` - Phaser 마운트 + 모바일 감지
- `apps/web/src/components/game/GameWrapper.tsx:25-150` - 게임 상태 관리
- `packages/game-core/src/index.ts:11-41` - Phaser 게임 설정 (1280x720, FIT)
- `packages/game-core/src/systems/InputController.ts:9-165` - 키보드/마우스 입력
- `packages/game-core/src/ui/GameHUD.ts` - 게임 HUD
- `packages/game-core/src/ui/WeaponSelector.ts` - 무기 선택 UI
- `packages/game-core/src/ui/AimingUI.ts` - 조준 UI

### 스타일/설정
- `apps/web/src/app/globals.css` - 전역 CSS (Tailwind v4, 커스텀 애니메이션)
- `apps/web/postcss.config.mjs` - PostCSS 설정
- `apps/web/src/app/layout.tsx` - 루트 레이아웃 (viewport 설정 없음)

## 아키텍처 문서화

### 현재 반응형 패턴
- Tailwind CSS v4 기본 breakpoints 사용 (sm: 640px, md: 768px)
- `md:` 사용 1개소, `sm:` 사용 3개소 (대부분 Dialog 컴포넌트)
- 로비는 절대 위치 기반 레이아웃 → 반응형 전환 어려움
- 게임은 Phaser FIT 모드로 자동 스케일 → 레이아웃 자체는 반응형이나 UI 크기 고정

### 현재 모바일 처리 패턴
- React state 기반 모바일/세로모드 감지 (CSS media query 미사용)
- User Agent sniffing + 화면 크기 비교
- 게임 화면에서만 세로모드 오버레이 표시
- 로비 화면에는 모바일 관련 처리 없음

### Phaser 입력 시스템
- `Phaser.Input.Pointer`로 마우스/터치 통합 처리
- `pointerdown`, `pointermove`, `pointerup` 이벤트
- WASD 키보드 입력 (모바일에서 사용 불가)
- 무기 선택: 1/2/3 키보드 + WeaponSelector 클릭
- 별도 가상 조이스틱/터치 버튼 없음

## 히스토리 컨텍스트 (thoughts/에서)

### 초기 아키텍처 연구
- `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md`
  - PWA + Capacitor 크로스 플랫폼 전략 논의
  - Phaser RESIZE 모드로 반응형 설계 계획 (현재는 FIT 모드 사용)
  - 모바일 세로모드 오버레이 추가 예정으로 기록

### MVP 구현 계획
- `thoughts/arta1069/plans/2026-01-23-worms-game-mvp-implementation.md`
  - 반응형 UI로 다양한 화면 크기 지원 계획
  - Phaser 캔버스 100% 채우기 + 모바일 세로모드 오버레이
  - 검증 항목: "반응형으로 화면 크기에 맞게 캔버스 조정됨"

### 최근 밸런스 조정
- `thoughts/arta1069/research/2026-02-16-game-balance-weapon-movement-research.md`
  - 이동 게이지 시스템 도입 (총량 250, 이동 1/프레임, 점프 20/회)
  - GameHUD에 게이지 바 추가 완료

## 관련 연구
- `thoughts/arta1069/research/2026-01-22-worms-game-architecture-research.md` - 초기 아키텍처 연구 (PWA, Capacitor 전략)
- `thoughts/arta1069/plans/2026-01-23-worms-game-mvp-implementation.md` - MVP 구현 계획 (반응형 설계)
- `thoughts/arta1069/research/2026-02-16-game-balance-weapon-movement-research.md` - 밸런스 연구 (이동 게이지)

## 미해결 질문

1. **CSS 회전 전략 채택 여부**: iOS에서 CSS transform rotate를 사용할 경우, Phaser의 입력 좌표 시스템과의 충돌을 어떻게 해결할 것인가?
2. **로비 모바일 레이아웃**: 절대 위치 기반의 현재 레이아웃을 flex/grid 기반으로 재구성할 것인가, 아니면 모바일 전용 레이아웃을 별도로 만들 것인가?
3. **게임 내 터치 조작 UI**: 가상 조이스틱/D-pad를 추가할 것인가, 화면 터치 영역 분할 방식을 사용할 것인가?
4. **PWA 설정**: manifest.json을 추가하여 Android 홈 화면 추가 시 자동 가로모드를 지원할 것인가?
5. **전략 선택**: 전략 A(조건부 lock), B(CSS 회전), C(유도 오버레이 강화) 중 어느 것을 채택할 것인가?
