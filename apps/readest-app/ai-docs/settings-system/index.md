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

## Version 0.9.79 - 0.9.82 Updates (cc3cc58d → e1691661)

### Variable Fonts Support (v0.9.80, #2007)

**Major Feature**: Full support for variable fonts (TTF/OTF files with multiple weights and styles).

**Overview**: Variable fonts contain multiple font weights, widths, and styles in a single file. Readest now properly detects and utilizes all available variations.

**Variable Font Detection**:
```typescript
import * as fontkit from '@pdf-lib/fontkit';

const analyzeVariableFont = async (fontFile: ArrayBuffer): Promise<FontInfo> => {
  const font = fontkit.create(Buffer.from(fontFile));

  const isVariable = font.variationAxes && Object.keys(font.variationAxes).length > 0;

  if (isVariable) {
    return {
      family: font.familyName,
      isVariable: true,
      axes: Object.keys(font.variationAxes),  // ['wght', 'ital', 'wdth', etc.]
      defaultWeight: font.variationAxes.wght?.default || 400,
      weightRange: {
        min: font.variationAxes.wght?.min || 100,
        max: font.variationAxes.wght?.max || 900
      },
      supportsItalic: 'ital' in font.variationAxes,
      supportsWidth: 'wdth' in font.variationAxes
    };
  }

  return {
    family: font.familyName,
    isVariable: false,
    weight: font.subfamilyName.includes('Bold') ? 700 : 400,
    isItalic: font.subfamilyName.includes('Italic')
  };
};
```

**Font Weight Slider**:
```typescript
const VariableFontControls = ({ font }: { font: VariableFontInfo }) => {
  const [weight, setWeight] = useState(font.defaultWeight);
  const [width, setWidth] = useState(100);
  const [italic, setItalic] = useState(0);

  const applyVariations = () => {
    const variations = {
      'font-weight': weight,
      'font-stretch': `${width}%`,
      'font-style': italic > 0.5 ? 'italic' : 'normal'
    };

    document.documentElement.style.fontVariationSettings =
      `"wght" ${weight}, "wdth" ${width}, "ital" ${italic}`;
  };

  return (
    <div className="variable-font-controls">
      <label>
        Weight: {weight}
        <input
          type="range"
          min={font.weightRange.min}
          max={font.weightRange.max}
          value={weight}
          onChange={(e) => setWeight(Number(e.target.value))}
        />
      </label>

      {font.supportsWidth && (
        <label>
          Width: {width}%
          <input
            type="range"
            min="75"
            max="125"
            value={width}
            onChange={(e) => setWidth(Number(e.target.value))}
          />
        </label>
      )}

      {font.supportsItalic && (
        <label>
          Italic
          <input
            type="checkbox"
            checked={italic > 0.5}
            onChange={(e) => setItalic(e.target.checked ? 1 : 0)}
          />
        </label>
      )}
    </div>
  );
};
```

**CSS Implementation**:
```css
@font-face {
  font-family: 'Inter Variable';
  src: url('/fonts/Inter-Variable.woff2') format('woff2-variations');
  font-weight: 100 900;  /* Supports all weights */
  font-stretch: 75% 125%;  /* Supports width variations */
  font-style: oblique 0deg 10deg;  /* Supports slant */
}

.reader-content {
  font-family: 'Inter Variable', sans-serif;
  font-variation-settings: 'wght' 450, 'wdth' 100, 'slnt' 0;
}
```

**Popular Variable Fonts**:
- **Inter**: Modern sans-serif with 9 axes
- **Recursive**: Monospace/Sans hybrid
- **Source Serif Variable**: Serif with weight axis
- **Roboto Flex**: Highly customizable
- **Anybody**: Display font with 9 axes

**Benefits**:
- One file for all weights (reduces storage)
- Smooth weight transitions
- Precise typography control
- Better performance (fewer font files)
- Fine-tuned readability adjustments

**Import Workflow**:
1. User selects variable font file (.ttf/.otf/.woff2)
2. App analyzes font with fontkit
3. Detects available axes (weight, width, slant, etc.)
4. Shows axis controls in font panel
5. Applies variations via CSS font-variation-settings
6. Saves settings per book

**Files**:
- `src/utils/fontAnalysis.ts` - Variable font detection
- `src/app/reader/components/settings/VariableFontPanel.tsx` - UI controls
- `src/types/fonts.ts` - Font type definitions
- `src/services/fontService.ts` - Font import and management

### Global Settings Access from Library (v0.9.82, #2151)

