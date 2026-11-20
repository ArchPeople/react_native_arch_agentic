# CLAUDE.md

Project instructions for Claude Code. This repo was created from the **Arch React Native project template**. Read this file first, then load the matching skill from `.claude/skills/` before you change anything.

## Project at a glance

| Item | Value |
| --- | --- |
| Type | Bare React Native CLI app (**no Expo**), New Architecture, Hermes |
| Versions | React Native 0.88.1, React 19.3.1, TypeScript 6.0.3 (see `package.json`) |
| Package manager | **yarn 1.22.22** (`yarn.lock`). Never install with npm, pnpm or bun. |
| Node / tooling | Node `>= 24.3.0` (`engines`), Java 17+, Xcode 16.4+, CocoaPods (`Gemfile`) |
| Navigation | `@react-navigation/native` + `native-stack`, **static API** (`createStaticNavigation`) |
| Server state | `@tanstack/react-query` (client in `src/core/services/query/query.ts`) |
| Client state | `zustand` |
| HTTP | `axios` wrapped by `apiFetch` (`src/core/services/api-fetch/apiFetch.ts`) |
| Storage | `react-native-mmkv` wrapped by `localStorage` (`src/core/services/local-storage/localStorage.ts`) |
| i18n | `i18next` + `react-i18next` + `react-native-localize` (`src/app/localization/`) |
| UI | In-house atomic components + theme tokens, `ds()` responsive sizing, dark mode via zustand, `react-native-linear-gradient`, `@react-native-vector-icons/material-icons`, `react-native-reanimated` 4 + `react-native-worklets` |
| Generator | `plop` (`plop-templates/feature/plop-feature.js`) |
| Tests | Jest 30 + `@react-native/jest-preset` + `react-test-renderer` |
| Flavors | `development`, `staging`, `production` (Android productFlavors, iOS schemes + `Debug-/Release-<flavor>` configs) |

**The app name, bundle id and repo name are placeholders.** `yarn arch:init` renames them, so don't hard-code them. Read the current values from:
- JS app name: `app.json` (`name`), `package.json` (`name`)
- Android: `android/app/build.gradle` (`namespace`, `applicationId`, and `resValue "string", "app_name"` in each flavor)
- iOS: `ios/<RepoName>.xcodeproj/project.pbxproj` (`PRODUCT_BUNDLE_IDENTIFIER` and `PRODUCT_NAME` in each build config)

## Folder map

```text
.arch/                      Setup scripts: init.js, packageName.js, generateEnv.js
plop-templates/feature/     Feature generator (plop-feature.js + *.hbs templates)
guide/                      Human docs (how-to-run, plop, change package/icon)
assets/images/              Raw image files (registered in src/app/assets/images/assetImages.ts)
__tests__/                  Jest tests
android/ ios/               Native projects (flavors, schemes, Podfile)
src/
├── App.tsx                 Root: QueryClientProvider → I18nextProvider → Navigator
├── app/                    App-wide UI and wiring
│   ├── assets/             assetColors (palette), assetImages (require() map)
│   ├── components/         Atomic design: atoms/ molecules/ organisms/ (each with index.ts barrel)
│   ├── global-state/       App-wide zustand stores (e.g. system-mode/systemMode.ts)
│   ├── localization/       i18n.ts + languages/<lang>.json
│   ├── navigation/         navigator.ts (RootStackParamList + screens + plop anchors), param/
│   └── themes/             themeFonts, themePadding, themeMargin, themeBorder, themeShadow,
│                           themeGradient, themeSystemMode, themeDeviceDimension
├── core/                   Non-UI logic
│   ├── common/             base/baseFailure.interface.ts (IBaseFailure), view-state/viewState.ts
│   ├── config/             config.ts (+ env.ts: GENERATED, gitignored)
│   ├── data/
│   │   ├── remote/         api-path.ts, <name>-data/<name>Data.ts + interface/*.interface.ts
│   │   └── local/          storageKey.ts, <name>-data/<name>Data.ts
│   ├── general-helpers/    extensions/ (ds) and utils/ (string, number, screenSize, localization)
│   └── services/           api-fetch/ (axios), local-storage/ (MMKV), query/ (QueryClient)
└── features/<kebab-name>/  One feature = one page (canonical example: src/features/demo-feature)
    ├── api/<camel>Api.ts           react-query hooks (useGetX / useMutation)
    ├── store/<camel>Store.ts       feature zustand store (use<Pascal>Store)
    ├── view/<Pascal>View.tsx       entry registered in the navigator
    ├── view/screen/<Pascal>Screen.tsx
    ├── view/section/*Section.tsx   large screen blocks
    ├── view/component/*.tsx        small feature-only components
    └── index.ts                    export { <Pascal>View }
```

