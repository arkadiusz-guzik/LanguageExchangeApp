# Backend Integration Rules

All communication with the Java Spring Boot backend follows these patterns. Do not deviate.

## Two Communication Channels

| Channel | Spring Boot Side | Frontend Client | Used For |
|---|---|---|---|
| REST/HTTP | `@RestController` | `httpClient` (axios) + TanStack Query | CRUD, auth, history, room management |
| WebSocket STOMP | `@MessageMapping` + SockJS | `stompClient` (@stomp/stompjs) | WebRTC signaling, presence, real-time events |

---

## REST: The HTTP Client

**File:** `src/shared/api/httpClient.ts`

```typescript
import axios from 'axios';
import { getToken, clearToken } from '../lib/auth/tokenStorage';
import { ENV } from '../config/env';

export const httpClient = axios.create({
  baseURL: ENV.API_URL,   // e.g. http://localhost:8080/api
  headers: { 'Content-Type': 'application/json' },
});

// Attach JWT to every request
httpClient.interceptors.request.use((config) => {
  const token = getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Handle Spring Security 401 globally
httpClient.interceptors.response.use(
  (res) => res,
  (err) => {
    if (err.response?.status === 401) {
      clearToken();
      window.location.href = '/login';
    }
    return Promise.reject(err.response?.data ?? err);
  }
);
```

**Rules:**
- ALL REST calls go through `httpClient` — never `fetch` directly
- Never set `Authorization` header manually in components or query hooks
- The interceptor handles it automatically
- Error handling for 401 is global — do not duplicate it in individual hooks

---

## WebSocket: The STOMP Client

**File:** `src/shared/api/stompClient.ts`

Spring Boot uses STOMP over SockJS — **not raw WebSocket**. Never use `new WebSocket(...)` to connect to Spring Boot endpoints.

```typescript
import { Client, type IMessage, type StompSubscription } from '@stomp/stompjs';
import SockJS from 'sockjs-client';
import { getToken } from '../lib/auth/tokenStorage';
import { ENV } from '../config/env';

let stompClient: Client | null = null;

export function getStompClient(): Client {
  if (stompClient?.active) return stompClient;

  stompClient = new Client({
    webSocketFactory: () => new SockJS(`${ENV.WS_URL}/ws`),
    // JWT in CONNECT headers — SockJS cannot send HTTP headers
    connectHeaders: { Authorization: `Bearer ${getToken() ?? ''}` },
    reconnectDelay: 5000,
    heartbeatIncoming: 4000,
    heartbeatOutgoing: 4000,
    onStompError: (frame) => {
      console.error('STOMP error:', frame.headers['message']);
    },
  });

  stompClient.activate();
  return stompClient;
}

export function subscribeToTopic<T>(
  destination: string,
  callback: (data: T) => void
): StompSubscription {
  return getStompClient().subscribe(destination, (msg: IMessage) => {
    callback(JSON.parse(msg.body) as T);
  });
}

export function publishMessage<T>(destination: string, body: T): void {
  getStompClient().publish({ destination, body: JSON.stringify(body) });
}
```

**Rules:**
- `getStompClient()` is the ONLY way to access the STOMP connection
- JWT is passed in `connectHeaders` — this is validated by Spring's `ChannelInterceptor`
- Always unsubscribe in `useEffect` cleanup: `return () => subscription.unsubscribe()`
- Always call `publishMessage('/app/call.leave', ...)` on cleanup to notify Spring

---

## Spring Boot STOMP Topic Map

```
CLIENT SUBSCRIBES TO (receives messages FROM Spring):
  /topic/call/{roomId}/events        → broadcast: USER_JOINED, USER_LEFT, CALL_ENDED
  /topic/call/{roomId}/participants  → broadcast: full participant list update
  /topic/call/{roomId}/chat          → broadcast: new chat messages
  /user/queue/call/offer             → user-specific: incoming SDP offer from peer
  /user/queue/call/answer            → user-specific: incoming SDP answer from peer
  /user/queue/call/ice               → user-specific: ICE candidate from peer

CLIENT PUBLISHES TO (sends messages TO Spring):
  /app/call.join    { roomId }                                → join a room
  /app/call.leave   { roomId }                                → leave a room
  /app/call.offer   { toUserId, sdp: RTCSessionDescriptionInit }
  /app/call.answer  { toUserId, sdp: RTCSessionDescriptionInit }
  /app/call.ice     { toUserId, candidate: RTCIceCandidateInit }
  /app/chat.send    { roomId, content }                       → send chat message
```

