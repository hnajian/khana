# Performance Considerations

## CSS Logical Properties

**Benefit**: Browser-native RTL support without JavaScript

```css
/* Fast: Browser handles RTL automatically */
margin-inline-start: 16px;

/* Slow: JavaScript must calculate RTL */
const margin = isRTL ? 'marginRight' : 'marginLeft';
element.style[margin] = '16px';
```

**Recommendation**: Prefer CSS logical properties over JS-based direction calculation.

## Component Re-rendering

**Avoid unnecessary re-renders when switching direction**:

```typescript
// Bad: Re-renders entire component on language change
const dir = getDirFromUILanguage();

// Good: Memoize direction
const dir = useMemo(() => getDirFromUILanguage(), [userLang]);
```

## Font Loading

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
