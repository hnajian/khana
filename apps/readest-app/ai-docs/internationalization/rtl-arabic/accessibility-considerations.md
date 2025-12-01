# Accessibility Considerations

## Screen Reader Support

**Proper `dir` and `lang` attributes**:
```html
<html dir="rtl" lang="ar">
  <!-- Screen readers announce Arabic content correctly -->
</html>
```

## Keyboard Navigation

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

## Focus Order

**Tab navigation in RTL**:
- Focus should move right-to-left in RTL layouts
- Browser handles this automatically when `dir="rtl"` is set
- Verify tab order matches visual order in RTL
