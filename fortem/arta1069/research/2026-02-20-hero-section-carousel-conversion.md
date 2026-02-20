---
date: 2026-02-20T18:00:00+09:00
researcher: arta1069@gmail.com
git_commit: 9ad7173a197eb194707def1f2c8b922a42da1cb4
branch: feat/web/store
repository: fortem
topic: "Hero Section을 Carousel 기반으로 변환 - 2개 영상 슬라이드"
tags: [research, codebase, hero-section, carousel, embla-carousel, main-page]
status: complete
last_updated: 2026-02-20
last_updated_by: arta1069
---

# 연구: Hero Section을 Carousel 기반으로 변환

**날짜**: 2026-02-20T18:00:00+09:00
**연구자**: arta1069@gmail.com
**Git 커밋**: 9ad7173a197eb194707def1f2c8b922a42da1cb4
**브랜치**: feat/web/store
**리포지토리**: fortem

## 연구 질문
현재 단일 영상인 HeroSection을 Carousel 컴포넌트를 활용하여 2개의 콘텐츠(영상) 슬라이드로 변환하기 위한 코드베이스 연구

## 요약

HeroSection(`apps/web/components/sections/hero-section.tsx`)은 현재 단일 배경 비디오 + 타이틀 + CTA 버튼 구조의 컴포넌트입니다. Carousel 컴포넌트(`apps/web/components/ui/carousel.tsx`)는 embla-carousel-react 기반으로, Context API를 통해 `current`, `count`, `scrollPrev`, `scrollNext` 등의 상태를 제공합니다. 프로젝트 내에서 이미 다양한 패턴으로 Carousel이 사용되고 있어, 기존 패턴을 참고하여 변환할 수 있습니다.

Figma 디자인(`5230:17058`)에서 "Hero Banner_Store" 인스턴스를 확인했으며, 좌우 CarouselPrevious/CarouselNext 화살표 버튼이 포함된 캐러셀 형태로 디자인되어 있습니다.

## 상세 발견 사항

### 1. 현재 HeroSection 구조

**파일**: `apps/web/components/sections/hero-section.tsx`

```
section (relative, 배경 영역)
  ├── div.absolute (배경 비디오 컨테이너, -z-1)
  │   └── video (playsInline, autoPlay, loop, muted)
  │       ├── source (main-hero.webm, video/webm)
  │       └── source (main-hero.mp4, video/mp4)
  ├── div.relative (콘텐츠 컨테이너, z-8)
  │   ├── div (텍스트 컨테이너)
  │   │   ├── h1 (heroTitle → "Get Ready to Play for Real")
  │   │   └── p (heroDescription → "Discover and collect items for your game")
  │   └── Link (browseItems → "Browse Items", href="/items")
```

- 크기: `md:h-[33.75rem] h-100` (모바일 400px, 데스크톱 540px)
- 그라디언트 오버레이: CSS `before:` 가상 요소로 `bg-gradient-to-b from-transparent to-background/70`
- 여백: `px-7.5 pb-7.5 mb-9.5 mt-2`
- 비디오 poster: `/images/banner/main.png`
- i18n: `useTranslations()` 훅으로 번역 키 사용

### 2. Figma 디자인 분석 (Hero Banner_Store)

**노드**: `5230:17058` - "Hero Banner_Store" 인스턴스
**크기**: 1560 x 540px (1920px 컨테이너에서 좌우 180px 패딩)

디자인에서 확인된 요소:
- **배경**: 컨베이어 벨트 위의 3D 게임 에셋 이미지/영상
- **좌측 화살표**: CarouselPrevious 버튼 (원형, 32x32px, 배경색 background, 테두리 border)
- **우측 화살표**: CarouselNext 버튼 (원형, 32x32px, 동일 스타일)
- **텍스트 컨테이너** (하단 좌측):
  - 타이틀: "Best Price, Zero Risk" (30px Bold, leading-9)
  - 설명: "Buy and sell verified game codes with confidence." (14px Medium, leading-5)
- **CTA 버튼**: "Browse Store" (bg-primary, 40px 높이, circle 아이콘 포함)
- **그라디언트 오버레이**: `linear-gradient(180deg, background/21% 60%, background/70% 100%)`

### 3. Carousel 컴포넌트 구조

**파일**: `apps/web/components/ui/carousel.tsx`

Export 목록:
| 컴포넌트 | 역할 |
|---------|------|
| `Carousel` | 루트, Context Provider, `opts`/`plugins` prop |
| `CarouselContent` | 슬라이드 트랙 (`overflow-hidden` + `flex`) |
| `CarouselItem` | 개별 슬라이드 (`basis-full` 기본) |
| `CarouselPrevious` | 이전 버튼 (`absolute`, 좌측 `-left-12`) |
| `CarouselNext` | 다음 버튼 (`absolute`, 우측 `-right-12`) |
| `CarouselDots` | 점 인디케이터 (1개 이하면 숨김) |
| `useCarousel` | Context 접근 훅 (`current`, `count`, `api`) |

Context 제공 값: `carouselRef`, `api`, `scrollPrev`, `scrollNext`, `canScrollPrev`, `canScrollNext`, `current`, `count`

### 4. 프로젝트 내 Carousel 사용 패턴

#### 패턴 A: AutoScroll 무한 스크롤 (featured-collection-section, just-listed-section)
```tsx
plugins={[AutoScroll({ speed: 1.1, startDelay: 300, stopOnInteraction: false, stopOnMouseEnter: true })]}
opts={{ align: "center", loop: true }}
```
- `embla-carousel-auto-scroll` 패키지 사용 (`^8.6.0`)
- Navigation 버튼 없음, 자동 스크롤만

