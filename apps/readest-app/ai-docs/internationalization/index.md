# Internationalization (i18n) Feature

**Feature Category**: Localization, User Interface
**Added**: December 2024 (commit 46127304)
**Dependencies**: i18next, react-i18next

## Overview

The Internationalization (i18n) feature enables Readest to support multiple languages, making the app accessible to users worldwide. The system uses **i18next** framework with automatic language detection, persistent user preferences, and comprehensive translation coverage across all UI components.

### Supported Languages (Updated v0.9.15)

Readest supports 15 languages with full translation coverage:

| Language | Code | Status | Added |
|----------|------|--------|-------|
| English | `en` | Default/Fallback | Initial |
| German | `de` | ✅ Complete | v0.9.0 |
| Spanish | `es` | ✅ Complete | v0.9.0 |
| French | `fr` | ✅ Complete | v0.9.0 |
| Italian | `it` | ✅ Complete | v0.9.0 |
| Japanese | `ja` | ✅ Complete | v0.9.0 |
| Korean | `ko` | ✅ Complete | v0.9.0 |
| Portuguese | `pt` | ✅ Complete | v0.9.0 |
| Russian | `ru` | ✅ Complete | v0.9.0 |
| Turkish | `tr` | ✅ Complete | v0.9.0 |
| Vietnamese | `vi` | ✅ Complete | v0.9.0 |
| Indonesian | `id` | ✅ Complete | v0.9.0 |
| Chinese (Simplified) | `zh-CN` | ✅ Complete | v0.9.0 |
| Chinese (Traditional) | `zh-TW` | ✅ Complete | v0.9.0 |
| **Arabic** | `ar` | ✅ Complete | **v0.9.15** (#432) |

### Additional Fallback Support

**Regional variants** automatically fall back to related languages:
- `zh-HK` (Hong Kong) → `zh-TW` → `en`
- Central Asian languages (`kk`, `ky`, `tk`, `uz`, `ug`, `tt`) → `ru` → `en`

### RTL (Right-to-Left) Language Support

**Added**: v0.9.15 (Commit #432)

Arabic is the first RTL language supported in Readest. The UI automatically adjusts layout direction based on the selected language.

**Implementation**:
- `dir="rtl"` applied to root HTML element when Arabic is selected
- Mirror layout for sidebars, menus, modals
- Text alignment switches to right-aligned
- Icon positions flip for RTL context

**Files Modified**:
- `public/locales/ar/translation.json` - Arabic translations
- `src/i18n/i18n.ts` - RTL detection logic
- `src/app/layout.tsx` - Dynamic `dir` attribute

**Future RTL Languages**: Hebrew and other RTL languages can be added following the same pattern established for Arabic.

**Detailed Documentation**: See **[RTL and Arabic Language Support](./rtl-arabic.md)** for comprehensive RTL implementation details, supported languages, UI direction switching, typography considerations, and AI agent modification guidelines.

## Architecture

### Technology Stack

| Component | Library | Version | Purpose |
|-----------|---------|---------|---------|
| Core Framework | i18next | 24.2.0 | Translation engine |
| React Integration | react-i18next | 15.2.0 | React hooks and components |
| Backend | i18next-http-backend | 3.0.1 | Load translation files |
| Language Detection | i18next-browser-languagedetector | 8.0.2 | Auto-detect user language |
| Key Extraction | i18next-scanner | 4.6.0 | Extract translation keys from code |

### File Structure

```
apps/readest-app/
├── public/locales/              # Translation files
│   ├── de/translation.json
│   ├── en/translation.json
│   ├── es/translation.json
│   ├── fr/translation.json
│   ├── id/translation.json
│   ├── it/translation.json
│   ├── ja/translation.json
│   ├── ko/translation.json
│   ├── pt/translation.json
│   ├── ru/translation.json
│   ├── tr/translation.json
│   ├── vi/translation.json
│   ├── zh-CN/translation.json
│   └── zh-TW/translation.json
├── src/
│   ├── i18n/
│   │   └── i18n.ts             # i18next initialization
│   └── hooks/
│       └── useTranslation.ts   # Custom translation hook
└── i18next-scanner.config.js   # Scanner configuration
```

## Key Components

### 1. i18n Initialization (`src/i18n/i18n.ts`)

**Purpose**: Configures and initializes i18next with all plugins.

**Configuration:**

```typescript
i18n
  .use(LanguageDetector)      // Auto-detect language
  .use(HttpBackend)           // Load translations via HTTP
  .use(initReactI18next)      // React integration
  .init({
    fallbackLng: 'en',
    debug: false,
    ns: ['translation'],
    defaultNS: 'translation',

    backend: {
      loadPath: '/locales/{{lng}}/{{ns}}.json',
    },

    detection: {
      order: ['querystring', 'localStorage', 'navigator'],
      caches: ['localStorage'],
      lookupQuerystring: 'lng',
      lookupLocalStorage: 'i18nextLng',
    },

    interpolation: {
      escapeValue: false,  // React already escapes
    },

    react: {
      useSuspense: false,  // Avoid loading delays
    },
  });
```

**Language Detection Order**:
1. **Query string**: `?lng=de` in URL
2. **localStorage**: Previously selected language
3. **Browser navigator**: System language

### 2. Custom Translation Hook (`src/hooks/useTranslation.ts`)

**Purpose**: Provides a simplified translation function with automatic fallback.

**Implementation:**

```typescript
import { useTranslation as _useTranslation } from 'react-i18next';

export const useTranslation = (namespace: string = 'translation') => {
  const { t } = _useTranslation(namespace);
  return (key: string, options = {}) =>
    t(key, { defaultValue: key, ...options });
};
```

**Key Feature**: If a translation key is missing, it returns the key itself as the default value, preventing blank UI elements.

### 3. Translation Files (`public/locales/{lang}/translation.json`)

**Format**: JSON key-value pairs with variable interpolation support.

**Example** (`public/locales/de/translation.json`):

```json
{
  "About Readest": "Über Readest",
  "Book": "Buch",
  "Books": "Bücher",
  "Settings": "Einstellungen",
  "Failed to import book(s): {{filenames}}": "Fehler beim Importieren von Buch/Büchern: {{filenames}}",
  "Logged in as {{userDisplayName}}": "Angemeldet als {{userDisplayName}}",
  "Are you sure you want to delete {{count}} book(s)?": "Möchten Sie {{count}} Buch/Bücher wirklich löschen?"
}
```

**Total Keys**: 115+ translation keys per language

### 4. Scanner Configuration (`i18next-scanner.config.js`)

**Purpose**: Automatically extracts translation keys from source code.

**Configuration:**

```javascript
module.exports = {
  input: [
    'src/**/*.{js,jsx,ts,tsx}',
    '!src/**/*.test.{js,jsx,ts,tsx}',
    '!**/node_modules/**',
  ],

  options: {
    func: {
      list: ['_'],  // Look for _() function calls
      extensions: ['.js', '.jsx', '.ts', '.tsx'],
    },

    lngs: ['en', 'de', 'es', 'fr', /* ... all 14 languages */],
    defaultLng: 'en',
    defaultValue: '__STRING_NOT_TRANSLATED__',

    resource: {
      loadPath: 'public/locales/{{lng}}/{{ns}}.json',
      savePath: 'public/locales/{{lng}}/{{ns}}.json',
    },

    keySeparator: false,       // Don't use dot notation
    nsSeparator: false,        // Don't use namespace separator
  },
};
```

**npm Script**:
```bash
npm run i18n:extract
```

## Usage Patterns

### Basic Translation

```typescript
import { useTranslation } from '@/hooks/useTranslation';

function MyComponent() {
  const _ = useTranslation();

  return (
    <div>
      <h1>{_('Welcome to Readest')}</h1>
      <button>{_('Settings')}</button>
    </div>
  );
}
```

### Translation with Variables

```typescript
function BookCard({ book }) {
  const _ = useTranslation();

  return (
    <span>
      {_('Written by {{author}}', { author: book.author })}
    </span>
  );
}
```

### Pluralization

```typescript
function BookList({ books }) {
  const _ = useTranslation();

  return (
    <p>
      {_('You have {{count}} book(s)', { count: books.length })}
    </p>
  );
}
```

### Conditional Translation

```typescript
function StatusMessage({ isSuccess }) {
  const _ = useTranslation();

  return (
    <div>
      {isSuccess ? _('Success!') : _('Failed!')}
    </div>
  );
}
```

## AI Agent Modification Guidelines

### Adding a New Language

**Step 1: Add language code to scanner config**

Edit `i18next-scanner.config.js`:

```javascript
lngs: ['en', 'de', 'es', 'fr', 'it', 'ja', 'ko', 'pt', 'ru', 'tr', 'vi', 'id', 'zh-CN', 'zh-TW', 'ar'],  // Added 'ar' for Arabic
```

**Step 2: Create translation file**

Create `public/locales/ar/translation.json`:

```json
{
  "About Readest": "حول Readest",
  "Book": "كتاب",
  "Settings": "الإعدادات"
}
```

**Step 3: Add fallback rules (if needed)**

Edit `src/i18n/i18n.ts`:

```typescript
const fallbackMapping: { [key: string]: string[] } = {
  'zh-HK': ['zh-TW', 'en'],
  'ar-EG': ['ar', 'en'],  // Add regional variant fallback
};
```

**Step 4: Test language detection**

Visit app with query string: `?lng=ar`

### Adding New Translation Keys

**Step 1: Use key in code**

```typescript
const message = _('New feature added!');
```

**Step 2: Extract keys**

```bash
npm run i18n:extract
```

This automatically adds the key to all language files with the placeholder `__STRING_NOT_TRANSLATED__`.

**Step 3: Translate in each language file**

Edit each `public/locales/{lang}/translation.json`:

```json
{
  "New feature added!": "Neue Funktion hinzugefügt!"  // German
}
```

### Handling Complex Translations

**HTML in translations** (avoid when possible):

```typescript
// BAD: Don't embed HTML
_('<strong>Bold text</strong>')

// GOOD: Use React components
<strong>{_('Bold text')}</strong>
```

**Dynamic keys** (avoid):

```typescript
// BAD: Dynamic keys can't be extracted
const key = isError ? 'Error' : 'Success';
const message = _(key);

// GOOD: Use conditional logic
const message = isError ? _('Error') : _('Success');
```

**Nested objects** (not supported):

```typescript
// BAD: Nested keys not supported
_('settings.general.theme')

// GOOD: Flat keys with descriptive names
_('Settings - General - Theme')
```

### Testing Translations

**Manual testing**:

1. Change language via query string: `?lng=de`
2. Verify all UI elements are translated
3. Check variable interpolation works
4. Test pluralization with different counts

**Automated testing**:

```typescript
// test/i18n.test.ts
import i18n from '@/i18n/i18n';

describe('i18n', () => {
  it('loads German translations', async () => {
    await i18n.changeLanguage('de');
    expect(i18n.t('Book')).toBe('Buch');
  });

  it('interpolates variables', async () => {
    await i18n.changeLanguage('en');
    const result = i18n.t('Hello {{name}}', { name: 'John' });
    expect(result).toBe('Hello John');
  });
});
```

## Components Using Translations

| Component | Location | Translated Elements |
|-----------|----------|---------------------|
| LibraryHeader | `src/app/library/components/LibraryHeader.tsx` | Menu items, buttons |
| Bookshelf | `src/app/library/components/Bookshelf.tsx` | Empty state, tooltips |
| BookCard | `src/app/library/components/BookCard.tsx` | Context menu, metadata |
| SettingsMenu | `src/app/library/components/SettingsMenu.tsx` | All menu items |
| SettingsDialog | `src/app/reader/components/settings/SettingsDialog.tsx` | Tab labels, panels |
| FontPanel | `src/app/reader/components/settings/FontPanel.tsx` | Font options, labels |
| LayoutPanel | `src/app/reader/components/settings/LayoutPanel.tsx` | Layout options |
| ColorPanel | `src/app/reader/components/settings/ColorPanel.tsx` | Color labels |
| MiscPanel | `src/app/reader/components/settings/MiscPanel.tsx` | Misc settings |
| FooterBar | `src/app/reader/components/FooterBar.tsx` | Progress info, tooltips |
| ViewMenu | `src/app/reader/components/ViewMenu.tsx` | View options |
| TOCView | `src/app/reader/components/sidebar/TOCView.tsx` | TOC labels |
| SearchView | `src/app/reader/components/sidebar/SearchView.tsx` | Search options |
| BooknoteView | `src/app/reader/components/sidebar/BooknoteView.tsx` | Note labels |
| Alert | `src/components/Alert.tsx` | Alert messages |

## Entry Points for Common Tasks

| Task | Primary Files | Steps |
|------|---------------|-------|
| **Add new language** | `i18next-scanner.config.js`, `src/i18n/i18n.ts` | Add to config → Create translation file → Test |
| **Add translation key** | Component files, translation JSONs | Use `_()` in code → Run extractor → Translate |
| **Update translations** | `public/locales/{lang}/translation.json` | Edit JSON file → Reload app |
| **Change language** | Browser URL, localStorage | Add `?lng=de` to URL or set in localStorage |
| **Extract new keys** | Terminal | Run `npm run i18n:extract` |
| **Debug missing translations** | Browser DevTools | Check console for `__STRING_NOT_TRANSLATED__` |

## Common Issues and Debugging

### Issue: Translation not appearing

**Symptoms**: UI shows key instead of translation

**Debug steps**:
1. Check translation file exists: `public/locales/{lang}/translation.json`
2. Verify key exists in translation file
3. Check for typos in key name
4. Ensure language is loaded (check browser DevTools console)
5. Verify `i18n.init()` completed successfully

**Common causes**:
- Key missing from translation file
- Typo in key name
- Translation file not loaded (network error)
- Language code mismatch

### Issue: Variable interpolation not working

**Symptoms**: Translation shows `{{variable}}` literally

**Debug steps**:
1. Verify variable is passed in options object: `_('key', { variable: value })`
2. Check variable name matches in translation file
3. Ensure value is not undefined

**Example fix**:

```typescript
// Wrong
_('Hello {{name}}', { username: 'John' })  // Variable name mismatch

// Correct
_('Hello {{name}}', { name: 'John' })
```

### Issue: Fallback language not working

**Symptoms**: UI blank instead of showing English

**Debug steps**:
1. Verify fallback is configured in `i18n.ts`: `fallbackLng: 'en'`
2. Check English translation file exists and is valid JSON
3. Verify key exists in English file
4. Check browser console for errors

### Issue: Scanner not extracting keys

**Symptoms**: New keys not added to translation files

**Debug steps**:
1. Verify function name is `_()` (not `t()` or other)
2. Check file extension is in scanner config: `.ts`, `.tsx`, `.js`, `.jsx`
3. Ensure key is a string literal (not variable)
4. Run scanner with debug flag: `DEBUG=* npm run i18n:extract`

**Common causes**:
- Using different function name
- Dynamic keys (can't be extracted)
- File excluded in scanner config

## Performance Considerations

### Bundle Size

- **Lazy loading**: Only loads translation files for selected language
- **HTTP caching**: Translation files cached in browser
- **Tree shaking**: Unused translations not included in bundle

### Loading Strategy

```typescript
// Disable Suspense for faster initial load
react: {
  useSuspense: false,
}
```

This prevents React from suspending during language changes, providing instant feedback.

### Language Detection Order

```typescript
detection: {
  order: ['querystring', 'localStorage', 'navigator'],
}
```

- **Query string** fastest (no storage read)
- **localStorage** cached (fast)
- **Navigator** slower (browser detection)

## Security Considerations

### XSS Prevention

**React escapes automatically**:

```typescript
interpolation: {
  escapeValue: false,  // React already sanitizes
}
```

Do not disable React's built-in XSS protection.

### Translation Integrity

**Verify translations** before deployment:
- Check for malicious code in translation files
- Validate JSON syntax
- Review contributor changes in PRs

### User-Generated Content

**Never translate user content**:

```typescript
// BAD: Don't translate user input
const userMessage = _('user_message_key');

// GOOD: Only translate UI labels
const label = _('Message from {{username}}', { username: user.name });
```

## Testing i18n

### Manual Testing Checklist

- [ ] All 14 languages load without errors
- [ ] Variable interpolation works
- [ ] Pluralization displays correctly
- [ ] Language persists after page reload
- [ ] Query string language override works
- [ ] Fallback language works for missing keys
- [ ] No `__STRING_NOT_TRANSLATED__` in UI
- [ ] RTL languages (if added) display correctly

### Automated Tests

```typescript
// test/i18n.test.ts
describe('i18n Configuration', () => {
  it('supports all 14 languages', () => {
    const languages = Object.keys(i18n.services.resourceStore.data);
    expect(languages).toHaveLength(14);
  });

  it('falls back to English', async () => {
    await i18n.changeLanguage('invalid');
    expect(i18n.language).toBe('en');
  });
});
```

## Future Enhancements

### Planned Features

1. **RTL Support**: Full right-to-left layout for Arabic, Hebrew
2. **Context-aware translations**: Different translations based on context
3. **Translation memory**: Reuse translations across similar keys
4. **Crowdsourced translations**: Allow community contributions
5. **Translation validation**: Automated checks for missing/incorrect translations
6. **Dynamic language switching**: Change language without page reload
7. **Locale-specific formats**: Dates, numbers, currencies per locale

### Integration Opportunities

1. **Weblate**: Professional translation management platform
2. **Crowdin**: Collaborative translation tool
3. **POEditor**: Localization management service
4. **Translation APIs**: DeepL, Google Translate for initial translations

---

**Last Updated**: Documentation for commits through 76c5f585 (Jan 2025)
**Related Documents**: [feature-settings-system.md](./feature-settings-system.md), [index.md](./index.md)
