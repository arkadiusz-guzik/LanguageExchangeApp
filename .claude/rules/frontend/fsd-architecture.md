# FSD Architecture Rules

Feature-Sliced Design is the ONLY architecture used in this project. Every file must fit into the FSD structure. If you cannot decide where a file goes, re-read this document.

## The Layer Stack (top to bottom)

```
app/        → Bootstrap, providers, router, global store wiring
pages/      → Full screens / route components
widgets/    → Self-contained UI blocks composed from features + entities
features/   → User actions / use cases (things users DO)
entities/   → Business domain objects (things that EXIST in the domain)
shared/     → Framework-agnostic primitives, zero business logic
```

## THE GOLDEN RULE — Dependency Direction

**Dependencies flow DOWNWARD ONLY. Never upward. Never sideways.**

```
app → pages → widgets → features → entities → shared
```

### What Is ALLOWED
- `widgets/` can import from `features/`, `entities/`, `shared/`
- `features/` can import from `entities/`, `shared/`
- `entities/` can import from `shared/`
- `shared/` imports from NOTHING in this project (no business logic)
- `pages/` can import from `widgets/`, `features/`, `entities/`, `shared/`
- `app/` can import from everything

### What Is FORBIDDEN — NEVER DO THIS
```
❌ features/join-call importing from features/mute-toggle   (same layer)
❌ entities/call importing from entities/participant         (same layer)
❌ entities/user importing from features/login               (upward)
❌ shared/ importing from entities/                          (upward)
❌ widgets/ importing directly from app/                     (upward)
```

If you need to share state between two slices on the same layer, move that state DOWN to entities/ or shared/. If you need to coordinate two features, do it in a widget or page ABOVE them.

## The Three FSD Concepts

### 1. Layers (top-level directories)
Always exactly these six: `app/`, `pages/`, `widgets/`, `features/`, `entities/`, `shared/`

### 2. Slices (subdirectories within a layer)
Slices represent domain areas within a layer.
- In `features/`: `join-call/`, `mute-toggle/`, `share-screen/`, `send-message/`, `webrtc-signaling/`
- In `entities/`: `call/`, `participant/`, `user/`, `audio-stream/`, `room/`
- In `pages/`: `lobby/`, `call-room/`, `settings/`, `auth/`
- In `widgets/`: `call-controls/`, `participant-grid/`, `chat-sidebar/`, `device-selector/`
- `app/` and `shared/` do NOT have slices — they use segments directly

### 3. Segments (leaf directories inside each slice)
Divide each slice by technical purpose:
- `ui/` — React components (both presentational and connected)
- `model/` — State: Zustand slices, Jotai atoms, TypeScript types, reducers
- `api/` — API calls: TanStack Query hooks, STOMP subscriptions, REST calls
- `lib/` — Pure utility functions, helpers (no React, no side effects)
- `config/` — Constants, feature flags, static config for this slice

**Not all segments are required in every slice.** A simple feature might only need `ui/` and `api/`.

## The Public API Rule — CRITICAL

Every slice MUST have an `index.ts` that explicitly exports ONLY what external layers need.

```typescript
// features/join-call/index.ts  ✅ CORRECT
export { JoinCallButton } from './ui/JoinCallButton';
export { useJoinCall } from './api/useJoinCall';
export type { JoinCallParams } from './model/types';

// Importing internally  ✅ CORRECT
// import { someHelper } from './lib/helpers'; (internal, fine)

// Cross-slice import  ❌ WRONG
// import { something } from '@/features/join-call/lib/helpers';
// Must import only from the public API:
// import { JoinCallButton } from '@/features/join-call';
```

**The internal structure of a slice is private. Only what is in `index.ts` is public.**

## Deciding Which Layer a File Belongs To

Ask these questions in order:

1. **Is it a full screen shown at a route?** → `pages/`
2. **Is it a self-contained UI section composed of multiple features?** → `widgets/`
3. **Does it represent something a USER DOES (an action, interaction, use case)?** → `features/`
4. **Does it represent something that EXISTS in the domain (a noun: call, user, participant)?** → `entities/`
5. **Is it a generic primitive with zero business knowledge (button, API client, date util)?** → `shared/`
6. **Is it bootstrap / global setup code?** → `app/`

## Layer Responsibilities — Voice App Examples

### `app/` — Bootstrap Only
- React app entry point (`index.tsx`)
- Global providers (QueryClientProvider, ThemeProvider, AuthProvider)
- Root router configuration
- Zustand store composition (`store/index.ts` — wires all slices together)
- Global CSS / theme tokens

### `pages/` — Screens
- `lobby/` — room browser, create/join room UI
- `call-room/` — active call screen (composes widgets)
- `settings/` — user settings screen
- `auth/` — login/register screens

Pages are THIN. They compose widgets and may read route params. They do NOT contain business logic directly.

### `widgets/` — Self-Contained UI Blocks
- `call-controls/` — mute, video, hang up, share screen bar
- `participant-grid/` — grid of all participant video/audio tiles
- `chat-sidebar/` — in-call chat panel
- `device-selector/` — mic/camera selector dropdown

Widgets ORCHESTRATE. They import from multiple features and compose them. A widget is the right place to coordinate feature interactions.

### `features/` — User Actions
- `join-call/` — logic + UI to join a call room
- `leave-call/` — logic + UI to leave/hang up
- `mute-toggle/` — toggle local microphone mute
- `share-screen/` — start/stop screen sharing
- `send-chat-message/` — compose + send a message in call chat
- `change-audio-device/` — switch microphone or speaker
- `webrtc-signaling/` — STOMP subscription management for WebRTC offer/answer/ICE
- `login/` — authentication form + mutation
- `create-room/` — create a new call room

Each feature = one user capability. Keep features small and focused.

### `entities/` — Domain Objects
- `call/` — Call type, call status state, call REST API hooks
- `participant/` — Participant type, participant list state, participant UI components
- `user/` — User type, current user state, user REST API
- `audio-stream/` — MediaStream handling, audio level atoms, stream type
- `room/` — Room type, room list queries

### `shared/` — Zero Business Logic
- `api/httpClient.ts` — axios instance + JWT interceptor
- `api/stompClient.ts` — @stomp/stompjs + SockJS singleton
- `api/types.ts` — SpringPage<T>, SpringApiError
- `lib/webrtc/` — RTCPeerConnection wrapper (framework-agnostic)
- `lib/auth/tokenStorage.ts` — in-memory JWT storage
- `ui/` — design system: Button, Modal, Avatar, Icon, etc.
- `config/env.ts` — typed environment variables
- `config/constants.ts` — ICE_SERVERS, MAX_PARTICIPANTS, etc.
- `types/global.d.ts` — global TypeScript declarations

## Path Aliases — Always Use These

```typescript
import { something } from '@/shared/ui/Button';
import { useCall } from '@/entities/call';
import { JoinCallButton } from '@/features/join-call';
import { CallControls } from '@/widgets/call-controls';
import { CallRoomPage } from '@/pages/call-room';
```

Never use relative paths that cross layer boundaries: `../../entities/call` is WRONG if you're in `features/`. Use `@/entities/call` instead.

## ESLint Enforcement

The `@feature-sliced/eslint-config` plugin automatically catches layer violations. If lint fails due to an import, the import is ARCHITECTURALLY WRONG — do not suppress the lint error, fix the structure.

```bash
pnpm lint  # will catch FSD violations
```
