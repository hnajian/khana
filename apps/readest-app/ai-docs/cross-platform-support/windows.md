# Windows Platform Support

**Platform Category**: Desktop, Cross-Platform
**Status**: Full Support
**Related Commits**: #707, #724, #726

## Overview

Readest provides comprehensive Windows support through Tauri v2, featuring custom window controls, Windows-specific optimizations, and full Windows integration including single-instance handling and file associations.

## Platform Detection

### AppService Configuration

```typescript
class NativeAppService extends AppService {
  osPlatform: 'windows';
  appPlatform: 'tauri';

  hasWindow: true;
  hasWindowBar: true;
  hasContextMenu: true;
  hasUpdater: true;
  isDesktopApp: true;
  distChannel: 'readest';
}
```

## Window Management

### Custom Window Controls

**File**: `src-tauri/src/lib.rs`

```rust
#[cfg(target_os = "windows")]
let win_builder = win_builder
    .decorations(false)      // Disable default title bar
    .transparent(true);      // Enable transparency for custom chrome
```

**Custom Title Bar** (`src/components/WindowButtons.tsx`):
```typescript
function WindowButtons() {
  const handleMinimize = () => getCurrentWindow().minimize();
  const handleMaximize = () => getCurrentWindow().toggleMaximize();
  const handleClose = () => getCurrentWindow().close();

  return (
    <div className="windows-controls">
      <button onClick={handleMinimize} aria-label="Minimize">
        <MinimizeIcon />
      </button>
      <button onClick={handleMaximize} aria-label="Maximize">
        <MaximizeIcon />
      </button>
      <button onClick={handleClose} className="close" aria-label="Close">
        <CloseIcon />
      </button>
    </div>
  );
}
```

**CSS** (`src/styles/globals.css`):
```css
.windows-controls {
  display: flex;
  height: 32px;
  -webkit-app-region: no-drag;
}

.windows-controls button {
  width: 46px;
  height: 100%;
  background: transparent;
  border: none;
  color: var(--text-color);
  transition: background 0.15s;
}

.windows-controls button:hover {
  background: rgba(255, 255, 255, 0.1);
}

.windows-controls button.close:hover {
  background: #e81123;
  color: white;
}
```

### Single Instance Handler