**Feature**: Access global (app-wide) settings directly from the library menu.

**Overview**: Previously, global settings were only accessible from within the reader. Now users can access app-wide settings from the library screen for quicker configuration.

**Implementation**:
```typescript
const LibraryHeader = () => {
  const [showGlobalSettings, setShowGlobalSettings] = useState(false);

  return (
    <header className="library-header">
      <h1>Library</h1>

      <Menu>
        <MenuItem icon={<BookIcon />}>
          Book Settings
          <SubMenu>
            <MenuItem onClick={() => openImportDialog()}>
              Import Books
            </MenuItem>
            <MenuItem onClick={() => openExportDialog()}>
              Export Library
            </MenuItem>
          </SubMenu>
        </MenuItem>

        <MenuItem icon={<SettingsIcon />} onClick={() => setShowGlobalSettings(true)}>
          Global Settings
        </MenuItem>

        <MenuItem icon={<SyncIcon />} onClick={() => syncLibrary()}>
          Sync Now
        </MenuItem>
      </Menu>

      {showGlobalSettings && (
        <GlobalSettingsDialog onClose={() => setShowGlobalSettings(false)} />
      )}
    </header>
  );
};
```

**Global Settings Dialog**:
```typescript
const GlobalSettingsDialog = ({ onClose }: { onClose: () => void }) => {
  const [settings, setSettings] = useGlobalSettings();

  return (
    <Dialog open onClose={onClose} fullScreen>
      <DialogTitle>Global Settings</DialogTitle>

      <Tabs>
        <Tab label="Appearance">
          <ThemeSettings />
          <DefaultFontSettings />
          <LayoutSettings />
        </Tab>

        <Tab label="Reading">
          <DefaultReadingSettings />
          <AutoSaveSettings />
          <PageTurnSettings />
        </Tab>

        <Tab label="Sync">
          <CloudSyncSettings />
          <SyncIntervalSettings />
          <ConflictResolutionSettings />
        </Tab>

        <Tab label="Advanced">
          <DataLocationSettings />
          <StorageQuotaSettings />
          <ExperimentalFeatures />
        </Tab>
      </Tabs>

      <DialogActions>
        <Button onClick={onClose}>Close</Button>
        <Button onClick={() => resetToDefaults()}>Reset to Defaults</Button>
      </DialogActions>
    </Dialog>
  );
};
```

**Global vs Book-Specific Settings**:

| Setting | Scope | Where to Change |
|---------|-------|-----------------|
| **Theme** (Light/Dark/Sepia) | Global | Library or Reader |
| **Default font** | Global | Library or Reader |
| **Font size for new books** | Global | Library Settings |
| **Current book's font size** | Book-specific | Reader |
| **Sync enabled** | Global | Library Settings |
| **Reading direction (LTR/RTL)** | Book-specific | Reader |
| **Data location** | Global | Library Settings |
| **Gestures** | Global | Library or Reader |

**Settings Persistence**:
```typescript
interface GlobalSettings {
  theme: 'light' | 'dark' | 'sepia';
  defaultFont: string;
  defaultFontSize: number;
  syncEnabled: boolean;
  syncInterval: number;
  dataLocation: string;
  experimentalFeatures: string[];
}

// Stored in: Readest/Data/global-settings.json
const saveGlobalSettings = async (settings: GlobalSettings) => {
  await appService.writeFile(
    'global-settings.json',
    JSON.stringify(settings, null, 2)
  );
};
```

**User Experience**:
- Quick access to app-wide settings without opening a book
- Changes apply to all books immediately
- New books inherit global defaults
- Existing books keep their customizations
- Clear indication of global vs book-specific settings

**Files**:
- `src/app/library/components/LibraryHeader.tsx` - Menu with settings option
- `src/components/GlobalSettingsDialog.tsx` - Settings dialog
- `src/hooks/useGlobalSettings.ts` - Settings state management
- `src/store/globalSettingsStore.ts` - Global settings store

### Font CSS Specificity Improvements (v0.9.80, #2081, #2113)

