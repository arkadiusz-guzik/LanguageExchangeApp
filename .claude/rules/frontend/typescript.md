# TypeScript Rules

TypeScript strict mode is non-negotiable. Every file in this project must be type-safe.

## Strictness Settings

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

**All of these are active. Do not suppress them.**

---

## Absolute Rules

```typescript
// ❌ NEVER
const data: any = fetchSomething();
const handler = (e: any) => {};
// @ts-ignore
// @ts-expect-error (only allowed in test files with a comment explaining why)

// ✅ ALWAYS — be explicit
const data: Room = fetchSomething();
const handler = (e: React.MouseEvent<HTMLButtonElement>) => {};
```

---

## Typing Patterns

### Component Props — Always an explicit interface

```typescript
// ✅ CORRECT
interface ParticipantAvatarProps {
  participant: Participant;
  size?: 'sm' | 'md' | 'lg';
  onAvatarClick?: (participantId: string) => void;
}

export function ParticipantAvatar({ participant, size = 'md', onAvatarClick }: ParticipantAvatarProps) {
  // ...
}

// ❌ WRONG — inline type object
export function ParticipantAvatar({ participant }: { participant: Participant }) {
  // Only acceptable for very simple single-prop components
}
```

### Discriminated Unions for State

```typescript
// ✅ Model complex state with discriminated unions
export type CallState =
  | { status: 'idle' }
  | { status: 'connecting'; roomId: string }
  | { status: 'connected'; roomId: string; startTime: number; peers: Map<string, RTCPeerConnection> }
  | { status: 'failed'; roomId: string; error: string }
  | { status: 'ended'; roomId: string; duration: number };

// Narrowing works correctly
function handleCallState(state: CallState) {
  if (state.status === 'connected') {
    console.log(state.startTime);  // TypeScript knows this exists
  }
}
```

### Generic Query Hook Return

```typescript
// ✅ Type the generic parameter explicitly
export function useRooms(page = 0) {
  return useQuery<SpringPage<Room>, SpringApiError>({
    queryKey: roomKeys.list(),
    queryFn: () => httpClient.get<SpringPage<Room>>('/rooms').then(r => r.data),
  });
}
```

### Zustand Slice Types

```typescript
// ✅ Always export the slice type interface
export interface CallSlice {
  callStatus: CallStatus;
  callRoomId: string | null;
  setCallStatus: (status: CallStatus) => void;
  setCallRoom: (roomId: string) => void;
  resetCall: () => void;
}

// ✅ StateCreator is typed with the full store type
export const createCallSlice: StateCreator<RootStore, [], [], CallSlice> = (set) => ({ ... });
```

### Jotai Atom Types

```typescript
// ✅ Type atom explicitly if inference is ambiguous
export const callStatusAtom = atom<CallStatus>('idle');
export const audioLevelAtomFamily = atomFamily((_id: string) => atom<number>(0));
```

### Event Handlers

```typescript
// ✅ Use React's built-in event types
const handleClick = (e: React.MouseEvent<HTMLButtonElement>) => { ... };
const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => { ... };
const handleSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); ... };
const handleKeyDown = (e: React.KeyboardEvent<HTMLDivElement>) => { ... };
```

### Optional Chaining and Nullish Coalescing

```typescript
// ✅ Use these — never unchecked access
const username = user?.profile?.username ?? 'Anonymous';
const level = audioLevelAtomFamily.get(participantId)?.() ?? 0;
const content = rooms?.content ?? [];
```

---

## Type vs Interface

Use `interface` for object shapes that may be extended or implemented.  
Use `type` for unions, aliases, and computed types.

```typescript
// interface for extendable shapes
interface BaseEntity { id: string; createdAt: string; }
interface User extends BaseEntity { username: string; email: string; }
interface Participant extends BaseEntity { userId: string; callId: string; }

// type for unions, computed, aliases
type CallStatus = 'idle' | 'connecting' | 'connected' | 'failed' | 'ended';
type ParticipantRole = 'host' | 'guest' | 'observer';
type PartialParticipant = Partial<Participant>;
type ParticipantMap = Map<string, Participant>;
```

---

## Importing Types

Always use `import type` for type-only imports. This improves build performance and prevents circular dependency issues.

```typescript
// ✅ CORRECT
import type { Participant } from '@/entities/participant';
import type { SpringPage, SpringApiError } from '@/shared/api/types';
import type { StateCreator } from 'zustand';

// Only omit 'type' when you need the value, not just the type
import { useStore } from '@/app/store';
import { httpClient } from '@/shared/api/httpClient';
```

---

## Non-Null Assertion

Never use `!` non-null assertion except for ref access where React guarantees a value.

```typescript
// ✅ Only for React refs after mount
const canvasRef = useRef<HTMLCanvasElement>(null);
// In useEffect (guaranteed non-null after mount):
const ctx = canvasRef.current!.getContext('2d');

// ❌ NEVER for business logic
const user = getUser()!;     // might actually be null — handle it
const room = rooms[0]!;      // might be undefined — check first
```

---

## Path Aliases — Required in tsconfig.json

```json
{
  "compilerOptions": {
    "paths": {
      "@/app/*":      ["./src/app/*"],
      "@/pages/*":    ["./src/pages/*"],
      "@/widgets/*":  ["./src/widgets/*"],
      "@/features/*": ["./src/features/*"],
      "@/entities/*": ["./src/entities/*"],
      "@/shared/*":   ["./src/shared/*"]
    }
  }
}
```

And in `vite.config.ts`:
```typescript
import { resolve } from 'path';

export default defineConfig({
  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
    },
  },
});
```

---

## After Any Change: Run These

```bash
pnpm typecheck   # tsc --noEmit across all packages
pnpm lint        # ESLint including FSD layer rules
```

Both must pass before a feature is considered done. A TypeScript error in CI is a build failure.
