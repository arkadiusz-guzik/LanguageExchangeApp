# File Structure

This is the canonical directory layout. When creating new files, place them exactly as specified here. When in doubt, check the FSD layer rules in `fsd-architecture.md`.

## Monorepo Root

```
voice-app/
├── apps/
│   ├── web/                          ← React + Vite
│   └── mobile/                       ← React Native + Expo
├── packages/
│   ├── shared-types/                 ← TypeScript types mirroring Spring DTOs
│   ├── shared-api/                   ← HTTP client + STOMP client (used by both apps)
│   ├── shared-webrtc/                ← WebRTC engine (framework-agnostic)
│   └── shared-ui/                    ← Cross-platform design system
├── .claude/
│   ├── CLAUDE.md                     ← Project overview (root)
│   ├── rules/                        ← Rule files (all loaded automatically)
│   └── commands/                     ← Custom slash commands
├── turbo.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
└── package.json
```

## Web App: apps/web/src/

```
src/
├── app/
│   ├── providers/
│   │   ├── QueryProvider.tsx         ← TanStack Query client setup
│   │   ├── ThemeProvider.tsx
│   │   └── index.ts
│   ├── store/
│   │   └── index.ts                  ← Zustand store: combines all slices
│   ├── router.tsx                    ← TanStack Router configuration
│   ├── styles/
│   │   └── global.css
│   └── index.tsx                     ← App entry point
│
├── pages/
│   ├── lobby/
│   │   ├── ui/
│   │   │   └── LobbyPage.tsx
│   │   └── index.ts
│   ├── call-room/
│   │   ├── ui/
│   │   │   └── CallRoomPage.tsx
│   │   └── index.ts
│   ├── settings/
│   │   ├── ui/
│   │   │   └── SettingsPage.tsx
│   │   └── index.ts
│   └── auth/
│       ├── ui/
│       │   ├── LoginPage.tsx
│       │   └── RegisterPage.tsx
│       └── index.ts
│
├── widgets/
│   ├── call-controls/
│   │   ├── ui/
│   │   │   ├── CallControls.tsx      ← orchestrates feature components
│   │   │   └── CallControlsBar.tsx
│   │   └── index.ts
│   ├── participant-grid/
│   │   ├── ui/
│   │   │   ├── ParticipantGrid.tsx
│   │   │   └── ParticipantTile.tsx
│   │   └── index.ts
│   ├── chat-sidebar/
│   │   ├── ui/
│   │   │   ├── ChatSidebar.tsx
│   │   │   └── ChatMessageList.tsx
│   │   └── index.ts
│   └── device-selector/
│       ├── ui/
│       │   └── DeviceSelector.tsx
│       └── index.ts
│
├── features/
│   ├── login/
│   │   ├── ui/
│   │   │   └── LoginForm.tsx
│   │   ├── api/
│   │   │   └── useLogin.ts           ← POST /api/auth/signin
│   │   └── index.ts
│   ├── join-call/
│   │   ├── ui/
│   │   │   └── JoinCallButton.tsx
│   │   ├── model/
│   │   │   ├── types.ts
│   │   │   └── joinCallSlice.ts
│   │   ├── api/
│   │   │   └── useJoinCall.ts        ← REST + then STOMP join
│   │   └── index.ts
│   ├── leave-call/
│   │   ├── ui/
│   │   │   └── LeaveCallButton.tsx
│   │   ├── api/
│   │   │   └── useLeaveCall.ts
│   │   └── index.ts
│   ├── mute-toggle/
│   │   ├── ui/
│   │   │   └── MuteButton.tsx
│   │   ├── model/
│   │   │   └── muteAtom.ts           ← Jotai atom
│   │   └── index.ts
│   ├── share-screen/
│   │   ├── ui/
│   │   │   └── ShareScreenButton.tsx
│   │   ├── model/
│   │   │   └── screenShareSlice.ts
│   │   ├── lib/
│   │   │   └── getDisplayMedia.ts    ← browser API wrapper
│   │   └── index.ts
│   ├── send-chat-message/
│   │   ├── ui/
│   │   │   └── ChatMessageInput.tsx
│   │   ├── api/
│   │   │   └── useSendMessage.ts     ← STOMP publish to /app/chat.send
│   │   └── index.ts
│   ├── change-audio-device/
│   │   ├── ui/
│   │   │   └── AudioDeviceSelect.tsx
│   │   ├── model/
│   │   │   └── audioDeviceSlice.ts
│   │   ├── api/
│   │   │   └── useAudioDevices.ts
│   │   └── index.ts
│   └── webrtc-signaling/
│       ├── api/
│       │   └── useSignaling.ts       ← STOMP subscriptions for offer/answer/ICE
│       ├── lib/
│       │   ├── offerFlow.ts          ← createOffer → publishMessage
│       │   └── iceHandler.ts
│       └── index.ts
│
├── entities/
│   ├── call/
│   │   ├── model/
│   │   │   ├── types.ts              ← Call, CallStatus, CallState interfaces
│   │   │   └── callSlice.ts          ← Zustand slice for call state
│   │   ├── ui/
│   │   │   └── CallStatusBadge.tsx
│   │   ├── api/
│   │   │   └── callApi.ts            ← useCall(), useRooms(), callKeys
│   │   └── index.ts
│   ├── participant/
│   │   ├── model/
│   │   │   ├── types.ts              ← Participant, ParticipantRole
│   │   │   └── participantSlice.ts
│   │   ├── ui/
│   │   │   ├── ParticipantAvatar.tsx
│   │   │   └── ParticipantAudioLevel.tsx
│   │   ├── api/
│   │   │   └── participantApi.ts
│   │   └── index.ts
│   ├── user/
│   │   ├── model/
│   │   │   ├── types.ts
│   │   │   └── userSlice.ts
│   │   ├── ui/
│   │   │   └── UserAvatar.tsx
│   │   ├── api/
│   │   │   └── userApi.ts            ← useCurrentUser(), useUser()
│   │   └── index.ts
│   ├── audio-stream/
│   │   ├── model/
│   │   │   ├── types.ts
│   │   │   └── audioStreamAtoms.ts   ← Jotai atomFamily per participantId
│   │   ├── lib/
│   │   │   └── analyzeAudioLevel.ts
│   │   └── index.ts
│   └── room/
│       ├── model/
│       │   ├── types.ts
│       │   └── roomSlice.ts
│       ├── ui/
│       │   └── RoomCard.tsx
│       ├── api/
│       │   └── roomApi.ts
│       └── index.ts
│
└── shared/
    ├── api/
    │   ├── httpClient.ts             ← axios instance with JWT interceptor
    │   ├── stompClient.ts            ← @stomp/stompjs + SockJS for Spring Boot
    │   └── types.ts                  ← SpringPage<T>, SpringApiError
    ├── lib/
    │   ├── webrtc/
    │   │   ├── PeerConnection.ts     ← RTCPeerConnection wrapper (no React)
    │   │   ├── ICECandidates.ts
    │   │   └── MediaStreamUtils.ts
    │   └── auth/
    │       └── tokenStorage.ts       ← in-memory JWT (NOT localStorage)
    ├── ui/
    │   ├── Button/
    │   │   ├── Button.tsx
    │   │   └── index.ts
    │   ├── Modal/
    │   │   ├── Modal.tsx
    │   │   └── index.ts
    │   ├── Avatar/
    │   │   ├── Avatar.tsx
    │   │   └── index.ts
    │   ├── Icon/
    │   │   ├── Icon.tsx
    │   │   └── index.ts
    │   └── index.ts                  ← re-exports all shared UI
    ├── config/
    │   ├── env.ts                    ← typed VITE_APP_* env variables
    │   └── constants.ts              ← ICE_SERVERS, MAX_PARTICIPANTS, etc.
    └── types/
        ├── springTypes.ts            ← Spring Boot DTO mirrors
        └── global.d.ts
```

## Mobile App: apps/mobile/src/

Identical FSD structure. Platform differences:
- `shared/ui/` contains React Native components instead of HTML
- `app/router.tsx` uses Expo Router file-based routing
- `shared/lib/webrtc/` uses `react-native-webrtc` instead of browser APIs
- Styling uses `StyleSheet` + `NativeWind` instead of Tailwind classes

## Rules for Creating New Files

1. **Identify the layer first.** Which layer? Re-read `fsd-architecture.md` if unsure.
2. **Identify the slice.** What domain/feature does this belong to?
3. **Identify the segment.** Is it `ui/`, `model/`, `api/`, `lib/`, or `config/`?
4. **Create the file** at `src/{layer}/{slice}/{segment}/{FileName}.ts(x)`
5. **Export from `index.ts`** if this is something external layers need.
6. **Add path alias** in `tsconfig.json` if a new layer alias is missing (should never happen).

## Where NOT to Put Files

- **Not in project root** — everything goes in `src/`
- **Not in `shared/` if it has business logic** — move to `entities/`
- **Not flat in `features/`** — always inside a slice: `features/my-feature/`
- **Not directly imported across layers** — always via `index.ts` public API
- **Not in `app/` if it's reusable** — `app/` is for bootstrap only
