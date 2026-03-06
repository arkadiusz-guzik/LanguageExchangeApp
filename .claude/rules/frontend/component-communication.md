# Component Communication Rules

How components, hooks, and slices communicate with each other — and what patterns are forbidden.

## The Communication Hierarchy

```
Pages / Widgets
   ↓ compose via imports
Features (User actions)
   ↓ read from / mutate via
Entities (Domain state and API)
   ↓ use primitives from
Shared (No business logic)
```

Each level can ONLY communicate downward. Communication "up" or "sideways" must be restructured.

---

## Pattern 1: Props (Parent → Child)

The default for all UI composition. Higher-layer components pass data and callbacks to lower-layer components they render.

```typescript
// widgets/participant-grid/ui/ParticipantGrid.tsx
// Widget reads state, passes data down to entity components via props
import { ParticipantAvatar } from '@/entities/participant';

export function ParticipantGrid() {
  const participants = useParticipants();   // Zustand selector

  return (
    <div className="grid grid-cols-3 gap-2">
      {participants.map(p => (
        <ParticipantAvatar
          key={p.id}
          participant={p}
          onAvatarClick={() => openParticipantMenu(p.id)}
        />
      ))}
    </div>
  );
}
```

**When to use:** Any time a parent renders a child and the data/handlers originate in the parent. Do not prop-drill more than 2 levels — if drilling is needed, use shared state instead.

---

## Pattern 2: Zustand (Global State Subscription)

When multiple unrelated components need the same state, they each subscribe independently. No prop drilling. No context needed.

```typescript
// These components are completely unrelated in the tree
// but both read from the same store slice

// In call-controls widget:
function MuteButton() {
  const isMuted = useStore(s => s.isMuted);          // ← selector
  const toggleMute = useStore(s => s.toggleMute);    // ← action
  return <button onClick={toggleMute}>{isMuted ? 'Unmute' : 'Mute'}</button>;
}

// Somewhere in participant-grid widget:
function ParticipantTile({ participant }: { participant: Participant }) {
  const isMuted = useStore(s => s.isMuted);          // ← same state, independent sub
  return <div>{isMuted && <MutedIcon />}</div>;
}
```

**Rule:** ALWAYS use a selector — `useStore(s => s.specificField)` — never `useStore()` without a selector. Without a selector, the component re-renders on every store change.

---

## Pattern 3: Jotai Atoms (Fine-Grained Subscriptions)

For per-item state that updates at high frequency. Each component subscribes to exactly one atom and only re-renders when that atom changes.

```typescript
// This component only re-renders when participant 'abc' audio level changes
// Other participants' level changes do NOT cause this to re-render

function ParticipantAudioMeter({ participantId }: { participantId: string }) {
  const [level] = useAtom(audioLevelAtomFamily(participantId));
  return <div style={{ height: `${level}%` }} />;
}

// Audio level is updated externally (e.g., from WebRTC audio analysis):
function updateAudioLevel(participantId: string, level: number) {
  const store = getDefaultStore();
  store.set(audioLevelAtomFamily(participantId), level);
}
```

---

## Pattern 4: TanStack Query (Server State)

Server data is accessed through query hooks. Components do not know about HTTP — they only know about the hook's result.

```typescript
// pages/lobby/ui/LobbyPage.tsx
function LobbyPage() {
  // Data comes from TanStack Query cache, auto-refetched when stale
  const { data: rooms, isLoading, isError } = useRooms();

  if (isLoading) return <LoadingSpinner />;
  if (isError) return <ErrorMessage />;

  return <RoomList rooms={rooms?.content ?? []} />;
}
```

---

## Pattern 5: Widget Orchestration (Cross-Feature Coordination)

**Features cannot import from each other.** When two features need to work together, a widget or page above them does the coordination by composing them.

```typescript
// widgets/call-controls/ui/CallControls.tsx
// This widget is the ORCHESTRATOR — it knows about multiple features

import { MuteButton } from '@/features/mute-toggle';          // feature A
import { ShareScreenButton } from '@/features/share-screen';  // feature B
import { LeaveCallButton } from '@/features/leave-call';      // feature C

export function CallControls() {
  // The widget composes features. Features don't know about each other.
  return (
    <div className="flex gap-2 items-center">
      <MuteButton />
      <ShareScreenButton />
      <LeaveCallButton />
    </div>
  );
}
```

---

## Pattern 6: Entity State as Cross-Feature Shared Ground

