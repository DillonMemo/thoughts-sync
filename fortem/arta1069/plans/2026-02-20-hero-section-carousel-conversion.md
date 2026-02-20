---
date: 2026-02-20T19:00:00+09:00
author: arta1069@gmail.com
branch: feat/web/store
repository: fortem
topic: "Hero Section Carousel 변환 구현 계획"
tags: [plan, hero-section, carousel, embla-carousel, main-page]
status: approved
research_ref: thoughts/arta1069/research/2026-02-20-hero-section-carousel-conversion.md
---

# Hero Section Carousel 변환 구현 계획

## 개요

현재 단일 배경 영상인 HeroSection을 embla-carousel 기반 2슬라이드 캐러셀로 변환한다. 기존 슬라이드(Items)를 유지하고 Store 슬라이드를 추가한다. 6초 자동 롤링, hover 시 일시정지, 좌우 화살표 버튼(hover 시 노출)을 구현한다.

## 현재 상태 분석

- `apps/web/components/sections/hero-section.tsx:1-65` — 서버 컴포넌트 (no `"use client"`), 단일 비디오 + 텍스트 + CTA 구조
- `apps/web/components/ui/carousel.tsx:1-283` — `"use client"` 클라이언트 컴포넌트, embla-carousel-react 기반
- `apps/web/app/examples/carousel/page.tsx:223-359` — 수동 autoplay 패턴 (setTimeout + hover pause/resume) 참조 코드
- `apps/web/messages/en.json` — i18n 번역 키 (단일 파일)
- `public/videos/main-hero-2.webm`, `public/videos/main-hero-2.mp4` — 두 번째 슬라이드 영상 에셋 이미 존재

### 핵심 발견:
- Carousel은 `"use client"` 필수 → HeroSection도 클라이언트 컴포넌트로 전환 필요
- `CarouselContent` 기본 `-ml-4`, `CarouselItem` 기본 `pl-3` → 풀 폭 슬라이드에 오버라이드 필요
- `CarouselPrevious`/`CarouselNext` 기본 위치 `-left-12`/`-right-12` (컨테이너 밖) → 내부 배치 오버라이드 필요
- 수동 autoplay 패턴이 `CarouselSidebar` 컴포넌트(examples)에 잘 구현되어 있어 그대로 차용 가능

## 원하는 최종 상태

HeroSection이 2개 슬라이드 캐러셀로 동작하며:
- 6초마다 자동으로 다음 슬라이드로 전환 (loop)
- hover 시 자동 롤링 일시정지, leave 시 남은 시간부터 재개
- 콘텐츠 영역 hover/enter 시에만 좌우 화살표 버튼 노출
- CarouselDots 미사용

## 하지 않을 것

- CarouselDots 추가 (요구사항에서 제외됨)
- 모바일 전용 UI 분기 (화살표 동작은 PC/모바일 동일)
- 그라디언트 스타일 변경 (기존 Tailwind 클래스 유지)
- 추가 패키지 설치 (수동 autoplay 패턴 사용)
- 기존 슬라이드 내용 변경
- `carousel.tsx` 공통 컴포넌트 수정

## 구현 접근 방식

단일 파일(`hero-section.tsx`) 전면 리라이트 + i18n 키 추가. 기존 `CarouselWithSidebar` 예시의 수동 autoplay 로직을 간소화하여 `HeroAutoplay` 내부 컴포넌트로 분리한다.

## 1단계: i18n 번역 키 추가

### 개요
두 번째 슬라이드(Store)에 필요한 번역 키를 `en.json`에 추가한다.

### 필요한 변경:

#### 1. i18n 파일
**파일**: `apps/web/messages/en.json`
**변경**: 3개 키 추가

```json
"heroStoreTitle": "Best Price, Zero Risk",
"heroStoreDescription": "Buy and sell verified game codes with confidence.",
"browseStore": "Browse Store"
```

`browseItems` 키(5번째 줄) 바로 아래에 삽입한다.

### 성공 기준:

#### 자동화된 검증:
- [ ] JSON 파싱 오류 없음
- [ ] 빌드 성공: `yarn build:web`

---

## 2단계: HeroSection 캐러셀 변환

### 개요
`hero-section.tsx`를 클라이언트 컴포넌트로 전환하고, Carousel 컴포넌트를 활용하여 2슬라이드 캐러셀로 재구성한다.

### 필요한 변경:

#### 1. HeroSection 컴포넌트 전면 리라이트
**파일**: `apps/web/components/sections/hero-section.tsx`

