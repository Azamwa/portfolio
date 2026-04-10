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
| 매 요청 최신 데이터 | 로그인 불필요, 매 요청마다 최신 데이터 필요 | 페이지 `force-dynamic` + fetch `cache: 'no-store'` |
| 인증 동적 데이터 | 사용자별 개인 데이터 | `force-dynamic` + fetch `cache: 'no-store'` |

> **Full Route Cache와 Data Cache는 독립 레이어다.**
> `force-dynamic`은 Full Route Cache(페이지 HTML 캐시)만 비활성화한다. 서버 컴포넌트 내부의 `fetch` 결과 캐시(Data Cache)는 별개 레이어로 `force-dynamic`과 무관하게 동작한다. 매 요청 최신 데이터를 보장하려면 fetch 옵션에 `cache: 'no-store'`를 함께 명시해야 한다. 이 구분을 놓치면 "페이지는 매번 새로 렌더링되는데 데이터는 옛날 것이 나오는" 현상이 발생한다.

---

## 서버 캐싱: fetch revalidate를 데이터 비즈니스 주기에 맞춤

"길게 or 짧게" 이분법이 아니라, **데이터가 실제로 바뀌는 비즈니스 주기**를 기준으로 설정했다.

> 아래 표의 revalidate 값은 **fetch 단위 Data Cache 설정**이다. 매장 페이지처럼 `force-dynamic`이 걸린 페이지에서도 Data Cache는 독립적으로 동작하므로, 페이지가 매 요청 렌더링되더라도 내부 fetch는 이 revalidate 값에 따라 캐시된다.

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

// 캐시 비활성화 — 항상 최신 (Server Action, 인증 API 등)
const fetchClientAPI = (url, options) =>
  fetch(url, { cache: 'no-store', ...options })
```

`fetchClientAPI`는 명칭과 달리 브라우저 전용이 아니다. Server Action(`'use server'`)에서도 호출되며, 역할은 **"Data Cache를 사용하지 않고 매번 최신 데이터를 가져오는 것"**이다. 인증 토큰이 포함된 요청, 결제 상태 조회 등 캐싱해선 안 되는 데이터에 사용한다.

새로운 API를 추가할 때 래퍼 선택만으로 캐싱 의도가 결정되고, 실수로 인증 데이터를 캐시하거나 공개 데이터를 no-store로 처리하는 실수를 타입 수준에서 차단한다.

---

## On-Demand Revalidation: 시간 기반 캐싱의 한계 보완

ISR만으로는 "관리자가 데이터를 수정했는데 최대 1시간을 기다려야 반영된다"는 한계가 있다.
이를 보완하기 위해 **admin에서 데이터 변경 시 즉시 캐시를 무효화**할 수 있는 On-Demand Revalidation 구조를 설계했다.

### API 설계

Next.js `revalidateTag`를 활용한 태그 기반 캐시 무효화 API Route (`POST /api/revalidate`)를 구현했다.

- **시크릿 키 인증**으로 외부 무단 호출 방지
- **허용 태그 화이트리스트**로 임의 캐시 무효화 차단
- `fetchServerAPI` 유틸에 `tags` 옵션을 추가하여, 서버 컴포넌트 fetch에 태그를 **선언적으로** 부여할 수 있는 구조

프론트 측에서는 무효화 API Route와 fetch 래퍼의 tags 전달 로직을 구현해 두었고, 백엔드 admin에서 데이터 변경 시 이 엔드포인트를 호출하는 방식으로 연동한다.

### revalidateTag vs revalidatePath — 태그 기반을 선택한 이유

Next.js의 On-Demand Revalidation은 두 가지 방식을 제공한다.

| 함수 | 무효화 대상 | 단위 |
|-----|----------|------|
| `revalidatePath('/agency')` | 특정 경로의 Full Route Cache + Data Cache | 경로(URL) |
| `revalidateTag('agency')` | 해당 태그가 부여된 모든 Data Cache 엔트리 | 태그 |

이 프로젝트에서는 **`revalidateTag`(태그 기반)만 사용**한다. 이유는 두 가지다.

**1) 같은 데이터가 여러 페이지에 걸쳐 사용된다.**
예를 들어 매장 데이터는 `/agency`(목록), `/agency/[agencyId]`(상세), `/reservation/[agencyId]`(예약) 세 경로에서 공유된다. `revalidatePath`로 처리하면 세 경로를 각각 호출해야 하지만, `revalidateTag('agency')` 한 번이면 세 페이지의 관련 캐시가 전부 무효화된다.

**2) 백엔드-프론트 결합도가 낮다.**
`revalidatePath`는 백엔드가 프론트의 URL 구조(`/agency/[agencyId]`)를 알아야 한다. 태그 기반이면 백엔드는 `"agency"`라는 도메인 태그만 알면 되고, 프론트가 URL을 리팩토링해도 백엔드 코드에 영향이 없다.

### 태그 설계

```
dashboard (상위 태그 — 하위 3개를 포괄)
├── dashboard-banners
├── dashboard-popups
└── dashboard-promotions

