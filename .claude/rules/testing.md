# Testing Rules (Project-Specific)

> General testing rules live in `~/.claude/rules/testing.md`. This file adds project-specific details only.

## Setup

- Jest with platform configs: `jest.config.js` (shared/RN), `jest.config.web.js`, `jest.config.android.js`
- `yarn test` | `yarn test:coverage` | `TZ=UTC` forced in all runs
- Platform file suffixes: `*.ios.spec.tsx`, `*.android.spec.tsx`, `*.web.spec.tsx`

## Mock Helpers (`src/mocks/`)

- `mockToken(value?)` — set auth token
- `mockStorage({ key: value })` — seed storage
- `mockClientId()` / `mockClientSecret()` — set API credentials
- `createMockFactory<T>(defaults)` — factory with partial overrides
- `mockBitmovinPlayerEvent(event)` / `emitNativePlayerEvent(event)` — player events

MSW handlers: `src/api/mocks/server/server.ts`

## Test Utilities (`src/testing/utils.tsx`)

- `renderWithProviders()` — all providers (ErrorBoundary, Navigation, DeviceType, ReactQuery, SpatialNavigation, Portal, Modal)
- `renderHookWithProviders()` — same, for hooks
- Never construct providers manually in test files

## Coverage (CI-tracked)

`src/api/**`, `src/data/**`, `src/app/features/player/**`
