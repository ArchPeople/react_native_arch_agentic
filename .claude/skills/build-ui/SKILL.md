---
name: build-ui
description: Builds and styles UI with the repo's design system - atomic components in src/app/components (atoms, molecules, organisms), feature sections/components, theme tokens (themeFonts, themePadding, themeMargin, themeBorder, themeShadow, themeGradient), the ds() responsive size helper, assetColors, assetImages, Material icons, gradients, Scaffold/AppBar layout, and light/dark mode via useSystemModeStore. Use when creating or restyling a screen, section or reusable component, adding colors, images or icons, fixing layout/responsiveness, or supporting dark mode. Triggers: UI, component, style, layout, theme, color, font, spacing, dark mode, icon, image, responsive, atom, molecule, organism.
---

# Build UI

## Where components go
| Level | Path | Rule | Examples |
| --- | --- | --- | --- |
| Atom | `src/app/components/atoms/<kebab>/<Pascal>.tsx` | wraps one primitive, no business logic | `TextBase`, `ButtonBase`, `PressableBase`, `MaterialIconBase`, `Gap`, `Spacer`, `GradientView` |
| Molecule | `src/app/components/molecules/<kebab>/<Pascal>.tsx` | a few atoms combined | `ActionPanel` |
| Organism | `src/app/components/organisms/<kebab>/<Pascal>.tsx` | larger blocks, may use navigation/store | `AppBar`, `Scaffold`, `StatusController` |
| Feature section | `src/features/<f>/view/section/<Name>Section.tsx` | a big block of one screen | `LandingIntroductionSection` |
| Feature component | `src/features/<f>/view/component/<Name>.tsx` | small, feature-only | plop's `SampleComponent.tsx` |

Promote a component to `src/app/components` only when two or more features use it. Always export new shared components from that level's `index.ts`.

## Component template (repo style)
```tsx
import { StyleSheet, View } from 'react-native';
import React, { memo } from 'react';
import { TextBase } from '@app/components/atoms';
import { assetColors } from '@app/assets';
import { useSystemModeStore } from '@app/global-state';
import { themeBorder, themeFonts, themePadding, themeSystemMode } from '@app/themes';
import { ds } from '@core/general-helpers/extensions';

type InfoCardProps = {
  title?: string;
  backgroundColor?: string;
  backgroundDarkColor?: string;
};

export const InfoCard: React.FC<InfoCardProps> = memo(
  ({ title, backgroundColor, backgroundDarkColor }) => {
    const systemMode = useSystemModeStore(state => state.systemMode);
    const isLight = systemMode == themeSystemMode.light;

    return (
      <View
        style={[
          styles.card,
          {
            backgroundColor: isLight
              ? backgroundColor ?? assetColors.lightMode
              : backgroundDarkColor ?? assetColors.darkMode,
          },
        ]}
      >
        <TextBase style={themeFonts.bodyMdSemibold}>{title}</TextBase>
      </View>
    );
  },
);

const styles = StyleSheet.create({
  card: {
    ...themePadding.pa16,
    ...themeBorder.radiusSM,
    minHeight: ds(64),
  },
});
```
Text passed in from features must already be translated with `t()` (see `add-localization`).

## Tokens cheat sheet (`@app/themes`)
- **Sizes**: `ds(n)` scales `n` from a 360-wide design baseline (`src/app/themes/device-dimension/themeDeviceDimension.ts`). Use it for every width, height, gap, radius and font size.
- **Fonts** (`themeFonts`):
  - Sizes: `fs8`–`fs96`; line heights: `fh1`, `fh1_5`, `fh2`, `fh2_5`.
  - Weights: `light`, `regular`, `medium`, `semiBold`, `bold`, `extraBold`, plus `italic`.
  - Presets: `d1`–`d6<Weight>`, `h1`–`h6<Weight>`, `sh1`–`sh6`, `bodyLg|Md|Sm|Xs<Weight>`, `caption<Weight>`.
  - Examples: `themeFonts.h4ExtraBold`, `themeFonts.bodySmLight`. The default `TextBase` style is `bodyMdRegular`.
