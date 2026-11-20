---
name: manage-environment-config
description: Manages environment configuration and build flavors - the root .env file, the arch generate-env script that writes the gitignored src/core/config/env.ts, exposing values through src/core/config/config.ts, and the development/staging/production flavors (Android productFlavors in build.gradle, iOS schemes and Debug-/Release-<flavor> build configurations, per-flavor app names and bundle id suffixes). Use when adding an env variable, base URL or API key per environment, switching flavor, adding a new flavor, or fixing "Cannot find module './env'". Triggers: env, .env, environment, config, flavor, staging, production, scheme, build variant, base url.
---

# Manage environment config

## Pipeline
```text
.env  →  yarn arch:generate-env (.arch/generateEnv.js)  →  src/core/config/env.ts  →  src/core/config/config.ts  →  import { config } from '@core/config'
```
`.env` and `src/core/config/env.ts` are both **gitignored** and must never be committed or edited by hand.

## Add a variable
1. Add it to `.env` as `KEY=value`. Keep `FLAVOR=` present:
   ```bash
   FLAVOR=development
   API_DOMAIN=https://api.dev.example.com
   ```
2. Run `yarn arch:generate-env`.
3. Expose it in `src/core/config/config.ts` (the only place that imports `env`):
   ```ts
   import { env } from './env';

   export const config = {
     demoDomain: 'https://reactnative.dev',
     apiDomain: env.api_domain,
     flavor: env.flavor,
   };
   ```
4. Consume it with `import { config } from '@core/config';`.

### Key format (actual script behavior)
`generateEnv.js` only **lowercases** keys and keeps underscores: `OTHER_KEY_1` → `other_key_1`, `API_DOMAIN` → `api_domain`.

> `guide/how-to-run.md` says keys become camelCase (`otherKey1`), but that's **wrong**. Follow the script.

- Every value is emitted as a double-quoted **string** (`api_domain: "https://…"`), so convert numbers and booleans in `config.ts` (`env.enable_logs === 'true'`).
- Values containing `"` break the generated file.
- Comment lines (`#`) and blank lines are skipped.

### Security
Everything in `env.ts` is bundled into the JS and readable from a release build. **Never put real secrets** (private keys, signing credentials) in `.env`. Public client IDs and base URLs are fine. README advice: delete `.env` after generating the config for a release.

### Sharing env with the team
Don't commit `.env`. Document the required keys in project docs or share them through your secret manager. Each developer runs `yarn arch:generate-env`.

## Flavors
| Flavor | JS (`.env`) | Android (`android/app/build.gradle` `productFlavors`) | iOS (`ios/<RepoName>.xcodeproj`) | Run |
| --- | --- | --- | --- | --- |
| development | `FLAVOR=development` | `applicationIdSuffix ".development"`, own `app_name` | scheme `development`, `Debug-/Release-development` | `yarn android:dev` / `yarn ios:dev` |
| staging | `FLAVOR=staging` | `applicationIdSuffix ".staging"`, own `app_name` | scheme `staging` | `yarn android:stg` / `yarn ios:stg` |
| production | `FLAVOR=production` | no suffix | scheme `production` | `yarn android:prod` / `yarn ios:prod` |

- **The JS flavor isn't tied to the native flavor.** Before running another flavor, set `FLAVOR` (and the other per-env values) in `.env` and re-run `yarn arch:generate-env`.
- Branch on `config.flavor === 'production'` in code when needed.
- Folders for flavor-specific native files, currently only placeholders:
  - Android: `android/app/src/<flavor>/` (e.g. `google-services.json`, res overrides).
  - iOS: `ios/config/<flavor>/` (e.g. `GoogleService-Info.plist`, copied by a build phase you add).
- Per-flavor display names: Android `resValue "string", "app_name"` in each flavor block; iOS `PRODUCT_NAME` in each build configuration.

### Adding a new flavor (e.g. `uat`)
1. Android: add a `uat { dimension "default"; applicationIdSuffix ".uat"; resValue "string", "app_name", "<App> UAT" }` block and a `package.json` script: `"android:uat": "react-native run-android --mode=uatDebug --appIdSuffix=uat"`.
2. iOS:
   - In Xcode, duplicate the `Debug-staging` / `Release-staging` configurations as `Debug-uat` / `Release-uat` and set the bundle id and product name.
   - Create a shared `uat` scheme.
   - Add the configs to the `project` mapping in `ios/Podfile`, run `pod install`, then add `"ios:uat": "react-native run-ios --scheme \"uat\""`.
3. Create `android/app/src/uat/` and `ios/config/uat/` if needed.

## Gotchas
- `Cannot find module './env'` (tsc/Metro) means `env.ts` was never generated. Run `yarn arch:generate-env`.
- In Jest, mock `@core/config` so tests don't depend on a local `env.ts` (see `write-tests`).

## Verify
```bash
yarn arch:generate-env && cat src/core/config/env.ts
git status --short .env src/core/config/env.ts   # must print nothing (ignored)
npx tsc --noEmit --ignoreDeprecations 6.0
```
