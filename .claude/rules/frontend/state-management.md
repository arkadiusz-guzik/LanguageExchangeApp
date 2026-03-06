# State Management Rules

Three tools. Three separate domains. They do NOT overlap.

## The Three-Tool Rule

| State Type | Tool | Where It Lives |
|---|---|---|
| Server / async data (from Spring Boot API) | **TanStack Query** | `entities/*/api/` and `features/*/api/` |
| Global UI + domain state | **Zustand** | `entities/*/model/*Slice.ts`, composed in `app/store/` |
| Fine-grained per-item reactive state | **Jotai** | `entities/*/model/*Atoms.ts`, `features/*/model/*Atom.ts` |
| Local ephemeral state | **useState / useReducer** | Inside component, not exported |

**NEVER mix these.** Never put server responses into Zustand manually. Never duplicate Zustand state in TanStack Query. Never use Jotai for global session state.

---

## TanStack Query — Server State Rules

### What Goes Here
- All REST API calls to Spring Boot
- Pagination of room lists, call history, user search
- User profile, device preferences stored server-side
- Any data that lives on the backend

### How to Structure Query Hooks

```typescript
// entities/call/api/callApi.ts

export const callKeys = {
  all: ['calls'] as const,
  list: () => ['calls', 'list'] as const,
  detail: (id: string) => ['calls', id] as const,
};

// Query hook
export function useCall(callId: string) {
  return useQuery({
    queryKey: callKeys.detail(callId),
    queryFn: () => httpClient.get<Call>(`/calls/${callId}`).then(r => r.data),
    staleTime: 30_000,           // 30s before considered stale
    enabled: !!callId,           // don't run with empty ID
  });
}

// Mutation hook
export function useCreateRoom() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: CreateRoomRequest) =>
      httpClient.post<Room>('/rooms', data).then(r => r.data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: callKeys.list() });
    },
  });
}
```

### Spring Boot Pagination

```typescript
// ALWAYS handle Spring's Page<T> format
export function useRooms(page = 0, size = 20) {
  return useQuery({
    queryKey: callKeys.list(),
    queryFn: () =>
      httpClient
        .get<SpringPage<Room>>(`/rooms?page=${page}&size=${size}&sort=createdAt,desc`)
        .then(r => r.data),
    placeholderData: keepPreviousData,   // smooth pagination UX
  });
}
```

### Rules
- Define query key objects (`xxxKeys`) alongside query hooks — always in `api/` segment
- `staleTime` should always be explicit — don't rely on defaults
- Use `enabled` to prevent queries from running with invalid params
- Use `invalidateQueries` after mutations that change server data
- NEVER manually set query data with `setQueryData` unless doing optimistic updates
- NEVER store the query result in Zustand — let TanStack Query cache it

---

## Zustand — Global UI State Rules

### What Goes Here
- Current call connection status (`'idle' | 'connecting' | 'connected' | 'failed'`)
- Active participant list (received from STOMP, not REST)
- Current user's audio/video device selection
- Current user's authentication state (after login)
- WebRTC peer connection map (keyed by participantId)
- Whether screen sharing is active
- Call start time (for duration timer)

### Store Architecture: Sliced Composition

```typescript
// app/store/index.ts — ONLY file that creates the store
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';
import { createCallSlice, type CallSlice } from '@/entities/call/model/callSlice';
import { createParticipantSlice, type ParticipantSlice } from '@/entities/participant/model/participantSlice';
import { createUserSlice, type UserSlice } from '@/entities/user/model/userSlice';
import { createAudioDeviceSlice, type AudioDeviceSlice } from '@/features/change-audio-device/model/audioDeviceSlice';

type RootStore = CallSlice & ParticipantSlice & UserSlice & AudioDeviceSlice;

export const useStore = create<RootStore>()(
  devtools(
    (...args) => ({
      ...createCallSlice(...args),
      ...createParticipantSlice(...args),
      ...createUserSlice(...args),
      ...createAudioDeviceSlice(...args),
    }),
    { name: 'VoiceAppStore' }
  )
);

// Selector hooks — ALWAYS create these to prevent unnecessary re-renders
export const useCallStatus = () => useStore(s => s.callStatus);
export const useParticipants = () => useStore(s => s.participants);
export const useCurrentUser = () => useStore(s => s.currentUser);
export const useIsScreenSharing = () => useStore(s => s.isScreenSharing);
export const usePeerConnections = () => useStore(s => s.peerConnections);
```

