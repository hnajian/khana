# Feature: Settings System

## Overview

The Settings System in Readest implements a three-tier hierarchy: **Global Settings** < **Book Settings** < **View Settings**. This allows users to set default preferences globally while overriding them per-book or per-view. At commit `d757555f`, settings cover fonts, colors, layout, and miscellaneous reading preferences.


## Sub-Features

- **[Custom CSS Editor](./custom-css-editor.md)** - Enhanced CSS editing with validation and draft mode (Jan 2025)
- **[Writing Mode Switch](./writing-mode-switch.md)** - Manual override for vertical/horizontal text direction (Jan 2025)
- **[Screen Wake Lock](./screen-wake-lock.md)** - Prevent screen sleep during reading (v0.9.13)

## Key Components

### Primary Files

- **`src/store/settingsStore.ts`** - Global system settings store
- **`src/app/reader/components/settings/SettingsDialog.tsx`** - Main settings modal
- **`src/app/reader/components/settings/FontPanel.tsx`** - Font configuration
- **`src/app/reader/components/settings/ColorPanel.tsx`** - Color and theme settings
- **`src/app/reader/components/settings/LayoutPanel.tsx`** - Layout and pagination settings
- **`src/app/reader/components/settings/MiscPanel.tsx`** - Miscellaneous settings (custom styles, etc.)

### Related Files

- **`src/types/settings.ts`** - TypeScript type definitions for settings
- **`src/types/book.ts`** - `ViewSettings`, `BookConfig` types
- **`src/services/appService.ts`** - `saveSettings()`, `loadSettings()` methods
- **`src/store/readerStore.ts`** - View-specific settings management
- **`src/store/bookDataStore.ts`** - Book-specific configuration storage

## Architecture

### Settings Hierarchy

```
Global Settings (SystemSettings)
└── Book Settings (BookConfig.viewSettings)
    └── View Settings (ViewState.viewSettings)
```

**Inheritance Rules**:
1. **View settings** override **book settings**
2. **Book settings** override **global settings**
3. **Global settings** are the fallback defaults

**Persistence**:
- **Global settings**: Saved in app configuration file via `appService.saveSettings()`
- **Book settings**: Stored in book config file alongside reading progress
- **View settings**: Stored in memory, primary view settings persist to book config

### Global Settings (`SystemSettings`)

Defined in `src/types/settings.ts`:

```typescript
interface SystemSettings {
  // Font settings
  fontFamily: string;
  fontSize: number;
  lineHeight: number;

  // Color settings
  theme: 'light' | 'dark' | 'auto';
  backgroundColor: string;
  foregroundColor: string;

  // Layout settings
  layout: 'paginated' | 'scrolled';
  columns: number;
  maxColumnWidth: number;
  gap: number;
  margin: number;

  // Misc
  customStyles: string;  // User CSS
  // ...
}
```

**Management**:
- Stored in `settingsStore` (Zustand)
- Loaded on app start via `appService.loadSettings()`
- Saved via `settingsStore.saveSettings(envConfig, settings)`

### Book Settings (`BookConfig.viewSettings`)

Part of `BookConfig` in `src/types/book.ts`:

```typescript
interface BookConfig {
  location: string;           // Last reading position
  progress: [number, number]; // [current, total] pages
  viewSettings: ViewSettings; // Book-specific overrides
  lastUpdated: number;
  // ...
}
```

**Behavior**:
- When opening a book, if no book settings exist, global settings are copied
- Book settings persist across sessions
- Updated when primary view settings change

### View Settings (`ViewSettings`)

Defined in `src/types/book.ts`:

```typescript
interface ViewSettings {
  fontFamily?: string;
  fontSize?: number;
  // ... (subset of SystemSettings)
  // Only overridden values are stored
}
```

**Behavior**:
- Each view in a multi-view layout can have independent settings
- Primary view settings are saved to book config
- Non-primary views exist only in memory during session

### Settings Dialog

