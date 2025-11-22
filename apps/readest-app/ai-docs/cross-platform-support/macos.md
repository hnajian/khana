# macOS Platform Support

**Platform Category**: Desktop, Cross-Platform
**Status**: Full Support
**Related Commits**: #297, #497

## Overview

Readest provides native macOS support through Tauri v2, featuring a polished macOS-native experience with traffic light window controls, title bar overlay, and full macOS integration while sharing the core Next.js codebase with other platforms.

## Platform Detection

### OS Platform Detection

**File**: `src/utils/ua.ts`

```typescript
export const getOSPlatform = (): OsPlatform => {
  if (typeof window === 'undefined') return 'unknown';

  const ua = navigator.userAgent.toLowerCase();

  if (ua.includes('mac os x')) return 'macos';
  // ... other platform checks
};
```

### AppService Configuration

**File**: `src/services/nativeAppService.ts`

```typescript
class NativeAppService extends AppService {
  osPlatform: 'macos';
  appPlatform: 'tauri';

  // macOS-specific capabilities
  hasTrafficLight: true;          // Native window controls
  hasWindow: true;                // Full window management
  hasWindowBar: true;             // Custom title bar
  hasContextMenu: true;           // Right-click menus
  hasRoundedWindow: true;         // Rounded corner support
  hasSafeAreaInset: false;        // No safe area needed
  hasHaptics: true;               // Force Touch trackpad
  hasUpdater: true;               // Sparkle auto-updater
  isDesktopApp: true;
  isMacOSApp: true;
  distChannel: 'readest';         // or 'appstore' for Mac App Store
}
```

## Window Management

### Traffic Light Positioning

