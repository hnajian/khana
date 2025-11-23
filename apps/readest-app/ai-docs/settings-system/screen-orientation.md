# Sub-Feature: Screen Orientation

## Overview

Screen Orientation control allows users to lock the screen orientation or follow system settings. Added in v0.9.39 (commit #1034), this feature provides flexibility for reading in different positions and on different devices.

## Key Components

### Primary Files

- **`src/app/reader/components/FoliateViewer.tsx`** - Screen orientation lock implementation
- **`src/types/settings.ts`** - Orientation setting type definition
- **`src/store/settingsStore.ts`** - Orientation preference storage

### Related Files

- **`src/services/nativeAppService.ts`** - Native platform orientation APIs
- **Platform-specific**:
  - iOS: CoreMotion framework for orientation detection
  - Android: Activity orientation settings
  - Web: Screen Orientation API

## Architecture

### Orientation Modes

**Three Orientation Options**:

1. **Auto** (System Settings)
   - Follows device auto-rotation settings
   - Default for library page
   - Allows natural device rotation

2. **Portrait**
   - Locks screen to portrait orientation
   - Useful for vertical reading
   - Common for mobile reading

3. **Landscape**
   - Locks screen to landscape orientation
   - Useful for tablets and wide screens
   - Better for two-column layouts

### Implementation

**Setting Type** (`src/types/settings.ts`):
```typescript
interface SystemSettings {
  // ... other settings
  screenOrientation: 'auto' | 'portrait' | 'landscape';
}
```

**Orientation Lock** (commit 5e04f6ae):
```typescript
// In FoliateViewer.tsx or reader component
useEffect(() => {
  const lockOrientation = async () => {
    if (settings.screenOrientation === 'auto') {
      await screen.orientation.unlock();
    } else {
      await screen.orientation.lock(
        settings.screenOrientation === 'portrait'
          ? 'portrait-primary'
          : 'landscape-primary'
      );
    }
  };

  lockOrientation();

  return () => {
    screen.orientation.unlock(); // Cleanup
  };
}, [settings.screenOrientation]);
```

### System Integration

**Auto Orientation Following System** (commit da49526f):

Issue: App orientation didn't respect device system settings
Solution: Implemented system settings listener

**Platform-Specific Implementation**:

#### iOS (CoreMotion):
```swift
// Monitor device orientation changes
NotificationCenter.default.addObserver(
  forName: UIDevice.orientationDidChangeNotification,
  object: nil,
  queue: .main
) { _ in
  // Update UI orientation
}
```

#### Android (Activity):
```kotlin
// In MainActivity.kt
if (orientation == "auto") {
  requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED
} else if (orientation == "portrait") {
  requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_PORTRAIT
} else {
  requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE
}
```

#### Web (Screen Orientation API):
```typescript
if ('orientation' in screen) {
  await screen.orientation.lock('portrait-primary');
} else {
  // Fallback: Use CSS or show message
  console.warn('Screen Orientation API not supported');
}
```

### Page-Specific Orientation

**Library Page Exception** (commit 6080f9e0):

Issue: Library page was locked to reader orientation
Fix: Unlock orientation when navigating to library

```typescript
// In library page component
useEffect(() => {
  // Always unlock on library page
  screen.orientation.unlock();

  return () => {
    // Restore reader orientation lock when leaving
    const readerOrientation = settingsStore.getState().screenOrientation;
    if (readerOrientation !== 'auto') {
      screen.orientation.lock(readerOrientation);
    }
  };
}, []);
```

## AI Agent Modification Guidelines

### Adding Orientation Options

To add more orientation modes (e.g., reverse portrait):

1. **Update type definition**:
   ```typescript
   type ScreenOrientation =
     | 'auto'
     | 'portrait'
     | 'portrait-reverse'
     | 'landscape'
     | 'landscape-reverse';
   ```

2. **Map to Screen Orientation API**:
   ```typescript
   const ORIENTATION_MAP = {
     'portrait': 'portrait-primary',
     'portrait-reverse': 'portrait-secondary',
     'landscape': 'landscape-primary',
     'landscape-reverse': 'landscape-secondary'
   };

   await screen.orientation.lock(ORIENTATION_MAP[orientation]);
   ```

3. **Add UI control**:
   ```typescript
   <select
     value={screenOrientation}
     onChange={(e) => setScreenOrientation(e.target.value)}
   >
     <option value="auto">Auto (System)</option>
     <option value="portrait">Portrait</option>
     <option value="portrait-reverse">Portrait (Reverse)</option>
     <option value="landscape">Landscape</option>
     <option value="landscape-reverse">Landscape (Reverse)</option>
   </select>
   ```

### Implementing Per-Book Orientation

To allow different orientation per book:

1. **Add to BookConfig**:
   ```typescript
   interface BookConfig {
     // ... existing
     screenOrientation?: ScreenOrientation; // Override global
   }
   ```

2. **Apply book-specific orientation**:
   ```typescript
   const effectiveOrientation =
     bookConfig.screenOrientation ||
     systemSettings.screenOrientation;

   await applyOrientation(effectiveOrientation);
   ```

3. **Add UI in book settings**:
   ```typescript
   <label>
     Book Orientation Override:
     <select
       value={bookConfig.screenOrientation || 'default'}
       onChange={(e) => updateBookConfig({
         screenOrientation: e.target.value
       })}
     >
       <option value="default">Use Global Setting</option>
       <option value="auto">Auto</option>
       <option value="portrait">Portrait</option>
       <option value="landscape">Landscape</option>
     </select>
   </label>
   ```

### Handling Orientation Change Events

To respond to orientation changes:

1. **Listen for orientation changes**:
   ```typescript
   useEffect(() => {
     const handleOrientationChange = () => {
       const currentOrientation = screen.orientation.type;
       console.log('Orientation changed to:', currentOrientation);

       // Adjust layout if needed
       if (currentOrientation.startsWith('landscape')) {
         setColumnsPerPage(2);
       } else {
         setColumnsPerPage(1);
       }
     };

     screen.orientation.addEventListener('change', handleOrientationChange);

     return () => {
       screen.orientation.removeEventListener('change', handleOrientationChange);
     };
   }, []);
   ```

2. **Debounce rapid changes**:
   ```typescript
   const debouncedOrientationChange = useMemo(
     () => debounce(handleOrientationChange, 300),
     []
   );

   screen.orientation.addEventListener('change', debouncedOrientationChange);
   ```

### Fallback for Unsupported Browsers

To handle browsers without Screen Orientation API:

1. **Feature detection**:
   ```typescript
   const supportsOrientationAPI = 'orientation' in screen;

   if (!supportsOrientationAPI) {
     // Show message or use alternative method
     console.warn('Screen Orientation API not supported');
   }
   ```

2. **CSS-based fallback**:
   ```typescript
   // Add meta tag for mobile browsers
   <meta
     name="screen-orientation"
     content={screenOrientation === 'portrait' ? 'portrait' : 'landscape'}
   />
   ```

3. **Show notification**:
   ```typescript
   {!supportsOrientationAPI && (
     <div className="info-banner">
       Screen orientation lock is not supported in your browser.
       Please rotate your device manually.
     </div>
   )}
   ```

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add orientation mode | `src/types/settings.ts` | `FoliateViewer.tsx` |
| Per-book orientation | `src/types/book.ts` | `BookConfig`, `readerStore.ts` |
| Orientation events | `FoliateViewer.tsx` | `readerStore.ts` |
| Platform-specific impl | `nativeAppService.ts` | Native bridge plugins |
| UI controls | Settings panels | `MiscPanel.tsx` |

## Common Issues and Debugging

### Problem: Orientation lock not working

- Check if Screen Orientation API is supported
- Verify browser permissions for orientation lock
- Check if page is in fullscreen mode (required for some browsers)
- Inspect console for orientation lock errors

### Problem: Orientation unlocks unexpectedly

- Verify cleanup functions are not called prematurely
- Check if navigation is unlocking orientation
- Ensure orientation lock is reapplied after suspension

### Problem: Library page locked to reader orientation

- Verify library page unlocks orientation in useEffect
- Check if orientation is restored properly on navigation
- Ensure page-specific logic is applied

### Problem: System auto-rotate not working

- Verify device auto-rotate is enabled in system settings
- Check if 'auto' mode properly unlocks orientation
- Test on different devices (some have rotation lock in quick settings)

## Platform Support

### Web
- **API**: Screen Orientation API
- **Support**: Modern browsers (Chrome, Firefox, Safari 16.4+)
- **Limitations**: Requires fullscreen or installed PWA on some browsers

### iOS
- **API**: CoreMotion framework
- **Support**: iOS 12+
- **Implementation**: Native bridge via Tauri plugin

### Android
- **API**: Activity.setRequestedOrientation()
- **Support**: Android 5.0+
- **Implementation**: Native bridge via Tauri plugin

### Desktop
- **Support**: Limited (windows can be rotated but display typically doesn't rotate)
- **Fallback**: Setting has no effect on desktop

## Dependencies

- **Screen Orientation API**: Browser API for orientation lock
- **Tauri plugins**: Native platform integration
- **Platform APIs**: CoreMotion (iOS), Activity (Android)

## Performance Considerations

- **Debounce orientation changes**: Prevent excessive re-renders
- **Cleanup locks**: Always unlock on component unmount
- **Cache orientation state**: Avoid repeated API calls
- **Lazy lock application**: Only lock when in reader, unlock elsewhere

---

**Implemented in commits:**
- 5e04f6ae: feat: support locking screen orientation, closes #860
- da49526f: fix: now auto orientation follows system settings, closes #1093 and closes #1098
- 6080f9e0: fix: revert to unlock screen orientation for library page, closes #1063