##### 1-1. `"use client"` 지시문 추가 및 import 변경

```tsx
"use client"

import { useEffect, useRef } from "react"
import { Link } from "@/i18n/navigation"
import {
  Carousel,
  CarouselContent,
  CarouselItem,
  CarouselPrevious,
  CarouselNext,
  useCarousel,
} from "@/components/ui/carousel"
import { cn } from "@/lib/utils"
import { useTranslations } from "next-intl"
```

##### 1-2. 슬라이드 데이터 정의

```tsx
const HERO_SLIDES = [
  {
    id: "hero-default",
    videoWebm: "/videos/main-hero.webm",
    videoMp4: "/videos/main-hero.mp4",
    poster: "/images/banner/main.png",
    titleKey: "heroTitle",
    descKey: "heroDescription",
    ctaKey: "browseItems",
    href: "/items",
  },
  {
    id: "hero-store",
    videoWebm: "/videos/main-hero-2.webm",
    videoMp4: "/videos/main-hero-2.mp4",
    poster: "/images/banner/main.png",
    titleKey: "heroStoreTitle",
    descKey: "heroStoreDescription",
    ctaKey: "browseStore",
    href: "/store",
  },
] as const
```

##### 1-3. HeroAutoplay 내부 컴포넌트 (수동 autoplay)

`examples/carousel/page.tsx:223-290`의 `CarouselSidebar` autoplay 로직을 간소화한 렌더리스 컴포넌트:

```tsx
function HeroAutoplay({ delay = 6000 }: { delay?: number }) {
  const { api, current } = useCarousel()
  const timerRef = useRef<ReturnType<typeof setTimeout>>(undefined)
  const startTimeRef = useRef(Date.now())
  const remainingRef = useRef(delay)
  const isHoveringRef = useRef(false)

  // Reset timer on slide change
  useEffect(() => {
    if (!api) return

    if (timerRef.current) clearTimeout(timerRef.current)

    remainingRef.current = delay

    if (!isHoveringRef.current) {
      startTimeRef.current = Date.now()
      timerRef.current = setTimeout(() => {
        api.scrollNext()
      }, delay)
    }

    return () => {
      if (timerRef.current) clearTimeout(timerRef.current)
    }
  }, [api, current, delay])

  // Pause/resume on mouse enter/leave
  useEffect(() => {
    if (!api) return
    const rootNode = api.rootNode()

    const handleMouseEnter = () => {
      isHoveringRef.current = true
      if (timerRef.current) clearTimeout(timerRef.current)
      const elapsed = Date.now() - startTimeRef.current
      remainingRef.current = Math.max(0, remainingRef.current - elapsed)
    }

    const handleMouseLeave = () => {
      isHoveringRef.current = false
      startTimeRef.current = Date.now()
      timerRef.current = setTimeout(() => {
        api.scrollNext()
      }, remainingRef.current)
    }

    rootNode.addEventListener("mouseenter", handleMouseEnter)
    rootNode.addEventListener("mouseleave", handleMouseLeave)

    return () => {
      rootNode.removeEventListener("mouseenter", handleMouseEnter)
      rootNode.removeEventListener("mouseleave", handleMouseLeave)
    }
  }, [api])

  return null
}
```

##### 1-4. HeroSection 메인 컴포넌트