**Feature**: Custom positioning of macOS window controls (#297, #497)

**File**: `src-tauri/src/tauri_traffic_light_positioner_plugin.rs`

```rust
use cocoa::appkit::{NSView, NSWindow};
use cocoa::foundation::NSRect;
use tauri::{plugin::Plugin, Runtime, Window};

pub struct TrafficLightPositioner;

impl<R: Runtime> Plugin<R> for TrafficLightPositioner {
    fn name(&self) -> &'static str {
        "traffic-light-positioner"
    }

    fn initialize(&mut self, app: &tauri::AppHandle<R>) -> Result<()> {
        // Position traffic lights when window is created
    }

    fn extend_api(&mut self, invoke: Invoke<R>) {
        match invoke.message.command() {
            "position_traffic_lights" => {
                let window: Window<R> = invoke.message.webview().window();
                position_traffic_lights(&window, x, y);
            }
            _ => {}
        }
    }
}

fn position_traffic_lights(window: &Window, x: f64, y: f64) {
    unsafe {
        let ns_window = window.ns_window() as cocoa::base::id;
        let close_button = ns_window.standardWindowButton_(NSWindowButton::NSWindowCloseButton);
        let miniaturize_button = ns_window.standardWindowButton_(NSWindowButton::NSWindowMiniaturizeButton);
        let zoom_button = ns_window.standardWindowButton_(NSWindowButton::NSWindowZoomButton);

        // Set custom positions
        close_button.setFrameOrigin(NSPoint::new(x, y));
        miniaturize_button.setFrameOrigin(NSPoint::new(x + 20.0, y));
        zoom_button.setFrameOrigin(NSPoint::new(x + 40.0, y));
    }
}
```

**TypeScript API**:
```typescript
import { invoke } from '@tauri-apps/api/core';

export const positionTrafficLights = async (x: number, y: number): Promise<void> => {
  if (isMacOSPlatform()) {
    await invoke('plugin:traffic-light-positioner|position_traffic_lights', { x, y });
  }
};
```

**Usage**:
```typescript
// Position traffic lights in custom title bar
useEffect(() => {
  if (isMacOSPlatform()) {
    positionTrafficLights(16, 22);  // Standard macOS position
  }
}, []);
```

### Title Bar Overlay

**File**: `src-tauri/src/lib.rs`

```rust
#[cfg(target_os = "macos")]
let win_builder = win_builder
    .decorations(true)                          // Enable native decorations
    .title_bar_style(TitleBarStyle::Overlay);   // Overlay content into title bar
```

**Effect**:
- Content extends into title bar area
- Traffic lights remain visible
- Custom UI can be placed in title bar
- Native macOS feel with custom design

### Hide Traffic Lights

**Feature**: Hide window controls when sidebar is invisible (#297, #497)

**File**: `src/app/library/components/LibraryHeader.tsx`

```typescript
import { getCurrentWindow } from '@tauri-apps/api/window';

const LibraryHeader = () => {
  const { sidebarVisible } = useSidebarStore();

  useEffect(() => {
    if (isMacOSPlatform()) {
      const window = getCurrentWindow();

      if (!sidebarVisible) {
        // Hide traffic lights
        window.setDecorations(false);
      } else {
        // Show traffic lights
        window.setDecorations(true);
      }
    }
  }, [sidebarVisible]);

  return (
    <header>
      {/* Header content */}
    </header>
  );
};
```

**Related Commits**:
- #297: Hide traffic lights when sidebar is invisible on macOS
- #497: Fix traffic light visibility logic

## Menu Integration

### Application Menu

**File**: `src-tauri/src/lib.rs`

```rust
use tauri::menu::{MenuBuilder, MenuItemBuilder, SubmenuBuilder};

pub fn build_menu<R: Runtime>(app: &AppHandle<R>) -> Result<Menu<R>> {
    let menu = MenuBuilder::new(app)
        // App Menu
        .item(&SubmenuBuilder::new(app, "Readest")
            .about(Some("About Readest".to_string()))
            .separator()
            .services()
            .separator()
            .hide()
            .hide_others()
            .show_all()
            .separator()
            .quit()
            .build()?)

        // File Menu
        .item(&SubmenuBuilder::new(app, "File")
            .text("open_file", "Open File...")
            .accelerator("Cmd+O")
            .text("import_folder", "Import Folder...")
            .accelerator("Cmd+Shift+O")
            .separator()
            .close_window()
            .build()?)

        // Edit Menu
        .item(&SubmenuBuilder::new(app, "Edit")
            .undo()
            .redo()
            .separator()
            .cut()
            .copy()
            .paste()
            .select_all()
            .build()?)

        // View Menu
        .item(&SubmenuBuilder::new(app, "View")
            .text("toggle_fullscreen", "Toggle Fullscreen")
            .accelerator("Cmd+Ctrl+F")
            .text("toggle_sidebar", "Toggle Sidebar")
            .accelerator("Cmd+B")
            .separator()
            .text("zoom_in", "Zoom In")
            .accelerator("Cmd+Plus")
            .text("zoom_out", "Zoom Out")
            .accelerator("Cmd+Minus")
            .text("reset_zoom", "Actual Size")
            .accelerator("Cmd+0")
            .build()?)

        // Window Menu
        .item(&SubmenuBuilder::new(app, "Window")
            .minimize()
            .zoom()
            .separator()
            .bring_all_to_front()
            .build()?)

        // Help Menu
        .item(&SubmenuBuilder::new(app, "Help")
            .text("privacy_policy", "Privacy Policy")
            .separator()
            .text("report_issue", "Report An Issue...")
            .text("readest_help", "Readest Help")
            .build()?)

        .build()?;

    Ok(menu)
}
```

**Menu Event Handling**:
```rust
app.on_menu_event(move |app, event| {
    match event.id().as_ref() {
        "open_file" => {
            // Open file picker
        }
        "toggle_fullscreen" => {
            let window = app.get_webview_window("main").unwrap();
            window.set_fullscreen(!window.is_fullscreen().unwrap()).unwrap();
        }
        "toggle_sidebar" => {
            // Emit event to toggle sidebar
            app.emit_all("toggle-sidebar", ()).unwrap();
        }
        "privacy_policy" => {
            let _ = app.shell().open("https://readest.com/privacy-policy", None);
        }
        "report_issue" => {
            let _ = app.shell().open("mailto:support@bilingify.com", None);
        }
        "readest_help" => {
            let _ = app.shell().open("https://readest.com/support", None);
        }
        _ => {}
    }
});
```

## File System Integration

### Storage Locations

**File**: `src/services/nativeAppService.ts`

```typescript
const getMacOSBasePath = async (base: BaseDir): Promise<string> => {
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

**macOS Paths**:
- **Books**: `~/Library/Application Support/com.readest.app/Books`
- **Settings**: `~/Library/Application Support/com.readest.app/Settings`
- **Cache**: `~/Library/Caches/com.readest.app`
- **Logs**: `~/Library/Logs/com.readest.app`

### File Associations

**Configuration** (`Info.plist`):
```xml
<key>CFBundleDocumentTypes</key>
<array>
    <dict>
        <key>CFBundleTypeName</key>
        <string>EPUB Book</string>
        <key>CFBundleTypeRole</key>
        <string>Viewer</string>
        <key>LSItemContentTypes</key>
        <array>
            <string>org.idpf.epub-container</string>
        </array>
    </dict>
    <dict>
        <key>CFBundleTypeName</key>
        <string>PDF Document</string>
        <key>CFBundleTypeRole</key>
        <string>Viewer</string>
        <key>LSItemContentTypes</key>
        <array>
            <string>com.adobe.pdf</string>
        </array>
    </dict>
</array>
```

**Result**: Double-clicking EPUB/PDF files opens them in Readest

## Build and Deployment

### Build Configuration

**File**: `apps/readest-app/src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "macOS": {
      "minimumSystemVersion": "10.15",
      "entitlements": "src-tauri/entitlements.plist",
      "exceptionDomain": null,
      "frameworks": [],
      "providerShortName": null,
      "signingIdentity": null
    }
  }
}
```

### Build Commands

**Development**:
```bash
pnpm tauri dev
```

**Production DMG (Universal Binary)**:
```bash
pnpm build-macos-universal
```

**Individual Architectures**:
```bash
# Intel (x86_64)
pnpm tauri build --target x86_64-apple-darwin

