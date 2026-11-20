---
name: add-localization
description: Localizes user-facing text with i18next/react-i18next as configured in src/app/localization - adding snake_case keys to every languages/<lang>.json, using t() with interpolation in components, translating outside React, switching language with localizationUtil, and adding a new language to i18n.ts resources. Use when adding or changing any visible text, translating screens, adding a locale, or replacing hard-coded strings (including plop-generated titles). Triggers: translation, i18n, localize, language, locale, t(), strings, copy, French, multi-language.
---

# Add localization

The setup lives in `src/app/localization/i18n.ts`:
- `resources = { en, fr }`, loaded from `languages/*.json`.
- The initial language comes from `RNLocalize.findBestLanguageTag`, falling back to `en`.
- `i18nInstance` is provided in `src/App.tsx`.

**Rule:** every user-facing string goes through `t()`, and every key exists in **every** language file.

## 1. Add keys
Add the same snake_case key to all files in `src/app/localization/languages/`:
```json
// en.json
{
  "order_detail_title": "Order Detail",
  "items_count": "{{count}} items"
}
```
```json
// fr.json
{
  "order_detail_title": "Détail de la commande",
  "items_count": "{{count}} articles"
}
```
- Keys stay flat and snake_case, like the existing `welcome_message` and `light_mode`. Prefix them by feature when that helps (`order_detail_…`).
- Interpolation uses `{{name}}`. `escapeValue: false` is set, which is safe in React Native.
- If you don't know the translation, add the English text and tell the user which keys need translating. Never leave a key missing in one file.

## 2. Use in components (as LandingIntroductionSection does)
```tsx
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();
<TextBase>{t('welcome_message', { name: 'Arch' })}</TextBase>
<AppBar title={t('order_detail_title')} />
<ButtonBase label={t('confirm')} />
```
Replace plop's hard-coded `AppBar` title in `view/section/AppBarSection.tsx`.

## 3. Outside React
```ts
import { i18nInstance } from '@app/localization/i18n';
const message = i18nInstance.t('generic_error');
```

## 4. Switch language
```ts
import { localizationUtil } from '@core/general-helpers/utils';
localizationUtil.changeLanguage('fr');
```
The choice isn't persisted. To remember it, store it with a `core/data/local` module (see `add-local-storage`) and pass it as `lng` on init.

## 5. Add a new language
1. Copy `en.json` to `src/app/localization/languages/<code>.json` and translate every key.
2. Register it in `src/app/localization/i18n.ts`:
   ```ts
   import id from './languages/id.json';
   const resources = {
     en: { translation: en },
     fr: { translation: fr },
     id: { translation: id },
   };
   ```
3. Add a switcher entry wherever languages are offered.

## Verify
```bash
node -e "const fs=require('fs'),d='src/app/localization/languages/';const f=fs.readdirSync(d);const k=f.map(x=>Object.keys(require('./'+d+x)).sort().join());console.log(new Set(k).size===1?'keys match':'KEY MISMATCH',f)"
npx tsc --noEmit --ignoreDeprecations 6.0
```
Then switch languages in the app and check the screen in each one.
