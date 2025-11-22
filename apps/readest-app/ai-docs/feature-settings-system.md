# Feature: Settings System

## Overview

The Settings System in Readest implements a three-tier hierarchy: **Global Settings** < **Book Settings** < **View Settings**. This allows users to set default preferences globally while overriding them per-book or per-view. At commit `d757555f`, settings cover fonts, colors, layout, and miscellaneous reading preferences.

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

## Custom CSS Editor Enhancements (Added Jan 2025)

### Overview

The Custom CSS editor received significant UX improvements in commit 2c9fe8e4, transforming it from an immediate-apply system to a draft-based editor with explicit validation and Apply button.

**Location**: `src/app/reader/components/settings/MiscPanel.tsx` (Misc settings panel)

### Key Improvements

| Feature | Before | After |
|---------|--------|-------|
| **Validation** | Regex-based, limited errors | Dedicated `cssValidate()` utility with detailed errors |
| **Application** | Immediate on every keystroke | Explicit "Apply" button |
| **State Management** | Direct state updates | Draft state + saved state separation |
| **Error Feedback** | Generic messages | Specific error messages per validation rule |
| **UX** | Confusing immediate changes | Clear save workflow |

### CSS Validation (`src/utils/css.ts`)

**Validation Rules**:
1. Comment removal (`/* ... */`)
2. Brace balancing (`{` equals `}`)
3. Rule structure (selector + declarations)
4. Selector validation (non-empty)
5. Declaration validation (non-empty)
6. Property validation (proper format with colons)

**Error Messages**:
- "Empty CSS"
- "Unbalanced curly braces"
- "Invalid CSS structure"
- "Missing selector"
- "Missing declarations for selector: {selector}"
- "Invalid property: {property}"

### Usage Pattern

```typescript
const [draftStylesheet, setDraftStylesheet] = useState(userStylesheet);
const [draftStylesheetSaved, setDraftStylesheetSaved] = useState(true);
const [error, setError] = useState<string | null>(null);

const handleUserStylesheetChange = (e) => {
  const css = e.target.value;
  setDraftStylesheet(css);
  setDraftStylesheetSaved(false);
  
  const { isValid, error } = cssValidate(css);
  setError(error);
};

const applyStyles = () => {
  if (error) return;
  
  const formatted = cssbeautify(draftStylesheet, {
    indent: '  ',
    openbrace: 'end-of-line',
  });
  
  setViewSettings({ userStylesheet: formatted });
  setDraftStylesheet(formatted);
  setDraftStylesheetSaved(true);
};
```

### Button State Management

- **Hidden**: When `draftStylesheetSaved === true` (no changes)
- **Disabled**: When `error !== null` (validation failed)
- **Enabled**: When changes exist and validation passes

---

## Vertical/Horizontal Layout Switch (Added Jan 2025)

### Overview

The vertical/horizontal layout switch (commit 3ad26d9d) allows users to manually override text direction for CJK (Chinese, Japanese, Korean) books.

**Location**: `src/app/reader/components/settings/LayoutPanel.tsx` (Writing Mode section)

### Writing Modes

| Mode | Value | Description | Icon |
|------|-------|-------------|------|
| **Auto** | `'auto'` | Uses document's natural writing direction | `MdOutlineAutoMode` |
| **Horizontal** | `'horizontal-tb'` | Forces horizontal text (left-to-right, top-to-bottom) | `MdOutlineTextRotationNone` |
| **Vertical** | `'vertical-rl'` | Forces vertical text (right-to-left, top-to-bottom) | `MdOutlineTextRotationDown` |

### Feature Characteristics

**Visibility**: Only shown for CJK books (detected via language code)

```typescript
const langCode = getBookLangCode(bookData.bookDoc?.metadata?.language);
const isCJKBook = langCode === 'zh' || langCode === 'ja' || langCode === 'ko';
```

**Type Definition**:

```typescript
export interface BookLayout {
  writingMode: string; // 'auto' | 'horizontal-tb' | 'vertical-rl'
  vertical: boolean;   // Auto-detected from document
  // ... other properties
}
```

**CSS Application** (`src/utils/style.ts`):

```css
html, body {
  ${writingMode === 'auto' ? '' : `writing-mode: ${writingMode};`}
}
```

### Differences from Existing `vertical` Property

| Aspect | `vertical` (existing) | `writingMode` (new) |
|--------|----------------------|---------------------|
| **Source** | Auto-detected from document | User-controlled setting |
| **Type** | Boolean | String enum |
| **Purpose** | Layout calculations (e.g., footnote positioning) | Override document text direction |
| **Persistence** | Not persisted | Saved per book |
| **When Set** | On document load | User selection in settings |

