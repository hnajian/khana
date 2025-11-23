# Android Platform Support

**Platform Category**: Mobile, Cross-Platform
**Status**: Full Support
**Related Commits**: #361, #653, #720, #788, #798, #799, #807, #829, #833

## Overview

Readest provides comprehensive Android support through Tauri's mobile capabilities, offering native Android integration while maintaining the shared Next.js codebase. The Android version includes platform-specific features like content URI handling, system UI control, hardware key interception, and optimized mobile UI patterns.

## Platform Detection

### OS Platform Detection

**File**: `src/utils/ua.ts`

```typescript
export const getOSPlatform = (): OsPlatform => {
  if (typeof window === 'undefined') return 'unknown';

  const ua = navigator.userAgent.toLowerCase();

  if (ua.includes('android')) return 'android';
  // ... other platform checks
};
```

### AppService Configuration

**File**: `src/services/nativeAppService.ts`

```typescript
class NativeAppService extends AppService {
  osPlatform: 'android';
  appPlatform: 'tauri';

  // Android-specific capabilities
  hasContextMenu: false;          // No native context menus
  hasRoundedWindow: false;        // No rounded corners
  hasSafeAreaInset: true;         // Safe area for status/nav bars
  hasHaptics: true;               // Haptic feedback support
  hasOrientationLock: true;       // Screen rotation lock
  hasScreenBrightness: true;      // Brightness control
  hasIAP: true;                   // Google Play In-App Purchase
  isMobileApp: true;
  isAndroidApp: true;
  distChannel: 'playstore';       // or 'readest' for sideload
}
```

## Native Bridge Integration

### Available Android APIs

**File**: `src/utils/bridge.ts`

| API Function | Purpose | Android Implementation |
|--------------|---------|------------------------|
| `copyURIToPath()` | Copy content URIs to cache | Tauri plugin command |
| `installPackage()` | Install APK files | Package manager integration |
| `setSystemUIVisibility()` | Show/hide status/nav bars | WindowInsetsController |
| `getStatusBarHeight()` | Get status bar height | WindowInsets API |
| `getSysFontsList()` | List installed fonts | System font directory scan |
| `interceptKeys()` | Capture hardware keys | KeyEvent interception |
| `lockScreenOrientation()` | Lock portrait/landscape | ActivityInfo orientation |
| `getSystemColorScheme()` | Dark/light mode detection | Configuration.uiMode |
| `getScreenBrightness()` | Current brightness level | WindowManager.LayoutParams |
| `setScreenBrightness()` | Set brightness | WindowManager.LayoutParams |
| `getExternalSDCardPath()` | External storage path | Environment.getExternalStorageDirectory |

### Content URI Handling

