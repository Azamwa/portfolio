# 상품 리스트 페이지 — 디바이스·필터 분기 아키텍처 설계

## 배경

GooseTire 상품 리스트는 두 가지 진입 방식이 있다.

- **MATCH**: 내 차량 정보 기반 타이어 자동 추천
- **SEARCH**: 직접 스펙(너비/편평비/인치) 입력 검색

그리고 디바이스별로 UI 자체가 다르다.

- **데스크탑**: 페이지네이션 + 사이드 필터 패널
- **모바일**: 무한 스크롤 + 바텀시트 필터

MATCH/SEARCH × 데스크탑/모바일 — 4가지 조합이 생기고, 각각 다른 쿼리, 다른 UI, 다른 파라미터 규칙을 가진다. 이걸 하나의 컴포넌트에서 분기 처리하면 조건문이 폭발한다.

---

## 설계 1: 컴포넌트 트리 완전 분리

CSS 반응형이 아니라 **디바이스별로 완전히 다른 컴포넌트 트리**를 렌더링한다.

```tsx
// product-wrapper.tsx
const ProductWrapper = ({ banner }) => {
  const { replaceComplete } = useProductUrlInitializer()

  return (
    <>
      <div className="block ts:hidden">
        <ProductMobile banner={banner} replaceComplete={replaceComplete} />
      </div>
      <div className="hidden ts:block">
        <ProductDesktop replaceComplete={replaceComplete} />
      </div>
    </>
  )
}
```

SSR에서는 서버가 디바이스 정보를 모르기 때문에 양쪽 HTML을 모두 내려준다. 클라이언트에서 CSS가 즉시 하나를 숨기는 구조다. JS로 `isMobile ? <Mobile /> : <Desktop />`을 분기하면 서버 렌더 결과와 클라이언트 렌더 결과가 달라져 hydration mismatch가 발생하기 때문에 CSS 분리 방식을 선택했다.

**이렇게 한 이유:**
- hydration mismatch 없이 SSR 지원
- 공유 상태 없음 — 컴포넌트 경계를 넘는 props drilling 없다
- 모바일 전용 로직(무한 스크롤, 드로어 상태)이 데스크탑 트리에 오염되지 않는다
- 각 트리가 독립적으로 자기 쿼리를 관리한다

---

## 설계 2: skipToken으로 4가지 쿼리 중 1개만 실행

MATCH/SEARCH × 데스크탑/모바일 조합에 따라 4개 쿼리 훅이 존재한다.
`skipToken`으로 현재 조합에 해당하는 훅 **1개만 실행**되도록 제어했다.

```ts
// 모바일 무한 스크롤 — MATCH
export const useMatchProductInfinite = (type, isDesktop) => {
  return useInfiniteQuery({
    queryFn: type !== 'MATCH' || isDesktop ? skipToken : async ({ pageParam }) => ...,
    ...
  })
}

// 모바일 무한 스크롤 — SEARCH
export const useSearchProductInfinite = (type, isDesktop) => {
  return useInfiniteQuery({
    queryFn: type !== 'SEARCH' || isDesktop ? skipToken : async ({ pageParam }) => ...,
    ...
  })
}

// 데스크탑 페이지네이션 — MATCH
export const useMatchProductPage = (type, isDesktop) => {
  return useQuery({
    queryFn: type !== 'MATCH' || !isDesktop ? skipToken : () => ...,
    placeholderData: keepPreviousData,
    ...
  })
}

// 데스크탑 페이지네이션 — SEARCH
export const useSearchProductPage = (type, isDesktop) => {
  return useQuery({
    queryFn: type !== 'SEARCH' || !isDesktop ? skipToken : () => ...,
    placeholderData: keepPreviousData,
    ...
  })
}
```

| | 모바일 | 데스크탑 |
|---|---|---|
| MATCH | `useMatchProductInfinite` ✅ | `useMatchProductPage` ✅ |
| SEARCH | `useSearchProductInfinite` ✅ | `useSearchProductPage` ✅ |
| 나머지 3개 | skipToken (실행 안 함) | skipToken (실행 안 함) |

`enabled: false`로도 구현 가능하지만 skipToken은 타입 수준에서 `queryFn`이 없는 쿼리임을 명시하고, 실수로 쿼리 키가 캐시에 남는 상황을 방지한다.

---

## 설계 3: replaceComplete — URL 정규화 완료 후 렌더

Product 페이지는 다양한 진입 경로가 있다.

- 로그인 사용자가 차량 등록 후 진입 → URL에 `type` 없음
- 외부 링크로 직접 진입 → URL에 잘못된 파라미터
- 모바일에서 MATCH 필터 적용 후 새로고침 → 허용되지 않는 파라미터 조합
- **사용자가 URL을 직접 조작** → `?type=MATCH`만 입력하거나 존재하지 않는 `carIdx` 입력 등

