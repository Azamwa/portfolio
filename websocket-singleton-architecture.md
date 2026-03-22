# WebSocket 싱글턴 아키텍처 — 단일 커넥션 공유 + 페이지별 구독 스코핑 설계

## 문제 상황

FMS 라이브맵은 WebSocket으로 차량의 GPS, TPMS, AI Feature 데이터를 실시간 수신한다.

초기에는 라이브맵 페이지에서만 소켓을 사용했지만, 대시보드·트립 등 다른 페이지에서도 실시간 데이터가 필요해지면서 문제가 생겼다.

- 페이지마다 소켓 커넥션을 새로 열면 서버 부하 + 중복 데이터 수신
- 페이지 전환 시마다 연결/해제 반복 → 연결 지연 + 데이터 공백
- 어떤 페이지가 어떤 이벤트를 구독하는지 관리 구조가 없음

---

## 설계 1: 싱글턴으로 단일 커넥션 유지

앱 전체에서 소켓 커넥션을 하나만 유지하고, 모든 페이지가 이를 공유한다.

```ts
export class SocketManager {
  private static instance: SocketManager;
  private socket: Socket | null = null;
  private connectPromise: Promise<void> | null = null;

  private constructor() {} // 외부 생성 차단

  static getInstance(): SocketManager {
    if (!SocketManager.instance) {
      SocketManager.instance = new SocketManager();
    }
    return SocketManager.instance;
  }
}

// 모듈 레벨에서 인스턴스 제공 — import만 하면 어디서든 동일 인스턴스
export const socketManager = SocketManager.getInstance();
```

**중복 연결 방지:**

React StrictMode에서 useEffect가 2번 호출되거나, 여러 페이지가 동시에 `connect()`를 호출해도 커넥션이 하나만 생성된다.

```ts
connect(url: string, sessionId?: string): Promise<void> {
  // 이미 연결됨 → 즉시 resolve
  if (this.socket?.connected) {
    return Promise.resolve();
  }

  // 연결 진행 중 → 같은 Promise 반환
  if (this.isConnecting && this.connectPromise) {
    return this.connectPromise;
  }

  // 최초 연결 → 새 Promise 생성
  this.connectPromise = new Promise((resolve, reject) => {
    this.socket = io(socketUrl, { /* 옵션 */ });
    this.socket.connect();
    this.socket.on("connect", () => resolve());
  });

  return this.connectPromise;
}
```

---

## 설계 2: 이벤트 리스너 풀링 — 옵저버 패턴 기반 브로드캐스트

소켓 이벤트 하나에 여러 컴포넌트가 구독할 수 있는 구조다.

`Map<eventType, Set<listener>>` 로 이벤트 타입별 리스너를 관리하고, 소켓 이벤트가 도착하면 등록된 리스너 전체에 브로드캐스트한다.

```ts
private eventListeners: Map<string, Set<(event: LiveMapWebSocketEvent) => void>> = new Map();

addEventListener(eventType: string, listener: (event: LiveMapWebSocketEvent) => void): () => void {
  if (!this.eventListeners.has(eventType)) {
    this.eventListeners.set(eventType, new Set());
  }
  this.eventListeners.get(eventType)?.add(listener);

  // unsubscribe 함수 반환 → useEffect cleanup에서 사용
  return () => {
    this.eventListeners.get(eventType)?.delete(listener);
  };
}
```

이벤트가 도착하면 해당 타입의 리스너 Set을 순회하며 전체에 브로드캐스트한다.

소켓 이벤트 핸들러는 이벤트 타입당 1개만 등록하고, 내부에서 리스너 풀에 분배한다.

```ts
// 소켓 → SocketManager → N개 리스너로 분배
this.socket.on("livemap:gps:updated", (data) => {
  this.notifyListeners("livemap:gps:updated", {
    type: "livemap:gps:updated",
    data,
  });
});
```

컴포넌트에서는 useEffect cleanup으로 구독을 해제한다.

```tsx
useEffect(() => {
  const unsubscribe = socketManager.addEventListener(
    "livemap:gps:updated",
    (event) => {
      // GPS 데이터 처리
    },
  );
  return unsubscribe; // 언마운트 시 리스너 제거
}, []);
```

---

## 설계 3: 연결 안정성 — 자동 재연결 + Transport 폴백

### Socket.io 내장 재연결

Socket.io의 옵션으로 재연결 전략과 transport 폴백을 설정했다.

| 전략            | 설정                    | 동작                                                           |
| --------------- | ----------------------- | -------------------------------------------------------------- |
| Transport 우선순위 | `["websocket", "polling"]` | WebSocket 우선, 실패 시 polling 폴백                        |
| 지수 백오프     | 1초 ~ 최대 5초          | 1초 → 2초 → 4초 → 5초(max) 간격으로 재시도                     |
| 최대 재시도     | 5회                     | 5회 실패 시 연결 포기                                          |
| Transport 폴백  | `upgrade: true`         | 기업 프록시/방화벽에서 WebSocket 차단 시 polling으로 통신 유지 |
| 업그레이드 기억 | `rememberUpgrade: true` | 다음 연결부터 polling 없이 바로 WebSocket 시도                 |

