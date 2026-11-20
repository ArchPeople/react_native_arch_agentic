---
name: project-setup
description: Sets up a freshly cloned or downloaded copy of this React Native template. Covers renaming the app, package/bundle id and repo with the arch init script, creating .env and env.ts, installing yarn deps and CocoaPods, running per flavor (development, staging, production) on Android/iOS, cleaning caches and changing app icons. Use when starting a new project from the template, onboarding a machine, after a fresh clone, or when the build is broken and needs a clean reinstall. Triggers: init, setup, rename app, bundle id, package name, first run, pod install, clean build, app icon.
---

# Project setup

## Prerequisites
- Node `>= 24.3.0` (`engines` in `package.json`), **yarn 1.22.22**, Java 17+, Android Studio, Xcode 16.4+, CocoaPods (Ruby `Gemfile` at the root).
- macOS is required for iOS.

## 1. Initialize the project (run once, on the untouched template)
```bash
yarn install
yarn arch:init
```
`.arch/init.js` asks for:
1. **App name**, the display name. It replaces the production `app_name` from `android/app/build.gradle` in gradle, `strings.xml`, the iOS project, schemes and `LaunchScreen.storyboard`.
2. **Package name**, e.g. `com.company.app`. It replaces the Android `namespace`/`applicationId` and the iOS `PRODUCT_BUNDLE_IDENTIFIER`, and moves `android/app/src/main/java/<old/path>` to the new path.
3. **Repository name**. It replaces the `package.json`/`app.json` name, `settings.gradle`, `Podfile`, and renames `ios/<old>`, `ios/<old>.xcodeproj` and `ios/<old>.xcworkspace`.

It also writes `.env` with `FLAVOR=development`.

> Gotcha: the script locates files using the *original* template name (`package.json` `name` and `android/app/src/main/java/com/<name>/`). It is meant to run **once**. To change only the package/bundle id later, use `yarn arch:change-package-name` (`.arch/packageName.js`).

Then run:
```bash
yarn clean               # wipes metro cache, builds, Pods; yarn install; gradlew clean; pod install
yarn arch:generate-env   # creates src/core/config/env.ts from .env
```

## 2. Environment
`.env` must exist and contain `FLAVOR=<flavor>`. Details are in the `manage-environment-config` skill. Never commit `.env` or `src/core/config/env.ts`.

## 3. Run
| Platform | development | staging | production |
| --- | --- | --- | --- |
| Android | `yarn android:dev` | `yarn android:stg` | `yarn android:prod` |
| iOS | `yarn ios:dev` | `yarn ios:stg` | `yarn ios:prod` |

- Android runs `--mode=<flavor>Debug`. The dev and staging builds get `.development` / `.staging` appId suffixes, so all three can be installed side by side.
- iOS uses the schemes in `ios/<RepoName>.xcodeproj/xcshareddata/xcschemes/`, which map to the `Debug-<flavor>` / `Release-<flavor>` build configs declared in `ios/Podfile`.
- The JS `FLAVOR` value comes from `.env`, **not** from the native flavor. Set it to match and re-run `yarn arch:generate-env` before switching.

## 4. App icon
- Follow `guide/change-app-icon.md`.
  - Android: `android/app/src/main/res/mipmap-*/ic_launcher*.png`.
  - iOS: `ios/<RepoName>/Images.xcassets/AppIcon.appiconset/`.
- `yarn icon:update` points at the placeholder `ios/AppName/Info.plist`. Replace `AppName` with your iOS folder name before using it (it registers vector-icon fonts; `MaterialIcons.ttf` is already listed under `UIAppFonts` in `Info.plist`).

## 5. When the build breaks
```bash
yarn clean
yarn start --reset-cache
```
For iOS, you can use `cd ios && bundle install && bundle exec pod install` to stay on the Gemfile's CocoaPods version.

## Gotchas
- `ios/.xcode.env.local` is local only (gitignored). Set `NODE_BINARY` there if Xcode can't find node.
- Release builds are signed with the debug keystore (`android/app/build.gradle`). Create your own keystore before shipping, and never commit it (`*.keystore` is gitignored except `debug.keystore`).

## Verify
```bash
test -f .env && test -f src/core/config/env.ts && echo "env ok"
npx tsc --noEmit --ignoreDeprecations 6.0   # only known baseline errors may remain (see CLAUDE.md)
yarn android:dev                            # or: yarn ios:dev
```
