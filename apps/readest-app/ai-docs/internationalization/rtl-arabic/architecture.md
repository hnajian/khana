# Architecture

## Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **RTL Detection** | `src/utils/rtl.ts` | Language-based direction detection |
| **Book Direction** | `src/utils/book.ts` | Book language and direction utilities |
| **Layout Controls** | `src/app/reader/components/settings/LayoutPanel.tsx` | UI for writing mode selection |
| **Sidebar RTL** | `src/app/reader/components/sidebar/*` | RTL layout for navigation |
| **Progress Bar RTL** | `src/app/reader/components/FooterBar.tsx` | RTL progress indicators |
| **Notebook RTL** | `src/app/reader/components/notebook/*` | RTL for annotations |

## RTL Detection Logic

**File**: `src/utils/rtl.ts`

```typescript
export const RTL_LANGUAGES = new Set([
  'ar',  // Arabic
  'he',  // Hebrew
  'fa',  // Persian
  'ur',  // Urdu
  'dv',  // Dhivehi
  'ps',  // Pashto
  'sd',  // Sindhi
  'yi',  // Yiddish
]);

export const getDirFromLanguage = (lang: string): 'rtl' | 'ltr' | 'auto' => {
  if (!lang) return 'auto';
  const primaryLang = lang.split('-')[0]!.toLowerCase();
  return RTL_LANGUAGES.has(primaryLang) ? 'rtl' : 'ltr';
};

export const getDirFromUILanguage = (): 'rtl' | 'ltr' => {
  const lang = getUserLang();
  const dir = getDirFromLanguage(lang);
  return dir === 'auto' ? 'ltr' : dir;
};
```

**Behavior**:
- Extracts primary language code (e.g., `ar` from `ar-EG`)
- Checks against RTL language set
- Returns `'rtl'` for RTL languages, `'ltr'` for LTR, or `'auto'` for unknown

## Book Direction Detection

**File**: `src/utils/book.ts`

```typescript
export const getBookLangCode = (lang: string | string[] | undefined): string => {
  try {
    const bookLang = typeof lang === 'string' ? lang : lang?.[0];
    return bookLang ? bookLang.split('-')[0]! : '';
  } catch {
    return '';
  }
};

export const getBookDirection = (book: Book): 'rtl' | 'ltr' | 'auto' => {
  const langCode = getBookLangCode(book.metadata.language);
  return getDirFromLanguage(langCode);
};
```

**Usage**:
```typescript
const book = await loadBook();
const direction = getBookDirection(book);
// Apply direction to reader layout
```