Path aliases are defined in **both** `tsconfig.json` (`paths`) and `babel.config.js` (`module-resolver`): `@app/*` → `src/app/*`, `@features/*` → `src/features/*`, `@core/*` → `src/core/*`.

## Data flow

```text
<Pascal>Screen / Section                      (src/features/x/view)
   │  useGetX() / useXMutation()              ← src/features/x/api/xApi.ts       (react-query)
   │  useXStore(s => s.field)                 ← src/features/x/store/xStore.ts   (zustand, feature)
   │  useSystemModeStore(s => s.systemMode)   ← src/app/global-state             (zustand, app-wide)
   ▼
xData.fetchX({ signal })                      ← src/core/data/remote/x-data/xData.ts
   │  Promise<IXResponse | IBaseFailure>       (never throws; catch → IBaseFailure)
   ▼
apiFetch.getApi/postApi/putApi/deleteApi      ← src/core/services/api-fetch (axios instance + interceptors)
   │  url = config.<domain> + apiPath.<key>   ← src/core/config/config.ts + src/core/data/remote/api-path.ts
   ▼
HTTP API

Local:  store / screen → xData (src/core/data/local) → localStorage (MMKV) with keys from storageKey.ts
Env:    .env → yarn arch:generate-env → src/core/config/env.ts → config.ts → import { config } from '@core/config'
```

The api hook narrows the union with `'errorStatus' in response` and **throws**, so the screen reads `isError` / `error` from react-query.

## Commands

| Purpose | Command |
| --- | --- |
| Install deps | `yarn install` (plus `cd ios && pod install` after native changes) |
| First-time init (rename app/package/repo, write `.env`) | `yarn arch:init`, then `yarn clean && yarn arch:generate-env` |
| Change package name / bundle id only | `yarn arch:change-package-name` |
| Generate `src/core/config/env.ts` from `.env` | `yarn arch:generate-env` |
| New feature (interactive) | `yarn plop:feature` |
| New feature (non-interactive) | `yarn plop:feature feature "order detail"` |
| Metro bundler | `yarn start` |
| Run Android | `yarn android:dev` · `yarn android:stg` · `yarn android:prod` |
| Run iOS | `yarn ios:dev` · `yarn ios:stg` · `yarn ios:prod` |
| Lint | `yarn lint` |
| Type-check | `npx tsc --noEmit` |
| Format | `npx prettier --write <files>` / `npx prettier --check <files>` |
| Tests | `yarn test` (single file: `yarn test __tests__/path/Name.test.tsx`) |
| Full clean + reinstall + pods | `yarn clean` |

### Known baseline issues (template as shipped)

These checks already fail on the untouched template. Don't fix them as a side effect of another task; fix them only when explicitly asked.

1. **`npx tsc --noEmit`** fails with TS5101 because `baseUrl` is deprecated in TS 6.
   - Workaround: `npx tsc --noEmit --ignoreDeprecations 6.0`.
   - That command still reports one existing error: `__tests__/App.test.tsx` imports `../App`, which doesn't exist (the app lives in `src/App.tsx`).