When feature A needs data that feature B produces, that data belongs in an entity, not in either feature.

```typescript
// ❌ WRONG — feature importing from feature
// features/share-screen/ui/ShareScreenButton.tsx
import { useIsCallActive } from '@/features/join-call';  // FORBIDDEN

// ✅ CORRECT — both features read from the entity below them
// features/share-screen/ui/ShareScreenButton.tsx
import { useCallStatus } from '@/entities/call';          // entity is below features

export function ShareScreenButton() {
  const callStatus = useCallStatus();
  const isInCall = callStatus === 'connected';
  // ...
}
```

---

## Pattern 7: STOMP Event → Zustand → UI

For real-time events from Spring Boot STOMP (participants joining/leaving, signaling messages), the flow is:

```
STOMP message arrives
    → features/webrtc-signaling/api/useSignaling.ts processes it
    → calls Zustand action: useStore.getState().addParticipant(participant)
    → Zustand notifies all subscribed components
    → UI re-renders
```

```typescript
// features/webrtc-signaling/api/useSignaling.ts
export function useSignaling(roomId: string) {
  useEffect(() => {
    const sub = subscribeToTopic<RoomEvent>(
      `/topic/call/${roomId}/events`,
      (event) => {
        const store = useStore.getState();    // ← imperative store access for side effects
        if (event.type === 'USER_JOINED') {
          store.addParticipant(event.participant);
        }
        if (event.type === 'USER_LEFT') {
          store.removeParticipant(event.userId);
          peerConnectionMap.get(event.userId)?.close();
        }
      }
    );
    return () => sub.unsubscribe();
  }, [roomId]);
}
```

**Rule:** Use `useStore.getState()` (imperative) inside `useEffect`, callbacks, and non-React code. Use `useStore(selector)` (reactive) inside React component render functions.

---

## Pattern 8: Callback Prop for Upward Communication

When a child needs to trigger something in its parent, use callback props. Do NOT reach for global state for this — it is a parent-child concern.

```typescript
// pages/call-room/ui/CallRoomPage.tsx
function CallRoomPage() {
  const navigate = useNavigate();

  const handleCallEnded = () => {
    navigate('/lobby');        // page-level navigation decision
  };

  return (
    <div>
      <CallControls onCallEnded={handleCallEnded} />
    </div>
  );
}

// widgets/call-controls/ui/CallControls.tsx
interface CallControlsProps {
  onCallEnded: () => void;   // callback up to the page
}

export function CallControls({ onCallEnded }: CallControlsProps) {
  return (
    <LeaveCallButton onLeave={onCallEnded} />
  );
}
```

---

## Forbidden Communication Patterns

```typescript
// ❌ Cross-slice import on same layer
import { useJoinCall } from '@/features/join-call';
// Inside features/mute-toggle/ — FORBIDDEN

// ❌ Upward import
import { useStore } from '@/app/store';
// Inside shared/ — FORBIDDEN

// ❌ Importing internal slice files directly
import { computeAudioLevel } from '@/entities/audio-stream/lib/analyzeAudioLevel';
// Should be: import { computeAudioLevel } from '@/entities/audio-stream';

// ❌ Using Zustand for rapidly-changing per-item state
const audioLevels = useStore(s => s.audioLevels); // if this is a Map that updates 10x/sec

// ❌ Using TanStack Query for UI state (not server state)
const { data: isMuted } = useQuery({ queryKey: ['isMuted'] });

// ❌ Prop drilling past 2 levels
<Parent>
  <Child muteState={muteState}>
    <GrandChild muteState={muteState}>
      <GreatGrandChild muteState={muteState} />  // use Zustand instead
    </GrandChild>
  </Child>
</Parent>

// ❌ Direct DOM manipulation
document.getElementById('audio-level-bar').style.width = '50%';
// Use Jotai atom instead

// ❌ Global mutable singleton for state
window.__appState = { isMuted: true };  // NEVER
```

---

## React Context — When to Use It

React Context is NOT a state management tool in this project. Use it ONLY for:
- Dependency injection of services that don't change (theme, i18n, analytics client)
- Avoiding passing stable values through many component levels

NEVER use Context for frequently changing values — it causes every consumer to re-render. For shared changing state, use Zustand.

```typescript
// ✅ OK — stable reference, rarely changes
const AnalyticsContext = createContext<AnalyticsService | null>(null);

// ❌ WRONG — use Zustand for this
const CallStatusContext = createContext<CallStatus>('idle');
```