### 연결 상태 리스너

연결 상태도 이벤트 리스너와 동일한 옵저버 패턴(`Set<listener>` + unsubscribe 반환)으로 구독한다.

재연결 중에는 disconnect 알림을 보류하고, 최종 실패 시에만 `false`를 전달해서 UI가 불필요하게 깜빡이는 것을 방지한다.

**알고 있는 한계:** 현재 Socket.io 내장 재연결(`reconnectionAttempts: 5`)과 `connect_error` 이벤트에서의 수동 카운팅이 동시에 동작한다. Socket.io가 아직 재연결을 시도 중인데 수동 카운터가 먼저 상한에 도달해 Promise를 reject할 수 있는 충돌 가능성이 있으며, 내장 재연결에 위임하고 수동 카운팅을 제거하는 방향으로 개선이 필요하다.

### 재연결 성공 시 자동 재구독

연결이 복원되면 `subscribeLiveMap`을 다시 호출해서 데이터 스트림을 복원한다.

```ts
// livemap-context.tsx
const unsubscribe = socketManager.addConnectionListener((connected) => {
  setIsConnected(connected);
  if (connected && effectiveSessionId) {
    socketManager.subscribeLiveMap(effectiveSessionId); // 재연결 시 자동 재구독
  }
});
```

---

## 설계 4: 페이지별 구독 스코핑 — 싱글턴 vs Context 역할 분리

싱글턴이 소켓 커넥션을 전역 관리하고, 각 페이지의 Context가 구독 범위와 상태를 스코핑한다.

```
┌─────────────────────────────────────────────────┐
│  SocketManager (싱글턴)                          │
│  - 단일 커넥션 관리                               │
│  - 이벤트 리스너 풀링                             │
│  - 연결/재연결/정리                               │
├─────────────────────────────────────────────────┤
│                    │                             │
│  LiveMapProvider   │  DashboardProvider (예정)    │
│  (Context)         │  (Context)                  │
│  - GPS 구독        │  - 요약 데이터 구독           │
│  - TPMS 구독       │                             │
│  - AI Feature 구독 │                             │
│  - RAF 배칭        │                             │
│  - 5초 쓰로틀      │                             │
└─────────────────────────────────────────────────┘
```

**왜 Zustand가 아닌 Context인가:**

페이지별로 구독 범위가 다르기 때문이다. 라이브맵은 GPS + TPMS + AI Feature를 모두 구독하지만, 대시보드는 요약 데이터만 필요할 수 있다. Context Provider가 언마운트되면 cleanup에서 해당 페이지의 구독만 자연스럽게 해제된다.

```tsx
// LiveMapProvider 언마운트 시 — 라이브맵 구독만 해제, 소켓 커넥션은 유지
useEffect(() => {
  return () => {
    if (rafIdRef.current !== null) {
      cancelAnimationFrame(rafIdRef.current);
    }
    if (currentSessionId && socketManager.isConnectedStatus()) {
      socketManager.unsubscribeLiveMap(currentSessionId); // 구독만 해제
    }
  };
}, [effectiveSessionId]);
```

Zustand는 전역 스토어라서 페이지별 구독 lifecycle을 분리하기 어렵다. Context는 Provider 마운트/언마운트에 구독 범위가 자연스럽게 바인딩된다.

**비정상 세션 시 전체 정리:**

세션 만료(401)나 중복 로그인(409)이 감지되면, Context 레벨에서 소켓을 끊고 상태를 일괄 초기화한다. 싱글턴은 커넥션만 관리하고, 세션 유효성 판단과 정리 트리거는 Context가 담당하는 역할 분리다.

```tsx
useEffect(() => {
  if (
    sessionError &&
    (sessionError.statusCode === 401 || sessionError.statusCode === 409)
  ) {
    socketManager.disconnect(); // 싱글턴에 연결 해제 요청
    setMapTrips(new Map()); // Context 상태 초기화
    setListTrips(new Map());
    setError(new Error("Session expired"));
  }
}, [sessionError]);
```

---

## 결과

| 구분        | Before                            | After                                              |
| ----------- | --------------------------------- | -------------------------------------------------- |
| 소켓 커넥션 | 페이지마다 새 커넥션              | 싱글턴으로 앱 전체 1개 커넥션 공유                 |
| 이벤트 구독 | 소켓에 직접 리스너 등록           | 이벤트 리스너 풀링으로 N개 컴포넌트에 브로드캐스트 |
| 페이지 전환 | 연결 해제 → 재연결 지연           | 커넥션 유지, 구독만 교체                           |
| 연결 끊김   | 네트워크 단절 시 재연결 로직 없음 | 지수 백오프 자동 재연결 + polling 폴백             |
| 구독 정리   | 전역 상태라 정리 누락 가능        | Context 언마운트에 바인딩, 자동 cleanup            |
| 중복 연결   | StrictMode에서 2중 연결 가능      | connectPromise 캐싱으로 방지                       |