- **Padding / Margin**: `themePadding.{ph|pv|pa|pt|pb|pl|pr}{0,2,4,8,12,16,20,24,32}` and the semantic forms `{...}{Null,XXS,XS,SM,MD,LG,XL,XXL}`. `themeMargin` follows the same pattern with `m`.
- **Border radius**: `themeBorder.bc{0..32}` (all corners) and `bt|bb|bl|br|btl|btr|bbl|bbr{4..32}`. Semantic: `radiusNull`, `radiusXXS`–`radiusXXL`.
- **Shadow**: `themeShadow.smallShadow | mediumShadow | largeShadow`.
- **Gradient points**: `themeGradient.topLeft` … `bottomRight`, for `GradientView` `start`/`end`.
- Tokens are **getters**, so spread them in StyleSheets (`...themePadding.ph16`) or pass them directly in style arrays.

## Colors and dark mode
- Every color comes from `src/app/assets/colors/assetColors.ts`. To add one, add it there (camelCase name) and never write hex in features.
- The backgrounds are `assetColors.lightMode` and `assetColors.darkMode`.
- `TextBase`, `ButtonBase` and `MaterialIconBase` take `color`/`darkColor` (or `labelColor`/`labelDarkColor`, `backgroundColor`/`backgroundDarkColor`) and pick the right one from `useSystemModeStore`.
- For custom views, branch on `systemMode == themeSystemMode.light` as `DemoFeatureScreen` does.
- Check every new UI in both modes. You can toggle the mode from the landing screen.

## Layout
- Root every screen in `Scaffold`. It adds a status-bar spacer from safe-area insets; pass `ignoreAppBarHeight` for full-bleed screens.
- Then add `<AppBar title=... leading=... actions=... />` and your content (`ScrollView` for long content).
- Spacing: use `<Gap height={ds(20)} />` / `<Gap width={...} />` between items and `<Spacer />` to push items apart (`flex: 1`).
- Device helpers: `screenSizeUtil.getScreenWidth`, `getScreenHeight` and `isTablet` (`src/core/general-helpers/utils/screen-size/screnSizeUtil.ts`).
  - `getStatusBarHeight` and `getBottomBarHeight` call a **hook** internally, so read them only during render, never in callbacks or at module scope.

## Images and icons
- Images: put the file in `assets/images/`, then register it in `src/app/assets/images/assetImages.ts`:
  ```ts
  myBanner: require('../../../../assets/images/my-banner.png'),
  ```
  Use it as `<Image source={assetImages.myBanner} style={{ height: ds(80), resizeMode: 'contain' }} />`. Provide light/dark variants if needed (see `archLogoBlack` / `archLogoWhite`).
- Icons: `<MaterialIconBase name="arrow-back" size={ds(22)} color={...} darkColor={...} onPress={...} />`. Names come from the Material Icons glyph map (`node_modules/@react-native-vector-icons/material-icons/glyphmaps/`).
  - Another icon family needs a new `@react-native-vector-icons/*` package, `UIAppFonts` in `Info.plist` and `pod install` (see `manage-dependencies`).
- Fonts: the template only uses system fonts with weights. Custom font files need native linking (Android `assets/fonts`, iOS `UIAppFonts`) and a rebuild.

## Interaction and accessibility
- Use `ButtonBase` / `PressableBase`. They prefix `accessibilityLabel` (`button-…`, `pressable-…`, `text-…`), so pass a meaningful label; tests and E2E rely on it.
- Animations use `react-native-reanimated` 4 (worklets Babel plugin already configured). Keep animations inside components and don't change `babel.config.js`.

## Verify
```bash
npx prettier --check <changed files>
npx tsc --noEmit --ignoreDeprecations 6.0
yarn ios:dev   # or android:dev: check light + dark mode and a small and large device
```
Add a render test for new shared components (see `write-tests`).
