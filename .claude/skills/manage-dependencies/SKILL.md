---
name: manage-dependencies
description: Adds, upgrades or removes npm packages in this yarn 1 React Native project while keeping exact version pins, installing CocoaPods for native modules, rebuilding native apps, keeping Babel plugin order (react-native-worklets/plugin last), and adding Jest mocks for new native modules. Use when installing a library, upgrading React Native or any package, removing a dependency, or fixing native linking/pod errors. Triggers: install, add package, yarn add, upgrade, update dependency, pod install, native module, version, remove package.
---

# Manage dependencies

The package manager is **yarn 1.22.22** (`yarn.lock`). Never run `npm install`, `pnpm` or `bun`, and never hand-edit `yarn.lock`.
All versions in `package.json` are **exact pins** (no `^`/`~`), and that must stay true.

## Add
```bash
yarn add <pkg>@<x.y.z> --exact
yarn add -D <pkg>@<x.y.z> --exact
```
- Check compatibility with React Native 0.88 / React 19 and the **New Architecture**: read the package's README and peer deps (`npm view <pkg> peerDependencies`).
- Prefer libraries that are already in the stack before adding a new one:
  - HTTP: `axios` via `apiFetch`
  - Storage: MMKV
  - State: zustand + react-query
  - i18n: i18next
  - Icons: `@react-native-vector-icons/*`
  - Gradients: `react-native-linear-gradient`
  - Animation: reanimated 4
- Pure-JS packages need no native step. Restart Metro with `yarn start --reset-cache` if resolution looks stale.

## Native modules
```bash
cd ios && pod install && cd ..      # or: bundle exec pod install
yarn android:dev                    # full native rebuild
yarn ios:dev
```
Autolinking handles registration. Follow any extra steps the library requires:
- Android: `android/app/build.gradle`, `AndroidManifest.xml`, `MainApplication.kt`.
- iOS: `Info.plist`, `AppDelegate.swift`, `PrivacyInfo.xcprivacy`.

Commit the `ios/Podfile.lock` changes.

## Babel plugins
If a library needs a Babel plugin, add it to `babel.config.js` **before** `'react-native-worklets/plugin'`, which must stay last. Restart Metro with `--reset-cache` afterwards.

## Jest
- A native module imported by tested code needs a mock in the test, or the test crashes with "native module not found". See `write-tests` for the existing mock patterns.
- ESM-only packages may need to be added to `transformIgnorePatterns`. Changing `jest.config.js` is a config change, so ask first.

## Upgrade
- Single package: `yarn add <pkg>@<new> --exact`, then pods, rebuild and run the tests.
- React Native itself: use the React Native Upgrade Helper diff for `0.88.1 → target`.
  - Bump `react-native`, `react`, every `@react-native/*` package and `@react-native-community/cli*` **together** to matching exact versions.
  - Then run `yarn clean`.
- Check `yarn why <pkg>` for duplicates. Nested mismatches cause the current Jest baseline failure: `@react-native/jest-preset` uses `jest-environment-node@29` while Jest is 30.

## Remove
```bash
yarn remove <pkg>
cd ios && pod install && cd ..
```
Remove its imports, mocks, Babel plugin and native config.

## Verify
```bash
node -e "const p=require('./package.json');const bad=Object.entries({...p.dependencies,...p.devDependencies}).filter(([,v])=>/^[\^~]/.test(v));console.log(bad.length?bad:'all pinned')"
npx tsc --noEmit --ignoreDeprecations 6.0
yarn android:dev && yarn ios:dev
```