**Lower Font-Family Specificity** (v0.9.80, #2081):

**Problem**: Readest's font settings were overriding publisher CSS with `!important`, causing issues with specialized fonts in books.

**Solution**: Use lower specificity CSS so publisher fonts can override when needed.

**Before**:
```css
/* Too specific - always overrides book */
.epub-content * {
  font-family: var(--user-font) !important;
}
```

**After**:
```css
/* Lower specificity - book can override */
.epub-content {
  font-family: var(--user-font);
}

/* Only use higher specificity when user explicitly sets "override publisher fonts" */
.epub-content.override-fonts * {
  font-family: var(--user-font) !important;
}
```

**User Control**:
```typescript
const FontSettings = () => {
  const [overridePublisherFonts, setOverridePublisherFonts] = useState(false);

  return (
    <div>
      <FontFamilyPicker />

      <Checkbox
        checked={overridePublisherFonts}
        onChange={(e) => setOverridePublisherFonts(e.target.checked)}
        label="Override publisher fonts"
      />

      <p className="help-text">
        When disabled, the book's built-in fonts will be used for specially
        formatted text (e.g., poetry, code, emphasis).
      </p>
    </div>
  );
};
```

**Hard-Coded Font Weight Override** (v0.9.80, #2113):

**Problem**: Some publishers hard-code `font-weight: bold` or `font-weight: 400`, interfering with variable font weight settings.

**Solution**: Override hard-coded weights when user has customized font weight.

**Implementation**:
```css
/* Override hard-coded weights if user has custom weight */
.epub-content.custom-weight * {
  font-weight: var(--user-weight) !important;
}

/* Preserve semantic bold/italic */
.epub-content.custom-weight strong,
.epub-content.custom-weight b {
  font-weight: calc(var(--user-weight) + 300) !important;
}

.epub-content.custom-weight em,
.epub-content.custom-weight i {
  font-style: italic !important;
}
```

**Files**:
- `packages/foliate-js/view.css` - CSS specificity rules
- `src/app/reader/components/settings/FontPanel.tsx` - Override toggle

---

## Version 0.9.83 - 0.9.90 Updates (e1691661 → dd5371d2)

### Screen Brightness Control (v0.9.83-0.9.85, #2197, #2297, #2338)

**Feature**: Manual and automatic screen brightness adjustment during reading.

**Overview**: Users can now adjust screen brightness directly from the reader interface, with options for manual control and automatic brightness based on ambient light.

**Settings Location**: Settings > Display Panel > "Screen Brightness"

**Brightness Modes**:

1. **Manual Brightness** (#2197):
   - Slider control for precise brightness adjustment
   - Range: 0% to 100%
   - Per-book brightness settings
   - Persistent across sessions

2. **Auto Brightness** (#2297):
   - Automatic adjustment based on ambient light sensors
   - Platform-specific implementation
   - Toggle on/off option
   - Manual override available

3. **Non-Linear Brightness Slider** (#2338):
   - Logarithmic scale for better low-value control
   - More precise control at lower brightness levels
   - Improved usability in dark environments

**Implementation** (`src/app/reader/components/settings/DisplayPanel.tsx`):
```typescript
const BrightnessControl = () => {
  const [brightness, setBrightness] = useState(100);
  const [autoBrightness, setAutoBrightness] = useState(false);

  // Non-linear mapping for better low-value control
  const mapBrightnessValue = (sliderValue: number): number => {
    // Logarithmic scale: more precision at lower values
    // 0-100 slider maps to 0-100% brightness non-linearly
    const normalized = sliderValue / 100;
    const mapped = Math.pow(normalized, 2); // Quadratic mapping
    return Math.round(mapped * 100);
  };

  const applyBrightness = async (value: number) => {
    const actualBrightness = mapBrightnessValue(value);

    // Apply to screen
    if (isNativePlatform()) {
      await invoke('set_screen_brightness', { brightness: actualBrightness / 100 });
    } else {
      // Web platform: adjust via CSS filter
      document.body.style.filter = `brightness(${actualBrightness}%)`;
    }
  };

  return (
    <div className="brightness-control">
      <div className="form-control">
        <label className="label">
          <span className="label-text">{t('Screen Brightness')}</span>
          <span className="label-text-alt">{mapBrightnessValue(brightness)}%</span>
        </label>

        <input
          type="range"
          min="0"
          max="100"
          value={brightness}
          onChange={(e) => {
            const value = Number(e.target.value);
            setBrightness(value);
            applyBrightness(value);
          }}
          className="range range-primary"
          disabled={autoBrightness}
        />
      </div>

      <div className="form-control">
        <label className="label cursor-pointer">
          <span className="label-text">{t('Auto Brightness')}</span>
          <input
            type="checkbox"
            className="toggle"
            checked={autoBrightness}
            onChange={(e) => {
              setAutoBrightness(e.target.checked);
              if (e.target.checked) {
                enableAutoBrightness();
              } else {
                disableAutoBrightness();
              }
            }}
          />
        </label>
      </div>
    </div>
  );
};
```

**Platform-Specific Implementation**:

**iOS/Android** (`src-tauri/src/brightness.rs`):
```rust
#[tauri::command]
pub async fn set_screen_brightness(brightness: f32) -> Result<(), String> {
    #[cfg(target_os = "ios")]
    {
        use objc::runtime::{Class, Object};
        use objc::{msg_send, sel, sel_impl};

        unsafe {
            let screen: *mut Object = msg_send![Class::get("UIScreen").unwrap(), mainScreen];
            let _: () = msg_send![screen, setBrightness: brightness];
        }
    }

    #[cfg(target_os = "android")]
    {
        // Android implementation using Android APIs
        let activity = get_android_activity();
        activity.set_screen_brightness(brightness);
    }

    Ok(())
}
```

**Desktop** (Linux/Windows/macOS):
- Platform-specific APIs for display brightness
- Fallback to CSS filter on web platform
- System brightness API integration

**Use Cases**:
- Reading in dark environments
- Reducing eye strain
- Battery conservation
- Accessibility requirements
- E-ink display optimization

**Files**:
- `src/app/reader/components/settings/DisplayPanel.tsx` - UI controls
- `src-tauri/src/brightness.rs` - Native brightness control
- `src/services/brightnessService.ts` - Service abstraction
- `src/types/settings.ts` - Settings type definitions

### Custom Background Images (v0.9.85-0.9.86, #2214, #2225)

**Feature**: Support for custom background images while reading.

**Overview**: Users can set custom background images globally or per-book, with options for opacity, blending, and positioning.

**Settings Location**: Settings > Appearance > "Background Image"

**Background Image Options**:

1. **Global Background Image** (#2214):
   - Set default background for all books
   - Image file selection from local storage
   - Opacity control (0-100%)
   - Blending modes (normal, multiply, screen)
   - Position and sizing options

2. **Per-Book Background Images** (#2225):
   - Override global background for specific books
   - Book-specific opacity and blending settings
   - Independent from global settings

**Implementation** (`src/app/reader/components/settings/AppearancePanel.tsx`):
```typescript
interface BackgroundImageSettings {
  enabled: boolean;
  imagePath: string;
  opacity: number;         // 0-100
  blendMode: 'normal' | 'multiply' | 'screen' | 'overlay';
  position: 'center' | 'tile' | 'stretch' | 'fit';
  fixed: boolean;          // Fixed while scrolling
}

const BackgroundImageSelector = () => {
  const [bgSettings, setBgSettings] = useState<BackgroundImageSettings>({
    enabled: false,
    imagePath: '',
    opacity: 20,
    blendMode: 'multiply',
    position: 'center',
    fixed: true
  });

  const selectBackgroundImage = async () => {
    const selected = await appService.selectFiles({
      filters: [{
        name: 'Images',
        extensions: ['jpg', 'jpeg', 'png', 'webp', 'svg']
      }],
      multiple: false
    });

    if (selected && selected.length > 0) {
      const imagePath = selected[0];

      // Copy to app data directory
      const savedPath = await appService.saveBackgroundImage(imagePath);

      setBgSettings({
        ...bgSettings,
        imagePath: savedPath,
        enabled: true
      });
    }
  };

  const applyBackgroundImage = () => {
    if (!bgSettings.enabled || !bgSettings.imagePath) {
      document.body.style.backgroundImage = 'none';
      return;
    }

    const { imagePath, opacity, blendMode, position, fixed } = bgSettings;

    // Apply CSS
    document.body.style.backgroundImage = `url("${imagePath}")`;
    document.body.style.backgroundSize = getSizeFromPosition(position);
    document.body.style.backgroundPosition = 'center';
    document.body.style.backgroundRepeat = position === 'tile' ? 'repeat' : 'no-repeat';
    document.body.style.backgroundAttachment = fixed ? 'fixed' : 'scroll';

    // Opacity via pseudo-element
    document.documentElement.style.setProperty('--bg-image-opacity', `${opacity / 100}`);
    document.documentElement.style.setProperty('--bg-blend-mode', blendMode);
  };

  const getSizeFromPosition = (pos: string): string => {
    switch (pos) {
      case 'stretch': return '100% 100%';
      case 'fit': return 'contain';
      case 'tile': return 'auto';
      default: return 'cover';
    }
  };

  return (
    <div className="background-image-settings">
      <div className="form-control">
        <label className="label cursor-pointer">
          <span className="label-text">{t('Enable Background Image')}</span>
          <input
            type="checkbox"
            className="toggle"
            checked={bgSettings.enabled}
            onChange={(e) => {
              setBgSettings({ ...bgSettings, enabled: e.target.checked });
              applyBackgroundImage();
            }}
          />
        </label>
      </div>

      {bgSettings.enabled && (
        <>
          <button onClick={selectBackgroundImage} className="btn btn-primary">
            {bgSettings.imagePath ? t('Change Image') : t('Select Image')}
          </button>

          {bgSettings.imagePath && (
            <>
              <div className="form-control">
                <label className="label">
                  <span className="label-text">{t('Opacity')}</span>
                  <span className="label-text-alt">{bgSettings.opacity}%</span>
                </label>
                <input
                  type="range"
                  min="0"
                  max="100"
                  value={bgSettings.opacity}
                  onChange={(e) => {
                    setBgSettings({ ...bgSettings, opacity: Number(e.target.value) });
                    applyBackgroundImage();
                  }}
                  className="range"
                />
              </div>

              <div className="form-control">
                <label className="label">
                  <span className="label-text">{t('Blend Mode')}</span>
                </label>
                <select
                  value={bgSettings.blendMode}
                  onChange={(e) => {
                    setBgSettings({ ...bgSettings, blendMode: e.target.value as any });
                    applyBackgroundImage();
                  }}
                  className="select select-bordered"
                >
                  <option value="normal">{t('Normal')}</option>
                  <option value="multiply">{t('Multiply')}</option>
                  <option value="screen">{t('Screen')}</option>
                  <option value="overlay">{t('Overlay')}</option>
                </select>
              </div>

              <div className="form-control">
                <label className="label">
                  <span className="label-text">{t('Position')}</span>
                </label>
                <select
                  value={bgSettings.position}
                  onChange={(e) => {
                    setBgSettings({ ...bgSettings, position: e.target.value as any });
                    applyBackgroundImage();
                  }}
                  className="select select-bordered"
                >
                  <option value="center">{t('Center (Cover)')}</option>
                  <option value="fit">{t('Fit')}</option>
                  <option value="stretch">{t('Stretch')}</option>
                  <option value="tile">{t('Tile')}</option>
                </select>
              </div>

              <div className="form-control">
                <label className="label cursor-pointer">
                  <span className="label-text">{t('Fixed While Scrolling')}</span>
                  <input
                    type="checkbox"
                    className="toggle"
                    checked={bgSettings.fixed}
                    onChange={(e) => {
                      setBgSettings({ ...bgSettings, fixed: e.target.checked });
                      applyBackgroundImage();
                    }}
                  />
                </label>
              </div>

              <button
                onClick={() => {
                  setBgSettings({ ...bgSettings, imagePath: '', enabled: false });
                  applyBackgroundImage();
                }}
                className="btn btn-error btn-outline"
              >
                {t('Remove Background Image')}
              </button>
            </>
          )}
        </>
      )}
    </div>
  );
};
```

**CSS Implementation** (`src/styles/background-image.css`):
```css
body::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-image: inherit;
  background-size: inherit;
  background-position: inherit;
  background-repeat: inherit;
  opacity: var(--bg-image-opacity, 0.2);
  mix-blend-mode: var(--bg-blend-mode, multiply);
  pointer-events: none;
  z-index: -1;
}
```

**Per-Book Settings** (BookConfig):
```typescript
interface BookConfig {
  // ...existing fields
  backgroundImage?: BackgroundImageSettings;
}

// Override global background with book-specific settings
const getEffectiveBackgroundSettings = (
  globalSettings: BackgroundImageSettings,
  bookSettings?: BackgroundImageSettings
): BackgroundImageSettings => {
  if (bookSettings && bookSettings.enabled) {
    return bookSettings;
  }
  return globalSettings;
};
```

**Use Cases**:
- Aesthetic customization
- Thematic reading experience (e.g., parchment texture for classics)
- Reduced eye strain with textured backgrounds
- Personal preference for visual ambiance

**Files**:
- `src/app/reader/components/settings/AppearancePanel.tsx` - UI controls
- `src/services/appService.ts` - Image file management
- `src/types/settings.ts` - Settings type definitions
- `src/styles/background-image.css` - Background styling

---

**Last Updated**: Documentation for commits through dd5371d2 (November 2025, v0.9.90)
**Related Documents**: [reader-ui-settings](./reader-ui-settings.md), [custom-css-editor](./custom-css-editor.md), [cross-platform-support](../cross-platform-support/index.md)
