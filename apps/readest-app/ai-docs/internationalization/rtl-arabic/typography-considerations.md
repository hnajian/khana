# Typography Considerations

## Font Selection

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

## Character Shaping

**Ligatures and Contextual Forms**:
```css
/* Enable proper Arabic character shaping */
font-feature-settings:
  "liga" 1,  /* Ligatures */
  "calt" 1,  /* Contextual alternates */
  "clig" 1;  /* Contextual ligatures */
```

**Files**: Applied in `src/utils/style.ts` for Arabic content.

## Bidirectional Text (Bidi)

**Mixed LTR/RTL Handling**:
```html
<!-- Explicit bidi control -->
<span dir="rtl">النص العربي</span>
<span dir="ltr">English text</span>

<!-- Inline isolation -->
<bdi>مختلط mixed نص</bdi>
```

**Browser Support**: Modern browsers handle Unicode bidirectional algorithm automatically.