# Apple Silicon (ARM64)
pnpm tauri build --target aarch64-apple-darwin
```

### Code Signing

**Entitlements** (`entitlements.plist`):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
</plist>
```

**Signing**:
```bash
codesign --sign "Developer ID Application: Your Name" \
         --entitlements entitlements.plist \
         --deep \
         --force \
         Readest.app
```

**Notarization**:
```bash
# Create DMG
hdiutil create -volname "Readest" -srcfolder Readest.app -ov -format UDZO Readest.dmg

# Submit for notarization
xcrun notarytool submit Readest.dmg \
  --apple-id your@email.com \
  --password app-specific-password \
  --team-id TEAM_ID \
  --wait

# Staple notarization
xcrun stapler staple Readest.dmg
```

## Performance Optimizations

### Metal Rendering

**Configuration**:
```rust
#[cfg(target_os = "macos")]
let win_builder = win_builder
    .user_agent("Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko)")
    .initialization_script("window.__TAURI_METAL__ = true;");
```

**Effect**: Enables GPU acceleration via Metal for smoother rendering

### Memory Management

**NSPurgeableData for Cache**:
```rust
use cocoa::foundation::{NSData, NSPurgeableData};

pub fn cache_with_purging(data: Vec<u8>) {
    let ns_data = NSPurgeableData::from_vec(data);
    ns_data.markAsDiscardable();  // Allow system to reclaim if needed
}
```

## Accessibility

### VoiceOver Support

**Enable full keyboard navigation**:
```typescript
if (isMacOSPlatform()) {
  document.addEventListener('keydown', (e) => {
    if (e.key === 'Tab') {
      // Ensure focus ring is visible
      document.body.classList.add('keyboard-nav');
    }
  });
}
```

**ARIA Labels**:
```typescript
<button
  aria-label={_('Go to next page')}
  onClick={goToNextPage}
>
  <ChevronRight />
</button>
```

## Known Issues

### Issue: Trackpad Gestures Conflict

**Symptom**: Swipe gestures conflict with page navigation

**Workaround**:
```rust
#[cfg(target_os = "macos")]
let win_builder = win_builder
    .accept_first_mouse(true)
    .tabbing_identifier("readest");
```

## Related Documentation

- **[Cross-Platform Support Index](./index.md)** - Overview of all platforms
- **[Windows Platform](./windows.md)** - Windows-specific features
- **[Linux Platform](./linux.md)** - Linux-specific features

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #297, #497