**Rule:** Never hardcode these strings inline. Always import from `shared/config/constants.ts`:

```typescript
// shared/config/constants.ts
export const STOMP_TOPICS = {
  ROOM_EVENTS: (roomId: string) => `/topic/call/${roomId}/events`,
  ROOM_PARTICIPANTS: (roomId: string) => `/topic/call/${roomId}/participants`,
  ROOM_CHAT: (roomId: string) => `/topic/call/${roomId}/chat`,
  OFFER: '/user/queue/call/offer',
  ANSWER: '/user/queue/call/answer',
  ICE: '/user/queue/call/ice',
} as const;

export const STOMP_DESTINATIONS = {
  JOIN: '/app/call.join',
  LEAVE: '/app/call.leave',
  OFFER: '/app/call.offer',
  ANSWER: '/app/call.answer',
  ICE: '/app/call.ice',
  CHAT: '/app/chat.send',
} as const;
```

---

## Authentication: JWT Flow

```typescript
// shared/lib/auth/tokenStorage.ts
// IN-MEMORY ONLY — NEVER localStorage (XSS risk)
let _token: string | null = null;

export const setToken = (token: string) => { _token = token; };
export const getToken = () => _token;
export const clearToken = () => { _token = null; };
export const isAuthenticated = () => _token !== null;
```

**Login flow:**
1. `POST /api/auth/signin` → Spring returns `{ token, userId, username, roles }`
2. `setToken(response.token)` → stored in memory only
3. `httpClient` interceptor auto-attaches to all REST calls
4. `stompClient` uses `getToken()` in `connectHeaders`

**Token refresh:**
- Refresh token lives in httpOnly cookie (set by Spring Boot — not accessible from JS)
- When 401 occurs: interceptor calls `POST /api/auth/refresh`
- If refresh fails: `clearToken()` + redirect to `/login`

**Rules:**
- NEVER `localStorage.setItem('token', ...)` — this is a critical security violation
- NEVER send the token as a URL query parameter except for STOMP initial handshake
- NEVER log the token to the console

---

## Spring Boot Response Types

Always use these typed interfaces when consuming Spring Boot responses:

```typescript
// shared/api/types.ts

// Standard Spring Boot error shape
export interface SpringApiError {
  timestamp: string;
  status: number;
  error: string;
  message: string;
  path: string;
}

// Spring Data Page<T> — paginated list responses
export interface SpringPage<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  number: number;      // current page, 0-indexed
  size: number;
  first: boolean;
  last: boolean;
  empty: boolean;
}
```

---

## Environment Variables

```typescript
// shared/config/env.ts
export const ENV = {
  API_URL: import.meta.env.VITE_APP_API_URL as string,
  WS_URL: import.meta.env.VITE_APP_WS_URL as string,
  ICE_SERVERS: JSON.parse(import.meta.env.VITE_APP_ICE_SERVERS) as RTCIceServer[],
} as const;
```

```bash
# .env.development
VITE_APP_API_URL=http://localhost:8080/api
VITE_APP_WS_URL=http://localhost:8080
VITE_APP_ICE_SERVERS=[{"urls":"stun:stun.l.google.com:19302"}]
```

**Rule:** Never use `import.meta.env.VITE_*` directly in components or hooks. Always import from `ENV` in `shared/config/env.ts`.

---

## CORS — Both Sides Must Be Configured

Spring Boot requires CORS in TWO places:
1. `WebMvcConfigurer` for REST endpoints (`/api/**`)
2. `WebSocketMessageBrokerConfigurer` for STOMP endpoint (`/ws`)

Missing either causes failures. If STOMP fails silently, check the WebSocket CORS config first.

---

## Typing Spring Boot DTOs

Create TypeScript interfaces in `shared/types/springTypes.ts` that mirror Spring Boot DTO classes:

```typescript
// shared/types/springTypes.ts
export interface UserDTO {
  id: string;
  username: string;
  email: string;
  roles: string[];
  createdAt: string;  // ISO datetime string from Java
}

export interface RoomDTO {
  id: string;
  name: string;
  hostId: string;
  participantCount: number;
  isPublic: boolean;
  createdAt: string;
}

export interface JoinCallResponseDTO {
  sessionId: string;
  roomId: string;
}

export interface SignalingMessageDTO {
  fromUserId: string;
  sdp?: RTCSessionDescriptionInit;
  candidate?: RTCIceCandidateInit;
}
```

**Rule:** When Spring Boot adds a new field to a DTO, update the corresponding TypeScript interface immediately. Type mismatches between frontend and backend are caught at `pnpm typecheck`.
