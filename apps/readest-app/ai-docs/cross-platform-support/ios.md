# iOS Platform Support

**Platform Category**: Mobile, Cross-Platform
**Status**: Full Support
**Related Commits**: #410, #411, #428, #433, #443, #447, #502, #505, #547, #822

## Overview

Readest provides comprehensive iOS support through Tauri's mobile capabilities, featuring native iOS integration including Sign in with Apple, haptic feedback, background audio playback, and iOS-optimized UI patterns. The iOS version maintains feature parity with Android while leveraging iOS-specific capabilities.

## Platform Detection

### OS Platform Detection

**File**: `src/utils/ua.ts`

```typescript
export const getOSPlatform = (): OsPlatform => {
  if (typeof window === 'undefined') return 'unknown';

  const ua = navigator.userAgent.toLowerCase();

  if (ua.includes('iphone') || ua.includes('ipad') || ua.includes('ipod')) {
    return 'ios';
  }
  // ... other platform checks
};
```

### AppService Configuration

**File**: `src/services/nativeAppService.ts`

```typescript
class NativeAppService extends AppService {
  osPlatform: 'ios';
  appPlatform: 'tauri';

  // iOS-specific capabilities
  hasContextMenu: false;          // No native context menus
  hasRoundedWindow: false;        // No rounded corners
  hasSafeAreaInset: true;         // Safe area for notch/home indicator
  hasHaptics: true;               // Taptic Engine support
  hasOrientationLock: true;       // Screen rotation lock
  hasScreenBrightness: true;      // Brightness control
  hasIAP: true;                   // Apple In-App Purchase
  hasUpdater: false;              // Updates via App Store
  isMobileApp: true;
  isIOSApp: true;
  distChannel: 'appstore';        // or 'testflight' for beta
}
```

## Native Bridge Integration

### Available iOS APIs

**File**: `src/utils/bridge.ts`

| API Function | Purpose | iOS Implementation |
|--------------|---------|-------------------|
| `copyURIToPath()` | Copy documents to cache | File coordination |
| `invokeUseBackgroundAudio()` | Background audio playback | AVFoundation session |
| `setSystemUIVisibility()` | Show/hide status bar | UIViewController prefersStatusBarHidden |
| `getStatusBarHeight()` | Get status bar height | UIApplication statusBarFrame |
| `getSysFontsList()` | List installed fonts | UIFont.familyNames |
| `interceptKeys()` | Capture hardware keys | Volume button events |
| `lockScreenOrientation()` | Lock portrait/landscape | UIDevice orientation |
| `getSystemColorScheme()` | Dark/light mode detection | UITraitCollection userInterfaceStyle |
| `getSafeAreaInsets()` | Get safe area dimensions | UIView safeAreaInsets |
| `getScreenBrightness()` | Current brightness level | UIScreen brightness |
| `setScreenBrightness()` | Set brightness | UIScreen brightness |
| `performHapticFeedback()` | Trigger haptics | UIImpactFeedbackGenerator |

### Background Audio Playback

