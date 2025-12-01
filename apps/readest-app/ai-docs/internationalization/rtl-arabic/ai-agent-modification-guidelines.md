# AI Agent Modification Guidelines

## Adding a New RTL Language

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

## Customizing RTL Behavior for a Component

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

## Debugging RTL Issues

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
