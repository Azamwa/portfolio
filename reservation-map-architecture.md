# 매장 선택 지도 페이지 — 네이버지도 클러스터링 + 상태 분리 설계

## 배경

타이어 설치 예약은 매장 선택이 선행된다. 사용자는 지도에서 근처 매장을 탐색하고, 원하는 매장을 선택해 예약 페이지로 이동한다.

요구사항을 정리하면:
- 전국 매장 수백 개를 지도에 표시 → 마커가 겹치지 않도록 클러스터링 필요
- 지도 이동 시 현재 화면 범위 내 매장만 목록에 표시
- 선택된 매장은 클러스터에서 분리해 별도 강조 표시
- PC: 사이드 패널에서 매장 선택 → 오버레이 카드 표시 / 모바일: 드로어로 표시

---

## 설계 1: 네이버지도 클러스터링 커스터마이징

네이버에서 제공하는 MarkerClustering 라이브러리를 래핑하여 사용한다.

```ts
const MarkerClustering = makeMarkerClustering(window.naver)

const cluster = new MarkerClustering({
  map,
  markers,           // 선택된 마커 제외한 목록
  gridSize: 240,     // 클러스터 반경 (px)
  maxZoom: 16,       // 줌 16 이상에서는 클러스터 해제
  minClusterSize: 2,
  icons: clusterIcons,          // SVG 커스텀 아이콘
  indexGenerator: [10, 30, 50], // 매장 수 기준 아이콘 단계
  stylingFunction: (clusterMarker, count) => {
    // 클러스터 위에 매장 수 텍스트 직접 주입
  },
})
```

**선택된 마커를 클러스터에서 분리:**

클러스터 라이브러리에 선택 상태 개념이 없어서, 선택된 매장을 클러스터 마커 배열에서 제외하고 별도 `<Marker>` 컴포넌트로 렌더링했다.

```tsx
// 클러스터 생성 시 선택된 마커 제외
const markers = filteredAgencyList
  .filter((agency) => agency.enseq !== selectedAgencyId) // 선택된 것 제외
  .map((agency) => new navermaps.Marker({ ... }))

// 선택된 마커는 별도 렌더링 — zIndex: 100으로 클러스터 위에 표시
{selectedAgency && (
  <Marker
    position={{ lat: ..., lng: ... }}
    icon={{ content: selectedMarkerSVG }}
    zIndex={100}
  />
)}
```

`selectedAgencyId`가 바뀔 때마다 클러스터를 전체 재생성한다. 이전 클러스터를 `setMap(null)`로 cleanup 후 새로 만드는 방식이다.

---

## 설계 2: 바운드 기반 실시간 필터링

지도 이동이 끝날 때마다 현재 화면 범위(bounds)를 갱신하고, 범위 내 매장만 목록에 표시한다.

```ts
// idle 이벤트 = 지도 이동/줌 완료 시점
navermaps.Event.addListener(map, 'idle', updateBoundsFromMap)

// 300ms 디바운스 — 빠른 연속 이동 시 불필요한 필터링 방지
const debouncedSetBounds = useMemo(
  () => debounce((newBounds) => {
    setBounds(newBounds)
    setIsFiltering(false)
  }, 300),
  []
)
```

필터링과 정렬은 `useMemo`로 파생 계산한다. bounds나 sortType이 바뀔 때만 재계산한다.

```ts
const filteredAgencyList = useMemo(() => {
  if (!bounds) return []

  return agencyList
    .filter((agency) => {
      // bounds 내 좌표 체크
      return lat >= bounds.sw.lat && lat <= bounds.ne.lat && ...
    })
    .map((agency) => ({
      ...agency,
      // Haversine 공식으로 사용자 위치 기준 거리 계산
      distance: userLocation
        ? getDistance(userLocation.lat, userLocation.lng, lat, lng)
        : 0,
    }))
    .sort((a, b) =>
      sortType === 'distance'
        ? a.distance - b.distance
        : parseFloat(b.star) - parseFloat(a.star)
    )
}, [agencyList, bounds, userLocation, sortType])
```