이 모든 케이스에서 `?type=MATCH&width=205&aspectRatio=55&inch=16` 형태의 유효한 URL이 보장되어야 상품 목록을 정확하게 불러올 수 있다.

`useProductUrlInitializer`는 이 정규화를 담당한다.

URL 정규화 로직은 **Rules Engine 패턴**으로 구현했다. 각 규칙을 `{ name, condition, action }` 구조로 선언하고, 엔진이 순차 실행한다. 새 케이스가 생겨도 배열에 규칙 하나만 추가하면 된다.

```ts
type Rule = {
  name: string
  condition: (state: StoreState) => boolean
  action: (state: StoreState, params: URLSearchParams) => boolean
}

// 규칙 선언 예시
const rules: Rule[] = [
  {
    name: 'no-type-with-valid-car',
    condition: (state) => !state.type && !!state.carIdx && state.hasValidCar,
    action: (state, params) => {
      params.set('type', 'MATCH')
      params.set('width', selectedCar.frontWidth)   // 차량에서 스펙 자동 추출
      params.set('aspectRatio', selectedCar.frontAspectRatio)
      params.set('inch', selectedCar.frontInch)
      return true
    },
  },
  // ... 11개 규칙 추가
]

// 엔진: 규칙 순차 실행, 각 실행 후 state 즉시 업데이트 (다음 규칙이 최신 상태 참조)
for (const rule of rules) {
  if (rule.condition(state)) {
    const changed = rule.action(state, params)
    if (changed) state = updateState(params)
  }
}
```

12개 규칙 중 대표 케이스:
- type 없음 + 유효한 차량 → MATCH + 차량 타이어 스펙 자동 설정
- type 없음 + 차량 없음 + 메인 차량 있음 → MATCH + 메인 차량 스펙
- type 없음 or 차량/스펙 없음 → SEARCH로 폴백
- 모바일 MATCH + 필터 파라미터(brand/price/season) → SEARCH 강제 전환
- MATCH일 때 허용되지 않은 파라미터 제거 (데스크탑/모바일 허용 목록 별도 관리)

```ts
const useProductUrlInitializer = () => {
  const [replaceComplete, setReplaceComplete] = useState(false)

  useEffect(() => {
    if (hasChanges) {
      startTransition(() => router.replace(`?${params.toString()}`))
    }
  }, [myCarList, searchParams])

  // URL이 목표 상태에 도달했을 때 완료 플래그
  useEffect(() => {
    if (type === 'SEARCH') setReplaceComplete(true)
    if (type === 'MATCH' && width && aspectRatio && inch) setReplaceComplete(true)
  }, [searchParams])

  return { replaceComplete }
}
```

`replaceComplete`가 `false`인 동안 실제 상품 목록 대신 스켈레톤을 표시한다.

```tsx
// product-desktop.tsx
{!replaceComplete ? (
  <ProductFilterSkeleton /> // URL 정규화 중 — 스켈레톤
) : (
  params.type === 'MATCH' ? <MatchProductDesktop /> : <SearchProductDesktop />
)}
```

**이 패턴이 필요한 이유:**
정규화 전 URL로 상품 API를 호출하면 잘못된 파라미터로 빈 결과나 에러가 반환된다.
URL이 확정되기 전까지 쿼리 자체를 실행하지 않음으로써 불필요한 API 호출과 레이아웃 시프트를 방지한다.

---

## 설계 4: HydrationBoundary로 필터 초기 로딩 제거

서버 컴포넌트에서 필터 리스트를 prefetch한다.

```tsx
// product/page.tsx (서버 컴포넌트)
const ProductPage = async () => {
  const queryClient = getQueryClient()

  await queryClient.prefetchQuery({
    queryKey: ['product-filter-list'],
    queryFn: () => fetchProductFilterList(),
  })

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Suspense>
        <ProductWrapper banner={banner} />
      </Suspense>
    </HydrationBoundary>
  )
}
```

클라이언트 컴포넌트가 mount되는 순간 `useProductFilterList()`는 이미 캐시에 데이터가 있어 즉시 렌더된다. 필터 영역 로딩 스피너가 사라졌다.

---

## 결과

| 구분 | Before | After |
|------|--------|-------|
| 디바이스 분기 | 하나의 컴포넌트에서 if/CSS 혼합 처리 | 모바일/데스크탑 독립 컴포넌트 트리 |
| 쿼리 실행 | 모든 쿼리 조건부 실행, 불필요한 요청 가능 | skipToken으로 4개 중 1개만 실행 보장 |
| 잘못된 URL 진입 | 잘못된 파라미터로 API 호출 → 빈 결과 | 정규화 완료 전 스켈레톤 → 확정 URL로만 호출 |
| 필터 리스트 로딩 | 클라이언트 mount 후 fetch → 로딩 표시 | 서버 prefetch → 즉시 렌더 |