2. **`yarn lint`** fails: ESLint 10 needs `eslint.config.js`, but the repo only has `.eslintrc.js`. There's no workaround short of migrating to flat config, so rely on tsc + Prettier until then.
3. **`yarn test`** fails with `this._moduleMocker.clearMocksOnScope is not a function`. The cause: `@react-native/jest-preset` bundles `jest-environment-node@29`, while Jest is 30.
   - Workaround without editing the repo: create `/tmp/rn-env-jest30.js` with the content below. Replace `<repo>` with the absolute project path.
     ```js
     const NodeEnv = require('<repo>/node_modules/jest-environment-node').TestEnvironment;
     module.exports = class extends NodeEnv {
       customExportConditions = ['require', 'react-native'];
     };
     ```
   - Then run `npx jest --testEnvironment=/tmp/rn-env-jest30.js <test-path>`.
   - `__tests__/App.test.tsx` still fails under the workaround because of the wrong import in item 1.

If your project has already fixed these, use the plain commands and delete this subsection.

## Non-negotiable rules

1. **Use the generator for new features/screens** (`yarn plop:feature`). Never copy a feature folder by hand.
2. **Never remove, move, rename or reformat the plop anchors** in `src/app/navigation/navigator.ts`:
   - `// @plop-navigation-import-anchor <-- Do not remove`
   - `// @plop-navigation-param-anchor <-- Do not remove`
   - `// @plop-navigation-screen-anchor <-- Do not remove`

   The generator matches them by exact text, including the leading spaces.
3. **Never hand-edit `src/core/config/env.ts`, and never commit it or `.env`** (both are gitignored).
   - To change a value, edit `.env` and run `yarn arch:generate-env`.
   - Never put real secrets in `.env`: the values are bundled into the JS and readable in a release build.
4. **Never edit generated or build output**: `node_modules/`, `ios/Pods/`, `ios/build/`, `android/build/`, `android/app/build/`, `android/.gradle/`. Never hand-edit `yarn.lock` or `ios/Podfile.lock`.
5. **Pin exact versions**: every dependency in `package.json` is exact (no `^`/`~`). Add packages with `yarn add <pkg>@<x.y.z> --exact`.
6. **Respect the layers**:
   - Screens never call `apiFetch`/axios or `localStorage`/MMKV directly. Remote calls go through `features/x/api` → `core/data/remote`; local storage goes through `core/data/local`.
   - `src/core` never imports from `src/features`.
7. **Imports**:
   - Use the aliases `@app/…`, `@core/…`, `@features/…` across folders.
   - Import from barrels where they exist: `@app/components/atoms|molecules|organisms`, `@app/themes`, `@app/assets`, `@app/global-state`, `@core/config`, `@core/data/remote`, `@core/data/local`, `@core/general-helpers/utils`, `@core/general-helpers/extensions`.
   - Use relative imports only inside the same feature/module.
   - A new alias must be added to **both** `tsconfig.json` and `babel.config.js`.
8. **Styling**:
   - Every size goes through `ds()`.
   - Use theme tokens (`themePadding`, `themeMargin`, `themeFonts`, `themeBorder`, `themeShadow`, `themeGradient`) and colors from `assetColors`. No raw hex values in features.
   - Prefer `TextBase`, `ButtonBase`, `PressableBase`, `MaterialIconBase`, `Scaffold` and `AppBar` over raw primitives.
   - Support light **and** dark mode, via `useSystemModeStore` + `themeSystemMode` or `color`/`darkColor` props.
9. **Localize every user-facing string**:
   - Use `const { t } = useTranslation()` with a snake_case key.
   - Add the key to **every** file in `src/app/localization/languages/`.
   - This includes the strings plop generates, such as the `AppBar` title.
10. **Data sources return failures as `IBaseFailure`; they never throw.** Api hooks turn failures into thrown errors for react-query.
11. Keep `react-native-worklets/plugin` as the **last** Babel plugin.
12. Formatting is Prettier (`.prettierrc.js`: single quotes, trailing commas, `arrowParens: 'avoid'`). `plop-templates/**/*.hbs` are excluded (`.prettierignore`).

## Naming conventions