**UI Structure** (`SettingsDialog.tsx`):
- Modal dialog with tabbed panels
- **Font Tab**: Font family, size, line height
- **Color Tab**: Theme, background/foreground colors
- **Layout Tab**: Layout mode, columns, margins, spacing
- **Misc Tab**: Custom CSS, advanced options

**Dialog Context**:
- Can be opened in "global" or "book/view" mode
- Mode controlled by `settingsStore.isFontLayoutSettingsGlobal`
- Global mode: Changes affect all books
- Book/view mode: Changes affect current book/view only

## AI Agent Modification Guidelines

### Adding a New Setting

To add a new setting (e.g., "justify text"):

1. **Update type definitions** (`src/types/settings.ts`):
   ```typescript
   interface SystemSettings {
     // ... existing settings
     textAlign: 'left' | 'justify' | 'right';
   }
   ```

2. **Update `ViewSettings`** in `src/types/book.ts`:
   ```typescript
   interface ViewSettings {
     // ... existing settings
     textAlign?: 'left' | 'justify' | 'right';
   }
   ```

3. **Add default value** in `appService.ts` or settings initialization:
   ```typescript
   const defaultSettings: SystemSettings = {
     // ... existing defaults
     textAlign: 'left',
   };
   ```

4. **Add UI control** in appropriate panel (e.g., `LayoutPanel.tsx`):
   ```typescript
   <select
     value={settings.textAlign}
     onChange={(e) => updateSetting('textAlign', e.target.value)}
   >
     <option value="left">Left</option>
     <option value="justify">Justify</option>
     <option value="right">Right</option>
   </select>
   ```

5. **Apply setting** in `FoliateViewer.tsx`:
   ```typescript
   useEffect(() => {
     if (view) {
       view.setStyles({
         textAlign: viewSettings.textAlign || globalSettings.textAlign,
       });
     }
   }, [viewSettings.textAlign, globalSettings.textAlign]);
   ```

### Modifying Setting Persistence

To change how settings are saved/loaded:

1. **Edit `appService.ts`**:
   - Modify `saveSettings(settings: SystemSettings)`
   - Modify `loadSettings(): SystemSettings`
   - For native app: File-based storage in `nativeAppService.ts`

2. **Add migration logic** for setting schema changes:
   ```typescript
   async loadSettings(): Promise<SystemSettings> {
     const saved = await this.readSettingsFile();
     const migrated = this.migrateSettings(saved);
     return migrated;
   }

   private migrateSettings(old: any): SystemSettings {
     // Handle old setting formats
     if (old.version < 2) {
       old.textAlign = 'left'; // Add new default
     }
     return old as SystemSettings;
   }
   ```

### Creating Setting Presets

To add preset configurations (e.g., "Reading Modes"):

1. **Define presets** in `settingsStore.ts`:
   ```typescript
   const PRESETS = {
     comfortable: {
       fontSize: 18,
       lineHeight: 1.6,
       maxColumnWidth: 700,
     },
     compact: {
       fontSize: 14,
       lineHeight: 1.4,
       maxColumnWidth: 600,
     },
   };
   ```

2. **Add preset selector** in settings dialog:
   ```typescript
   <select onChange={(e) => applyPreset(PRESETS[e.target.value])}>
     <option value="comfortable">Comfortable</option>
     <option value="compact">Compact</option>
   </select>
   ```

3. **Apply preset** function:
   ```typescript
   const applyPreset = (preset: Partial<SystemSettings>) => {
     settingsStore.setSettings({ ...settings, ...preset });
   };
   ```

### Implementing Setting Validation

To validate setting values:

1. **Add validation functions**:
   ```typescript
   // In settingsStore.ts or utils
   function validateFontSize(size: number): number {
     return Math.max(8, Math.min(72, size)); // Clamp between 8-72
   }

   function validateColor(color: string): string {
     // Validate hex color format
     return /^#[0-9A-F]{6}$/i.test(color) ? color : '#000000';
   }
   ```

