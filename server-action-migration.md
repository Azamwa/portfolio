# API 레이어 재설계 — Route Handler → Server Action + 3-레이어 컨벤션

## 문제 상황

**전환 규모: Route Handler 약 50개 함수 → Server Action 전환**

초기 구현은 클라이언트에서 외부 API를 직접 호출하거나, Next.js Route Handler를 경유하는 방식이 혼재해 있었다.

```
[Route Handler 경유]
클라이언트 → /api/cart (Next.js Route Handler) → 외부 API
                   (불필요한 중간 홉)

[직접 호출]
클라이언트 → 외부 API (인증 토큰이 브라우저에 노출)
```

문제는 구조만이 아니었다. 인증 헤더 주입, 토큰 갱신, 에러 핸들링이 파일마다 다르게 구현되어 있었다. API를 하나 추가할 때마다 기존 코드를 참고해서 패턴을 맞춰야 했다.

---

## 설계 결정: Server Action으로 전환

Next.js 14에서 안정화된 Server Action은 Route Handler 없이 클라이언트에서 서버 함수를 직접 호출할 수 있다.

**Route Handler 대비 이점:**

| 항목 | Route Handler | Server Action |
|------|--------------|---------------|
| 네트워크 홉 | 클라이언트 → Next 서버(Route Handler) → 외부 API | 서버에서 외부 API 직접 호출 (홉 1개) |
| 인증 토큰 | 브라우저 전달 또는 Route Handler 내 처리 | `cookies()`로 서버에서만 접근 |
| 엔드포인트 노출 | `/api/*` URL 노출 | 함수 직접 호출, URL 없음 |
| 번들 크기 | Route Handler 코드 클라이언트 제외 필요 | Server Action은 클라이언트 번들에 포함 안 됨 |

약 50개 Route Handler 함수를 Server Action으로 전환했다.

---

## 3-레이어 컨벤션

전환 과정에서 레이어를 3개로 명확히 분리했다.

```
[서버 컴포넌트 prefetch 전용]  src/lib/api-server/
  → cookies() 직접 접근, fetchServerAPI로 ISR 캐시

[Server Action]               src/lib/api-options/  ('use server')
  → fetchServerAPI / fetchClientAPI / withAuthAction 조합

[클라이언트 훅]                src/lib/query-options/
  → api-options 함수를 useQuery / useMutation으로 소비
```

레이어 경계가 명확하기 때문에 "이 API 호출이 서버에서 일어나는가, 클라이언트인가"를 파일 위치만 봐도 알 수 있다.

---

## withAuthAction — 인증 로직 캡슐화

인증이 필요한 모든 Server Action은 `withAuthAction` 래퍼를 통과한다.

```ts
// src/lib/actions.ts
export async function withAuthAction<T>(
  handler: (tokenData: NonNullable<SocialLoginResponse['data']>) => Promise<T>,
): Promise<T> {
  const refreshTokenData = await refreshTokenIfNeeded()

  if (!refreshTokenData) {
    throw new Error('Unauthorized')
  }

  const cookieStore = await cookies()
  const result = await handler(refreshTokenData)

  // 성공 시 최신 토큰으로 쿠키 갱신
  cookieStore.set('userToken', refreshTokenData.accessToken, cookieOptions)
  cookieStore.set('refreshToken', refreshTokenData.refreshToken, cookieOptions)
  cookieStore.set('expire', refreshTokenData.expire, cookieOptions)

  return result
}
```

`refreshTokenIfNeeded`는 토큰 만료 5분 전 자동으로 갱신 API를 호출하고, 갱신된 토큰을 반환한다. 실패 시 `null` 반환 → `withAuthAction`이 `Unauthorized` throw.

**이 설계의 핵심:**
- 토큰 유효성 검사, 갱신, 쿠키 업데이트를 한 곳에 모아 중복 제거
- handler는 "유효한 tokenData가 보장된 상태"에서 실행 — 인증 걱정 없이 비즈니스 로직만 작성
- 갱신된 토큰을 쿠키에 다시 저장하므로 사용자가 페이지를 이동할 때도 세션 유지

```ts
// 소비 예시 — src/lib/api-options/cart/index.ts
'use server'

export const fetchCartList = async () => {
  return withAuthAction(async (tokenData) => {
    const response = await fetchClientAPI(apiUrl, {
      headers: { Authorization: `Bearer ${tokenData.accessToken}` },
    })
    if (!response.ok) return undefined
    const result: CartListResponse = await response.json()
    return result.data
  })
}
```

새 API 함수를 추가할 때 패턴이 고정되어 있어서 인증 처리를 빠뜨리거나 다르게 구현할 여지가 없다.

**Unauthorized 에러 처리 전략:**

인증 에러는 발생 시점에 따라 두 레이어에서 처리한다.

```
[페이지 진입 시] 미들웨어
  → /mypage, /checkout, /reservation 접근 시 토큰 없으면
  → /login?returnUrl=... 으로 redirect (페이지 렌더 이전에 차단)

[뮤테이션 실행 중 토큰 만료 시] useMutation onError
  → withAuthAction이 throw new Error('Unauthorized')
  → onError: (error) => toast.error(error.message)
```

미들웨어가 페이지 단위 인증을 보장하므로, 뮤테이션 중 Unauthorized는 세션이 만료된 엣지 케이스다. 토스트로 인지시키고 사용자가 재로그인하도록 유도한다.

---

## fetchClientAPI — 타임아웃 + 디버그 로그 내장

단순 fetch 래퍼가 아니라 운영에 필요한 두 가지를 기본 탑재했다.

```ts
export const fetchClientAPI = async (input, options) => {
  const { timeout = 10000, ...init } = options || {}
  const controller = new AbortController()
  const timeoutId = setTimeout(() => controller.abort(), timeout)  // 10초 타임아웃

  const response = await fetch(input, {
    ...init,
    signal: controller.signal,
    cache: 'no-store',
  })

  // 개발 환경 전용 — endpoint, method, request/response body 로깅
  debugLog(response.ok ? '🟢 [fetchClientAPI]' : '🟠 [fetchClientAPI]', { ... })

  return response
}
```

- **타임아웃**: AbortController로 10초 초과 시 자동 abort. 외부 API가 응답 없이 hanging되는 상황 방어.
- **debugLog**: 개발 환경에서 모든 API 요청/응답을 콘솔에 출력. 프로덕션 빌드에서는 제거.

---

## 결과

| 구분 | Before | After |
|------|--------|-------|
| API 호출 구조 | Route Handler 경유 or 클라이언트 직접 호출 혼재 | Server Action 단일 경로 |
| 인증 처리 | 파일마다 토큰 주입 방식 상이 | withAuthAction 래퍼로 통일 |
| 토큰 갱신 | 만료 후 재로그인 유도 | 만료 5분 전 자동 갱신 + 쿠키 업데이트 |
| 외부 API 엔드포인트 | 브라우저 네트워크 탭에 노출 | 서버에서만 호출, 클라이언트 노출 없음 |
| 타임아웃 처리 | 없음 | 10초 자동 abort |
