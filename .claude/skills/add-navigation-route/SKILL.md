---
name: add-navigation-route
description: Adds or changes routes in the React Navigation static native-stack configured in src/app/navigation/navigator.ts - typed RootStackParamList entries, param types in src/app/navigation/param, screen registration around the plop anchor comments, navigating with typed useNavigation, reading params with useRoute, and screen options. Use when adding a route for an existing view, passing params between screens, changing the initial route, adding nested/tab navigators, or fixing navigation types. Triggers: navigation, route, navigate, params, stack, screen registration, initialRouteName, goBack, deep link, tabs.
---

# Add or change a navigation route

The setup is `src/app/navigation/navigator.ts`: a **static** `createNativeStackNavigator<RootStackParamList>({...})` rendered with `createStaticNavigation(RootStack)`. It uses `headerShown: false` because screens draw their own `AppBar`.

## New screen → use plop
`yarn plop:feature` registers the import, the `RootStackParamList` entry (`undefined`) and the `screens` entry automatically (see `create-feature`). Use the manual steps below only for routes that point at an existing view, or to add params.

## Anchors (never touch)
```ts
// @plop-navigation-import-anchor <-- Do not remove
  // @plop-navigation-param-anchor <-- Do not remove
    // @plop-navigation-screen-anchor <-- Do not remove
```
Add manual entries **below** the anchor lines, in the same shape plop uses. Never edit or remove the anchor lines themselves.

## Add typed params
1. Create `src/app/navigation/param/<camel>NavigationParam.ts`, using the naming of `demoFeatureNavigationParam.ts`:
   ```ts
   export type orderDetailNavigationParam = {
     orderId: string;
   };
   ```
2. Use it in `RootStackParamList`, replacing the `undefined` that plop wrote:
   ```ts
   import { orderDetailNavigationParam } from './param/orderDetailNavigationParam';

   export type RootStackParamList = {
     // @plop-navigation-param-anchor <-- Do not remove
     OrderDetail: orderDetailNavigationParam;
     Landing: demoFeatureNavigationParam;
     Demo: undefined;
   };
   ```
   Note: the template's `Landing` reuses `demoFeatureNavigationParam` (`undefined`). Give new routes their own param type.

## Navigate (as LandingIntroductionSection does)
```ts
import { useNavigation } from '@react-navigation/native';
import { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { RootStackParamList } from '@app/navigation/navigator';

type NavigationProp = NativeStackNavigationProp<RootStackParamList, 'Landing'>;

const navigation = useNavigation<NavigationProp>();
navigation.navigate('OrderDetail', { orderId: '42' });
```
`AppBar` already handles back navigation (`navigation.canGoBack() && navigation.goBack()`).

## Read params
```ts
import { RouteProp, useRoute } from '@react-navigation/native';

const route = useRoute<RouteProp<RootStackParamList, 'OrderDetail'>>();
const { orderId } = route.params;
```
Read params in the **View** or the **Screen** and pass them down as props, so sections stay reusable and testable.

## Screen options
Per screen: `OrderDetail: { screen: OrderDetailView, options: { animation: 'slide_from_right', gestureEnabled: false } }`.
Global options go in `screenOptions`. Change the start screen with `initialRouteName`.

## Nested navigators (tabs, modals)
- Define the nested navigator in `src/app/navigation/` (e.g. `homeTabs.ts`) with the same static API and register it as a `screen`.
- Bottom tabs need `@react-navigation/bottom-tabs`, which isn't installed. Add it with an exact version (see `manage-dependencies`).
- Keep the root anchors in `navigator.ts` intact.

## Verify
```bash
grep -c "@plop-navigation-" src/app/navigation/navigator.ts   # must print 3
npx prettier --check src/app/navigation
npx tsc --noEmit --ignoreDeprecations 6.0   # wrong route names/params fail here
```
Then run the app and navigate forward and back.