### Persistence

- **Per-Book Only**: Not available in global settings
- **Storage**: Saved in `BookConfig.viewSettings.writingMode`
- **Default**: `'auto'` (defined in `src/services/constants.ts`)

### Use Case

Useful when:
- Book has incorrect metadata about text direction
- User prefers alternate reading direction (e.g., horizontal for traditionally vertical text)
- Testing different reading modes for CJK content

---


## Screen Wake Lock Setting (Added v0.9.13)

### Overview
Commit #403 added an option to prevent the device screen from sleeping during reading sessions, improving the reading experience for users who prefer not to interact with the screen frequently.

### Implementation

**Location**: `src/app/reader/components/settings/MiscPanel.tsx`

**Type Definition**:
```typescript
interface SystemSettings {
  // ... existing settings
  keepScreenAwake: boolean; // Added v0.9.13
}
```

### API Support

The feature uses different APIs based on the platform:

**Web Platform** (Wake Lock API):
```typescript
let wakeLock: WakeLockSentinel | null = null;

const requestWakeLock = async () => {
  if ('wakeLock' in navigator) {
    try {
      wakeLock = await navigator.wakeLock.request('screen');
    } catch (err) {
      console.error('Wake Lock request failed:', err);
    }
  }
};

const releaseWakeLock = async () => {
  if (wakeLock) {
    await wakeLock.release();
    wakeLock = null;
  }
};
```

**Native Platform (Tauri)**:
```rust
// In src-tauri/src/lib.rs or custom module
#[tauri::command]
fn keep_screen_awake(enable: bool) -> Result<(), String> {
    // Platform-specific implementation
    #[cfg(target_os = "macos")]
    {
        // Use caffeinate or IOKit
    }

    #[cfg(target_os = "windows")]
    {
        // Use SetThreadExecutionState
    }

    #[cfg(target_os = "linux")]
    {
        // Use systemd-inhibit
    }

    Ok(())
}
```

### Behavior

**Activation**:
- Enabled only when a book is open in reader
- Activated when user enables the setting in Misc Panel

**Deactivation**:
- Automatically disabled when:
  - Book is closed
  - User navigates away from reader
  - App moves to background (mobile)
  - User disables the setting
  - Browser tab becomes inactive (web)

**Persistence**:
- Setting persists across sessions
- Can be configured globally or per-book
- Default: `false` (respect system sleep settings)

### Platform Support

| Platform | API Used | Status |
|----------|----------|--------|
| **Web (Modern Browsers)** | Wake Lock API | ✅ Supported |
| **macOS** | caffeinate / IOKit | ✅ Supported |
| **Windows** | SetThreadExecutionState | ✅ Supported |
| **Linux** | systemd-inhibit | ✅ Supported |
| **iOS** | UIApplication.isIdleTimerDisabled | ✅ Supported |
| **Android** | PowerManager.WakeLock | ✅ Supported |

### Browser Compatibility

**Wake Lock API Support**:
- Chrome/Edge: ✅ Version 84+
- Firefox: ✅ Version 126+
- Safari: ✅ Version 16.4+
- Opera: ✅ Version 70+

**Fallback**: For unsupported browsers, setting is disabled/hidden in UI.

### AI Modification Guidelines

**To customize wake lock behavior**:

1. **Add wake lock timeout** (auto-release after period):
   ```typescript
   let wakeLockTimeout: NodeJS.Timeout;

   const requestWakeLock = async (duration: number) => {
     await requestWakeLock();
     wakeLockTimeout = setTimeout(async () => {
       await releaseWakeLock();
     }, duration);
   };
   ```

2. **Add battery level check** (disable on low battery):
   ```typescript
   const battery = await navigator.getBattery();
   if (battery.level < 0.20) {
     // Disable wake lock when battery < 20%
     await releaseWakeLock();
   }
   ```

3. **Add page visibility handling**:
   ```typescript
   document.addEventListener('visibilitychange', async () => {
     if (document.hidden) {
       await releaseWakeLock();
     } else if (keepScreenAwake) {
       await requestWakeLock();
     }
   });
   ```

### Common Issues

**Issue**: Wake lock not working on mobile

**Debug steps**:
1. Check if browser supports Wake Lock API
2. Verify HTTPS connection (required for Wake Lock API)
3. Check if screen is locked manually by user
4. Verify permission is granted (some browsers require permission)

**Solution**: Ensure HTTPS and check browser compatibility.

**Issue**: Wake lock releases unexpectedly

**Debug steps**:
1. Check browser console for Wake Lock errors
2. Verify page visibility state
3. Check if system sleep settings override
4. Test battery saver mode impact

**Solution**: Reacquire wake lock on visibility change.

---