| Thing | Convention | Example |
| --- | --- | --- |
| Feature folder | kebab-case | `src/features/order-detail` |
| Component file / export | PascalCase, `React.FC<Props>` wrapped in `memo` | `OrderDetailScreen.tsx`, `ActionPanel.tsx` |
| Shared component folder | kebab-case, file PascalCase | `components/molecules/action-panel/ActionPanel.tsx` |
| Props type | `type <Name>Props = {…}` in the same file | `type ButtonBaseProps` |
| Non-component module | camelCase file exporting an **object literal** | `demoData`, `apiFetch`, `localStorage`, `stringUtil`, `themeFonts` |
| Interfaces | `I` prefix, file `*.interface.ts` | `IBaseFailure`, `IDemoDataResponse` |
| Query hooks | `useGet<Thing>` / `use<Action><Thing>` | `useGetDemoData` |
| Zustand stores | `use<Pascal>Store`, state type `<camel>State` | `useDemoFeatureStore` |
| Routes | PascalCase key in `RootStackParamList` | `OrderDetail` |
| i18n keys | snake_case | `welcome_message` |
| Storage keys | `<name>StorageKey: '<name>_storage_key'` | `appStatusStorageKey` |
| Utils | `<domain>Util` in `general-helpers/utils/<domain>/` | `screenSizeUtil` |

Styles go in `const styles = StyleSheet.create({…})` at the bottom of the file and spread theme tokens (`...themePadding.ph16`).

## Definition of done

- [ ] Code follows the folder map, layering and naming above, and the feature was created with plop.
- [ ] No hard-coded user-facing strings; new keys exist in every language JSON.
- [ ] No raw colors or unscaled sizes, and the UI works in both light and dark mode.
- [ ] `RootStackParamList` is updated, and the plop anchors are intact.
- [ ] `npx prettier --check <changed files>` passes.
- [ ] Type-check is clean apart from the known baseline errors above.
- [ ] Tests are added or updated for new logic (see `write-tests`) and pass.
- [ ] `git status` shows no `.env`, `env.ts`, secrets or build output staged.
- [ ] After native or dependency changes, pods are installed and the app runs on the affected platform(s) and flavor.

## Skills and subagents index

| Skill | Use when |
| --- | --- |
| [project-setup](.claude/skills/project-setup/SKILL.md) | First run after cloning: init/rename, env, pods, run per flavor, clean, icons |
| [create-feature](.claude/skills/create-feature/SKILL.md) | Adding a new screen/feature with the plop generator |
| [add-api-endpoint](.claude/skills/add-api-endpoint/SKILL.md) | Calling a new REST endpoint: apiPath, interfaces, data source, react-query hook |
| [add-local-storage](.claude/skills/add-local-storage/SKILL.md) | Persisting values on the device with MMKV |
| [manage-state](.claude/skills/manage-state/SKILL.md) | Choosing and implementing state: react-query vs feature store vs global store |
| [add-navigation-route](.claude/skills/add-navigation-route/SKILL.md) | Routes, typed params, navigating between screens |
| [build-ui](.claude/skills/build-ui/SKILL.md) | Components, theme tokens, `ds()`, dark mode, colors, images, icons |
| [add-localization](.claude/skills/add-localization/SKILL.md) | Translating strings, adding a language, switching language |
| [manage-environment-config](.claude/skills/manage-environment-config/SKILL.md) | `.env`, `env.ts`, `config.ts`, flavors/schemes |
| [manage-dependencies](.claude/skills/manage-dependencies/SKILL.md) | Adding, upgrading or removing packages (exact pins, pods, native rebuild) |
| [write-tests](.claude/skills/write-tests/SKILL.md) | Writing Jest tests with mocked native modules and providers |

| Subagent | Use when |
| --- | --- |
| [rn-architect](.claude/agents/rn-architect.md) | Read-only planning of a feature or change before coding |
| [rn-feature-developer](.claude/agents/rn-feature-developer.md) | Implementing features end to end (plop, api, state, navigation, i18n) |
| [rn-ui-developer](.claude/agents/rn-ui-developer.md) | Building or restyling screens and shared components |
| [rn-test-engineer](.claude/agents/rn-test-engineer.md) | Writing and running Jest tests |
| [rn-code-reviewer](.claude/agents/rn-code-reviewer.md) | Read-only review of a diff against these rules |
