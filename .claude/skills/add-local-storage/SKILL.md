---
name: add-local-storage
description: Persists data on the device using the repo's MMKV wrapper - add a key to storageKey.ts, create a typed local data module in src/core/data/local with set/get/delete methods that serialize to strings, export it from the barrel, and optionally hydrate a zustand store from it. Use when saving user preferences, flags, onboarding status, cached values, session data, or anything that must survive app restarts. Triggers: persist, save locally, local storage, MMKV, cache on device, remember, AsyncStorage, preferences.
---

# Add local storage

Canonical example:
- `src/core/data/local/app-status-data/appStatusData.ts`
- `src/core/data/local/storageKey.ts`

The service is `src/core/services/local-storage/localStorage.ts`. It wraps `createMMKV()` and **stores strings only**:
`localStorage.setValue(key, string)`, `getValue(key): string | null`, `deleteValue(key)`, `clearAllValue()`.

Never call `localStorage` or MMKV from features or components. Always go through a `core/data/local` module.

## 1. Add the key
`src/core/data/local/storageKey.ts`:
```ts
export const storageKey = {
  appStatusStorageKey: 'app_status_storage_key',
  userPreferenceStorageKey: 'user_preference_storage_key',
};
```

## 2. Create the data module
`src/core/data/local/<name>-data/<name>Data.ts`:
```ts
import { localStorage } from '@core/services/local-storage';
import { storageKey } from '../storageKey';

const key = storageKey.userPreferenceStorageKey;

export interface IUserPreference {
  language: string;
  notifications: boolean;
}

export const userPreferenceData = {
  setUserPreferenceData: (value: IUserPreference) => {
    localStorage.setValue(key, JSON.stringify(value));
  },
  getUserPreferenceData: (): IUserPreference | null => {
    const value = localStorage.getValue(key);
    if (value == null) {
      return null;
    }
    try {
      return JSON.parse(value) as IUserPreference;
    } catch {
      return null;
    }
  },
  deleteUserPreferenceData: () => {
    localStorage.deleteValue(key);
  },
};
```
- Booleans and numbers: `String(value)` on write, then parse back on read, as `appStatusData` does with `value === 'true'`.
- Always return a safe default when the value is missing or corrupt.

## 3. Export it
`src/core/data/local/index.ts`:
```ts
export { userPreferenceData } from './user-preference-data/userPreferenceData';
```

## 4. Make it reactive (optional)
MMKV reads are synchronous, so a zustand store can hydrate at creation time and write through in its setter:
```ts
export const useUserPreferenceStore = create<userPreferenceState>(set => ({
  preference: userPreferenceData.getUserPreferenceData(),
  setPreference: preference => {
    userPreferenceData.setUserPreferenceData(preference);
    set({ preference });
  },
}));
```

## Security
- `createMMKV()` is created **without an encryption key**. Don't store secrets (tokens, PII) in plain MMKV.
- If you need to, create a separate encrypted instance in `src/core/services/local-storage` (`createMMKV({ id, encryptionKey })`) or add a keychain library (see `manage-dependencies`). Flag this to the user.
- On logout, delete user-scoped keys (or use `clearAllValue()` if appropriate).

## Verify
```bash
npx prettier --check src/core/data/local
npx tsc --noEmit --ignoreDeprecations 6.0
```
Add a round-trip test with the Map-backed `react-native-mmkv` mock from `write-tests`.
