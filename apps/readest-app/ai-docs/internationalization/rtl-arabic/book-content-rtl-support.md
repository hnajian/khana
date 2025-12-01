# Book Content RTL Support

## EPUB RTL Detection

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

## CSS Application

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

## Foliate.js Integration

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
