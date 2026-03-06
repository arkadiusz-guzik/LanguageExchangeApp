---
paths: 
    - ""
---

# Project Rules

See `.claude/rules/frontend/` for all project rules.


# Voice App — Project Intelligence

## What This Project Is
Cross-platform voice communication app. React (Vite) for web, React Native (Expo) for mobile, Java Spring Boot backend. Monorepo via pnpm + Turborepo.

## Architecture
Feature-Sliced Design (FSD). Read `.claude/rules/` before writing any code.

## Stack at a Glance
- **Web**: React 19 + Vite + TanStack Router + Tailwind CSS v4
- **Mobile**: React Native + Expo SDK 52 + Expo Router + NativeWind
- **State**: Zustand (global) + Jotai (atomic/per-participant) + TanStack Query (server)
- **Backend comms**: axios + @stomp/stompjs + SockJS (Spring Boot STOMP)
- **Voice**: WebRTC (browser native)
- **Language**: TypeScript strict mode everywhere, zero `any`

## Commands
```bash
pnpm dev:web        # start web app
pnpm dev:mobile     # start Expo mobile
pnpm build          # build all apps
pnpm typecheck      # run tsc across all packages
pnpm lint           # run ESLint (includes @feature-sliced/eslint-config)
pnpm test           # run Vitest
pnpm test:watch     # watch mode
```

## IMPORTANT Rules
- ALWAYS read `.claude/rules/` files before touching any code
- NEVER skip layer dependency rules — the ESLint plugin will catch violations in CI
- NEVER use `any` in TypeScript
- NEVER store JWT in localStorage — use in-memory tokenStorage only
- ALWAYS run `pnpm typecheck && pnpm lint` after changes

## Rule Files Index
- `.claude/rules/fsd-architecture.md` — FSD layers, dependency rules, what goes where
- `.claude/rules/naming-conventions.md` — file, folder, component, hook, store naming
- `.claude/rules/state-management.md` — Zustand, Jotai, TanStack Query usage rules
- `.claude/rules/backend-integration.md` — Spring Boot REST, STOMP/WebSocket, JWT auth
- `.claude/rules/component-communication.md` — how components talk to each other
- `.claude/rules/file-structure.md` — exact directory layout and where new files go
- `.claude/rules/typescript.md` — TypeScript strictness rules and patterns
- `.claude/rules/webrtc.md` — WebRTC peer connection patterns and signaling
- `.claude/rules/testing.md` — testing conventions and what to test
- `.claude/rules/anti-patterns.md` — explicit list of things NEVER to do