event     (단일 태그)
product   (단일 태그)
agency    (단일 태그)
```

**dashboard 계열만 상위/하위로 세분화**한 이유: 메인 페이지의 배너·팝업·프로모션은 각각 독립적으로 운영되며, 관리자가 배너만 수정하는 경우가 대부분이다. 상위 태그 `dashboard`로 전체를 한 번에 무효화할 수도 있고, `dashboard-banners`로 배너 캐시만 핀포인트로 무효화할 수도 있다. admin 측에서 호출 범위를 선택할 수 있어 **불필요한 캐시 버스팅을 최소화**한다.

**나머지(event, product, agency)를 단일 태그로 유지**한 이유: 데이터 변경 빈도가 낮고(상품 2,000개 미만, 매장·이벤트도 소규모), 전체 일괄 무효화의 비용이 크지 않다. 태그를 ID 단위(`product-{id}`)까지 세분화하면 관리 복잡도만 증가하고 실질적 이득이 없다고 판단했다. 일괄 무효화 시에도 **실제로 백엔드를 다시 호출하는 건 유저가 접속한 페이지뿐**이므로, 미접속 페이지의 캐시가 비어있더라도 백엔드 부하로 이어지지 않는다.

### 이중 캐싱 전략의 핵심

| | 시간 기반 (ISR) | 이벤트 기반 (On-Demand) |
|---|---|---|
| 역할 | 안전망 — 최악의 경우에도 일정 주기 내 갱신 보장 | 즉시성 — 데이터 변경 시점에 캐시 무효화 |
| 트리거 | revalidate 시간 경과 | admin API에서 `/api/revalidate` 웹훅 호출 |
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

## 트러블슈팅 1: force-dynamic 페이지에서 데이터가 갱신되지 않는 문제

### 증상

매장 목록 페이지(`/agency`)에 `force-dynamic`을 설정했는데, 백엔드에서 API 응답을 변경한 후에도 배포 서버에서 옛날 데이터가 그대로 표시되었다. 로컬 개발 환경에서는 최신 데이터가 정상 출력.

### 원인

`force-dynamic`이 **모든 캐시를 끄는 것**이라고 오해하고 있었다.

Next.js는 서버 측 캐시가 두 개의 독립 레이어로 나뉜다.

| 레이어 | 대상 | `force-dynamic` 영향 |
|-------|------|---------------------|
| Full Route Cache | 페이지 HTML/RSC 페이로드 | 비활성화됨 |
| Data Cache | 개별 `fetch()` 응답 | **영향 없음 — 독립 동작** |

매장 목록 페이지는 지도 기반 UI로 클라이언트 위치에 따라 시작점이 달라 페이지 캐싱이 무의미해서 `force-dynamic`을 적용했다. 그런데 서버 컴포넌트 내부의 `getAgencyList()`는 `next: { revalidate: 604800, tags: ['agency'] }` 옵션이 걸려 있어 Data Cache에 7일간 고정되어 있었다.

**페이지는 매 요청 새로 렌더링되지만, 렌더링 중에 호출되는 fetch가 캐시된 옛날 응답을 반환**하는 구조였다.

로컬에서 문제가 없었던 이유는 `next dev`(개발 모드)에서는 Data Cache가 기본 비활성화되어 매번 실제 API를 호출하기 때문이다.

### 해결

On-Demand Revalidation 웹훅 연동으로 해결했다. 백엔드 admin에서 매장 데이터를 변경하면 `POST /api/revalidate { tag: "agency" }` 웹훅이 호출되어, 해당 태그의 Data Cache가 즉시 무효화된다. 다음 유저 요청 시 백엔드에서 새 데이터를 가져와 Data Cache를 재생성한다.

이 경험을 통해 **설계 기준 표에서 `force-dynamic`만으로는 "매 요청 최신 데이터"를 보장할 수 없다**는 점을 반영했다. 매 요청 최신이 필요한 경우 fetch 옵션에 `cache: 'no-store'`를 명시하거나, On-Demand Revalidation으로 Data Cache를 관리해야 한다.

---

## 트러블슈팅 2: Next.js 미들웨어 쿠키 오염 문제

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