```tsx
export default function HeroSection() {
  const translate = useTranslations()

  return (
    <Carousel
      opts={{ loop: true }}
      className="group/hero relative mb-9.5 mt-2"
    >
      <CarouselContent className="ml-0">
        {HERO_SLIDES.map((slide) => (
          <CarouselItem key={slide.id} className="pl-0">
            <div
              className={cn(
                "relative w-full md:h-[33.75rem] h-100 rounded-md overflow-hidden",
                "before:content-[''] before:absolute before:inset-0",
                "before:bg-gradient-to-b before:from-transparent before:to-background/70",
                "before:z-1",
                "flex flex-col justify-end",
                "px-7.5 pb-7.5"
              )}
            >
              {/* Background Video */}
              <div className="absolute inset-0">
                <video
                  playsInline
                  autoPlay
                  loop
                  muted
                  preload="metadata"
                  width="100%"
                  height="100%"
                  className="w-full h-full object-cover"
                  poster={slide.poster}
                >
                  <source src={slide.videoWebm} type="video/webm" />
                  <source src={slide.videoMp4} type="video/mp4" />
                  {translate("yourBrowserDoesNotSupportTheVideoTag")}
                </video>
              </div>

              {/* Content */}
              <div className="relative z-8 flex flex-col gap-5">
                <div className="flex flex-col gap-2.5">
                  <h1 className="md:text-3xl text-2xl font-extrabold text-foreground leading-[1.2]">
                    {translate(slide.titleKey)}
                  </h1>
                  <p className="text-sm md:font-medium font-normal text-foreground leading-[1.428571]">
                    {translate(slide.descKey)}
                  </p>
                </div>

                <Link
                  href={slide.href}
                  className={cn(
                    "flex items-center justify-center gap-2",
                    "h-10 px-4 rounded-md",
                    "bg-primary text-primary-foreground",
                    "text-sm font-medium",
                    "w-fit transition-colors",
                    "hover:bg-primary/90"
                  )}
                >
                  <span>{translate(slide.ctaKey)}</span>
                </Link>
              </div>
            </div>
          </CarouselItem>
        ))}
      </CarouselContent>

      {/* Navigation Arrows - visible on hover */}
      <CarouselPrevious
        className={cn(
          "absolute left-4 top-1/2 -translate-y-1/2",
          "opacity-0 group-hover/hero:opacity-100",
          "transition-opacity duration-300"
        )}
      />
      <CarouselNext
        className={cn(
          "absolute right-4 top-1/2 -translate-y-1/2",
          "opacity-0 group-hover/hero:opacity-100",
          "transition-opacity duration-300"
        )}
      />

      {/* Autoplay Controller */}
      <HeroAutoplay delay={6000} />
    </Carousel>
  )
}
```

##### 구조 변경 요약

| 항목 | Before | After |
|------|--------|-------|
| 루트 요소 | `<section>` | `<Carousel>` (`group/hero`, `mb-9.5 mt-2`) |
| 비디오 컨테이너 | 단일 `div.absolute` `-z-1` | 각 슬라이드 내 `div.absolute` |
| 콘텐츠 | 단일 텍스트+CTA | 슬라이드별 텍스트+CTA (데이터 배열 기반) |
| 네비게이션 | 없음 | CarouselPrevious/CarouselNext (hover 노출) |
| 자동 전환 | 없음 | HeroAutoplay (6초, hover pause) |
| 컴포넌트 유형 | 서버 컴포넌트 | 클라이언트 컴포넌트 (`"use client"`) |
| rounded-md | section에 직접 | 슬라이드 내부 div + `overflow-hidden` |

### 성공 기준:

#### 자동화된 검증:
- [ ] 빌드 성공: `yarn build:web`
- [ ] 타입 체크 통과 (빌드에 포함)

#### 수동 검증:
- [ ] 메인 페이지 접속 시 첫 번째 슬라이드(Items) 정상 표시
- [ ] 6초 후 두 번째 슬라이드(Store)로 자동 전환
- [ ] 마지막 슬라이드 → 첫 슬라이드 순환 (loop)
- [ ] 콘텐츠 hover 시 좌우 화살표 노출, 클릭 시 슬라이드 전환
- [ ] hover 시 자동 롤링 일시정지, leave 시 재개
- [ ] 첫 번째 슬라이드 "Browse Items" → `/items` 이동
- [ ] 두 번째 슬라이드 "Browse Store" → `/store` 이동
- [ ] 두 영상 모두 autoPlay, loop, muted 정상 동작
- [ ] 모바일 뷰포트에서 스와이프 정상 동작

---

## 테스트 전략

### 수동 테스트 단계:
1. `yarn dev:web`으로 로컬 실행
2. `/` 메인 페이지 접속
3. 첫 번째 슬라이드 내용 확인 ("Get Ready to Play for Real")
4. 6초 대기 → 자동 전환 확인 ("Best Price, Zero Risk")
5. 다시 6초 대기 → 첫 번째로 순환 확인
6. 영역에 마우스 hover → 화살표 노출 + 자동 전환 정지 확인
7. 화살표 클릭 → 슬라이드 전환 확인
8. 마우스 leave → 자동 전환 재개 확인
9. CTA 버튼 클릭 → 올바른 링크로 이동 확인
10. 브라우저 너비 축소 → 모바일 레이아웃 + 스와이프 동작 확인

## 참조

- 연구 문서: `thoughts/arta1069/research/2026-02-20-hero-section-carousel-conversion.md`
- 현재 HeroSection: `apps/web/components/sections/hero-section.tsx:1-65`
- Carousel 컴포넌트: `apps/web/components/ui/carousel.tsx:1-283`
- 수동 autoplay 참조 코드: `apps/web/app/examples/carousel/page.tsx:223-359`
- i18n 파일: `apps/web/messages/en.json`