**Feature**: Open files in existing instance (#724, #726)

**File**: `src-tauri/src/lib.rs`

```rust
use tauri::Manager;

tauri::Builder::default()
    .plugin(tauri_plugin_single_instance::init(|app, argv, cwd| {
        println!("Single instance activated with args: {:?}", argv);

        // Extract file paths from arguments
        let files: Vec<String> = argv.iter()
            .filter(|arg| is_book_file(arg))
            .cloned()
            .collect();

        if !files.is_empty() {
            // Focus window
            if let Some(window) = app.get_webview_window("main") {
                let _ = window.set_focus();

                // Send files to frontend
                let _ = app.emit_all("open-files", files);
            }
        }
    }))
    .build(tauri::generate_context!())
    .expect("error while building tauri application")
    .run(|_app_handle, event| match event {
        tauri::RunEvent::ExitRequested { api, .. } => {
            api.prevent_exit();
        }
        _ => {}
    });
```

**Frontend Handler** (`src/app/layout.tsx`):
```typescript
useEffect(() => {
  if (isWindowsPlatform()) {
    const unlisten = listen('open-files', (event) => {
      const files = event.payload as string[];
      files.forEach(file => importBook(file));
    });

    return () => {
      unlisten.then(fn => fn());
    };
  }
}, []);
```

**Related Commits**:
- #724: Should listen single-instance event on Windows
- #726: Fix single-instance handler

### Style Tweaks for Windows

**Feature**: Platform-specific styling (#707)

**File**: `src/styles/windows.css`

```css
/* Windows-specific styles */
body.windows {
  font-family: 'Segoe UI', system-ui, sans-serif;
}

/* Rounded corners for Windows 11 */
.windows-11 .window-container {
  border-radius: 8px;
  overflow: hidden;
}

/* Acrylic effect for Windows 10+ */
.windows-10 .sidebar,
.windows-11 .sidebar {
  background: rgba(255, 255, 255, 0.7);
  backdrop-filter: blur(20px);
}

/* Dark mode adjustments */
.windows.dark .sidebar {
  background: rgba(0, 0, 0, 0.7);
}

/* Custom scrollbars */
.windows ::-webkit-scrollbar {
  width: 12px;
}

.windows ::-webkit-scrollbar-track {
  background: transparent;
}

.windows ::-webkit-scrollbar-thumb {
  background: rgba(128, 128, 128, 0.5);
  border-radius: 6px;
  border: 3px solid transparent;
  background-clip: padding-box;
}

.windows ::-webkit-scrollbar-thumb:hover {
  background: rgba(128, 128, 128, 0.7);
  background-clip: padding-box;
}
```

**Related Commits**:
- #707: Style tweaks for Windows

## File System Integration

### Storage Locations

**macOS Paths**:
- **Books**: `%APPDATA%\com.readest.app\Books`
- **Settings**: `%APPDATA%\com.readest.app\Settings`
- **Cache**: `%LOCALAPPDATA%\com.readest.app\cache`
- **Logs**: `%LOCALAPPDATA%\com.readest.app\logs`

### File Associations

**Configuration** (`tauri.conf.json`):
```json
{
  "bundle": {
    "windows": {
      "fileAssociations": [
        {
          "ext": ["epub"],
          "name": "EPUB Book",
          "description": "Electronic Publication",
          "role": "Viewer",
          "mimeType": "application/epub+zip"
        },
        {
          "ext": ["pdf"],
          "name": "PDF Document",
          "description": "Portable Document Format",
          "role": "Viewer",
          "mimeType": "application/pdf"
        }
      ]
    }
  }
}
```

**Registry Entries** (created automatically):
```
HKEY_CURRENT_USER\Software\Classes\.epub
HKEY_CURRENT_USER\Software\Classes\.pdf
HKEY_CURRENT_USER\Software\Classes\Readest.EPUB
HKEY_CURRENT_USER\Software\Classes\Readest.PDF
```

## Build and Deployment

### Build Configuration

**File**: `apps/readest-app/src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "windows": {
      "certificateThumbprint": null,
      "digestAlgorithm": "sha256",
      "timestampUrl": "",
      "wix": {
        "language": "en-US"
      },
      "nsis": {
        "displayLanguageSelector": true,
        "languages": ["English", "German", "French", "Spanish"],
        "installerIcon": "icons/icon.ico",
        "installMode": "currentUser",
        "allowDowngrades": false,
        "deleteAppDataOnUninstall": false
      }
    }
  }
}
```

### Build Commands

**Development**:
```bash
pnpm tauri dev
```

**Production NSIS Installer**:
```bash
pnpm build-win-x64
```

**MSI Installer (WiX)**:
```bash
pnpm tauri build --target x86_64-pc-windows-msvc --bundles msi
```

**Portable Build**:
```bash
# Build without installer
pnpm tauri build --target x86_64-pc-windows-msvc --bundles app
```

### Code Signing

**Sign with signtool**:
```bash
signtool sign /f certificate.pfx /p password /tr http://timestamp.digicert.com /td sha256 /fd sha256 Readest.exe
```

## Known Issues

### Issue: WebView2 Not Installed

**Symptom**: App fails to launch on older Windows versions

**Workaround**: Bundle WebView2 installer
```json
{
  "bundle": {
    "windows": {
      "webviewInstallMode": {
        "type": "embedBootstrapper"
      }
    }
  }
}
```

## Related Documentation

- **[Cross-Platform Support Index](./index.md)** - Overview of all platforms
- **[macOS Platform](./macos.md)** - macOS-specific features
- **[Linux Platform](./linux.md)** - Linux-specific features

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #707, #724, #726