**Feature**: Open files from file managers and other apps (#829, #833)

**File**: `src/utils/bridge.ts`

```typescript
export const copyURIToPath = async (
  uri: string,
  destPath: string
): Promise<void> => {
  if (isAndroidPlatform()) {
    await invoke('plugin:native-bridge|copy_uri_to_path', {
      uri,
      destPath,
    });
  }
};
```

**Use Case**:
```typescript
// User shares an EPUB from another app
const contentUri = 'content://com.android.providers.downloads/...';
const cachePath = 'cache/temp.epub';

await copyURIToPath(contentUri, cachePath);
const book = await loadBook(cachePath);
```

**Implementation** (`src-tauri/plugins/tauri-plugin-native-bridge/src/mobile.rs`):
```rust
#[cfg(target_os = "android")]
pub fn copy_uri_to_path(uri: String, dest_path: String) -> Result<(), String> {
    // Use Android ContentResolver to copy content:// URI to file
}
```

**Related Commits**:
- #829: Support opening content URIs
- #833: Copy to cache when direct access fails

### System UI Control

**Feature**: Fullscreen immersive mode

**File**: `src/utils/bridge.ts`

```typescript
export const setSystemUIVisibility = async (visible: boolean): Promise<void> => {
  if (isAndroidPlatform()) {
    await invoke('plugin:native-bridge|set_system_ui_visibility', {
      visible,
    });
  }
};
```

**Usage**:
```typescript
// Enter fullscreen mode
await setSystemUIVisibility(false);

// Exit fullscreen mode
await setSystemUIVisibility(true);
```

**Implementation**:
- Hides status bar and navigation bar
- Uses `WindowInsetsController` (API 30+)
- Falls back to `SYSTEM_UI_FLAG_FULLSCREEN` (API < 30)

### Hardware Key Interception

**Feature**: Capture volume buttons and back button

**File**: `src/utils/bridge.ts`

```typescript
export const interceptKeys = async (
  keys: string[],
  intercept: boolean
): Promise<void> => {
  if (isAndroidPlatform()) {
    await invoke('plugin:native-bridge|intercept_keys', {
      keys,       // ['VOLUME_UP', 'VOLUME_DOWN', 'BACK']
      intercept,  // true = block default action
    });
  }
};
```

**Use Case**:
```typescript
// Use volume buttons for page navigation
await interceptKeys(['VOLUME_UP', 'VOLUME_DOWN'], true);

// Listen for key events
document.addEventListener('hardwarekey', (e) => {
  if (e.detail.key === 'VOLUME_UP') {
    goToNextPage();
  } else if (e.detail.key === 'VOLUME_DOWN') {
    goToPreviousPage();
  }
});
```

### Screen Orientation Lock

**Feature**: Lock screen to portrait or landscape

**File**: `src/utils/bridge.ts`

```typescript
export const lockScreenOrientation = async (
  orientation: 'portrait' | 'landscape' | 'auto'
): Promise<void> => {
  if (isAndroidPlatform()) {
    await invoke('plugin:native-bridge|lock_screen_orientation', {
      orientation,
    });
  }
};
```

**Usage**:
```typescript
// Lock to portrait mode
await lockScreenOrientation('portrait');

// Allow rotation
await lockScreenOrientation('auto');
```

### Brightness Control

**Feature**: Adjust screen brightness (#653)

**File**: `src/utils/bridge.ts`

```typescript
export const setScreenBrightness = async (brightness: number): Promise<void> => {
  if (isAndroidPlatform()) {
    await invoke('plugin:native-bridge|set_screen_brightness', {
      brightness, // 0.0 - 1.0
    });
  }
};

export const getScreenBrightness = async (): Promise<number> => {
  if (isAndroidPlatform()) {
    return await invoke('plugin:native-bridge|get_screen_brightness');
  }
  return 0.5;
};
```

**Usage**:
```typescript
// Dim screen for night reading
await setScreenBrightness(0.2);

// Restore brightness
const current = await getScreenBrightness();
await setScreenBrightness(current + 0.1);
```

## File System Integration

### Storage Locations

**File**: `src/services/nativeAppService.ts`

```typescript
const getAndroidBasePath = async (base: BaseDir): Promise<string> => {
  switch (base) {
    case 'Books':
      return await appDataDir() + '/Books';
    case 'Settings':
      return await appConfigDir() + '/Settings';
    case 'Data':
      return await appDataDir() + '/Data';
    case 'Fonts':
      return await appDataDir() + '/Fonts';
    case 'Images':
      return await appCacheDir() + '/Images';
    case 'Log':
      return await appLogDir();
    case 'Cache':
      return await appCacheDir();
    case 'Temp':
      return await tempDir();
    default:
      return await appDataDir();
  }
};
```

**Android Paths**:
- **Books**: `/data/data/com.readest.app/files/Books`
- **Settings**: `/data/data/com.readest.app/config/Settings`
- **Cache**: `/data/data/com.readest.app/cache`
- **External SD Card**: `/storage/emulated/0` or custom path

### External Storage Access

**File**: `src/utils/bridge.ts`

```typescript
export const getExternalSDCardPath = async (): Promise<string | null> => {
  if (isAndroidPlatform()) {
    return await invoke('plugin:native-bridge|get_external_sd_card_path');
  }
  return null;
};
```

**Usage**:
```typescript
// Allow users to import books from SD card
const sdCardPath = await getExternalSDCardPath();
if (sdCardPath) {
  const filePicker = await openFilePicker({
    directory: sdCardPath,
  });
}
```

**Permissions Required**:
- `READ_EXTERNAL_STORAGE`
- `WRITE_EXTERNAL_STORAGE` (API < 30)
- `MANAGE_EXTERNAL_STORAGE` (API 30+)

### File Picker

**Default File Chooser** (#798, #799, #807)

**File**: `src/services/nativeAppService.ts`

```typescript
async selectFiles(): Promise<string[]> {
  const selected = await open({
    multiple: true,
    filters: [
      {
        name: 'Books',
        extensions: ['epub', 'pdf', 'mobi', 'azw3', 'cbz', 'fb2', 'txt'],
      },
    ],
  });

  return Array.isArray(selected) ? selected : [selected];
}
```

**Behavior**:
- Opens Android's native file picker (DocumentsUI)
- Supports multiple file selection
- Filters by book format extensions
- Returns file paths or content URIs

**Related Commits**:
- #798: Open default file chooser on Android
- #799: Fix file picker integration
- #807: Handle both file paths and content URIs

## UI Optimizations

### Keyboard Handling

**Soft Keyboard in Custom CSS Textarea** (#653, #720, #762, #763)

**File**: `src/app/reader/components/settings/CustomCSSPanel.tsx`

**Issue**: Textarea hidden behind keyboard in fullscreen mode

**Solution**:
```typescript
<textarea
  className="custom-css-editor"
  onFocus={() => {
    if (isAndroidPlatform()) {
      // Scroll textarea into view when keyboard opens
      setTimeout(() => {
        element.scrollIntoView({ behavior: 'smooth', block: 'center' });
      }, 300);
    }
  }}
/>
```

**CSS**:
```css
/* Ensure textarea is visible when keyboard is open */
@media (max-height: 500px) {
  .custom-css-editor {
    max-height: 150px;
  }
}
```

### Safe Area Handling

**Status Bar and Navigation Bar Insets**

**File**: `src/styles/globals.css`

```css
:root {
  --sat: env(safe-area-inset-top);
  --sab: env(safe-area-inset-bottom);
  --sal: env(safe-area-inset-left);
  --sar: env(safe-area-inset-right);
}

/* Apply safe area insets */
.app-container {
  padding-top: var(--sat);
  padding-bottom: var(--sab);
  padding-left: var(--sal);
  padding-right: var(--sar);
}
```

**JavaScript Detection**:
```typescript
export const getSafeAreaInsets = async (): Promise<SafeAreaInsets> => {
  if (isAndroidPlatform()) {
    const insets = await invoke('plugin:native-bridge|get_safe_area_insets');
    return insets;
  }
  return { top: 0, bottom: 0, left: 0, right: 0 };
};
```

### Mobile UI Patterns

**Action Bar at Bottom** (#681)

**File**: `src/app/reader/components/MobileActionBar.tsx`

```typescript
function MobileActionBar() {
  const isAndroid = isAndroidPlatform();

  return (
    <div className={`action-bar ${isAndroid ? 'bottom' : 'top'}`}>
      {/* Annotation tools */}
    </div>
  );
}
```

**CSS**:
```css
.action-bar.bottom {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  padding-bottom: env(safe-area-inset-bottom);
}
```

## OAuth Integration

### Custom Tabs for OAuth

**Feature**: Initiate OAuth using Custom Tabs (#361, #788)

**File**: `src/services/auth.ts`

```typescript
async function initiateOAuth(provider: 'google' | 'apple' | 'github') {
  if (isAndroidPlatform()) {
    // Use Chrome Custom Tabs instead of external browser
    await invoke('plugin:native-bridge|open_custom_tab', {
      url: authUrl,
    });
  } else {
    window.open(authUrl);
  }
}
```

**Benefits**:
- Seamless in-app browser experience
- Pre-warmed for faster loading
- Shared cookie jar with Chrome
- Automatic credential autofill

**Implementation** (`src-tauri/plugins/tauri-plugin-native-bridge/src/mobile.rs`):
```rust
#[cfg(target_os = "android")]
pub fn open_custom_tab(url: String) -> Result<(), String> {
    // Launch Chrome Custom Tab with CustomTabsIntent
}
```

**Related Commits**:
- #361: Initiate OAuth using Custom Tabs
- #788: Fix OAuth flow on Android

## Font Management

### System Fonts Detection

**File**: `src/utils/bridge.ts`

```typescript
export const getSysFontsList = async (): Promise<string[]> => {
  if (isAndroidPlatform()) {
    return await invoke('plugin:native-bridge|get_sys_fonts_list');
  }
  return [];
};
```

**Implementation**:
- Scans `/system/fonts/` directory
- Parses `fonts.xml` for font families
- Returns list of available font names

**Android Font Families**:
```typescript
const ANDROID_FONTS = [
  'Roboto', 'Roboto Condensed', 'Roboto Mono', 'Roboto Slab',
  'Noto Sans', 'Noto Serif', 'Noto Sans CJK', 'Noto Serif CJK',
  'Droid Sans', 'Droid Serif', 'Droid Sans Mono',
];
```

### Free Fonts Filter

**File**: `src/components/settings/FontPanel.tsx`

```typescript
const NON_FREE_FONTS = ['Times New Roman', 'Arial', 'Verdana', 'Georgia'];

const filterNonFreeFonts = (font: string) => {
  return !['android', 'linux'].includes(getOSPlatform()) ||
         !NON_FREE_FONTS.includes(font);
};

const androidFonts = allFonts.filter(filterNonFreeFonts);
```

**Rationale**: Avoid licensing issues with proprietary fonts on Android

## Build and Deployment

### Build Configuration

**File**: `apps/readest-app/src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "android": {
      "minSdkVersion": 24,
      "targetSdkVersion": 34,
      "versionCode": 1,
      "permissions": [
        "android.permission.READ_EXTERNAL_STORAGE",
        "android.permission.WRITE_EXTERNAL_STORAGE",
        "android.permission.INTERNET",
        "android.permission.ACCESS_NETWORK_STATE",
        "android.permission.WAKE_LOCK"
      ]
    }
  }
}
```

### Build Commands

**Development**:
```bash
pnpm tauri android dev
```

**Production APK**:
```bash
pnpm tauri android build
```

**Google Play AAB**:
```bash
pnpm tauri android build --target aab
```

### GitHub Actions

**File**: `.github/workflows/android-release.yml` (#695)

```yaml
name: Android Release

on:
  push:
    tags:
      - 'v*'

jobs:
  build-android:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Setup Android SDK
        uses: android-actions/setup-android@v2
      - name: Build APK
        run: pnpm tauri android build
      - name: Sign APK
        run: |
          jarsigner -keystore ${{ secrets.KEYSTORE_PATH }} \
                    -storepass ${{ secrets.KEYSTORE_PASSWORD }} \
                    app-release-unsigned.apk release
      - name: Upload to Play Store
        uses: r0adkll/upload-google-play@v1
```

**Related Commits**:
- #695: Add Android release in GitHub Actions

## Performance Optimizations

### Mobile-Specific Settings

**File**: `src/services/constants.ts`

```typescript
export const DEFAULT_MOBILE_VIEW_SETTINGS: Partial<ViewSettings> = {
  fullJustification: false,       // Better performance
  animated: true,                 // Enable page animations
  defaultFont: 'Sans-serif',      // Roboto on Android
  marginBottomPx: 16,
  disableDoubleClick: true,       // Prevent accidental zooms
  fontSize: 16 * 1.25,            // 25% larger for mobile
};
```

### Rendering Optimizations

**File**: `src/app/reader/components/FoliateViewer.tsx`

```typescript
const isMobile = ['android', 'ios'].includes(getOSPlatform());

const viewConfig = {
  // Disable CSS transitions on Android for better performance
  animated: isMobile ? false : animated,

  // Reduce reflow on scroll
  fixedLayout: isMobile,

  // Lazy load images
  lazyImages: isMobile,
};
```

## Testing on Android

### Emulator Setup

```bash
# Create Android Virtual Device
avdmanager create avd -n Readest_Test -k "system-images;android-34;google_apis;x86_64"

# Launch emulator
emulator -avd Readest_Test

# Run app
pnpm tauri android dev
```

### Physical Device Testing

```bash
# Enable USB debugging on device
adb devices

# Install and run
pnpm tauri android dev --device <device-id>
```

### Debug Logging

**File**: `src-tauri/tauri.conf.json`

```json
{
  "plugins": {
    "log": {
      "targets": [
        {
          "target": "logcat",
          "level": "debug"
        }
      ]
    }
  }
}
```

**View logs**:
```bash
adb logcat | grep Readest
```

## Known Issues and Workarounds

### Issue: Keyboard Covers Input

**Symptom**: Soft keyboard hides text input fields

**Workaround**:
```typescript
window.addEventListener('resize', () => {
  if (isAndroidPlatform()) {
    const activeElement = document.activeElement;
    if (activeElement && activeElement.tagName === 'INPUT') {
      activeElement.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
  }
});
```

### Issue: Back Button Exits App

**Symptom**: Hardware back button closes app instead of navigating

**Workaround**:
```typescript
if (isAndroidPlatform()) {
  await interceptKeys(['BACK'], true);

  document.addEventListener('hardwarekey', (e) => {
    if (e.detail.key === 'BACK') {
      e.preventDefault();
      router.back();
    }
  });
}
```

### Issue: Content URI Permission Denied

**Symptom**: Cannot open files from file managers

**Workaround**:
```typescript
try {
  const content = await readFile(contentUri);
} catch (error) {
  // Copy to cache and retry
  const cachePath = `cache/${filename}`;
  await copyURIToPath(contentUri, cachePath);
  const content = await readFile(cachePath);
}
```

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### Native Android TTS Engine Integration (v0.9.56-0.9.57, #1376, #1387, #1394)

**Major Feature**: Integration with Android's native Text-to-Speech engine.

**Overview**: Users can now use the system's built-in TTS engine alongside Edge TTS and Web Speech API, with better offline support and battery efficiency.

**See**: [text-to-speech/index.md](../text-to-speech/index.md#native-android-tts) for full documentation.

**Benefits**:
- Offline TTS support with pre-installed voices
- Better battery efficiency
- Respects system TTS settings
- Works with Google TTS, Samsung TTS, and other system engines

**Implementation** (`src-tauri/src/android/tts.rs`):
```rust
use android_speech::TextToSpeech;

#[tauri::command]
pub async fn speak_android(text: String, voice: String, rate: f32) -> Result<()> {
    let tts = TextToSpeech::new()?;
    tts.set_voice(&voice)?;
    tts.set_speech_rate(rate)?;
    tts.speak(&text, QueueMode::Flush, None)?;
    Ok(())
}

#[tauri::command]
pub async fn get_android_voices() -> Result<Vec<Voice>> {
    let tts = TextToSpeech::new()?;
    Ok(tts.get_available_voices())
}
```

**Compatibility Fix** (v0.9.57, #1394):
- Resolved compatibility issues on Android 8-12
- Fixed voice enumeration on older devices
- Improved error handling and fallback logic

**Files**:
- `src-tauri/src/android/tts.rs`
- `src/app/reader/utils/tts/AndroidTTSBackend.ts`

### Overlay Scrollbar for TOC (v0.9.62, #1506)

**Feature**: Overlay scrollbar for Table of Contents on Android for better UI consistency.

**Implementation**:
- Scrollbar overlays content instead of taking space
- Auto-hides when not scrolling
- Touch-friendly scrollbar thumb

**CSS** (`src/styles/android-scrollbar.css`):
```css
.toc-list::-webkit-scrollbar {
  width: 8px;
}

.toc-list::-webkit-scrollbar-track {
  background: transparent;
}

.toc-list::-webkit-scrollbar-thumb {
  background: rgba(0, 0, 0, 0.3);
  border-radius: 4px;
}

.toc-list::-webkit-scrollbar-thumb:hover {
  background: rgba(0, 0, 0, 0.5);
}
```

**File**: `src/app/reader/components/sidebar/TOCView.tsx`

---

## Related Documentation

- **[Cross-Platform Support Index](./index.md)** - Overview of all platforms
- **[iOS Platform](./ios.md)** - iOS-specific features
- **[PWA Enhancements](./pwa-enhancements.md)** - Web platform support

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #361, #653, #695, #720, #762, #763, #788, #798, #799, #807, #829, #833
