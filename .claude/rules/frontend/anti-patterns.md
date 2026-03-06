# Anti-Patterns — NEVER DO These

This file is a running log of patterns that are explicitly forbidden. When Claude does something wrong, it gets added here so it never happens again.

## Architecture Anti-Patterns

```typescript
// ❌ Same-layer cross-slice import
// In features/share-screen/:
import { useJoinCall } from '@/features/join-call';
// FIX: Both features should read from entities/call/model/callSlice

// ❌ Upward layer import
// In entities/call/:
import { useStore } from '@/app/store';
// FIX: The store is composed from entity slices, not the other way around

// ❌ Bypassing public API (index.ts)
import { analyzeLevel } from '@/entities/audio-stream/lib/analyzeAudioLevel';
// FIX: import { analyzeLevel } from '@/entities/audio-stream';  (via index.ts)

// ❌ Business logic in shared/
// In shared/lib/someHelper.ts:
import type { Participant } from '@/entities/participant';  // FORBIDDEN — shared has no business logic
// FIX: Move to entities/participant/lib/ or features/*/lib/

// ❌ UI components in model/ segment
// features/join-call/model/JoinCallButton.tsx  ← WRONG segment
// FIX: features/join-call/ui/JoinCallButton.tsx
```

---

## State Management Anti-Patterns

```typescript
// ❌ useStore() without a selector
const store = useStore();  // re-renders on EVERY state change
// FIX: const callStatus = useStore(s => s.callStatus);

// ❌ Storing server responses in Zustand manually
const { data } = useQuery(...);
useEffect(() => { setParticipants(data); }, [data]);  // duplicating TanStack Query cache
// FIX: Read from useQuery directly, let TanStack Query cache it

// ❌ Using TanStack Query for non-server state
const { data: isMuted } = useQuery({ queryKey: ['isMuted'], queryFn: () => false });
// FIX: const [isMuted, setIsMuted] = useAtom(isMutedAtom);

// ❌ Using Zustand for high-frequency per-participant state
// In participantSlice.ts:
audioLevels: Map<string, number>  // this Map update re-renders ALL subscribers
// FIX: Use atomFamily from Jotai — each atom is independent

// ❌ useStore.setState() called from inside a component render
function MyComponent() {
  useStore.setState({ isMuted: true });  // side effect in render — NEVER
  // FIX: Put in useEffect or event handler
}

// ❌ Storing RTCPeerConnection instances in Zustand
peerConnections: Map<string, RTCPeerConnection>  // these are not serializable
// FIX: Store in useRef inside the signaling hook; store only derived state in Zustand
```

---

## TypeScript Anti-Patterns

```typescript
// ❌ any
const response: any = await fetch(...);
// FIX: const response: Room = await fetch(...);

// ❌ Unnecessary type assertions
const user = getUser() as User;  // if it might not be a User, handle it
// FIX: const user = getUser(); if (!user) return null;

// ❌ Non-null assertion on uncertain values
const first = arr[0]!;  // arr could be empty
// FIX: const first = arr[0]; if (!first) return;

// ❌ @ts-ignore without explanation
// @ts-ignore
someWeirdAPI();
// FIX: Fix the type issue or use @ts-expect-error with a comment

// ❌ Missing return type on complex functions
function processSignal(msg) { ... }  // no parameter type, no return type
// FIX: function processSignal(msg: SignalingMessageDTO): Promise<void> { ... }
```

---

## Security Anti-Patterns

```typescript
// ❌ CRITICAL: JWT in localStorage
localStorage.setItem('token', jwt);
sessionStorage.setItem('token', jwt);
// FIX: Use in-memory tokenStorage.ts ONLY

// ❌ Token in URL
navigate(`/call?token=${jwt}`);
fetch(`/api/data?token=${jwt}`);
// FIX: Always use Authorization header (handled by httpClient interceptor)

// ❌ Logging sensitive data
console.log('User token:', token);
console.log('Auth response:', response);  // response contains token
// FIX: Never log tokens, passwords, or auth responses

// ❌ Using fetch() directly with manual auth
fetch('/api/rooms', { headers: { Authorization: `Bearer ${token}` } });
// FIX: Use httpClient which attaches the token automatically

// ❌ Using raw WebSocket for Spring Boot STOMP
new WebSocket('ws://localhost:8080/ws');
// FIX: Use getStompClient() which uses @stomp/stompjs + SockJS
```

---

## Component Anti-Patterns

```typescript
// ❌ Prop drilling more than 2 levels
<Page callStatus={callStatus}>
  <Widget callStatus={callStatus}>
    <Feature callStatus={callStatus}>  // use Zustand here
      <Component callStatus={callStatus} />
    </Feature>
  </Widget>
</Page>

// ❌ useEffect for derived state
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(`${firstName} ${lastName}`); }, [firstName, lastName]);
// FIX: const fullName = `${firstName} ${lastName}`;  (just compute it)

// ❌ Missing cleanup in useEffect with subscriptions
useEffect(() => {
  const sub = subscribeToTopic('/topic/events', handler);
  // ← forgot return () => sub.unsubscribe()
  // This leaks the subscription!
});
// FIX: Always return cleanup

// ❌ Initializing state with a function result (runs every render)
const [rooms, setRooms] = useState(fetchRooms());  // calls fetchRooms on every render!
// FIX: Use TanStack Query for async data

// ❌ Key prop using array index
{participants.map((p, i) => <ParticipantTile key={i} ... />)}
// FIX: key={p.id}  — use stable, unique IDs

// ❌ console.log left in committed code
console.log('DEBUG participant:', participant);
// FIX: Remove before commit. Use devtools instead.
```

---

## File Organization Anti-Patterns

```
❌ Flat files directly in features/
features/JoinCallButton.tsx        ← no slice, no segment
FIX: features/join-call/ui/JoinCallButton.tsx

❌ Missing index.ts public API
features/join-call/ui/JoinCallButton.tsx  (no index.ts)
FIX: create features/join-call/index.ts exporting JoinCallButton

❌ Business logic in shared/
shared/lib/callHelpers.ts  (contains call state logic)
FIX: entities/call/lib/callHelpers.ts

❌ Components in model/ segment
entities/call/model/CallStatusBadge.tsx
FIX: entities/call/ui/CallStatusBadge.tsx

❌ Types scattered everywhere
features/join-call/ui/types.ts  (types at wrong level)
FIX: features/join-call/model/types.ts
```

---

## WebRTC Anti-Patterns

```typescript
// ❌ Storing RTCPeerConnection in React state (causes double-instantiation)
const [pc, setPc] = useState(new RTCPeerConnection());
// FIX: const pcRef = useRef(new VoicePeerConnection(participantId));

// ❌ Creating peer connections outside the signaling hook
// In a button click handler:
const pc = new RTCPeerConnection();
// FIX: Call useSignaling hook's initiateCallWithPeer() method

// ❌ Ignoring connection state failures
pc.onconnectionstatechange = () => {
  if (pc.connectionState === 'connected') { ... }
  // ← missing handling for 'failed', 'disconnected', 'closed'
};

// ❌ Not cleaning up AudioContext
const analyser = createAudioLevelAnalyzer(stream, (level) => {...});
// ← forgot to call the returned cleanup function
// FIX: const cleanup = createAudioLevelAnalyzer(...); return cleanup;
```
