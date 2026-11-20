---
name: write-tests
description: Writes and runs Jest tests for this React Native template using @react-native/jest-preset and react-test-renderer - rendering screens with their required providers (QueryClientProvider, SafeAreaProvider), mocking native modules (MMKV, safe-area, vector icons, linear-gradient, localize), mocking navigation, config/env and data sources, testing zustand stores, local data modules and remote data sources. Includes the workaround for the template's Jest 30 environment mismatch. Use when adding or fixing tests, testing a screen, component, hook, store or data module, or when a test fails on native modules. Triggers: test, jest, unit test, render test, mock, coverage, react-test-renderer, failing test.
---

# Write tests

The stack is Jest 30, `@react-native/jest-preset` (`jest.config.js`) and `react-test-renderer` 19. `@testing-library/react-native` is **not** installed; don't add it unless asked.

## Where
- Put tests in root `__tests__/`, mirroring `src/`: `__tests__/features/<feature>/<Name>.test.tsx`, `__tests__/core/data/<name>Data.test.ts`.
- Jest's default `testMatch` also picks up `*.test.ts(x)` anywhere.
- Path aliases (`@app`, `@core`, `@features`) work in tests through `babel.config.js`.

## Running (template baseline)
`yarn test` currently fails for **every** test because the preset's `jest-environment-node@29` doesn't match Jest 30. Until the project fixes it, run with a Jest 30 environment kept outside the repo. Replace `<repo>` with the absolute project path:
```js
// /tmp/rn-env-jest30.js
const NodeEnv = require('<repo>/node_modules/jest-environment-node').TestEnvironment;
module.exports = class extends NodeEnv {
  customExportConditions = ['require', 'react-native'];
};
```
```bash
npx jest --testEnvironment=/tmp/rn-env-jest30.js __tests__/path/Name.test.tsx
```
Once the project fixes the environment, use `yarn test __tests__/path/Name.test.tsx`. Don't change `jest.config.js` without asking.

## Screen/component test (verified against `DemoFeatureScreen`)
```tsx
import React from 'react';
import { Text } from 'react-native';
import ReactTestRenderer from 'react-test-renderer';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { DemoFeatureScreen } from '@features/demo-feature/view/screen/DemoFeatureScreen';
import { demoData } from '@core/data/remote';
import { useSystemModeStore } from '@app/global-state';
import { themeSystemMode } from '@app/themes';

// Native modules
jest.mock('react-native-safe-area-context', () =>
  require('react-native-safe-area-context/jest/mock').default, // .default is required
);
jest.mock('@react-native-vector-icons/material-icons', () => 'MaterialIcons');
jest.mock('react-native-linear-gradient', () => 'LinearGradient');
jest.mock('react-native-localize', () => ({
  findBestLanguageTag: () => ({ languageTag: 'en', isRTL: false }),
}));
jest.mock('react-native-mmkv', () => {
  const store = new Map<string, string>();
  return {
    createMMKV: () => ({
      set: (k: string, v: string) => store.set(k, v),
      getString: (k: string) => store.get(k),
      remove: (k: string) => store.delete(k),
      clearAll: () => store.clear(),
    }),
  };
});
// No dependency on the gitignored, generated env.ts
jest.mock('@core/config', () => ({
  config: { demoDomain: 'https://example.test', flavor: 'development' },
}));
// Mock navigation fully. requireActual fails: @react-navigation is ESM and not transformed.
jest.mock('@react-navigation/native', () => ({
  useNavigation: () => ({
    navigate: jest.fn(),
    goBack: jest.fn(),
    canGoBack: () => false,
  }),
}));
// Fake the data layer, never the network
jest.mock('@core/data/remote', () => ({
  demoData: { fetchDemoData: jest.fn() },
}));

let mounted: ReactTestRenderer.ReactTestRenderer | null = null;
afterEach(async () => {
  // Unmount, or store updates in later tests trigger act() warnings
  await ReactTestRenderer.act(async () => mounted?.unmount());
  mounted = null;
});

const renderWithProviders = async (ui: React.ReactElement) => {
  const client = new QueryClient({
    defaultOptions: { queries: { retry: false, gcTime: Infinity } },
  });
  let renderer!: ReactTestRenderer.ReactTestRenderer;
  await ReactTestRenderer.act(async () => {
    renderer = ReactTestRenderer.create(
      <SafeAreaProvider>
        <QueryClientProvider client={client}>{ui}</QueryClientProvider>
      </SafeAreaProvider>,
    );
  });
  mounted = renderer;
  return renderer;
};

describe('DemoFeatureScreen', () => {
  beforeEach(() => {
    useSystemModeStore.setState({ systemMode: themeSystemMode.light });
    jest.mocked(demoData.fetchDemoData).mockReset();
  });

  it('fetches on mount and refetches on press', async () => {
    jest
      .mocked(demoData.fetchDemoData)
      .mockResolvedValue({ title: 't', description: 'd', movies: [] });
    const renderer = await renderWithProviders(<DemoFeatureScreen />);

    expect(demoData.fetchDemoData).toHaveBeenCalledTimes(1);
    const texts = renderer.root.findAllByType(Text).map(t => t.props.children);
    expect(texts).toContain('Demo Feature');

    const button = renderer.root.findByProps({ label: 'Check' });
    await ReactTestRenderer.act(async () => {
      button.props.onPress();
    });
    expect(demoData.fetchDemoData).toHaveBeenCalledTimes(2);
  });
});
```
Notes:
- Mock only what the component tree imports. Copy the blocks you need.
- If the tree uses `useRoute`, add `useRoute: () => ({ params: {...} })` to the navigation mock.
- If it uses `useTranslation`, add `import '@app/localization/i18n';` (with `react-native-localize` mocked). Assertions then see the English strings from `en.json`.
- If it uses reanimated, add `jest.mock('react-native-reanimated', () => require('react-native-reanimated/mock'));`.

