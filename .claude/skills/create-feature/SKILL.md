---
name: create-feature
description: Creates a new feature (one page/screen) with the repo's plop generator, which scaffolds src/features/<kebab-name> (api, store, view/screen, sections, component, View, index) and auto-registers the route in src/app/navigation/navigator.ts via anchor comments. Explains exactly what is generated, which placeholders must be replaced, and how to finish wiring. Use when adding a new screen, page, flow or feature module, or when asked to scaffold or generate a feature. Triggers: new screen, new page, new feature, scaffold, generate, plop.
---

# Create a feature

A **feature** is one page plus everything it owns (api hooks, store, UI). The canonical example is `src/features/demo-feature`.

## 1. Generate (never copy folders by hand)
```bash
yarn plop:feature                          # interactive prompt: "What is the feature name?"
yarn plop:feature feature "order detail"   # non-interactive
```
Type the name in words. Plop derives `kebab-case` (folder), `camelCase` (files) and `PascalCase` (components/route) from it.

## 2. What it creates (`plop-templates/feature/plop-feature.js`)
Example input: `"order detail"`:

| File | Template | Content |
| --- | --- | --- |
| `src/features/order-detail/index.ts` | `index.ts.hbs` | `export { OrderDetailView } from './view/OrderDetailView';` |
| `.../store/orderDetailStore.ts` | `store/store.ts.hbs` | zustand `useOrderDetailStore` with placeholder `data`/`setdata` |
| `.../api/orderDetailApi.ts` | `api/api.ts.hbs` | **copy of the demo hook** (`useGetDemoData`, `queryKey: ['demoData']`, calls `demoData`) |
| `.../view/OrderDetailView.tsx` | `view/view.tsx.hbs` | renders `<OrderDetailScreen />` |
| `.../view/screen/OrderDetailScreen.tsx` | `view/screen/screen.tsx.hbs` | `Scaffold` → `ScrollView` → `AppBarSection` + `BodySection` |
| `.../view/section/AppBarSection.tsx` | `view/section/appBar.tsx.hbs` | `<AppBar title="Order Detail" />` (hard-coded) |
| `.../view/section/BodySection.tsx` | `view/section/body.tsx.hbs` | renders `SampleWidget` |
| `.../view/component/SampleComponent.tsx` | `view/component/component.tsx.hbs` | exports `SampleWidget` (file and export names differ) |

It also **edits `src/app/navigation/navigator.ts`** by appending after the three anchors:
```ts
// @plop-navigation-import-anchor <-- Do not remove
import { OrderDetailView } from '@features/order-detail';
...
  // @plop-navigation-param-anchor <-- Do not remove
  OrderDetail: undefined,
...
    // @plop-navigation-screen-anchor <-- Do not remove
    OrderDetail: {
      screen: OrderDetailView,
    },
```
Never touch the anchor lines. The regexes in `plop-feature.js` match them exactly, including indentation.

## 3. Finish the scaffold (required)
1. **Format** the generated code. The param line ends with `,`, and Prettier normalizes it:
   `npx prettier --write src/features/<kebab> src/app/navigation/navigator.ts`
2. **API hook**: replace the demo copy in `api/<camel>Api.ts` with a real data source (see `add-api-endpoint`), and use a **unique** `queryKey` (e.g. `['orderDetail', id]`). Delete the file if the feature has no remote data.
3. **Store**: replace the `data`/`setdata` placeholder with real state, or delete the store if it isn't needed (see `manage-state`). Keep the `use<Pascal>Store` naming.
4. **Localize** the AppBar title and any other text: `title={t('order_detail_title')}`, with keys in every `src/app/localization/languages/*.json` (see `add-localization`).
5. **Rename** `SampleComponent.tsx` / `SampleWidget` to something meaningful, or delete them. Keep feature-only components in `view/component/` and big blocks in `view/section/`.
6. **Route params**: plop always registers `undefined`. If the screen needs params, see `add-navigation-route`.
7. **Navigate** to it from somewhere:
   ```ts
   type NavigationProp = NativeStackNavigationProp<RootStackParamList, 'OrderDetail'>;
   const navigation = useNavigation<NavigationProp>();
   navigation.navigate('OrderDetail');
   ```

## Conventions in feature code (copy from demo-feature)
```tsx
type OrderDetailScreenProps = {};

export const OrderDetailScreen: React.FC<OrderDetailScreenProps> = memo(() => {
  const systemMode = useSystemModeStore(state => state.systemMode);
  const { data, isFetching, isError, refetch } = useGetOrderDetail();
  return (
    <Scaffold>
      <AppBar title={t('order_detail_title')} />
      ...
    </Scaffold>
  );
});

const styles = StyleSheet.create({ container: { flex: 1, ...themePadding.ph16 } });
```
- The screen root is `Scaffold`. Pass `ignoreAppBarHeight` only for full-bleed screens such as the landing screen.
- `<Pascal>View` stays a thin wrapper. Put logic in the screen and sections.
- Import other features only through their `index.ts`, and avoid it when you can. Shared things belong in `src/app` or `src/core`.

## Removing a feature
Delete `src/features/<kebab>`. Then remove its import, its `RootStackParamList` entry and its `screens` entry from `navigator.ts`, leaving the anchors in place. Finally delete any unused i18n keys and param types.

## Verify
```bash
npx prettier --check src/features/<kebab> src/app/navigation/navigator.ts
npx tsc --noEmit --ignoreDeprecations 6.0   # only known baseline errors may remain
grep -c "@plop-navigation-" src/app/navigation/navigator.ts   # must print 3
yarn android:dev   # or ios:dev, then navigate to the new screen
```
