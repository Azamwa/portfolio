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
| 공개 정적 데이터 | 로그인 불필요, 갱신 주기 예측 가능 | ISR (revalidate) |
| 매 요청 최신 데이터 | 로그인 불필요, 매 요청마다 최신 데이터 필요 | `force-dynamic` |
| 인증 동적 데이터 | 사용자별 개인 데이터 | `force-dynamic` + `cache: 'no-store'` |

---

## 서버 캐싱: ISR revalidate를 데이터 비즈니스 주기에 맞춤

"길게 or 짧게" 이분법이 아니라, **데이터가 실제로 바뀌는 비즈니스 주기**를 기준으로 설정했다.

```
상품 상세 / 이벤트   → revalidate: 3600  (관리자 수작업 업데이트, 1시간 캐시 충분)
메인 배너 / 추천 타이어 → revalidate: 300   (당일 프로모션 반영, 5분 내 갱신 필요)
팝업 목록            → revalidate: 86400  (캠페인 단위 변경, 하루 캐시 가능)
매장 정보 / 마이페이지  → force-dynamic     (매 요청마다 최신 재고·개인 데이터 필요)
```

### 레이어 분리로 컨벤션 강제

두 종류의 fetch 래퍼로 **함수 이름만 봐도 캐싱 의도가 드러나도록** 설계했다.

```ts
// 서버 전용 — ISR 옵션 래퍼
const fetchServerAPI = (url, { revalidate }) =>
  fetch(url, { next: { revalidate } })

// 클라이언트 호출용 — 항상 최신
const fetchClientAPI = (url, options) =>
  fetch(url, { cache: 'no-store', ...options })
```

새로운 API를 추가할 때 래퍼 선택만으로 캐싱 의도가 결정되고, 실수로 인증 데이터를 캐시하거나 공개 데이터를 no-store로 처리하는 실수를 타입 수준에서 차단한다.

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

### skipToken으로 조건부 쿼리 제어

```ts
// 차량 미선택 시 모델·타이어 쿼리 실행 자체를 막음
useCarModels(carMakeId ?? skipToken)

// 검색어 없으면 주소 검색 안 함
useSearchAddressList(query?.length > 0 ? query : skipToken)

// 결제 타입에 따라 하나만 실행
useOrderSheetNormal(isNormal ? orderId : skipToken)
useOrderSheetReservation(isReservation ? orderId : skipToken)
```

enabled 조건 분기 대신 skipToken을 사용하면 타입 안전성이 보장되고 쿼리 키 오염이 없다.

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
| 공개 데이터 | 동시 방문자마다 API 호출 | ISR 서버 캐시 공유 |
| 마이페이지 초기 로딩 | 클라이언트 fetch → 스켈레톤 표시 | prefetchQuery → 스켈레톤 없이 즉시 렌더 (LCP 수치 측정 예정) |
| 창 포커스 복귀 | 무한 스크롤 위치 초기화, 불필요한 refetch | refetchOnWindowFocus 선별 제어 |
| 페이지 전환 | 새 데이터 오기 전까지 빈 화면 | keepPreviousData로 이전 목록 유지 |
| 캐시 컨벤션 | 파일마다 제각각 | fetchServerAPI / fetchClientAPI 레이어로 강제 |
