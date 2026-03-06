# Testing Rules

## Stack
- **Unit/integration:** Vitest + React Testing Library
- **Test files:** Co-located with source: `Button.test.tsx` next to `Button.tsx`
- **Mocking:** `vi.mock()` for modules, `vi.fn()` for functions

## What to Test by FSD Layer

| Layer | What to test | How |
|---|---|---|
| `shared/lib/` | Pure functions: audio analysis, token storage, formatting | Unit test — no React |
| `shared/ui/` | Rendered output, accessibility, user interactions | RTL render |
| `entities/*/model/` | Zustand slice actions and state transitions | Unit test the slice creator |
| `entities/*/api/` | Query key shapes, data transformation | Mock httpClient |
| `features/*/api/` | Mutation flows, onSuccess callbacks | Mock httpClient + Zustand |
| `features/*/ui/` | User interactions, form submission | RTL render + userEvent |
| `widgets/` | Composition of features (integration) | RTL render with mocked child features |

Do NOT write tests for `pages/` or `app/`. These are integration surfaces tested via E2E (Playwright) if needed.

---

## Test File Location

```
features/join-call/
├── ui/
│   ├── JoinCallButton.tsx
│   └── JoinCallButton.test.tsx    ← co-located
├── api/
│   ├── useJoinCall.ts
│   └── useJoinCall.test.ts        ← co-located
└── index.ts
```

---

## Test Naming Convention

```typescript
// Pattern: "should [action] when [condition]"
describe('JoinCallButton', () => {
  it('should render as disabled when user is not authenticated', () => { ... });
  it('should call joinCall mutation when clicked', () => { ... });
  it('should show loading spinner while mutation is pending', () => { ... });
  it('should navigate to call room on successful join', () => { ... });
});
```

---

## Mocking Patterns

### Mock httpClient (not axios itself)
```typescript
vi.mock('@/shared/api/httpClient', () => ({
  httpClient: {
    get: vi.fn(),
    post: vi.fn(),
    put: vi.fn(),
    delete: vi.fn(),
  },
}));
```

### Mock stompClient
```typescript
vi.mock('@/shared/api/stompClient', () => ({
  subscribeToTopic: vi.fn(() => ({ unsubscribe: vi.fn() })),
  publishMessage: vi.fn(),
  getStompClient: vi.fn(),
}));
```

### Mock Zustand store for components
```typescript
vi.mock('@/app/store', () => ({
  useStore: vi.fn(),
  useCallStatus: vi.fn(() => 'idle'),
  useParticipants: vi.fn(() => []),
}));
```

### Testing Zustand slices in isolation
```typescript
import { createCallSlice } from '@/entities/call/model/callSlice';

describe('callSlice', () => {
  it('should set call status', () => {
    const store = create(createCallSlice);
    store.getState().setCallStatus('connecting');
    expect(store.getState().callStatus).toBe('connecting');
  });

  it('should reset call state', () => {
    const store = create(createCallSlice);
    store.getState().setCallStatus('connected');
    store.getState().resetCall();
    expect(store.getState().callStatus).toBe('idle');
  });
});
```

---

## TanStack Query Testing

Wrap components with a fresh QueryClient in each test:

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function createWrapper() {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
}

test('useRooms returns rooms list', async () => {
  vi.mocked(httpClient.get).mockResolvedValue({
    data: { content: [mockRoom], totalElements: 1 } as SpringPage<Room>,
  });

  const { result } = renderHook(() => useRooms(), { wrapper: createWrapper() });
  await waitFor(() => expect(result.current.isSuccess).toBe(true));
  expect(result.current.data?.content).toHaveLength(1);
});
```

---

## Rules
- One `describe` block per component/hook file
- One concept per `it` block — do not test multiple things in one test
- Never test implementation details — test behavior from the user's perspective
- Mock at the boundary (httpClient, stompClient) — never mock internal helpers
- `beforeEach` for setup, `afterEach` for cleanup
- Never use `setTimeout` in tests — use `vi.useFakeTimers()` or `waitFor()`
- Test error states and loading states, not just happy path