2. **Apply validation** in `setSettings`:
   ```typescript
   setSettings: (settings: SystemSettings) => {
     const validated = {
       ...settings,
       fontSize: validateFontSize(settings.fontSize),
       foregroundColor: validateColor(settings.foregroundColor),
     };
     set({ settings: validated });
   }
   ```

### Adding Per-Book Setting Overrides

To improve book-specific settings:

1. **Update UI** to show which settings are overridden:
   ```typescript
   <div>
     Font Size: {viewSettings.fontSize ?? globalSettings.fontSize}
     {viewSettings.fontSize && <span>(overridden)</span>}
   </div>
   ```

2. **Add reset button** to clear override:
   ```typescript
   <button onClick={() => {
     const newSettings = { ...viewSettings };
     delete newSettings.fontSize;
     updateViewSettings(newSettings);
   }}>
     Reset to Global
   </button>
   ```

### Synchronizing Settings with Foliate-js

To ensure settings are properly applied to the reader:

1. **In `FoliateViewer.tsx`**, track settings changes:
   ```typescript
   useEffect(() => {
     if (!view) return;

     const effectiveSettings = {
       ...globalSettings,
       ...viewSettings, // Override with view-specific
     };

     view.setStyles({
       fontFamily: effectiveSettings.fontFamily,
       fontSize: `${effectiveSettings.fontSize}px`,
       lineHeight: effectiveSettings.lineHeight,
       // ... other settings
     });
   }, [view, globalSettings, viewSettings]);
   ```

2. **Handle setting application timing**:
   - Apply settings after view is initialized
   - Re-apply when switching between views
   - Debounce rapid setting changes

### Adding Import/Export for Settings

To allow users to backup/share settings:

1. **Add export function**:
   ```typescript
   // In settingsStore.ts
   exportSettings: () => {
     const json = JSON.stringify(get().settings, null, 2);
     return new Blob([json], { type: 'application/json' });
   }
   ```

2. **Add import function**:
   ```typescript
   importSettings: async (file: File) => {
     const text = await file.text();
     const imported = JSON.parse(text) as SystemSettings;
     const validated = validateSettings(imported);
     set({ settings: validated });
   }
   ```

3. **Add UI buttons** in settings dialog

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add new setting | `src/types/settings.ts` | Panel components, `FoliateViewer.tsx` |
| Modify persistence | `appService.ts` | `nativeAppService.ts` |
| Change UI | Panel components | `SettingsDialog.tsx` |
| Add presets | `settingsStore.ts` | Settings dialog |
| Setting validation | `settingsStore.ts` | Type definitions |
| Import/export | `settingsStore.ts` | Settings dialog, file utils |

## Common Issues and Debugging

### Problem: Setting changes not applying

- Check if setting is being propagated to `viewSettings` or `globalSettings`
- Verify `FoliateViewer` useEffect dependencies include the setting
- Check if view is initialized before applying settings
- Inspect foliate-js API for correct method to apply setting

### Problem: Settings not persisting

- Verify `saveSettings()` is being called after changes
- Check `appService` file write permissions (native app)
- Verify book config is being saved with `lastUpdated` timestamp
- Check for errors in console during save

### Problem: Book settings overriding global incorrectly

- Check inheritance logic in `readerStore.initViewState()`
- Verify `ViewSettings` only contains overridden values
- Check if settings are being merged correctly before application

### Problem: Settings dialog not showing current values

- Verify dialog is reading from correct store (global vs. view)
- Check `isFontLayoutSettingsGlobal` flag in `settingsStore`
- Ensure dialog state syncs with store changes

## Dependencies

- **zustand**: State management for settings stores
- **Tauri APIs**: File system access for settings persistence (native)
- **foliate-js**: Reader view that applies settings

## Performance Considerations

- **Debounce saves**: Don't save on every slider movement
- **Batch updates**: Apply multiple setting changes in one operation
- **Lazy load panels**: Only render active settings panel
- **Memoize computations**: Use `useMemo` for derived settings values

