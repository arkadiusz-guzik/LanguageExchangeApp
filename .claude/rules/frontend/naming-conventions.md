# Naming Conventions

Consistent naming is enforced across all files. When in doubt, follow this file exactly.

## Folders and Directories

| Location | Convention | Examples |
|---|---|---|
| FSD layers | lowercase | `app/`, `pages/`, `widgets/`, `features/`, `entities/`, `shared/` |
| FSD slices | kebab-case | `join-call/`, `audio-stream/`, `call-room/`, `participant-grid/` |
| Segments | lowercase | `ui/`, `model/`, `api/`, `lib/`, `config/` |
| Sub-folders | kebab-case | `shared/lib/webrtc/`, `shared/lib/auth/` |

**NEVER use PascalCase for folders. NEVER use camelCase for folders.**

## Files

### React Components → PascalCase.tsx
```
JoinCallButton.tsx
ParticipantAvatar.tsx
CallControls.tsx
AudioDeviceSelect.tsx
```
The filename MUST match the exported component name exactly.

### Hooks → use + PascalCase.ts
```
useJoinCall.ts         (TanStack Query mutation)
useParticipants.ts     (TanStack Query query)
useWebRTC.ts           (custom WebRTC hook)
usePeerConnection.ts   (custom hook)
useSignaling.ts        (STOMP subscription hook)
```

### Zustand Store Slices → camelCase + Slice.ts
```
callSlice.ts
participantSlice.ts
audioDeviceSlice.ts
```

### Jotai Atoms → camelCase + Atom(s).ts or camelCase + AtomFamily.ts
```
muteAtom.ts
audioLevelAtoms.ts
isSpeakingAtomFamily.ts
```

### API / Query Key Files → camelCase + Api.ts or just the hook file
```
callApi.ts
participantApi.ts
userApi.ts
```

### Type definition files → types.ts (always this name inside a slice)
```
features/join-call/model/types.ts
entities/call/model/types.ts
```

### Utility/helper files → camelCase.ts
```
analyzeAudioLevel.ts
formatDuration.ts
getDisplayMedia.ts
roomKey.ts
```

### Constants files → constants.ts or config.ts
```
shared/config/constants.ts
shared/config/env.ts
```

### Public API entry → index.ts (always this name)
```
features/join-call/index.ts
entities/call/index.ts
widgets/call-controls/index.ts
```

## TypeScript Identifiers

### React Components → PascalCase
```typescript
export function CallControls() { ... }
export function ParticipantAvatar({ participant }: Props) { ... }
export const MuteButton = () => { ... }  // also fine
```

### Hooks → camelCase starting with `use`
```typescript
export function useJoinCall() { ... }
export function usePeerConnection(peerId: string) { ... }
```

### Zustand Store and Selectors → camelCase
```typescript
export const useStore = create<RootStore>(...)

// Selector hooks — use + descriptive name
export const useCallStatus = () => useStore(s => s.callStatus);
export const useParticipants = () => useStore(s => s.participants);
export const useCurrentUser = () => useStore(s => s.currentUser);
```

### Zustand Slice Creators → create + PascalCase + Slice
```typescript
export function createCallSlice(...): CallSlice { ... }
export function createParticipantSlice(...): ParticipantSlice { ... }
```

### Jotai Atoms → camelCase + Atom suffix
```typescript
export const isMutedAtom = atom(false);
export const callStatusAtom = atom<CallStatus>('idle');
export const audioLevelAtomFamily = atomFamily((participantId: string) => atom(0));
```

### TypeScript Interfaces and Types → PascalCase
```typescript
export interface Call { id: string; roomId: string; status: CallStatus; }
export interface Participant { id: string; userId: string; isMuted: boolean; }
export type CallStatus = 'idle' | 'connecting' | 'connected' | 'failed' | 'ended';
export type ParticipantRole = 'host' | 'guest';
```

### Constants → SCREAMING_SNAKE_CASE
```typescript
export const MAX_PARTICIPANTS = 50;
export const ICE_SERVERS: RTCIceServer[] = [...];
export const STOMP_RECONNECT_DELAY = 5000;
export const CALL_HEARTBEAT_INTERVAL = 4000;
```

### TanStack Query Key Objects → camelCase + Keys suffix
```typescript
export const callKeys = {
  all: ['calls'] as const,
  list: () => ['calls', 'list'] as const,
  detail: (id: string) => ['calls', id] as const,
};

export const participantKeys = {
  all: ['participants'] as const,
  byCall: (callId: string) => ['participants', callId] as const,
};
```

### Spring Boot DTO Mirror Types → PascalCase with DTO suffix (in shared/types)
```typescript
export interface UserDTO { id: string; username: string; email: string; roles: string[]; }
export interface RoomDTO { id: string; name: string; hostId: string; participantCount: number; }
export interface JoinCallResponseDTO { token: string; roomId: string; iceServers: RTCIceServer[]; }
```

### Event handler props → on + PascalCase
```typescript
interface Props {
  onJoin: (roomId: string) => void;
  onLeave: () => void;
  onMuteToggle: (muted: boolean) => void;
}
```

## Naming Anti-Patterns — NEVER DO

```typescript
// ❌ Vague names
const data = useQuery(...)           // what data?
const handleClick = () => {}         // handle what?
const Component = () => {}           // component of what?
const utils = { ... }               // utils for what?

// ✅ Descriptive names
const callDetails = useCall(callId)
const handleMuteToggle = () => {}
const ParticipantAvatar = () => {}
const audioLevelUtils = { analyzeLevel, formatDb }

// ❌ Abbreviations that obscure meaning
const usr = useCurrentUser()
const ptcp = useParticipants()
const cfg = useConfig()

// ✅ Full words
const currentUser = useCurrentUser()
const participants = useParticipants()
const config = useConfig()

// ❌ Boolean naming without is/has/can/should
const muted = atom(false)
const loaded = useQuery(...)
const connected = useStore(s => s.connected)

// ✅ Explicit boolean semantics
const isMutedAtom = atom(false)
const isLoading = query.isLoading
const isCallConnected = useStore(s => s.isCallConnected)
```

## File Organization Within a Component File

Always in this order:
1. Imports (external libs → internal FSD layers top to bottom → relative)
2. Types / interfaces
3. Constants (file-local)
4. Component function
5. Sub-components (if small and tightly coupled)
6. Default export (if needed)

```typescript
// 1. External imports
import { useState, useEffect } from 'react';
import { useAtom } from 'jotai';

// 2. Internal imports — high layers first
import { useStore } from '@/app/store';
import { useJoinCall } from '@/features/join-call';
import { useParticipants } from '@/entities/participant';
import { Button } from '@/shared/ui/Button';

// 3. Types
interface CallControlsProps {
  roomId: string;
  onLeave: () => void;
}

// 4. Component
export function CallControls({ roomId, onLeave }: CallControlsProps) {
  // hooks first
  const participants = useParticipants(roomId);
  const [isMuted, setIsMuted] = useAtom(isMutedAtom);
  
  // derived state
  const participantCount = participants.length;
  
  // handlers
  const handleMuteToggle = () => setIsMuted(prev => !prev);
  
  // render
  return ( ... );
}
```
