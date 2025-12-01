# UI Direction Switching

## Root HTML Element

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

## CSS Logical Properties

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