### Querying gotchas
- Shared components are `memo()`-wrapped, so `findAllByType(TextBase)` finds **nothing**. Query host components (`Text`, `View`) or props (`findByProps({ label: 'Check' })`).
- `accessibilityLabel` is prefixed (`'button-' + label`, `'text-' + label`, `'pressable-' + label`), so `findByProps({ accessibilityLabel: 'button-submit' })` works when a label is passed.
- Wrap renders and events in `await ReactTestRenderer.act(async () => …)`.
- Use a **new** `QueryClient` per test with `retry: false`. Never use the app singleton `query.client`.

## Store test (zustand)
```ts
beforeEach(() => useSystemModeStore.setState({ systemMode: themeSystemMode.light }));

it('switches mode', () => {
  useSystemModeStore.getState().setsystemMode(themeSystemMode.dark);
  expect(useSystemModeStore.getState().systemMode).toBe('dark');
});
```

## Local data module test
Use the Map-backed `react-native-mmkv` mock above, then:
```ts
appStatusData.setAppStatusData(true);
expect(appStatusData.getAppStatusData()).toBe(true);
appStatusData.deleteAppStatusData();
expect(appStatusData.getAppStatusData()).toBe(false);
```

## Remote data source test
```ts
import { apiFetch } from '@core/services/api-fetch';
import { demoData } from '@core/data/remote';

jest.mock('@core/services/api-fetch', () => ({ apiFetch: { getApi: jest.fn() } }));
jest.mock('@core/config', () => ({ config: { demoDomain: 'https://example.test' } }));

it('maps axios errors to IBaseFailure', async () => {
  jest.mocked(apiFetch.getApi).mockRejectedValue({
    response: { status: 500 },
    code: 'ERR_BAD_RESPONSE',
    message: 'boom',
  });
  await expect(demoData.fetchDemoData()).resolves.toEqual({
    errorStatus: 500,
    errorCode: 'ERR_BAD_RESPONSE',
    errorMessage: 'boom',
  });
});
```

## What to test
- Data-source mapping (success and failure).
- Store actions.
- Local data round-trips and defaults.
- Screens' loading/error/success branches and user interactions.

Don't test theme constants or barrel exports.

## Verify
```bash
npx jest --testEnvironment=/tmp/rn-env-jest30.js <your test files>   # or `yarn test <files>` once fixed
npx prettier --check <your test files>
```
`__tests__/App.test.tsx` fails in the template (it imports `../App`, which doesn't exist). That's a known baseline issue; don't count it against your change.