---

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Invert Image Color in Dark Mode (v0.9.48, #1223)

**Feature**: Option to automatically invert image colors when using dark mode for better readability.

**Implementation**:
- CSS filter applied to images in dark mode
- User toggle in settings panel
- Per-book preference saved in BookConfig

**Settings Location**: Color Panel > "Invert images in dark mode"

**CSS Applied**:
```css
.dark-mode img {
  filter: invert(1) hue-rotate(180deg);
}
```

**Use Cases**:
- Reading technical books with diagrams
- Comics/manga with dark backgrounds
- Books with white-background screenshots

**File**: `src/app/reader/components/settings/ColorPanel.tsx`

### Opt-Out Telemetry (v0.9.48, #1236)

**Feature**: Privacy option to disable usage telemetry and analytics.

**Settings**:
- Toggle in Misc Panel > "Send anonymous usage data"
- Default: Enabled (can be disabled)
- Preference persisted globally

**What's Tracked (when enabled)**:
- Feature usage statistics
- Performance metrics
- Crash reports
- No personal information or reading content

**Implementation** (`src/utils/analytics.ts`):
```typescript
const sendAnalytics = (event: string, data: any) => {
  const telemetryEnabled = settingsStore.getState().telemetryEnabled;

  if (!telemetryEnabled) {
    return; // No tracking when disabled
  }

  // Send to PostHog/analytics service
  posthog.capture(event, data);
};
```

**File**: `src/store/settingsStore.ts`

### Enable JavaScript in EPUB (v0.9.52, #1295)

**Feature**: Security option to allow JavaScript execution in EPUB files for interactive content.

**Settings Location**: Settings > Control Panel > Security section > "Allow JavaScript"

**Default**: Disabled (for security reasons)

**Security Warning**: UI displays "Enable only if you trust the file" warning message

**Implementation** (`src/app/reader/components/settings/ControlPanel.tsx`):
```typescript
<div className="form-control">
  <label className="label cursor-pointer">
    <span className="label-text">{t('Allow JavaScript')}</span>
    <input
      type="checkbox"
      className="toggle"
      checked={viewSettings.allowJavaScript}
      onChange={(e) => updateViewSettings({ allowJavaScript: e.target.checked })}
    />
  </label>
  <span className="text-xs text-warning opacity-70">
    {t('Enable only if you trust the file.')}
  </span>
</div>
```

**Applied to Reader** (`src/libs/document.ts`):
```typescript
// Create EPUB view with JavaScript enabled/disabled
const epubView = await makeEPUBView(file, {
  allowScript: viewSettings.allowJavaScript || false,
  // ... other options
});
```

**Use Cases**:
- Interactive educational books with quizzes
- Enhanced EPUB3 content with JavaScript animations
- Technical documentation with live code examples
- Books with embedded widgets or calculators

**Security Considerations**:
- Disabled by default to prevent malicious scripts
- Per-book setting (not global)
- User must explicitly enable for each book
- Warning message reminds users of security implications

**Files**:
- `src/app/reader/components/settings/ControlPanel.tsx`
- `src/libs/document.ts`
- `src/types/book.ts` (ViewSettings.allowJavaScript)
- `packages/foliate-js/epub.js`

### TOC Sort by Page Number (v0.9.51, #1308)

**Feature**: Option to sort Table of Contents by page number instead of hierarchical structure.

**Settings Location**: Sidebar > TOC tab > Sort options

**Modes**:
- **Hierarchical** (default): Show TOC as nested structure
- **By Page**: Flatten and sort entries by page number

**Implementation** (`src/app/reader/components/sidebar/TOCView.tsx`):
```typescript
const sortedTOC = useMemo(() => {
  if (sortByPage) {
    return [...tocItems].sort((a, b) => a.pageNum - b.pageNum);
  }
  return tocItems; // Keep original hierarchy
}, [tocItems, sortByPage]);
```

**Use Cases**:
- Quickly finding a specific page reference
- Linear reading without nested sections
- Reference books with non-hierarchical structure

**File**: `src/app/reader/components/sidebar/TOCView.tsx`

### Remaining Time Display (v0.9.52-0.9.61, #1326, #1478)

**Feature**: Display estimated reading time remaining in current chapter or entire book.

**Settings Options**:
1. **Remaining Pages in Chapter** (#1478)
   - Shows: "23 pages left in chapter"
   - Location: Footer status bar
   - Calculation: Total chapter pages - current page

2. **Remaining Minutes in Chapter** (#1326)
   - Shows: "15 min left in chapter"
   - Calculation: (Remaining pages × average read time per page)
   - Adapts to user's reading speed over time

**Implementation** (`src/app/reader/components/Footer.tsx`):
```typescript
const calculateRemainingTime = (currentPage: number, totalPages: number, avgWPM: number) => {
  const remainingPages = totalPages - currentPage;
  const avgWordsPerPage = 300; // Estimated
  const remainingWords = remainingPages * avgWordsPerPage;
  const minutesRemaining = Math.ceil(remainingWords / avgWPM);

  return minutesRemaining;
};
```

**Settings Location**: Settings > Layout Panel > "Show remaining time"

**Display Modes**:
- Pages only
- Minutes only
- Both pages and minutes
- Hide (default)

**Files**:
- `src/app/reader/components/Footer.tsx`
- `src/store/settingsStore.ts`

### Always Show Status Bar (v0.9.58, #1417)

**Feature**: Option to keep status bar visible at all times, even in fullscreen mode.

**Settings Location**: Settings > Layout Panel > "Always show status bar"

**Behavior**:
- **Enabled**: Status bar remains visible in all modes
- **Disabled**: Status bar auto-hides in fullscreen/immersive mode

**Implementation**:
- CSS visibility override
- Platform-specific handling (iOS safe areas)
- Persisted per-user preference

**File**: `src/app/reader/components/Footer.tsx`

### Override Book Foreground/Background Color (v0.9.52, #1335)

**Feature**: Force override book's embedded color scheme with user-selected colors.

**Settings Location**: Settings > Color Panel > "Override book colors"

**Options**:
- Override background color
- Override text (foreground) color
- Override both
- Respect book colors (default)

**Implementation** (`src/app/reader/components/FoliateViewer.tsx`):
```typescript
const applyColorOverrides = (view: FoliateView, settings: ViewSettings) => {
  if (settings.overrideBackgroundColor) {
    view.setStyles({
      background: settings.backgroundColor + ' !important'
    });
  }

  if (settings.overrideForegroundColor) {
    view.setStyles({
      color: settings.foregroundColor + ' !important'
    });
  }
};
```

**Use Cases**:
- Books with poor color contrast
- Accessibility requirements
- Consistent reading experience across books

**Files**:
- `src/app/reader/components/settings/ColorPanel.tsx`
- `src/types/settings.ts`

### Parallel Reading Toggle (v0.9.62, #1504)

**Feature**: Toggle parallel reading mode when viewing multiple books simultaneously.

**Settings Location**: View menu > "Parallel Reading"

**Modes**:
- **Parallel Reading ON**: Books scroll/turn pages together synchronously
- **Parallel Reading OFF**: Books navigate independently

**Implementation** (`src/store/readerStore.ts`):
```typescript
const syncPageTurn = (direction: 'next' | 'prev') => {
  const parallelReading = settingsStore.getState().parallelReading;
  const activeViews = readerStore.getState().views;

  if (parallelReading && activeViews.length > 1) {
    // Turn page in all views simultaneously
    activeViews.forEach(view => {
      view.turnPage(direction);
    });
  } else {
    // Turn page only in focused view
    focusedView.turnPage(direction);
  }
};
```

**Use Cases**:
- Comparing translations side-by-side
- Reference material with main text
- Bilingual reading

**Files**:
- `src/store/settingsStore.ts`
- `src/app/reader/components/ReaderContent.tsx`

### Reset Settings Option (v0.9.61, #1475)

**Feature**: Reset all settings to factory defaults.

**Settings Location**: Settings > About/Advanced > "Reset All Settings"

**Reset Options**:
1. **Reset Global Settings**: Restore default global preferences
2. **Reset Book Settings**: Clear all per-book overrides
3. **Reset All**: Complete reset (requires confirmation)

**Implementation** (`src/store/settingsStore.ts`):
```typescript
const resetSettings = (scope: 'global' | 'book' | 'all') => {
  switch (scope) {
    case 'global':
      settingsStore.setState(DEFAULT_SETTINGS);
      break;
    case 'book':
      bookDataStore.clearAllBookConfigs();
      break;
    case 'all':
      settingsStore.setState(DEFAULT_SETTINGS);
      bookDataStore.clearAllBookConfigs();
      localStorage.clear();
      break;
  }

  // Save to disk
  appService.saveSettings(settingsStore.getState());
};
```

**Confirmation Dialog**:
- Warning message about data loss
- Checkbox: "I understand this cannot be undone"
- Confirm/Cancel buttons

**What Gets Reset**:
- Font preferences
- Color themes
- Layout settings
- Reading preferences
- Custom CSS
- Keyboard shortcuts (optional)

**What's Preserved**:
- User account
- Library (books)
- Reading progress
- Annotations and notes

**Files**:
- `src/app/settings/components/ResetSettings.tsx`
- `src/store/settingsStore.ts`

### Individual Margin Adjustment (v0.9.58, #1410)

**Feature**: Separate controls for top, bottom, left, and right margins instead of a single margin slider.

**Settings Location**: Settings > Layout Panel > Margin Controls

**UI Changes**:
```
Before: [========] Margin: 24px

After:
Top:    [========] 16px
Bottom: [========] 16px
Left:   [========] 32px
Right:  [========] 32px
```

**Implementation** (`src/types/settings.ts`):
```typescript
interface ViewSettings {
  // Old (deprecated):
  // margin: number;

  // New (v0.9.58+):
  marginTop: number;
  marginBottom: number;
  marginLeft: number;
  marginRight: number;
}
```

**Migration**: Existing `margin` value split equally to all four sides on first load.

**Constraints** (v0.9.59, #1428):
- Top/Bottom: 0-100px
- Left/Right: 0-200px
- More reasonable limits based on typical use cases

**Platform-Specific** (iOS, #1408):
- Respect safe area insets
- Additional padding for notch/home indicator
- Automatic adjustment for device orientation

**Files**:
- `src/app/reader/components/settings/LayoutPanel.tsx`
- `src/types/settings.ts`

### Multiple Columns in Portrait Mode (v0.9.58, #1413)

**Feature**: Allow more than 1 column even in portrait orientation.

**Settings Location**: Settings > Layout Panel > Columns

**Previous Limitation**: Portrait mode locked to 1 column

**New Behavior**:
- Portrait: 1-3 columns selectable
- Landscape: 1-4 columns selectable
- User preference respected regardless of orientation

**Implementation** (`src/app/reader/components/FoliateViewer.tsx`):
```typescript
const determineColumns = (orientation: 'portrait' | 'landscape', userPref: number) => {
  // Old logic:
  // return orientation === 'portrait' ? 1 : userPref;

  // New logic (v0.9.58+):
  const maxColumns = orientation === 'portrait' ? 3 : 4;
  return Math.min(userPref, maxColumns);
};
```

**Use Cases**:
- Large phones/tablets in portrait
- Split-screen comparison
- Dense text layouts

**Files**:
- `src/app/reader/components/settings/LayoutPanel.tsx`
- `src/app/reader/components/FoliateViewer.tsx`

---

## Version 0.9.68 - 0.9.78 Updates (33b2ba16 → cc3cc58d)

### Custom Fonts Support (v0.9.75, #1864)

**Major Feature**: Import and use custom TTF/OTF fonts in Readest.

**Overview**: Users can now import their own font files and use them for reading, providing greater customization and support for specialized fonts.

**Supported Font Formats**:
- TrueType fonts (.ttf)
- OpenType fonts (.otf)
- Font collections (.ttc) - extracts individual fonts

**Implementation** (`src/app/reader/components/settings/FontPanel.tsx`):
```typescript
const CustomFontImporter = () => {
  const handleImport = async () => {
    // Open file picker
    const files = await appService.selectFiles({
      filters: [{
        name: 'Fonts',
        extensions: ['ttf', 'otf', 'ttc']
      }],
      multiple: true
    });

    // Import each font
    for (const file of files) {
      try {
        // Parse font metadata
        const fontData = await parseFontFile(file);

        // Store font in app storage
        await appService.saveFontFile(file, fontData.fontFamily);

        // Add to available fonts list
        customFonts.push({
          fontFamily: fontData.fontFamily,
          fontStyle: fontData.fontStyle,
          fontWeight: fontData.fontWeight,
          filePath: fontData.filePath
        });

        toast.success(`Imported "${fontData.fontFamily}"`);
      } catch (error) {
        toast.error(`Failed to import ${file.name}: ${error.message}`);
      }
    }

    // Refresh font list
    refreshFonts();
  };

  return (
    <button onClick={handleImport}>
      Import Custom Fonts
    </button>
  );
};
```

**Font Parsing** (#1876, #1881):
- Extracts font family name from font file metadata
- Parses font style (Regular, Bold, Italic, Bold Italic)
- Detects font weight variants (100-900)
- Handles font collections (.ttc) with multiple fonts

**UI Features**:

1. **Font Preview**:
   - Preview text shown in each font
   - Different preview text for different scripts (Latin, CJK, Arabic)
   - Font weight variants displayed

2. **Font Management**:
   - List of all imported custom fonts
   - Delete unwanted fonts
   - Rename font display names
   - Organize into font families

3. **Application**:
   - Select custom font from font dropdown
   - Available in all font selection contexts
   - Works across all book formats (EPUB, PDF)

**Files**:
- `src/app/reader/components/settings/FontPanel.tsx` - Font import UI
- `src/utils/fontParser.ts` - Font metadata extraction
- `src/services/appService.ts` - Font file storage
- `src/store/settingsStore.ts` - Custom fonts state

### Grouped Custom Fonts (v0.9.76, #1945)

**Feature**: Group custom fonts into families for better organization.

**Overview**: Custom fonts with the same family name but different styles (Regular, Bold, Italic) are now grouped together in the font selector.

**Font Family Structure**:
```typescript
interface FontFamily {
  familyName: string;
  fonts: Array<{
    style: 'Regular' | 'Bold' | 'Italic' | 'Bold Italic';
    weight: number;
    filePath: string;
  }>;
}
```

**UI Display**:
```
▼ Noto Serif
  ├─ Regular (400)
  ├─ Bold (700)
  ├─ Italic (400)
  └─ Bold Italic (700)

▼ Roboto
  ├─ Thin (100)
  ├─ Light (300)
  ├─ Regular (400)
  ├─ Medium (500)
  └─ Bold (700)
```

**Auto-Grouping Logic** (`src/utils/fontGrouping.ts`):
```typescript
const groupFonts = (fonts: CustomFont[]): FontFamily[] => {
  const families = new Map<string, FontFamily>();

  for (const font of fonts) {
    const familyName = font.fontFamily;

    if (!families.has(familyName)) {
      families.set(familyName, {
        familyName,
        fonts: []
      });
    }

    families.get(familyName).fonts.push({
      style: font.fontStyle,
      weight: font.fontWeight,
      filePath: font.filePath
    });
  }

  return Array.from(families.values());
};
```

**Benefits**:
- Cleaner font selector UI
- Easy identification of font variants
- Automatic variant selection (bold, italic)
- Better organization for large font collections

**Files**:
- `src/utils/fontGrouping.ts` - Font family grouping logic
- `src/app/reader/components/settings/FontPanel.tsx` - Grouped display UI

### Font Panel Layout Improvements (v0.9.75, #1870, #1871, #1903)

**Layout Enhancements**:

1. **Custom Fonts Panel** (#1870):
   - Dedicated panel for managing custom fonts
   - Grid layout for font cards
   - Font preview in each card
   - Quick apply button

2. **Layout Tweaks** (#1871):
   - Better spacing and alignment
   - Responsive grid columns
   - Improved mobile layout
   - Touch-friendly controls

3. **Font Name Overflow** (#1903):
   - Long font names truncated with ellipsis
   - Tooltip shows full font name on hover
   - Prevents layout breaking

**Implementation** (`src/app/reader/components/settings/CustomFontsPanel.tsx`):
```typescript
<div className="custom-fonts-grid">
  {customFonts.map(font => (
    <div key={font.id} className="font-card">
      <div
        className="font-preview"
        style={{ fontFamily: font.fontFamily }}
      >
        The quick brown fox jumps over the lazy dog
      </div>

      <div className="font-name" title={font.fontFamily}>
        {truncateText(font.fontFamily, 20)}
      </div>

      <div className="font-actions">
        <button onClick={() => applyFont(font)}>
          Apply
        </button>
        <button onClick={() => deleteFont(font)}>
          Delete
        </button>
      </div>
    </div>
  ))}
</div>
```

**Files**:
- `src/app/reader/components/settings/CustomFontsPanel.tsx` - Custom fonts UI
- `src/styles/fonts.css` - Font panel styling

### Purge Custom Fonts on Reset (v0.9.76, #1906)

**Feature**: Custom fonts are now removed when resetting font configuration.

**Behavior**:
- "Reset to Defaults" button in font settings
- Confirmation dialog warns about custom font deletion
- Removes all custom font files from storage
- Clears custom fonts from font list
- Reverts to default system fonts

**Implementation**:
```typescript
const resetFontsConfig = async () => {
  const confirmed = await confirm(
    'Reset font configuration?',
    'This will remove all custom fonts and reset to default settings.'
  );

  if (!confirmed) return;

  // Delete all custom font files
  for (const font of customFonts) {
    await appService.deleteFontFile(font.filePath);
  }

  // Clear custom fonts list
  setCustomFonts([]);

  // Reset font settings to defaults
  settingsStore.setSettings({
    fontFamily: DEFAULT_FONT,
    fontSize: DEFAULT_FONT_SIZE,
    lineHeight: DEFAULT_LINE_HEIGHT
  });

  toast.success('Font configuration reset to defaults.');
};
```

**Files**:
- `src/app/reader/components/settings/FontPanel.tsx` - Reset functionality

### Fixed Broken CJK Font Links (v0.9.67, #1687)

**Fix**: Resolved broken links for online CJK fonts.

**Problem**: Some CJK fonts were loading from deprecated or broken CDN URLs, causing fallback to system fonts.

**Solution**: Updated font URLs to reliable CDN sources:

```typescript
const CJK_FONTS = {
  'Noto Serif CJK': 'https://fonts.googleapis.com/css2?family=Noto+Serif+SC',
  'Noto Sans CJK': 'https://fonts.googleapis.com/css2?family=Noto+Sans+SC',
  'Source Han Serif': 'https://cdn.jsdelivr.net/npm/source-han-serif-sc/dist/SourceHanSerifSC-Regular.otf',
  'LXGW WenKai': 'https://cdn.jsdelivr.net/npm/lxgw-wenkai-webfont@latest/style.css'
};
```

**Affected Fonts**:
- Noto Serif JP
- Noto Sans CJK
- Source Han Serif
- LXGW WenKai

**Files**:
- `src/utils/fonts.ts` - Font URL configuration
- `src/components/FontLoader.tsx` - Online font loading

---

**Last Updated**: Documentation for commits through cc3cc58d (November 2025, v0.9.78)
**Related Documents**: [reader-ui-settings](./reader-ui-settings.md), [custom-css-editor](./custom-css-editor.md), [cross-platform-support](../cross-platform-support/index.md)