**Feature**: TTS works in background (#547, #822)

**File**: `src/utils/bridge.ts`

```typescript
export const invokeUseBackgroundAudio = async (): Promise<void> => {
  if (isIOSPlatform()) {
    await invoke('plugin:native-bridge|use_background_audio');
  }
};
```

**Implementation** (`src-tauri/plugins/tauri-plugin-native-bridge/src/mobile.rs`):
```rust
#[cfg(target_os = "ios")]
pub fn use_background_audio() -> Result<(), String> {
    // Configure AVAudioSession for background playback
    let audio_session = AVAudioSession::sharedInstance();
    audio_session.setCategory(
        AVAudioSessionCategoryPlayback,
        AVAudioSessionModeSpokenAudio,
        AVAudioSessionCategoryOptionMixWithOthers
    )?;
    audio_session.setActive(true)?;
    Ok(())
}
```

**Effect**:
- TTS continues playing when app is in background
- Audio ducking when other apps play
- Control Center integration
- Lock screen playback controls

**Related Commits**:
- #547: TTS now works in background in iOS
- #822: Fix TTS now works in background in iOS

**User Instructions**:
> To enable background playback, go to Settings > Readest > Background App Refresh, and enable it.

### Haptic Feedback

**Feature**: Taptic Engine integration (#428)

**File**: `src/utils/bridge.ts`

```typescript
export const performHapticFeedback = async (
  style: 'light' | 'medium' | 'heavy' | 'soft' | 'rigid' | 'success' | 'warning' | 'error'
): Promise<void> => {
  if (isIOSPlatform()) {
    await invoke('plugin:native-bridge|perform_haptic_feedback', {
      style,
    });
  }
};
```

**Usage**:
```typescript
// Page turn feedback
await performHapticFeedback('light');

// Bookmark added
await performHapticFeedback('success');

// Error occurred
await performHapticFeedback('error');
```

**Implementation**:
- **Light**: `UIImpactFeedbackGenerator` with `.light` style
- **Medium**: `UIImpactFeedbackGenerator` with `.medium` style
- **Heavy**: `UIImpactFeedbackGenerator` with `.heavy` style
- **Soft**: `UIImpactFeedbackGenerator` with `.soft` style (iOS 13+)
- **Rigid**: `UIImpactFeedbackGenerator` with `.rigid` style (iOS 13+)
- **Success**: `UINotificationFeedbackGenerator` with `.success` type
- **Warning**: `UINotificationFeedbackGenerator` with `.warning` type
- **Error**: `UINotificationFeedbackGenerator` with `.error` type

**Related Commits**:
- #428: Haptics feedback for interactions

### Safe Area Insets

**Feature**: Notch and home indicator handling

**File**: `src/utils/bridge.ts`

```typescript
export interface SafeAreaInsets {
  top: number;
  bottom: number;
  left: number;
  right: number;
}

export const getSafeAreaInsets = async (): Promise<SafeAreaInsets> => {
  if (isIOSPlatform()) {
    return await invoke('plugin:native-bridge|get_safe_area_insets');
  }
  return { top: 0, bottom: 0, left: 0, right: 0 };
};
```

**Implementation**:
```rust
#[cfg(target_os = "ios")]
pub fn get_safe_area_insets() -> Result<SafeAreaInsets, String> {
    let window = UIApplication::sharedApplication().keyWindow();
    let insets = window.safeAreaInsets();
    Ok(SafeAreaInsets {
        top: insets.top,
        bottom: insets.bottom,
        left: insets.left,
        right: insets.right,
    })
}
```

**CSS Application**:
```css
:root {
  --sat: env(safe-area-inset-top);
  --sab: env(safe-area-inset-bottom);
  --sal: env(safe-area-inset-left);
  --sar: env(safe-area-inset-right);
}

.app-container {
  padding-top: var(--sat);
  padding-bottom: max(var(--sab), 16px); /* At least 16px for home indicator */
  padding-left: var(--sal);
  padding-right: var(--sar);
}
```

### Screen Brightness Control

**Feature**: Adjust screen brightness (#505)

**File**: `src/utils/bridge.ts`

```typescript
export const setScreenBrightness = async (brightness: number): Promise<void> => {
  if (isIOSPlatform()) {
    await invoke('plugin:native-bridge|set_screen_brightness', {
      brightness, // 0.0 - 1.0
    });
  }
};

export const getScreenBrightness = async (): Promise<number> => {
  if (isIOSPlatform()) {
    return await invoke('plugin:native-bridge|get_screen_brightness');
  }
  return 0.5;
};
```

**Implementation**:
```rust
#[cfg(target_os = "ios")]
pub fn set_screen_brightness(brightness: f64) -> Result<(), String> {
    let screen = UIScreen::mainScreen();
    screen.setBrightness(brightness);
    Ok(())
}
```

**Related Commits**:
- #505: Release wakelock when window or tab lost focus

### Screen Orientation Lock

**Feature**: Lock portrait or landscape mode

**File**: `src/utils/bridge.ts`

```typescript
export const lockScreenOrientation = async (
  orientation: 'portrait' | 'landscape' | 'auto'
): Promise<void> => {
  if (isIOSPlatform()) {
    await invoke('plugin:native-bridge|lock_screen_orientation', {
      orientation,
    });
  }
};
```

**Implementation**:
```rust
#[cfg(target_os = "ios")]
pub fn lock_screen_orientation(orientation: String) -> Result<(), String> {
    let mask = match orientation.as_str() {
        "portrait" => UIInterfaceOrientationMaskPortrait,
        "landscape" => UIInterfaceOrientationMaskLandscape,
        "auto" => UIInterfaceOrientationMaskAll,
        _ => return Err("Invalid orientation".to_string()),
    };

    // Update supported orientations
    UIDevice::currentDevice().setValue(
        NSNumber::numberWithInt(UIDeviceOrientationUnknown),
        forKey: "orientation"
    );
    UIDevice::currentDevice().setValue(
        NSNumber::numberWithInt(mask),
        forKey: "orientation"
    );

    Ok(())
}
```

## Authentication

### Sign in with Apple

**Feature**: Native Apple ID authentication (#411, #433, #443)

**File**: `src/services/auth.ts`

```typescript
async function signInWithApple(): Promise<User> {
  if (isIOSPlatform()) {
    // Use native Sign in with Apple
    const result = await invoke('plugin:safari-auth|sign_in_with_apple');
    return processAppleAuth(result);
  } else {
    // Fall back to web OAuth
    return signInWithAppleWeb();
  }
}
```

**Implementation** (`src-tauri/plugins/tauri-plugin-safari-auth/src/mobile.rs`):
```rust
#[cfg(target_os = "ios")]
pub fn sign_in_with_apple() -> Result<AppleAuthResult, String> {
    use AuthenticationServices::*;

    let provider = ASAuthorizationAppleIDProvider::new();
    let request = provider.createRequest();
    request.setRequestedScopes([
        ASAuthorizationScopeFullName,
        ASAuthorizationScopeEmail,
    ]);

    let controller = ASAuthorizationController::new([request]);
    controller.performRequests();

    // Wait for completion
    // Returns: { identityToken, authorizationCode, user }
}
```

**Benefits**:
- Native iOS UI (Face ID / Touch ID support)
- Seamless user experience
- Privacy-preserving (hide email option)
- Fast authentication

**Related Commits**:
- #411: Native Sign in with Apple
- #433: Safari-auth plugin for native OAuth
- #443: Fix Safari-auth plugin for OAuth flow

### Deep Link OAuth

**Feature**: Custom URL scheme for OAuth callbacks (#433, #443)

**File**: `src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "iOS": {
      "cfBundleURLTypes": [
        {
          "CFBundleURLSchemes": ["readest"],
          "CFBundleURLName": "com.readest.app"
        }
      ]
    }
  }
}
```

**Implementation**:
```typescript
// Handle deep link
window.addEventListener('deeplink', (event) => {
  const url = new URL(event.detail.url);
  if (url.pathname === '/auth/callback') {
    const code = url.searchParams.get('code');
    processOAuthCallback(code);
  }
});
```

**OAuth Flow**:
1. App opens Safari with OAuth provider URL
2. User authenticates
3. Provider redirects to `readest://auth/callback?code=...`
4. iOS opens app with deep link
5. App processes OAuth code

## UI Optimizations

### iOS-Specific Modals

**Feature**: iOS-optimized modals and annotation tools (#447)

**File**: `src/app/reader/components/annotator/AnnotatorModal.tsx`

```typescript
function AnnotatorModal({ open, onClose }: AnnotatorModalProps) {
  const isIOS = isIOSPlatform();

  return (
    <Modal
      open={open}
      onClose={onClose}
      className={isIOS ? 'ios-modal' : 'default-modal'}
    >
      {/* Modal content */}
    </Modal>
  );
}
```

**CSS**:
```css
/* iOS-specific modal styling */
.ios-modal {
  border-radius: 14px;          /* iOS corner radius */
  backdrop-filter: blur(20px);  /* iOS frosted glass */
  -webkit-backdrop-filter: blur(20px);
}

.ios-modal .modal-header {
  padding-top: max(16px, env(safe-area-inset-top));
}

.ios-modal .modal-footer {
  padding-bottom: max(16px, env(safe-area-inset-bottom));
}
```

**Related Commits**:
- #447: iOS-optimized modals and annotation tools

### Paging Animations

**Feature**: Page turn animations (#410)

**File**: `src/app/reader/components/FoliateViewer.tsx`

```typescript
const viewSettings = {
  animated: isIOSPlatform() ? true : animated,  // Enabled by default on iOS
  // ... other settings
};
```

**CSS Animations**:
```css
@keyframes page-turn {
  from {
    transform: translateX(100%);
    opacity: 0;
  }
  to {
    transform: translateX(0);
    opacity: 1;
  }
}

.page-turn-animation {
  animation: page-turn 0.3s cubic-bezier(0.4, 0.0, 0.2, 1);
}
```

**Performance**:
- Uses GPU-accelerated transforms
- 60 FPS on iOS devices
- Disabled on older devices (< iOS 14)

**Related Commits**:
- #410: Paging animations enabled by default

### Pull-Down to Dismiss

**Feature**: Pull-down gesture to dismiss modals (#440)

**File**: `src/components/Modal.tsx`

```typescript
function Modal({ open, onClose }: ModalProps) {
  const isIOS = isIOSPlatform();
  const [startY, setStartY] = useState(0);
  const [currentY, setCurrentY] = useState(0);

  const handleTouchStart = (e: TouchEvent) => {
    if (isIOS) {
      setStartY(e.touches[0].clientY);
    }
  };

  const handleTouchMove = (e: TouchEvent) => {
    if (isIOS) {
      const y = e.touches[0].clientY;
      setCurrentY(y);

      // Allow pull-down if scrolled to top
      const modalContent = e.currentTarget as HTMLElement;
      if (modalContent.scrollTop === 0 && y > startY) {
        e.preventDefault();
        // Apply drag transform
        modalContent.style.transform = `translateY(${y - startY}px)`;
      }
    }
  };

  const handleTouchEnd = () => {
    if (isIOS) {
      const dragDistance = currentY - startY;
      if (dragDistance > 100) {
        // Dismiss modal
        onClose();
        // Add haptic feedback
        performHapticFeedback('light');
      } else {
        // Snap back
        const modalContent = document.querySelector('.modal-content');
        modalContent.style.transform = 'translateY(0)';
      }
    }
  };

  return (
    <div
      className="modal"
      onTouchStart={handleTouchStart}
      onTouchMove={handleTouchMove}
      onTouchEnd={handleTouchEnd}
    >
      {/* Modal content */}
    </div>
  );
}
```

**Related Commits**:
- #440: Pull-down gesture to dismiss modals

### iOS Font Scaling

**File**: `src/utils/style.ts`

```typescript
const isIOS = getOSPlatform() === 'ios';
const fontScale = isIOS ? 1.25 : 1;  // 25% larger on iOS

const fontSize = baseFontSize * fontScale;
```

**Rationale**: iOS devices have higher pixel density, requiring larger font sizes for readability.

## File System Integration

### Storage Locations

**File**: `src/services/nativeAppService.ts`

```typescript
const getIOSBasePath = async (base: BaseDir): Promise<string> => {
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

**iOS Paths**:
- **Books**: `~/Library/Application Support/com.readest.app/Books`
- **Settings**: `~/Library/Application Support/com.readest.app/Settings`
- **Cache**: `~/Library/Caches/com.readest.app`
- **Documents**: `~/Documents` (user-accessible via Files app)

### iCloud Drive Integration

**Configuration** (`Info.plist`):
```xml
<key>NSUbiquitousContainers</key>
<dict>
    <key>iCloud.com.readest.app</key>
    <dict>
        <key>NSUbiquitousContainerName</key>
        <string>Readest</string>
    </dict>
</dict>
```

**Implementation**:
```typescript
export const syncToiCloud = async (): Promise<void> => {
  if (isIOSPlatform()) {
    await invoke('plugin:native-bridge|sync_to_icloud');
  }
};
```

**Files Synced**:
- Reading progress
- Annotations and highlights
- Settings and preferences
- Book metadata (not book files)

### File Picker

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
- Opens iOS Files app (UIDocumentPickerViewController)
- Supports iCloud Drive, local files, and third-party providers
- Handles file coordination for sandboxed access

## Build and Deployment

### Build Configuration

**File**: `apps/readest-app/src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "iOS": {
      "minimumSystemVersion": "13.0",
      "developmentTeam": "YOUR_TEAM_ID",
      "bundleIdentifier": "com.readest.app",
      "infoPlist": {
        "NSCameraUsageDescription": "Readest needs camera access to scan book covers",
        "NSMicrophoneUsageDescription": "Readest needs microphone access for voice notes",
        "NSPhotoLibraryUsageDescription": "Readest needs photo library access to import book covers"
      }
    }
  }
}
```

### Build Commands

**Development (Simulator)**:
```bash
pnpm tauri ios dev
```

**Development (Physical Device)**:
```bash
pnpm tauri ios dev --device <device-id>
```

**Production IPA**:
```bash
pnpm tauri ios build
```

**App Store Build**:
```bash
pnpm tauri ios build --target ios-arm64 --release
```

### Code Signing

**Xcode Configuration**:
1. Open `src-tauri/gen/apple/` in Xcode
2. Select project → Signing & Capabilities
3. Set Team and Bundle Identifier
4. Enable required capabilities:
   - Push Notifications
   - Background Modes (Audio, Airplay, Picture in Picture)
   - App Groups (for widget support)

### TestFlight Deployment

**File**: `fastlane/Fastfile`

```ruby
platform :ios do
  desc "Upload to TestFlight"
  lane :beta do
    build_app(
      scheme: "readest",
      export_method: "app-store"
    )
    upload_to_testflight
  end
end
```

**Run**:
```bash
fastlane ios beta
```

## Performance Optimizations

### Scroll Performance

**File**: `src/app/reader/components/FoliateViewer.tsx`

```typescript
const viewConfig = {
  // Enable momentum scrolling on iOS
  scrollMode: isIOSPlatform() ? 'momentum' : 'normal',

  // Use native scroll on iOS for 60 FPS
  nativeScroll: isIOSPlatform(),
};
```

### Rendering Optimizations

**GPU Acceleration**:
```css
.reader-content {
  /* Force GPU layer on iOS */
  -webkit-transform: translateZ(0);
  transform: translateZ(0);

  /* Enable hardware acceleration */
  -webkit-perspective: 1000;
  perspective: 1000;

  /* Smooth scrolling */
  -webkit-overflow-scrolling: touch;
}
```

### Memory Management

**File**: `src/app/reader/components/FoliateViewer.tsx`

```typescript
useEffect(() => {
  // Cleanup on iOS to prevent memory leaks
  return () => {
    if (isIOSPlatform()) {
      // Clear image caches
      view.destroy();

      // Force garbage collection (iOS only)
      window.webkit?.messageHandlers?.gc?.postMessage(null);
    }
  };
}, []);
```

## Testing on iOS

### Simulator Setup

```bash
# List available simulators
xcrun simctl list devices

# Boot simulator
xcrun simctl boot "iPhone 15 Pro"

# Run app
pnpm tauri ios dev
```

### Physical Device Testing

```bash
# List connected devices
xcrun xctrace list devices

# Run on device
pnpm tauri ios dev --device "Your iPhone"
```

### Debug Logging

**Safari Web Inspector**:
1. Enable on iPhone: Settings → Safari → Advanced → Web Inspector
2. Connect to Mac
3. Safari → Develop → [Your iPhone] → Readest

**Xcode Console**:
1. Open Xcode
2. Window → Devices and Simulators
3. Select device → View Device Logs

## Known Issues and Workarounds

### Issue: Keyboard Covers Input

**Symptom**: Keyboard hides text input on smaller iPhones

**Workaround**:
```typescript
const [keyboardHeight, setKeyboardHeight] = useState(0);

useEffect(() => {
  const handleKeyboardWillShow = (e: KeyboardEvent) => {
    setKeyboardHeight(e.keyboardHeight);
  };

  const handleKeyboardWillHide = () => {
    setKeyboardHeight(0);
  };

  if (isIOSPlatform()) {
    window.addEventListener('keyboardWillShow', handleKeyboardWillShow);
    window.addEventListener('keyboardWillHide', handleKeyboardWillHide);
  }

  return () => {
    window.removeEventListener('keyboardWillShow', handleKeyboardWillShow);
    window.removeEventListener('keyboardWillHide', handleKeyboardWillHide);
  };
}, []);

// Apply padding
<div style={{ paddingBottom: keyboardHeight }}>
  {/* Content */}
</div>
```

### Issue: Safari 100vh Issue

**Symptom**: `100vh` includes browser UI, causing content to be cut off

**Workaround**:
```css
/* Use dvh (dynamic viewport height) on iOS */
@supports (height: 100dvh) {
  .full-height {
    height: 100dvh;
  }
}

@supports not (height: 100dvh) {
  .full-height {
    height: calc(100vh - env(safe-area-inset-bottom));
  }
}
```

### Issue: Tap Delay on iOS

**Symptom**: 300ms delay on tap events

**Workaround**:
```css
/* Disable tap delay */
* {
  -webkit-tap-highlight-color: transparent;
  touch-action: manipulation;
}
```

## Version 0.9.44 - 0.9.63 Updates (def157ca → f5b686ab)

### iPad Split-Screen Mode (v0.9.58, #1416)

**Major Feature**: Native iPad split-screen and slide-over support.

**Overview**: Readest now fully supports iPad's multitasking features, allowing users to run Readest side-by-side with other apps.

**Supported Modes**:
1. **Split View**: Readest alongside another app (50/50 or 70/30 split)
2. **Slide Over**: Readest as floating window over another app
3. **Picture in Picture**: Video/media playback in floating window (if applicable)

**Implementation** (`src-tauri/src/ios/multitasking.rs`):
```rust
// Enable multitasking in Info.plist
<key>UIRequiresFullScreen</key>
<false/>

<key>UISupportedInterfaceOrientations~ipad</key>
<array>
  <string>UIInterfaceOrientationPortrait</string>
  <string>UIInterfaceOrientationPortraitUpsideDown</string>
  <string>UIInterfaceOrientationLandscapeLeft</string>
  <string>UIInterfaceOrientationLandscapeRight</string>
</array>
```

**Responsive Layout**:
- Adaptive sidebar width based on available space
- Reflow content when split ratio changes
- Hide/show UI elements based on compact vs regular size classes

**Window Size Detection** (`src/hooks/useWindowSize.ts`):
```typescript
const useWindowSize = () => {
  const [size, setSize] = useState({ width: window.innerWidth, height: window.innerHeight });

  useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });

      // Adjust layout for iPad split-screen
      if (isIPad && window.innerWidth < 768) {
        // Compact mode: single column, hide sidebar
        adjustForCompactMode();
      }
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return size;
};
```

**Files**:
- `src-tauri/Info.plist` - Multitasking enablement
- `src/components/Layout.tsx` - Responsive layout
- `src/hooks/useWindowSize.ts` - Size detection

### Resizable Sidebars on iPad (v0.9.58, #1415)

**Feature**: Drag to resize sidebar width on iPad for better multitasking.

**Implementation**:
- Drag handle on sidebar edge
- Min width: 200px, Max width: 400px
- Preference persisted per device
- Smooth animation during resize

**Drag Handler** (`src/components/Sidebar.tsx`):
```typescript
const handleDrag = (e: TouchEvent) => {
  const deltaX = e.touches[0].clientX - dragStartX;
  const newWidth = Math.max(200, Math.min(400, initialWidth + deltaX));

  setSidebarWidth(newWidth);
  localStorage.setItem('sidebarWidth_iPad', String(newWidth));
};
```

**UI**:
- Vertical drag handle with subtle indicator
- Touch-friendly hit area (44px min)
- Visual feedback during drag

**Fix** (v0.9.58, #1415):
- Previously, sidebars couldn't be dragged on iPad due to touch event conflicts
- Resolved by separating touch handlers for drag vs scroll

**File**: `src/components/Sidebar.tsx`

### Safe Area Insets Support (v0.9.58, #1408)

**Feature**: Responsive safe area insets for notch and home indicator on modern iPads.

**Implementation**:
- Use `env(safe-area-inset-*)` CSS variables
- Automatic padding adjustment
- Per-orientation handling

**CSS** (`src/styles/ios-safe-area.css`):
```css
.reader-container {
  padding-top: env(safe-area-inset-top);
  padding-bottom: env(safe-area-inset-bottom);
  padding-left: env(safe-area-inset-left);
  padding-right: env(safe-area-inset-right);
}

/* iPad Pro 12.9" landscape safe areas */
@media (orientation: landscape) and (min-width: 1024px) {
  .header {
    padding-left: max(24px, env(safe-area-inset-left));
    padding-right: max(24px, env(safe-area-inset-right));
  }
}
```

**Platform Detection** (`src/utils/platform.ts`):
```typescript
const hasNotch = () => {
  // Detect iPad models with notch/Face ID
  const safeAreaTop = getComputedStyle(document.documentElement)
    .getPropertyValue('--sat') || '0px';

  return parseInt(safeAreaTop) > 20;
};
```

**Affected Components**:
- Header bar
- Footer bar
- Sidebar
- Modal dialogs
- Full-screen reader view

**Files**:
- `src/styles/ios-safe-area.css`
- `src/components/Header.tsx`
- `src/components/Footer.tsx`

### Import Reliability Improvements (v0.9.59, #1439)

**Fix**: More reliable book imports on iOS at first launch.

**Issue**: Books imported from Files app or shared via Share Sheet occasionally failed to import on fresh install.

**Root Cause**:
- File permissions not granted before import attempt
- Async initialization race condition
- Temporary file cleanup too aggressive

**Solution**:
```typescript
// Wait for file system ready before allowing import
const initializeImport = async () => {
  // Request file access permission first
  await requestFilePermission();

  // Ensure temp directory exists
  await ensureTempDirectory();

  // Initialize import handlers
  registerImportHandlers();
};

// Call on app start
await initializeImport();
```

**Additional Improvements**:
- Longer timeout for file copy operations
- Better error messages for permission issues
- Automatic retry on transient failures

**Files**:
- `src-tauri/src/ios/import.rs`
- `src/services/importService.ts`

### Smoother Orientation Changes (v0.9.59, #1441)

**Enhancement**: Smoother transition when rotating iPad between portrait and landscape.

**Issues Fixed**:
- Flash of unstyled content during rotation
- Layout shift and reflow jank
- Reading position lost during orientation change

**Implementation**:
```typescript
const handleOrientationChange = () => {
  // Save current reading position
  const currentCFI = view.getCurrentCFI();

  // Prevent layout until transition complete
  view.freeze();

  // Update layout for new orientation
  requestAnimationFrame(() => {
    view.updateLayout();

    // Restore reading position
    view.goToCFI(currentCFI);

    // Unfreeze after layout stable
    setTimeout(() => view.unfreeze(), 100);
  });
};

window.addEventListener('orientationchange', handleOrientationChange);
```

**CSS Transitions**:
```css
/* Smooth orientation transition */
@media (prefers-reduced-motion: no-preference) {
  .reader-view {
    transition: width 0.3s ease-out, height 0.3s ease-out;
  }
}
```

**Files**:
- `src/app/reader/components/FoliateViewer.tsx`
- `src/hooks/useOrientation.ts`

### Splash Screen and Icon Improvements (v0.9.60, #1450)

**Enhancement**: Updated splash screen and app icon for better iOS integration.

**Changes**:
1. **Splash Screen**:
   - Dismiss splash screen programmatically when app ready
   - Smooth fade-out transition
   - Proper background color matching app theme

2. **Icon Background**:
   - Changed from transparent to solid color
   - Better visibility in task switcher
   - Consistent with iOS design guidelines

**Implementation** (`src-tauri/src/ios/splash.rs`):
```rust
use tauri_plugin_splash::SplashExt;

#[tauri::command]
fn dismiss_splash(app: AppHandle) {
    // Dismiss after app initialization complete
    app.splash().dismiss();
}
```

**Frontend** (`src/App.tsx`):
```typescript
useEffect(() => {
  const initialize = async () => {
    // Load critical resources
    await loadSettings();
    await loadLibrary();

    // Dismiss splash screen
    if (isIOS) {
      invoke('dismiss_splash');
    }
  };

  initialize();
}, []);
```

**Assets**:
- `src-tauri/icons/AppIcon.iconset/` - Updated icon set
- `src-tauri/assets/LaunchScreen.storyboard` - Splash screen

**Files**:
- `src-tauri/src/ios/splash.rs`
- `src-tauri/icons/` - Icon assets

### iPad Layout Tweaks (v0.9.59, #1446, #1447)

**Enhancements**: Various layout optimizations specifically for iPad.

**Changes**:
1. **Ignore System Insets When No Header/Footer** (#1447):
   - When header/footer hidden, use full screen space
   - No unnecessary padding in immersive mode
   - Better space utilization

2. **Adaptive Font Sizes**:
   - Larger fonts for iPad vs iPhone
   - Scale based on screen size
   - Respect accessibility settings

3. **Touch Target Sizes**:
   - Minimum 44×44pt touch targets
   - Increased spacing for iPad
   - Better one-handed operation

**Implementation** (`src/styles/ipad-layout.css`):
```css
@media (min-width: 768px) {
  /* iPad and larger */
  .button {
    min-height: 44px;
    min-width: 44px;
    padding: 12px 24px;
  }

  .text-base {
    font-size: 17px; /* vs 15px on iPhone */
  }
}

@media (min-width: 1024px) {
  /* iPad Pro and larger */
  .sidebar {
    width: 320px; /* vs 280px on smaller iPads */
  }
}
```

**Files**:
- `src/styles/ipad-layout.css`
- `src/components/Layout.tsx`

---

## Related Documentation

- **[Cross-Platform Support Index](./index.md)** - Overview of all platforms
- **[Android Platform](./android.md)** - Android-specific features
- **[PWA Enhancements](./pwa-enhancements.md)** - Web platform support

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #410, #411, #428, #433, #443, #447, #502, #505, #547, #822
