# Sub-Feature: Theme Editor

## Overview

The Theme Editor allows users to create custom color themes with a primary color input and theme customization options. Added in v0.9.37 (commit #970), it enables full control over the reading interface color scheme beyond the built-in themes.

## Key Components

### Primary Files

- **`src/app/reader/components/settings/ColorPanel.tsx`** - Theme editor UI with primary color input
- **`src/store/settingsStore.ts`** - Custom theme storage and sync
- **`src/types/settings.ts`** - Theme type definitions

## Architecture

### Theme Structure

**Custom Theme Definition**:
```typescript
interface CustomTheme {
  name: string;
  primaryColor: string;      // User-defined primary color
  backgroundColor: string;
  foregroundColor: string;
  linkColor?: string;
  // ... other color properties
}
```

### Primary Color Input

**Feature** (commit 9303ec8c):
- Color picker input for primary brand color
- Real-time preview of theme with selected color
- Automatic calculation of complementary colors
- Integration with existing theme system

**UI Component** (`ColorPanel.tsx`):
- HTML5 color input: `<input type="color" />`
- Hex color value display
- Theme preview section
- Save/Reset buttons

### Theme Synchronization

**Local Storage Sync** (commit ee93885c):

Issue: Custom themes were not syncing between global settings and localStorage
Solution: Implemented bidirectional sync

```typescript
// Sync custom themes from global settings to localStorage
useEffect(() => {
  if (customThemes) {
    localStorage.setItem('customThemes', JSON.stringify(customThemes));
  }
}, [customThemes]);

// Load from localStorage on init
useEffect(() => {
  const stored = localStorage.getItem('customThemes');
  if (stored) {
    setCustomThemes(JSON.parse(stored));
  }
}, []);
```

### Spell Check Disabled

**Fix** (commit 048867e5):
- Disabled browser spell check on color input fields
- Prevents red squiggly lines on hex color codes
- Attribute: `spellCheck={false}` on color inputs

## AI Agent Modification Guidelines

### Adding Color Properties to Theme

To add new customizable colors to themes:

1. **Update theme type** in `src/types/settings.ts`:
   ```typescript
   interface CustomTheme {
     // ... existing
     accentColor?: string;      // New property
     highlightColor?: string;   // New property
   }
   ```

2. **Add color input** in `ColorPanel.tsx`:
   ```typescript
   <label>
     Accent Color:
     <input
       type="color"
       value={theme.accentColor || '#0066cc'}
       onChange={(e) => updateTheme({ accentColor: e.target.value })}
       spellCheck={false}
     />
   </label>
   ```

3. **Apply color** in reader CSS:
   ```css
   .reader-accent {
     background-color: var(--accent-color);
   }
   ```

### Implementing Theme Presets

To add predefined theme presets:

1. **Define presets** in `ColorPanel.tsx`:
   ```typescript
   const THEME_PRESETS: Record<string, Partial<CustomTheme>> = {
     'Sepia': {
       backgroundColor: '#f4ecd8',
       foregroundColor: '#5c4a3a',
       primaryColor: '#8b7355'
     },
     'Night': {
       backgroundColor: '#1a1a1a',
       foregroundColor: '#e0e0e0',
       primaryColor: '#4a90e2'
     },
     'Forest': {
       backgroundColor: '#e8f5e9',
       foregroundColor: '#1b5e20',
       primaryColor: '#4caf50'
     }
   };
   ```

2. **Add preset selector**:
   ```typescript
   <select onChange={(e) => applyPreset(e.target.value)}>
     <option value="">Custom</option>
     {Object.keys(THEME_PRESETS).map(name => (
       <option key={name} value={name}>{name}</option>
     ))}
   </select>
   ```

3. **Apply preset function**:
   ```typescript
   const applyPreset = (presetName: string) => {
     const preset = THEME_PRESETS[presetName];
     if (preset) {
       updateTheme(preset);
     }
   };
   ```

### Enhancing Color Picker

To improve the color picker UX:

1. **Add color palette suggestions**:
   ```typescript
   const SUGGESTED_COLORS = [
     '#2196f3', '#4caf50', '#ff9800', '#e91e63',
     '#9c27b0', '#00bcd4', '#ff5722', '#795548'
   ];

   return (
     <div className="color-suggestions">
       {SUGGESTED_COLORS.map(color => (
         <button
           key={color}
           style={{ backgroundColor: color }}
           onClick={() => updateTheme({ primaryColor: color })}
         />
       ))}
     </div>
   );
   ```

2. **Add color contrast validation**:
   ```typescript
   function validateColorContrast(
     bgColor: string,
     fgColor: string
   ): boolean {
     const contrast = calculateContrastRatio(bgColor, fgColor);
     return contrast >= 4.5; // WCAG AA standard
   }

   // Show warning if contrast is low
   {!validateColorContrast(backgroundColor, foregroundColor) && (
     <div className="warning">
       Warning: Low contrast ratio. Text may be hard to read.
     </div>
   )}
   ```

### Implementing Theme Import/Export

To allow users to share themes:

1. **Export theme function**:
   ```typescript
   const exportTheme = (theme: CustomTheme) => {
     const json = JSON.stringify(theme, null, 2);
     const blob = new Blob([json], { type: 'application/json' });
     const url = URL.createObjectURL(blob);

     const a = document.createElement('a');
     a.href = url;
     a.download = `${theme.name}.json`;
     a.click();
   };
   ```

2. **Import theme function**:
   ```typescript
   const importTheme = async (file: File) => {
     const text = await file.text();
     const theme = JSON.parse(text) as CustomTheme;

     // Validate theme structure
     if (!theme.name || !theme.primaryColor) {
       throw new Error('Invalid theme file');
     }

     addCustomTheme(theme);
   };
   ```

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add color property | `src/types/settings.ts` | `ColorPanel.tsx` |
| Add theme presets | `ColorPanel.tsx` | `settingsStore.ts` |
| Improve color picker | `ColorPanel.tsx` | - |
| Theme import/export | `ColorPanel.tsx` | `settingsStore.ts` |
| Sync custom themes | `settingsStore.ts` | `ColorPanel.tsx` |

## Common Issues and Debugging

### Problem: Theme not persisting

- Verify localStorage is enabled
- Check localStorage quota limits
- Ensure `customThemes` key exists in localStorage
- Verify JSON serialization is working

### Problem: Color input showing spell check

- Ensure `spellCheck={false}` attribute is set
- Check browser settings for forced spell check
- Verify input type is `"color"`

### Problem: Theme not applying to reader

- Check if theme is selected in settings
- Verify CSS variables are set correctly
- Inspect element styles in DevTools
- Ensure theme sync completed before reader init

## Dependencies

- **HTML5 Color Input**: Native browser color picker
- **localStorage**: Custom theme persistence
- **Zustand**: Settings state management

## Performance Considerations

- **Debounce color changes**: Don't update on every color picker drag
- **Lazy theme application**: Only apply when user clicks "Apply"
- **Cache theme calculations**: Pre-compute theme variations

---

**Implemented in commits:**
- 9303ec8c: feat: add primary color input in theme editor
- ee93885c: fix: sync custom themes from global settings to localstorage
- 048867e5: settings: disable spell check in color input
