# UX 개선 — Optimistic Update + Suspense 섹션별 독립 로딩

## 배경

장바구니와 재입고 알림은 사용자 조작 빈도가 높은 영역이다.
수량 변경, 삭제, 재입고 신청·취소 — 이 모든 동작이 서버 응답 후에야 UI가 갱신되면 반응이 느리다고 느낀다.

기존 구현은 뮤테이션 완료 후 `invalidateQueries`로 데이터를 다시 받아 화면을 갱신했다.
네트워크 지연 동안 사용자는 아무 피드백 없이 기다려야 했다.

---

## Optimistic Update

### 공통 패턴

`onMutate → onError → onSettled` 3단계로 구성된다.

```ts
onMutate: async (payload) => {
  // 1. 진행 중인 쿼리 취소 — race condition 방지
  await queryClient.cancelQueries({ queryKey: [...] })

  // 2. 현재 캐시 백업 — rollback용
  const previous = queryClient.getQueryData([...])

  // 3. 캐시 즉시 업데이트 — 서버 응답 전 UI 반영
  queryClient.setQueryData([...], (old) => /* 변경 로직 */)

  return { previous }  // context에 전달
},
onError: (_error, _payload, context) => {
  // 서버 실패 시 백업 데이터로 롤백
  queryClient.setQueryData([...], context.previous)
  toast.error(error.message)
},
onSettled: () => {
  // 성공/실패 모두 서버와 최종 동기화
  queryClient.invalidateQueries({ queryKey: [...] })
},
```

### 장바구니 — 동시 뮤테이션 처리

수량 변경, 휠 얼라인먼트 변경, 삭제 — 3개 뮤테이션이 모두 동일한 `cart-list` 캐시를 바라본다.
사용자가 연속으로 수량 +/−를 빠르게 누를 때, 각 요청이 완료될 때마다 invalidate가 발생하면 화면이 여러 번 깜빡인다.

이를 `mutationKey`로 해결했다.

```ts
const CART_MUTATION_KEY = ['cart-mutation'] as const

// 수량 변경
useMutation({ mutationKey: [...CART_MUTATION_KEY, 'amount'], ... })

// 휠 얼라인먼트 변경
useMutation({ mutationKey: [...CART_MUTATION_KEY, 'wheel-alignment'], ... })

// 삭제
useMutation({ mutationKey: [...CART_MUTATION_KEY, 'delete'], ... })

// onSettled — 마지막 뮤테이션만 invalidate
onSettled: () => {
  if (queryClient.isMutating({ mutationKey: CART_MUTATION_KEY }) === 1) {
    queryClient.invalidateQueries({ queryKey: ['cart-list'] })
  }
}
```

`isMutating`은 현재 진행 중인 해당 키의 뮤테이션 수를 반환한다. `=== 1`이면 마지막 요청만 남은 상태.
동시에 3개 요청이 진행 중이면 1, 2번 완료 시 invalidate를 건너뛰고, 마지막 3번이 끝날 때 한 번만 서버 동기화한다.

### 재입고 알림 — 두 캐시 동시 동기화

재입고 신청/취소는 두 개의 독립 캐시를 동시에 업데이트해야 한다.
- `stock-status` — 개별 상품의 신청 여부 (boolean)
- `stock-status-list` — 신청한 상품 ID 목록 (string[])

```ts
onMutate: async (tireId) => {
  await queryClient.cancelQueries({ queryKey: ['stock-status', tireId] })
  await queryClient.cancelQueries({ queryKey: ['stock-status-list'] })

  const previousStatus = queryClient.getQueryData<boolean>(['stock-status', tireId])
  const previousList = queryClient.getQueryData<string[]>(['stock-status-list'])

  // 두 캐시 즉시 반영
  queryClient.setQueryData<boolean>(['stock-status', tireId], true)
  queryClient.setQueryData<string[]>(['stock-status-list'], (old) =>
    old && !old.includes(tireId) ? [...old, tireId] : old
  )

  return { previousStatus, previousList }
},
onError: (error, tireId, context) => {
  // 두 캐시 독립적으로 rollback
  if (context?.previousStatus !== undefined) {
    queryClient.setQueryData(['stock-status', tireId], context.previousStatus)
  }
  if (context?.previousList) {
    queryClient.setQueryData(['stock-status-list'], context.previousList)
  }
  toast.error(error.message)
},
```

두 캐시를 개별 `previous`로 백업하기 때문에 하나가 실패해도 각각 독립적으로 rollback된다.

---

## Suspense — 섹션별 독립 로딩

비동기 섹션마다 `<Suspense fallback={<Skeleton />}>`으로 감싸 독립 로딩 단위를 만들었다.

```tsx
// 마이페이지 대시보드 — 주문/예약 섹션이 서로를 기다리지 않음
<Suspense fallback={<ActiveOrderSkeleton length={3} />}>
  <ActiveOrderContainer />       {/* 주문 섹션 독립 로딩 */}
</Suspense>
<Suspense fallback={<ActiveReservationSkeleton length={2} />}>
  <ActiveReservationContainer /> {/* 예약 섹션 독립 로딩 */}
</Suspense>

// 인덱스 — 배너와 추천 섹션이 서로를 기다리지 않음
<Suspense fallback={<BannerSkeleton />}>
  <BannerSection />
</Suspense>
<Suspense fallback={<BestItemsSkeleton />}>
  <BestItemsSection />
</Suspense>
```

**핵심 설계 의도**: 느린 API 하나가 전체 페이지 렌더를 막지 않는다.
배너 API가 1초 걸리고 추천 섹션 API가 200ms면, 추천 섹션은 200ms에 표시되고 배너만 1초 기다린다.
`<Suspense>` 없이 구현하면 두 섹션 모두 1초를 기다린다.

---

## 결과

| 구분 | Before | After |
|------|--------|-------|
| 장바구니 수량 변경 | 서버 응답까지 UI 변화 없음 | 즉시 반영, 실패 시 자동 rollback |
| 연속 수량 클릭 | 요청마다 invalidate → 화면 깜빡임 | 마지막 요청 완료 시 한 번만 동기화 |
| 재입고 신청 | 응답 후 갱신 | 신청 버튼 누르자마자 상태 전환 |
| 페이지 초기 로딩 | 가장 느린 API 기준으로 전체 대기 | 섹션별 독립 로딩 (200ms~1s 순차 표시) |
