# LCP 최적화 시도 — 병목 분석 기록 (2026-03-16)

## 측정 환경
- 대상: dev.goosetire.com (index 페이지)
- 도구: Chrome Lighthouse (시크릿 모드, 3회 측정)
- 모드: 모바일 (Slow 4G: 1.6 Mbps, RTT 150ms, CPU 4x slowdown)

## LCP 수치 변화
| 시점 | LCP | LCP breakdown (이미지) |
|---|---|---|
| 개선 전 | 4.1s | 미측정 |
| 개선 후 | 4.4s (오차 범위) | TTFB 50ms + load delay 50ms + load 20ms + render 270ms = **390ms** |

→ **이미지 로딩 자체는 390ms로 빠름.** 4.4s 중 나머지 ~4s는 이미지가 아닌 다른 곳에서 소요.

## LCP 요소
- `img.object-contain` — 메인 배너 이미지 (Embla Carousel 첫 번째 슬라이드)
- CloudFront CDN → `/_next/image` 최적화 경유, WebP 포맷
- fetchPriority: high 정상 적용 확인 (request header `priority: u=1, i`)

## 적용한 최적화 (코드 레벨)

### 1. 데스크톱 배너 `<img loading="lazy">` → `next/image` + `priority` 전환
- LCP 요소에 lazy loading이 걸려있던 치명적 문제 수정
- `fetchPriority="high"` 명시 추가 (Lighthouse 권고 반영)

### 2. 배너 Suspense 경계 분리
- 기존: `BannerSection` 하나에서 `Promise.all`로 데스크톱(sharp 처리) + 모바일 동시 fetch
- 문제: 데스크톱 배너의 sharp 엣지컬러 추출(이미지별 fetch + 픽셀 분석)이 끝날 때까지 모바일 배너 HTML 스트리밍이 블로킹됨
- 해결: `BannerSectionDesktop` / `BannerSectionMobile`로 분리, 각각 독립 Suspense 경계 적용

### 3. Noto Sans KR 폰트 weight 제한
- weight 미지정 → `['400', '500', '700']`만 로드
- Network dependency tree에서 woff2 파일 수 12개 → 감소
- Render blocking requests: 1,520ms → 870ms

### 4. 레거시 CSS 제거 + Footer lazy load
- globals.css에서 미사용 Ant Design 셀렉터 ~80줄 제거
- Footer를 dynamic import로 전환

## LCP를 더 줄이지 못한 이유 — 병목 분석

### 핵심: render-blocking CSS 870ms가 지배적
```
HTML 도착 (TTFB 50ms)
  → CSS 5개 파일 다운로드 대기 (93.9 KiB, 870ms 블로킹)
    → 72.3 KiB: Tailwind CSS 출력물 (전체 유틸리티 클래스)
    → 나머지: 컴포넌트 CSS 청크
  → CSS 파싱 완료 후 렌더 시작
  → LCP 이미지 렌더 (390ms)
```
- Slow 4G (1.6 Mbps)에서 93.9 KiB CSS 다운로드에 ~870ms
- Next.js가 자동 생성하는 CSS 청크 → 코드 레벨에서 분할/로딩 전략 제어 불가
- Tailwind CSS는 사용하는 유틸리티 클래스 수에 비례 → 프로젝트 규모에 따른 구조적 크기

### CSS → 폰트 체인 (LCP에 직접 영향 없음 확인)
- CSS 파싱 완료 후에야 한국어 폰트(unicode-range) 다운로드 시작
- `next/font/google`이 latin 서브셋만 preload, 한국어 글리프는 on-demand 로드
- 단, `font-display: swap` 적용되어 있고 **LCP 요소가 텍스트가 아닌 이미지**이므로 LCP에 직접 영향 없음

### experimental.optimizeCss 시도 → 실패
- critters 기반 critical CSS 인라이닝 적용
- 결과: 오히려 악화 (Performance 45점, LCP 6.3s) → 즉시 롤백
- 원인 추정: critters의 critical CSS 추출이 Embla Carousel + Tailwind 조합에서 부정확하게 동작

## 주요 서비스 동일 조건 비교 (Lighthouse 모바일)

| 사이트 | Performance |
|---|---|
| **구스타이어** | 55 |
| **마켓컬리** | 50점대 |
| **무신사** | 40점대 |
| **당근** | 15점대 |

→ Lighthouse 모바일(Slow 4G + CPU 4x)에서 국내 주요 서비스 대부분 낮은 점수. **LCP의 지배 요소가 코드가 아닌 CSS/JS 총량 + 네트워크 조건**이기 때문.

## 결론
- **코드 레벨에서 할 수 있는 LCP 최적화는 완료** — 이미지 로딩 자체는 390ms로 충분히 빠름
- **LCP 4.4s의 병목은 render-blocking CSS(870ms)** — Next.js + Tailwind 프레임워크 구조에서 코드 레벨로 제어할 수 없는 영역
- **추가 개선은 인프라 레벨이 필요** — CDN 엣지 렌더링, CSS 번들 전략 변경(프레임워크 레벨), 또는 Tailwind → 경량 CSS 솔루션 전환 등