### Slice Definition Pattern

```typescript
// entities/call/model/callSlice.ts
import type { StateCreator } from 'zustand';
import type { CallStatus } from './types';

export interface CallSlice {
  callStatus: CallStatus;
  callRoomId: string | null;
  callStartTime: number | null;
  // Actions
  setCallStatus: (status: CallStatus) => void;
  setCallRoom: (roomId: string) => void;
  resetCall: () => void;
}

const initialState = {
  callStatus: 'idle' as CallStatus,
  callRoomId: null,
  callStartTime: null,
};

export const createCallSlice: StateCreator<CallSlice> = (set) => ({
  ...initialState,
  setCallStatus: (status) => set({ callStatus: status }),
  setCallRoom: (roomId) => set({ callRoomId: roomId, callStartTime: Date.now() }),
  resetCall: () => set(initialState),
});
```

### Zustand Rules
- NEVER call `useStore` without a selector — `useStore()` re-renders on ANY state change
- ALWAYS use selector hooks: `const status = useCallStatus()` not `const { callStatus } = useStore()`
- Slices are defined in their owning layer's `model/` segment, composed ONLY in `app/store/`
- Actions are defined on the slice, not as standalone functions
- Use `devtools` middleware in development for debugging
- Use `persist` middleware (with `partialize`) only for settings that should survive page refresh

---

## Jotai — Fine-Grained Atomic State Rules

### What Goes Here
- Per-participant audio level (0–100, updates 10x/second)
- Per-participant mute state (boolean toggle)
- Per-participant video enabled state
- Per-participant speaking indicator (boolean, changes frequently)
- Any state that updates at high frequency for individual items

### Why Jotai for This

When a single participant's audio level changes, ONLY the component showing that participant's level should re-render. With Zustand, changing any part of the participants array would re-render all participants. `atomFamily` solves this by creating one independent atom per participantId.

### AtomFamily Pattern

```typescript
// entities/audio-stream/model/audioStreamAtoms.ts
import { atom } from 'jotai';
import { atomFamily } from 'jotai/utils';

// One atom per participant — only the subscriber for that ID re-renders
export const audioLevelAtomFamily = atomFamily(
  (_participantId: string) => atom(0),  // initial level 0
);

export const isMutedAtomFamily = atomFamily(
  (_participantId: string) => atom(false),
);

export const isSpeakingAtomFamily = atomFamily(
  (_participantId: string) => atom(false),
);

export const videoEnabledAtomFamily = atomFamily(
  (_participantId: string) => atom(true),
);
```

### Usage in Components

```typescript
// entities/participant/ui/ParticipantAudioLevel.tsx
import { useAtom } from 'jotai';
import { audioLevelAtomFamily } from '@/entities/audio-stream';

interface Props { participantId: string; }

export function ParticipantAudioLevel({ participantId }: Props) {
  const [audioLevel] = useAtom(audioLevelAtomFamily(participantId));
  // Only re-renders when THIS participant's level changes
  return <div style={{ width: `${audioLevel}%` }} className="audio-bar" />;
}
```

### Jotai Rules
- Use `atomFamily` for per-item state — never store arrays of state in a single atom
- Clean up atomFamily atoms when participants leave: `audioLevelAtomFamily.remove(participantId)`
- Don't use Jotai for state that multiple unrelated components need to coordinate on — use Zustand
- Simple toggles shared globally (like "is mic muted") can use a plain `atom` in the feature model

---

## useState / useReducer — Local State Rules

Use for:
- Dropdown/modal open state
- Form input values (unless using react-hook-form)
- Hover/focus states
- Animation state
- Temporary UI feedback (copied!, success!, etc.)

NEVER lift these to Zustand. If you think you need to, ask whether another component actually needs that state. If not, keep it local.

---

## The Interaction Pattern: Mutation → Zustand → Jotai

When a user joins a call:
1. `useJoinCall()` mutation (TanStack Query) fires `POST /api/rooms/{id}/join`
2. On success: `useStore.getState().setCallStatus('connecting')` (Zustand)
3. STOMP subscription fires when WebRTC signaling begins
4. `setJoinCallStatus` actions update Zustand as ICE state changes
5. Per-participant audio levels feed into `audioLevelAtomFamily` (Jotai)
6. UI reacts: call status badge re-renders (Zustand), audio meters re-render (Jotai), call history refetches (TanStack Query)

Each tool handles exactly what it is designed for. They compose cleanly.
