# 캐싱 전략 — 데이터 성격별 서버·클라이언트 이중 캐싱 설계

## 문제 상황

초기 구현은 데이터 성격 구분 없이 대부분을 매 요청마다 외부 API에서 가져오고 있었다.

- 하루에 수십 번 바뀌지 않는 상품·이벤트 데이터도 동시 방문자마다 각자 API 호출
- TanStack Query 기본 옵션 일괄 적용 → 탭 전환·창 포커스마다 불필요한 refetch 발생
- 개발자마다 캐시 옵션을 다르게 쓰는 컨벤션 부재

---

## 설계 기준: 데이터를 3가지로 분류

모든 데이터를 캐시하거나 모두 no-store로 처리하는 건 틀렸다고 판단했다.
**"얼마나 자주 바뀌는가"와 "누구의 데이터인가"** 두 축으로 분류했다.

| 유형 | 기준 | 전략 |
|------|------|------|
| 공개 정적 데이터 | 로그인 불필요, 갱신 주기 예측 가능 | ISR + On-Demand Revalidation |
| 매 요청 최신 데이터 | 로그인 불필요, 매 요청마다 최신 데이터 필요 | `force-dynamic` |
| 인증 동적 데이터 | 사용자별 개인 데이터 | `force-dynamic` + `cache: 'no-store'` |

---

## 서버 캐싱: ISR revalidate를 데이터 비즈니스 주기에 맞춤

"길게 or 짧게" 이분법이 아니라, **데이터가 실제로 바뀌는 비즈니스 주기**를 기준으로 설정했다.

| 페이지 | revalidate | 근거 |
|--------|-----------|------|
| 메인 (배너, 팝업, 추천상품) | 24시간 + On-Demand | 캠페인 단위 변경, 긴급 시 즉시 무효화 |
| 이벤트 목록/상세 | 1시간 + On-Demand | 관리자 수작업 업데이트, 변경 시 즉시 반영 |
| 상품 상세/이미지/유사상품 | 1시간 + On-Demand | 가격·재고 변동 대응, 변경 시 즉시 반영 |
| 매장 목록/상세 | 1주일 + On-Demand | 거의 안 바뀌지만 신규 매장 추가 시 즉시 반영 |
| 이용약관/개인정보처리방침 | 1주일 | 요청 시에만 변경, 시간 기반으로 충분 |
| 공지사항/FAQ | 1주일 | 요청 시에만 변경, 즉시성 불필요 |
| 마이페이지 | `force-dynamic` | 사용자별 개인 데이터, 캐시 불가 |

시간 기반 revalidate는 **안전망**이다. On-Demand가 실패하거나 누락되더라도 최대 revalidate 주기 내에 갱신이 보장된다.

### 레이어 분리로 컨벤션 강제

두 종류의 fetch 래퍼로 **함수 이름만 봐도 캐싱 의도가 드러나도록** 설계했다.

```ts
// 서버 전용 — ISR 옵션 + 태그 래퍼
const fetchServerAPI = (url, { revalidate, tags }) =>
  fetch(url, { next: { revalidate, tags } })

// 클라이언트 호출용 — 항상 최신
const fetchClientAPI = (url, options) =>
  fetch(url, { cache: 'no-store', ...options })
```

새로운 API를 추가할 때 래퍼 선택만으로 캐싱 의도가 결정되고, 실수로 인증 데이터를 캐시하거나 공개 데이터를 no-store로 처리하는 실수를 타입 수준에서 차단한다.

---

## On-Demand Revalidation: 시간 기반 캐싱의 한계 보완

ISR만으로는 "관리자가 데이터를 수정했는데 최대 1시간을 기다려야 반영된다"는 한계가 있다.
이를 보완하기 위해 **admin에서 데이터 변경 시 즉시 캐시를 무효화**할 수 있는 On-Demand Revalidation API를 설계했다.

### API 설계

Next.js `revalidateTag`를 활용한 태그 기반 캐시 무효화 API Route (`POST /api/revalidate`)를 구현했다.

- **시크릿 키 인증**으로 외부 무단 호출 방지
- **허용 태그 화이트리스트**로 임의 캐시 무효화 차단
- `fetchServerAPI` 유틸에 `tags` 옵션을 추가하여, 서버 컴포넌트 fetch에 태그를 **선언적으로** 부여할 수 있는 구조

### 태그 설계: 계층적 구조

```
dashboard (상위)
├── dashboard-banners
├── dashboard-popups
└── dashboard-promotions

event (상위)
├── event-list
└── event-detail-{id}

product (상위)
├── product-detail-{id}
└── product-images-{id}

agency (상위)
├── agency-list
└── agency-detail-{id}
```

- **상위 태그로 전체 무효화**: 관리자가 대시보드 전체를 갱신하고 싶을 때 `dashboard` 하나만 호출
- **하위 태그로 개별 무효화**: 특정 상품 하나만 수정했을 때 `product-detail-{id}`만 호출
- admin 측에서 호출 범위를 선택할 수 있어 **불필요한 캐시 버스팅을 최소화**

### 이중 캐싱 전략의 핵심

