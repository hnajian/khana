# RTL (Right-to-Left) and Arabic Language Support

**Feature Category**: Internationalization, Layout, Typography
**Added**: v0.9.19-0.9.20 (February 2025)
**Related Commits**: #500, #504, #519, #535, #551
**Dependencies**: i18next, CSS logical properties

## Overview

The RTL (Right-to-Left) language support enables Readest to properly display and interact with languages that are read from right to left, with **Arabic** being the first fully supported RTL language. This feature includes automatic UI direction switching, mirrored layouts, and proper text alignment for both the application interface and book content.

### Supported RTL Languages

| Language | Code | Status | Added |
|----------|------|--------|-------|
| **Arabic** | `ar` | ✅ Complete | v0.9.15 (#432) |
| Hebrew | `he` | ⚙️ Partial | Awaiting translations |
| Persian (Farsi) | `fa` | ⚙️ Partial | Awaiting translations |
| Urdu | `ur` | ⚙️ Partial | Awaiting translations |
| Dhivehi | `dv` | ⚙️ Partial | Awaiting translations |
| Pashto | `ps` | ⚙️ Partial | Awaiting translations |
| Sindhi | `sd` | ⚙️ Partial | Awaiting translations |
| Yiddish | `yi` | ⚙️ Partial | Awaiting translations |

**Note**: Partial support means the RTL layout logic is in place, but translations are not yet available.

## Architecture

### Key Components

| Component | File | Purpose |
|-----------|------|---------|
| **RTL Detection** | `src/utils/rtl.ts` | Language-based direction detection |
| **Book Direction** | `src/utils/book.ts` | Book language and direction utilities |
| **Layout Controls** | `src/app/reader/components/settings/LayoutPanel.tsx` | UI for writing mode selection |
| **Sidebar RTL** | `src/app/reader/components/sidebar/*` | RTL layout for navigation |
| **Progress Bar RTL** | `src/app/reader/components/FooterBar.tsx` | RTL progress indicators |
| **Notebook RTL** | `src/app/reader/components/notebook/*` | RTL for annotations |

### RTL Detection Logic

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

### Book Direction Detection

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

## Writing Mode Options

### Available Modes

**File**: `src/types/book.ts`

```typescript
export type WritingMode =
  | 'auto'           // Automatic detection from book metadata
  | 'horizontal-tb'  // Horizontal LTR, top-to-bottom
  | 'horizontal-rl'  // Horizontal RTL, top-to-bottom (Arabic default)
  | 'vertical-rl';   // Vertical RTL, right-to-left (CJK)
```

### Mode Selection UI

**File**: `src/app/reader/components/settings/LayoutPanel.tsx` (lines 382-419)

**UI Labels**:
- **Auto**: Uses the document's natural writing direction
- **Horizontal (LTR)**: Horizontal text, left-to-right, top-to-bottom
- **Vertical (RTL)**: Vertical text, right-to-left, top-to-bottom (for CJK)
- **RTL Direction**: Horizontal text, right-to-left, top-to-bottom (for Arabic/Hebrew)


### Configuration Storage

**File**: `src/types/book.ts`

```typescript
export interface BookLayout {
  writingMode: WritingMode;
  rtl: boolean;           // Derived from writingMode
  vertical: boolean;      // Derived from writingMode
  // ... other layout properties
}

export interface ViewSettings {
  layout: BookLayout;
  // ... other view settings
}
```

**Derivation Logic**:
```typescript
const rtl = writingMode === 'horizontal-rl' || writingMode === 'vertical-rl';
const vertical = writingMode === 'vertical-rl';
```

## UI Direction Switching

### Root HTML Element

**File**: `src/app/layout.tsx`

```typescript
export default function RootLayout({ children }: { children: React.ReactNode }) {
  const dir = getDirFromUILanguage();

  return (
    <html lang={getUserLang()} dir={dir}>
      <body>{children}</body>
    </html>
  );
}
```

**Effect**: When Arabic is selected, `<html dir="rtl">` causes all UI elements to mirror automatically.

### CSS Logical Properties

Readest uses CSS logical properties for automatic RTL support:

```css
/* Instead of margin-left/right, use logical properties */
margin-inline-start: 16px;  /* Left in LTR, right in RTL */
margin-inline-end: 16px;    /* Right in LTR, left in RTL */

padding-inline: 12px;       /* Horizontal padding (both sides) */
border-inline-start: 1px;   /* Left border in LTR, right in RTL */
```

**Files Using Logical Properties**:
- `src/styles/globals.css`
- Component-specific CSS modules
- Tailwind utility classes with RTL plugin

## RTL-Specific Components

### 1. Sidebar Navigation

**File**: `src/app/reader/components/sidebar/Sidebar.tsx`

**RTL Behavior** (#504, #519):
- Sidebar position switches sides (left in LTR, right in RTL)
- Tab navigation icons mirror
- TOC indentation reverses
- Search results align to the right
- Bookmark list aligns to the right

**Conditional RTL Application**:
```typescript
// Only apply RTL to sidebar for mandatory RTL languages
const sidebarDir = MANDATORY_RTL_LANGS.includes(uiLang) ? 'rtl' : 'ltr';
```

**Mandatory RTL Languages**: `ar`, `he`, `fa`, `ur`

**Rationale**: Some languages (like Arabic) require strict RTL layout, while others may prefer LTR UI even if the book is RTL.

### 2. Progress Bar

**File**: `src/app/reader/components/FooterBar.tsx` (#498)

**RTL Behavior**:
- Progress bar fills from right to left in RTL books
- Page numbers display in reversed order (e.g., "100 of 1" instead of "1 of 100")
- Navigation buttons swap positions

**Implementation**:
```typescript
const isRTL = bookLayout.rtl;

<div className={`progress-bar ${isRTL ? 'rtl' : 'ltr'}`}>
  {isRTL ? (
    <>{totalPages} {_('of')} {currentPage}</>
  ) : (
    <>{currentPage} {_('of')} {totalPages}</>
  )}
</div>
```

### 3. Notebook and Annotations

**File**: `src/app/reader/components/notebook/NotebookView.tsx` (#535)

**RTL Behavior**:
- Annotation list aligns to the right
- Note text aligns to the right
- Timestamps align to the left (reversed)
- Action buttons swap positions

**Conditional Rendering**:
```typescript
const dir = getDirFromUILanguage();

<div className="notebook-container" dir={dir}>
  {/* Content automatically mirrors in RTL */}
</div>
```

### 4. Go Back/Forward Buttons

**File**: `src/app/reader/components/FooterBar.tsx` (#551)

**RTL Behavior**:
- Button order consistent with reading direction
- In RTL: Forward (→) on left, Back (←) on right
- In LTR: Back (←) on left, Forward (→) on right

**Implementation**:
```typescript
const buttons = isRTL
  ? [forwardButton, backButton]
  : [backButton, forwardButton];
```

## Book Content RTL Support

### EPUB RTL Detection

**File**: `src/libs/document.ts`

**Metadata Detection**:
```typescript
// Read from EPUB metadata
const pageProgressionDirection = opf.metadata['page-progression-direction'];
const isRTL = pageProgressionDirection === 'rtl';

// Alternative: language-based detection
const lang = opf.metadata.language || opf.metadata['dc:language'];
const direction = getDirFromLanguage(lang);
```

**Override Mechanism**:
Users can manually override the detected direction via the Layout Panel.

### CSS Application

**File**: `src/utils/style.ts`

**Generated CSS for RTL Books**:
```css
/* Applied when writingMode === 'horizontal-rl' */
body {
  writing-mode: horizontal-tb;
  direction: rtl;
}

/* Text alignment */
p, div {
  text-align: start;  /* Right in RTL */
}

/* Indentation */
p {
  text-indent: 2em;
  padding-inline-start: 0;  /* No extra padding */
  padding-inline-end: 2em;  /* Indent on the right side */
}

/* Images */
img {
  float: inline-start;  /* Right in RTL */
}
```

### Foliate.js Integration

**File**: `packages/foliate-js/view.js`

**RTL Configuration**:
```javascript
const view = new View({
  rtl: bookLayout.rtl,
  // ... other options
});
```

**Effect**: Foliate.js reading engine:
- Reverses page navigation (swipe left to go forward)
- Mirrors column layout (right column first)
- Adjusts scroll direction

## Typography Considerations

### Font Selection

**Arabic Font Recommendations**:
- **Noto Naskh Arabic** (Google Fonts)
- **Noto Sans Arabic** (modern, clean)
- **Amiri** (traditional, elegant)
- **Scheherazade New** (extended character support)

**Configuration**:
```typescript
// User can select Arabic fonts in FontPanel
const arabicFonts = [
  'Noto Naskh Arabic',
  'Noto Sans Arabic',
  'Amiri',
  'Scheherazade New',
  'Traditional Arabic',
  'Simplified Arabic',
];
```

### Character Shaping

**Ligatures and Contextual Forms**:
```css
/* Enable proper Arabic character shaping */
font-feature-settings:
  "liga" 1,  /* Ligatures */
  "calt" 1,  /* Contextual alternates */
  "clig" 1;  /* Contextual ligatures */
```

**Files**: Applied in `src/utils/style.ts` for Arabic content.

### Bidirectional Text (Bidi)

**Mixed LTR/RTL Handling**:
```html
<!-- Explicit bidi control -->
<span dir="rtl">النص العربي</span>
<span dir="ltr">English text</span>

<!-- Inline isolation -->
<bdi>مختلط mixed نص</bdi>
```

**Browser Support**: Modern browsers handle Unicode bidirectional algorithm automatically.

## AI Agent Modification Guidelines

### Adding a New RTL Language

**Step 1: Add language to RTL set**

Edit `src/utils/rtl.ts`:
```typescript
export const RTL_LANGUAGES = new Set([
  'ar', 'he', 'fa', 'ur', 'dv', 'ps', 'sd', 'yi',
  'ku',  // Add Kurdish
]);
```

**Step 2: Add translations**

Create `public/locales/ku/translation.json` following Arabic translation pattern.

**Step 3: (Optional) Add to mandatory RTL languages**

Edit `src/utils/rtl.ts`:
```typescript
export const MANDATORY_RTL_LANGS = ['ar', 'he', 'fa', 'ur', 'ku'];
```

**Step 4: Test**

```bash
# Change language to Kurdish
http://localhost:3000?lng=ku

# Verify:
# - UI mirrors correctly
# - Sidebar on the right
# - Text aligns to the right
# - Progress bar fills right-to-left
```

### Customizing RTL Behavior for a Component

**Example: Custom RTL Menu**

```typescript
import { getDirFromUILanguage } from '@/utils/rtl';

function CustomMenu() {
  const dir = getDirFromUILanguage();
  const isRTL = dir === 'rtl';

  return (
    <div className={`menu ${isRTL ? 'rtl-menu' : 'ltr-menu'}`} dir={dir}>
      {/* Menu items */}
    </div>
  );
}
```

**CSS**:
```css
.menu {
  display: flex;
}

.rtl-menu {
  flex-direction: row-reverse;
}

.ltr-menu {
  flex-direction: row;
}
```

### Debugging RTL Issues

**Common Issues**:

1. **Text not aligning correctly**
   - Check `dir="rtl"` is set on parent element
   - Verify `text-align: start` instead of `left`
   - Use logical properties (`inline-start` vs `left`)

2. **Icons not mirroring**
   - Apply `transform: scaleX(-1)` for directional icons
   - Use bidirectional icon variants (e.g., `chevron-start` instead of `chevron-left`)

3. **Layout breaking in RTL**
   - Check for hardcoded `left/right` properties
   - Replace with `inline-start/inline-end`
   - Test with `dir="rtl"` on `<html>` element

4. **Progress bar filling wrong direction**
   - Verify `rtl` flag is passed to component
   - Check CSS `flex-direction` is reversed
   - Ensure `transform` is applied if using absolute positioning

**Debugging Tools**:
```typescript
// Log current direction
console.log('UI Direction:', getDirFromUILanguage());
console.log('Book Direction:', getBookDirection(book));

// Force RTL for testing
<html dir="rtl" lang="ar">
```

## Entry Points for Common Tasks

| Task | Primary Files | Steps |
|------|---------------|-------|
| **Add new RTL language** | `src/utils/rtl.ts`, `public/locales/{lang}/` | Update RTL_LANGUAGES set → Add translations → Test |
| **Customize RTL layout** | Component files, `src/utils/rtl.ts` | Import direction utilities → Apply conditional classes → Test |
| **Fix RTL alignment** | Component CSS, `src/styles/globals.css` | Replace fixed properties with logical → Test in RTL mode |
| **Override book direction** | `src/app/reader/components/settings/LayoutPanel.tsx` | Select writing mode → Apply to view settings → Persist |
| **Debug RTL issues** | Browser DevTools, component files | Inspect dir attribute → Check CSS logical properties → Verify flexbox direction |

## Testing RTL Support

### Manual Testing Checklist

- [ ] Switch UI language to Arabic (`?lng=ar`)
- [ ] Verify UI mirrors completely
- [ ] Open an Arabic EPUB book
- [ ] Confirm book content displays RTL
- [ ] Check progress bar fills right-to-left
- [ ] Verify sidebar appears on the right
- [ ] Test navigation buttons are swapped
- [ ] Check annotations align to the right
- [ ] Verify TOC indentation reverses
- [ ] Test search results align properly
- [ ] Confirm book cards align to the right in library
- [ ] Verify modal dialogs mirror correctly

### Automated Tests

```typescript
// test/rtl.test.ts
import { getDirFromLanguage, getDirFromUILanguage } from '@/utils/rtl';

describe('RTL Detection', () => {
  it('detects Arabic as RTL', () => {
    expect(getDirFromLanguage('ar')).toBe('rtl');
    expect(getDirFromLanguage('ar-SA')).toBe('rtl');
  });

  it('detects English as LTR', () => {
    expect(getDirFromLanguage('en')).toBe('ltr');
  });

  it('handles unknown languages', () => {
    expect(getDirFromLanguage('')).toBe('auto');
    expect(getDirFromLanguage('xyz')).toBe('ltr');
  });
});
```

## Performance Considerations

### CSS Logical Properties

**Benefit**: Browser-native RTL support without JavaScript

```css
/* Fast: Browser handles RTL automatically */
margin-inline-start: 16px;

/* Slow: JavaScript must calculate RTL */
const margin = isRTL ? 'marginRight' : 'marginLeft';
element.style[margin] = '16px';
```

**Recommendation**: Prefer CSS logical properties over JS-based direction calculation.

### Component Re-rendering

**Avoid unnecessary re-renders when switching direction**:

```typescript
// Bad: Re-renders entire component on language change
const dir = getDirFromUILanguage();

// Good: Memoize direction
const dir = useMemo(() => getDirFromUILanguage(), [userLang]);
```

### Font Loading

**Optimize Arabic font loading**:

```typescript
// Preload Arabic fonts when language is Arabic
if (getUserLang() === 'ar') {
  const link = document.createElement('link');
  link.rel = 'preload';
  link.as = 'font';
  link.href = '/fonts/NotoNaskhArabic.woff2';
  link.type = 'font/woff2';
  link.crossOrigin = 'anonymous';
  document.head.appendChild(link);
}
```

## Accessibility Considerations

### Screen Reader Support

**Proper `dir` and `lang` attributes**:
```html
<html dir="rtl" lang="ar">
  <!-- Screen readers announce Arabic content correctly -->
</html>
```

### Keyboard Navigation

**RTL Keyboard Mapping**:
- Arrow Left → Next page (reversed in RTL)
- Arrow Right → Previous page (reversed in RTL)
- Home → End of book (reversed in RTL)
- End → Start of book (reversed in RTL)

**Implementation**:
```typescript
const handleKeyPress = (e: KeyboardEvent) => {
  const isRTL = bookLayout.rtl;

  if (e.key === 'ArrowLeft') {
    isRTL ? goToPreviousPage() : goToNextPage();
  } else if (e.key === 'ArrowRight') {
    isRTL ? goToNextPage() : goToPreviousPage();
  }
};
```

### Focus Order

**Tab navigation in RTL**:
- Focus should move right-to-left in RTL layouts
- Browser handles this automatically when `dir="rtl"` is set
- Verify tab order matches visual order in RTL

## Future Enhancements

### Planned Features

1. **Additional RTL Languages**: Hebrew, Persian, Urdu translations
2. **Improved Bidi Algorithm**: Better mixed LTR/RTL text handling
3. **RTL Footnotes**: Proper RTL positioning for footnote popovers
4. **RTL Annotations**: Enhanced RTL support for highlighting tools
5. **Arabic Typography**: Advanced shaping and diacritics support
6. **RTL PDF Support**: Proper RTL rendering for PDF documents

### Known Limitations

1. **PDF RTL**: PDF books may not respect RTL direction setting
2. **Embedded LTR Text**: Some EPUB books with mixed content may display incorrectly
3. **Custom CSS**: User custom CSS may conflict with RTL layout
4. **Third-party Widgets**: Some embedded widgets may not support RTL

## Related Documentation

- **[Internationalization (i18n)](./index.md)** - Translation and language support
- **[Settings System](../settings-system/index.md)** - Writing mode configuration
- **[Document Reading Engine](../document-reading-engine/index.md)** - Book direction detection

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #432, #451, #498, #500, #504, #512, #519, #535, #551