외부 API를 다시 호출하지 않고 **서버에서 받아온 전체 매장 목록을 클라이언트에서 필터링**한다. 지도 이동마다 API를 치는 대신 한 번 받은 데이터를 재활용하는 구조다.

---

## 설계 3: Context 4분할로 구독 범위 최소화

지도 관련 상태를 하나의 Context에 몰아넣으면, 선택된 매장이 바뀔 때 지도 중심 좌표를 구독하는 컴포넌트까지 리렌더된다.

관심사별로 4개 Context로 분리했다.

| Context | 담당 상태 | 주요 구독처 |
|---------|---------|---------|
| `MapBoundsContext` | bounds, filteredAgencyList | 매장 목록, 클러스터 |
| `FilteringContext` | isFiltering, sortType | 목록 헤더 (로딩 표시, 정렬 UI) |
| `MapCenterContext` | mapCenter, mapZoom, userLocation | 지도 컴포넌트, 검색 |
| `SelectedAgencyContext` | selectedAgencyId | 매장 카드/드로어, 클러스터 |

```tsx
// 단일 Provider가 4개 Context를 중첩 제공
<MapBoundsContext.Provider value={boundsValue}>
  <FilteringContext.Provider value={filteringValue}>
    <MapCenterContext.Provider value={mapCenterValue}>
      <SelectedAgencyContext.Provider value={selectedAgencyValue}>
        {children}
      </SelectedAgencyContext.Provider>
    </MapCenterContext.Provider>
  </FilteringContext.Provider>
</MapBoundsContext.Provider>
```

매장 선택(`selectedAgencyId`) 변경은 `SelectedAgencyContext` 구독 컴포넌트만 리렌더한다. 지도 드래그로 `mapCenter`가 바뀌어도 매장 목록 컴포넌트는 영향받지 않는다.

---

## 설계 4: useQueries combine으로 3개월치 예약 날짜 병렬 fetch

예약 날짜 선택기는 3개월치 예약 가능일을 보여줘야 한다. 월별로 API가 분리되어 있어서 3번 호출해야 한다.

순차 호출 대신 `useQueries`로 병렬 fetch하고, `combine`으로 하나의 결과로 병합했다.

```ts
export const useReservationDateList = (type, enseq, baseYear, baseMonth) => {
  const months = Array.from({ length: 3 }, (_, i) => {
    const totalMonth = baseMonth + i
    return {
      year: baseYear + Math.floor((totalMonth - 1) / 12),
      month: ((totalMonth - 1) % 12) + 1,
    }
  })

  return useQueries({
    queries: months.map(({ year, month }) => ({
      queryKey: ['reservation-date-list', type, enseq, year, month],
      queryFn: () => fetchReservationDateList(type, enseq, year, month),
    })),
    combine: (results) => ({
      data: results.flatMap((r) => r.data ?? []),  // 3개월 데이터 병합
      isLoading: results.some((r) => r.isLoading),
      isError: results.some((r) => r.isError),
    }),
  })
}
```

**이 구조의 이점:**
- 3개 요청이 병렬 실행 → 순차 대비 응답 시간 단축
- 월별 캐시 키가 독립적 → 1월 데이터를 캐시하면 다음 달 넘어가도 해당 월 데이터 재사용
- 소비 컴포넌트는 `combine` 결과만 받아 단일 쿼리처럼 사용

---

## 결과

| 구분 | Before | After |
|------|--------|-------|
| 마커 밀집 지역 | 마커 수백 개 겹침 | 클러스터로 그룹화, 줌 레벨 따라 해제 |
| 선택 마커 | 클러스터에 묻힘 | 클러스터 분리 + 별도 SVG 아이콘 + zIndex 100 |
| 지도 이동 시 목록 | 전체 매장 목록 고정 | bounds 기준 실시간 필터링 + 거리순/평점순 정렬 |
| 상태 변경 리렌더 | 전체 지도 UI 리렌더 | Context 분리로 변경된 상태 구독 컴포넌트만 리렌더 |
| 예약 날짜 로딩 | 월별 순차 3회 호출 | useQueries 병렬 fetch → combine 병합 |