| | 시간 기반 (ISR) | 이벤트 기반 (On-Demand) |
|---|---|---|
| 역할 | 안전망 — 최악의 경우에도 일정 주기 내 갱신 보장 | 즉시성 — 데이터 변경 시점에 캐시 무효화 |
| 트리거 | revalidate 시간 경과 | admin API 호출 |
| 실패 시 | 이전 캐시 유지 (stale-while-revalidate) | ISR이 백업으로 동작 |

두 전략이 서로의 약점을 보완한다. On-Demand가 정상 동작하면 사용자는 항상 최신 데이터를 보고, 실패하더라도 ISR 주기 내에 갱신된다.

---

## 클라이언트 캐싱: TanStack Query 옵션을 데이터 특성별로 튜닝

### staleTime 기본값 선정 이유

```ts
defaultOptions: {
  queries: {
    staleTime: 60_000,  // 60초
    retry: false,
  }
}
```

`staleTime: 0`이면 창 포커스마다 refetch, `Infinity`면 최신성 보장 불가.
**60초는 "같은 탭에서 짧게 다른 페이지 갔다 돌아오는" 시나리오를 커버**하면서 데이터 신선도도 유지하는 트레이드오프다.

개별 훅에서는 데이터 특성에 따라 오버라이드한다.

```ts
useAuthInit:  { staleTime: 300_000 }  // 인증 정보 — 토큰 만료 주기보다 짧게 5분
usePassAuth:  { staleTime: 0, gcTime: 0 }  // 본인인증 — PASS 결과는 재사용 불가
useCartList:  { staleTime: 0 }        // 장바구니 — 수량 변경 즉시 반영 필수
```

### refetchOnWindowFocus 선별 비활성화

기본값(true)은 창 복귀 시마다 refetch를 유발한다. 아래는 의도적으로 끄는 게 맞다고 판단한 케이스다.

```ts
// 무한 스크롤 목록 — 창 복귀 시 스크롤 위치 초기화 방지
useMatchProductInfinite: { refetchOnWindowFocus: false }

// 상품 필터 리스트 — 거의 안 바뀌는 메타데이터, 불필요한 네트워크 낭비
useProductFilterList:    { refetchOnWindowFocus: false }
```

### keepPreviousData로 페이지 전환 깜빡임 제거

```ts
useMatchProductPage:  { placeholderData: keepPreviousData }
useSearchProductPage: { placeholderData: keepPreviousData }
```

페이지 번호 클릭 시 새 데이터가 올 때까지 이전 목록을 유지한다.
빈 화면 없이 데이터가 부드럽게 교체되는 UX.

---

## 트러블슈팅: Next.js 미들웨어 쿠키 오염 문제

마이페이지를 서버 컴포넌트로 전환하면서 예상치 못한 버그를 발견했다.

### 증상

서버 컴포넌트에서 `cookies().get('userToken')`이 value가 없는 빈 객체를 반환.
인증 API 호출 시 401 에러.

### 원인

미들웨어에서 도메인 쿠키 정리를 위해 `response.cookies.set()`을 사용하고 있었다.

```ts
// middleware.ts
response.cookies.set(cookieName, '', { domain: 'www.goosetire.com', maxAge: 0 })
```

`response.cookies.set()`은 응답 Set-Cookie 헤더에만 영향을 줄 것 같지만,
**실제로는 서버 컴포넌트로 전달되는 request cookies도 덮어쓴다.**

서버 액션은 별도 분기로 스킵되어 오랫동안 잠복해 있었던 버그다.

### 해결

```ts
// before: request cookies까지 오염
response.cookies.set(cookieName, '', { domain: 'www.goosetire.com', maxAge: 0 })

// after: 브라우저에만 삭제 지시, request cookies 영향 없음
response.headers.append(
  'Set-Cookie',
  `${cookieName}=; domain=www.goosetire.com; maxAge=0; path=/`
)
```

`response.cookies.set()` 대신 `response.headers.append('Set-Cookie', ...)`를 직접 사용하면 request cookies를 오염시키지 않는다.

이 수정으로 서버 컴포넌트에서 인증 API 호출이 가능해졌고,
마이페이지 9개 라우트에 `prefetchQuery + HydrationBoundary` 패턴을 적용할 수 있었다.

---

## 결과

| 구분 | Before | After |
|------|--------|-------|
| 공개 데이터 | 동시 방문자마다 API 호출 | ISR + On-Demand로 서버 캐시 공유, 변경 시 즉시 반영 |
| 관리자 데이터 수정 | 최대 revalidate 주기까지 대기 | On-Demand Revalidation으로 즉시 캐시 무효화 |
| 마이페이지 초기 로딩 | 클라이언트 fetch → 스켈레톤 표시 | prefetchQuery → 스켈레톤 없이 즉시 렌더 (LCP 수치 측정 예정) |
| 창 포커스 복귀 | 무한 스크롤 위치 초기화, 불필요한 refetch | refetchOnWindowFocus 선별 제어 |
| 페이지 전환 | 새 데이터 오기 전까지 빈 화면 | keepPreviousData로 이전 목록 유지 |
| 캐시 컨벤션 | 파일마다 제각각 | fetchServerAPI / fetchClientAPI 레이어로 강제 |
