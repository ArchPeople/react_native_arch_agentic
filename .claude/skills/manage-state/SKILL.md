---
name: manage-state
description: Chooses and implements state in this repo - TanStack react-query for server data, a per-feature zustand store (src/features/<f>/store) for feature UI state shared across sections, an app-wide zustand store in src/app/global-state for cross-feature state like theme or session, MMKV-backed persistence, and plain useState for local component state. Use when adding state, sharing data between screens or sections, handling loading/error status, persisting state, or deciding where state belongs. Triggers: state, store, zustand, global state, react-query cache, useState, selector, dark mode state, session.
---

# Manage state

## Decision table
| The data is… | Put it in | Example |
| --- | --- | --- |
| From the server (fetch/mutate, cache, loading, error) | react-query hook in `src/features/<f>/api/` | `useGetDemoData` in `src/features/demo-feature/api/demoFeatureApi.ts` |
| UI state used by one component | `useState` / `useReducer` | toggles, input text |
| UI state shared by sections of one feature | feature store `src/features/<f>/store/<camel>Store.ts` | `useDemoFeatureStore` |
| Used across features or app-wide | `src/app/global-state/<kebab>/<camel>.ts` + barrel | `useSystemModeStore` (`src/app/global-state/system-mode/systemMode.ts`) |
| Must survive restarts | the global/feature store **plus** a `core/data/local` module | see `add-local-storage` |

Never copy server data into zustand. Read it from the query; react-query is the cache.

## Feature store (plop generates this shape)
```ts
import { create } from 'zustand';

type orderDetailState = {
  selectedItemId: string | null;
  setSelectedItemId: (id: string | null) => void;
  reset: () => void;
};

export const useOrderDetailStore = create<orderDetailState>(set => ({
  selectedItemId: null,
  setSelectedItemId: selectedItemId => set({ selectedItemId }),
  reset: () => set({ selectedItemId: null }),
}));
```
- Replace the generated `data` / `setdata` placeholder with real fields. Use camelCase `setX` names for new actions, and don't rename existing template ones (`setsystemMode`).
- Feature stores are module singletons, so they outlive the screen. Call `reset()` on unmount or when leaving the flow if stale state matters.

## Global store
1. Create `src/app/global-state/<kebab>/<camel>.ts` following `systemMode.ts`.
2. Export it from `src/app/global-state/index.ts`:
   ```ts
   export { useSessionStore } from './session/session';
   ```
3. Import it with `import { useSessionStore } from '@app/global-state';`.

## Reading state
Use one selector per field so components only re-render when that field changes:
```ts
const systemMode = useSystemModeStore(state => state.systemMode);
const setSystemMode = useSystemModeStore(state => state.setsystemMode);
```
Outside React (in services or interceptors), use `useSessionStore.getState().token`.

## Status without react-query
For manual async flows, use `viewState` from `src/core/common/view-state/viewState.ts` (`initial | loading | success | error`). Don't invent new status strings.

## Dark mode
- App theme: `useSystemModeStore` with `themeSystemMode.light | dark` from `@app/themes`. It isn't persisted and defaults to `light`.
- To persist it, combine it with a `core/data/local` module (see `add-local-storage`).

## Verify
```bash
npx tsc --noEmit --ignoreDeprecations 6.0
```
Test stores directly (`useXStore.getState()` / `setState`) and reset them in `beforeEach` (see `write-tests`).
