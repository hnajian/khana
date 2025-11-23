# Screen Wake Lock Setting

**Added**: v0.9.13 (Commit #403)
**Updated**: v0.9.22 (Commit #505) - Focus/visibility handling

## Overview

The Screen Wake Lock setting prevents the device screen from sleeping during reading sessions, improving the reading experience for users who prefer not to interact with the screen frequently. The feature intelligently releases the wake lock when the app loses focus or becomes hidden to conserve battery.

## Implementation

**Location**: `src/app/reader/components/settings/MiscPanel.tsx`

**Type Definition**:
```typescript
interface SystemSettings {
  // ... existing settings
  keepScreenAwake: boolean; // Added v0.9.13
}
```

## API Support

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

## Behavior

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
  - Window or tab loses focus (added v0.9.22, #502, #505)

**Persistence**:
- Setting persists across sessions
- Can be configured globally or per-book
- Default: `false` (respect system sleep settings)

## Focus and Visibility Handling (v0.9.22 Update)

**Problem Solved** (#502, #505):
Prior to v0.9.22, the wake lock remained active even when the user switched to a different window or tab, causing unnecessary battery drain.

**Implementation** (`src/hooks/useScreenWakeLock.ts`):

**For Web Platform**:
```typescript
useEffect(() => {
  const handleVisibilityChange = () => {
    if (document.visibilityState === 'hidden') {
      // Release wake lock when tab becomes hidden
      releaseWakeLock();
    } else if (document.visibilityState === 'visible' && keepScreenAwake) {
      // Reacquire wake lock when tab becomes visible again
      requestWakeLock();
    }
  };

  document.addEventListener('visibilitychange', handleVisibilityChange);

  return () => {
    document.removeEventListener('visibilitychange', handleVisibilityChange);
  };
}, [keepScreenAwake]);
```

**For Tauri Desktop Apps**:
```typescript
useEffect(() => {
  if (!isTauriApp()) return;

  const unlisten = getCurrentWindow().onFocusChanged(({ payload: focused }) => {
    if (!focused) {
      // Release wake lock when window loses focus
      releaseWakeLock();
    } else if (focused && keepScreenAwake) {
      // Reacquire wake lock when window regains focus
      requestWakeLock();
    }
  });

  return () => {
    unlisten.then((fn) => fn());
  };
}, [keepScreenAwake]);
```

**Benefits**:
- **Battery Conservation**: Wake lock only active when app is actually visible/focused
- **Multi-Tasking Friendly**: Allows device to sleep when user switches away
- **Automatic Recovery**: Wake lock reacquired when user returns to the app
- **No User Intervention Required**: Handles focus changes transparently

**Behavior Examples**:

| Scenario | Wake Lock Status |
|----------|------------------|
| Reading in active tab/window | ✅ Active |
| Switched to different browser tab | ❌ Released |
| Switched to different application | ❌ Released |
| Returned to Readest | ✅ Reacquired |
| Minimized window | ❌ Released |
| Restored window | ✅ Reacquired |

## Platform Support

| Platform | API Used | Status |
|----------|----------|--------|
| **Web (Modern Browsers)** | Wake Lock API | ✅ Supported |
| **macOS** | caffeinate / IOKit | ✅ Supported |
| **Windows** | SetThreadExecutionState | ✅ Supported |
| **Linux** | systemd-inhibit | ✅ Supported |
| **iOS** | UIApplication.isIdleTimerDisabled | ✅ Supported |
| **Android** | PowerManager.WakeLock | ✅ Supported |

## Browser Compatibility

**Wake Lock API Support**:
- Chrome/Edge: ✅ Version 84+
- Firefox: ✅ Version 126+
- Safari: ✅ Version 16.4+
- Opera: ✅ Version 70+

**Fallback**: For unsupported browsers, setting is disabled/hidden in UI.

## AI Modification Guidelines

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

## Common Issues

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

**Related**: [index.md](./index.md) (Main settings system documentation)