#### 패턴 B: 카드 캐러셀 + Dots (examples/carousel/page.tsx)
```tsx
opts={{ align: "center", loop: true, containScroll: false }}
```
- `CarouselDots`로 네비게이션
- 반응형 `basis` 클래스 (85% → 75% → 70% → 65%)

#### 패턴 C: 사이드바 연동 + 수동 autoplay (examples/carousel/page.tsx)
```tsx
opts={{ loop: true }}
// setTimeout + api.scrollNext()로 수동 autoplay
// api.rootNode() 이벤트 리스너로 hover pause
```
- `useCarousel()` 훅으로 `current`, `api` 접근
- CSS animation 기반 프로그레스 바

### 5. 비디오 에셋 현황

**`public/videos/`**:
| 파일 | 용도 |
|------|------|
| `main-hero.webm` | 히어로 배경 (Chrome/Firefox/Edge) |
| `main-hero.mp4` | 히어로 배경 (Safari) |
| `dna-chart-bg.webm` | DNA 차트 배경 |

**비디오 태그 공통 속성**: `playsInline autoPlay loop muted`

**히어로 전용 속성**: `preload="metadata"`, `poster="/images/banner/main.png"`, 듀얼 source (webm + mp4)

### 6. 메인 페이지 섹션 구성

**파일**: `apps/web/app/[locale]/page.tsx`

| 순서 | 컴포넌트 | 설명 |
|------|----------|------|
| 1 | `HeroSection` | 배경 비디오 + CTA |
| 2 | `StatisticsSection` | 거래량 통계 |
| 3 | `FeaturedCollectionSection` | 추천 컬렉션 캐러셀 |
| 4 | `MarketingSection` | 마케팅 카드 |
| 5 | `JustListedSection` | 최근 등록 아이템 캐러셀 |
| 6 | `LiveTransactionsSection` | 실시간 트랜잭션 |
| 7 | `SocialBannerSection` | 소셜 링크 배너 |

### 7. i18n 번역 키 (현재)

**파일**: `apps/web/messages/en.json`

| 키 | 현재 값 |
|----|--------|
| `heroTitle` | "Get Ready to Play for Real" |
| `heroDescription` | "Discover and collect items for your game" |
| `browseItems` | "Browse Items" |
| `yourBrowserDoesNotSupportTheVideoTag` | "Your browser does not support the video tag." |

Figma 디자인의 Store 슬라이드에서 확인된 텍스트 (새로 추가 필요):
- 타이틀: "Best Price, Zero Risk"
- 설명: "Buy and sell verified game codes with confidence."
- CTA: "Browse Store"

## 코드 참조

- `apps/web/components/sections/hero-section.tsx:1-65` - 현재 HeroSection 전체
- `apps/web/components/ui/carousel.tsx:1-283` - Carousel 컴포넌트 정의
- `apps/web/app/[locale]/page.tsx` - 메인 페이지에서 HeroSection import/사용
- `apps/web/messages/en.json:3-5` - hero 관련 번역 키
- `apps/web/components/sections/featured-collection-section.tsx:145-169` - AutoScroll 캐러셀 패턴
- `apps/web/app/examples/carousel/page.tsx:437-495` - Dots 캐러셀 패턴
- `apps/web/app/examples/carousel/page.tsx:360-424` - 사이드바 연동 패턴

## 아키텍처 문서화

### 캐러셀 기술 스택
- `embla-carousel-react` `^8.6.0` - 코어 캐러셀 라이브러리
- `embla-carousel-auto-scroll` `^8.6.0` - 자동 스크롤 플러그인
- Context API 기반 상태 공유 (`CarouselContext`)

### Carousel 컴포넌트 관계
```
Carousel (Provider)
  ├── CarouselContent (ref=carouselRef, overflow-hidden)
  │   └── CarouselItem[] (basis-full 또는 커스텀 basis)
  ├── CarouselPrevious (optional, absolute positioned)
  ├── CarouselNext (optional, absolute positioned)
  └── CarouselDots (optional, count <= 1이면 hidden)
```

### 히어로 섹션 스타일 패턴
- 배경: absolute positioned video + CSS gradient overlay (before pseudo-element)
- 콘텐츠: relative positioned, flex-col justify-end (하단 배치)
- 반응형: `md:h-[33.75rem] h-100` (540px/400px)

## 히스토리 컨텍스트 (thoughts/에서)

hero section, carousel, banner 관련 기존 문서는 없습니다.
store 관련으로는 navbar 게임 상품 스토어 가드 관련 문서 2개가 존재합니다:
- `thoughts/arta1069/plans/2026-02-20-navbar-game-product-store-guard.md`
- `thoughts/arta1069/research/2026-02-20-navbar-game-product-store-guard.md`

## 미해결 질문

1. 두 번째 슬라이드(Store 슬라이드)의 배경 영상 파일은 아직 `public/videos/`에 존재하지 않음 - 새 영상 에셋이 필요
2. 두 번째 슬라이드의 poster 이미지도 새로 필요할 수 있음
3. 캐러셀 자동 전환(autoplay) 필요 여부 - Figma에서는 수동 네비게이션(화살표)만 표시
4. 모바일 반응형에서 캐러셀 화살표 표시 여부 (현재 CarouselPrevious/Next는 `md:hidden` 등 처리 없음)
5. CarouselDots 사용 여부 (Figma 디자인에서는 dots가 보이지 않으나, 모바일에서 필요할 수 있음)
