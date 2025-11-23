# Feature: Cross-Platform Support

## Overview

Readest supports two deployment modes:

1. **Native Applications** (via Tauri v2): Desktop apps for macOS, Windows, and Linux
2. **Web Platform** (via Browser): Progressive Web App (PWA) running in modern browsers

Both modes share the same Next.js codebase but use different service implementations (`NativeAppService` vs `WebAppService`) for platform-specific operations. **Web platform support was added in commit aa16bc09 (Dec 2024)**.

## Platform-Specific Documentation

### Mobile Platforms
- **[Android Platform](./android.md)** - Android-specific features, native bridge APIs, and optimizations
- **[iOS Platform](./ios.md)** - iOS-specific features, Sign in with Apple, haptics, and background audio

### Desktop Platforms
- **[macOS Platform](./macos.md)** - macOS-specific features, traffic light positioning, and title bar overlay
- **[Windows Platform](./windows.md)** - Windows-specific features, custom window controls, and single-instance handling
- **[Linux Platform](./linux.md)** - Linux-specific features, F-Droid integration, and GTK theming

### Web Platform
- **[Progressive Web App (PWA) Enhancements](./pwa-enhancements.md)** - PWA capabilities, service workers, and offline support

##
## Key Components

### Primary Files

- **`apps/readest-app/src-tauri/src/lib.rs`** - Main Tauri setup and plugin configuration
- **`apps/readest-app/src-tauri/src/main.rs`** - Application entry point
- **`apps/readest-app/src-tauri/tauri.conf.json`** - Tauri configuration
- **`apps/readest-app/src-tauri/Cargo.toml`** - Rust dependencies

### Related Files

- **`apps/readest-app/src-tauri/src/tauri_traffic_light_positioner_plugin.rs`** - macOS window controls plugin
- **`apps/readest-app/src-tauri/capabilities/`** - Security capability definitions
- **`apps/readest-app/src/services/nativeAppService.ts`** - TypeScript wrapper for Tauri APIs
- **`apps/readest-app/src/services/environment.ts`** - Platform detection

## Architecture

### Platform Detection

**Environment Configuration** (`src/services/environment.ts`):
```typescript
// Platform detection via environment variable
export const isTauriAppPlatform = () =>
  process.env['NEXT_PUBLIC_APP_PLATFORM'] === 'tauri';

export const isWebAppPlatform = () =>
  process.env['NEXT_PUBLIC_APP_PLATFORM'] === 'web';

// Service factory pattern
export const getAppService = async () => {
  if (isTauriAppPlatform()) {
    const { NativeAppService } = await import('@/services/nativeAppService');
    return new NativeAppService();
  } else {
    const { WebAppService } = await import('@/services/webAppService');
    return new WebAppService();
  }
};
```

**Environment Files:**
- `.env.tauri`: Sets `NEXT_PUBLIC_APP_PLATFORM=tauri`
- `.env.web`: Sets `NEXT_PUBLIC_APP_PLATFORM=web`

Build system determines which service to use at compile time.

### Tauri Setup (`src-tauri/src/lib.rs`)

**Plugin Registration**:
```rust
let builder = tauri::Builder::default()
    .plugin(tauri_plugin_shell::init())      // Shell commands
    .plugin(tauri_plugin_http::init())       // HTTP client
    .plugin(tauri_plugin_os::init())         // OS info
    .plugin(tauri_plugin_dialog::init())     // File dialogs
    .plugin(tauri_plugin_fs::init());        // File system access

#[cfg(target_os = "macos")]
let builder = builder.plugin(tauri_traffic_light_positioner_plugin::init());
```

**Window Configuration**:
```rust
let win_builder = WebviewWindowBuilder::new(app, "main", WebviewUrl::default())
    .title("")
    .inner_size(800.0, 600.0)
    .resizable(true)
    .maximized(true);

#[cfg(target_os = "macos")]
let win_builder = win_builder
    .decorations(true)
    .title_bar_style(TitleBarStyle::Overlay);

#[cfg(not(target_os = "macos"))]
let win_builder = win_builder
    .decorations(false)
    .transparent(true);
```

**Platform-Specific Behavior**:
- **macOS**: Native title bar with overlay style, decorations enabled
- **Windows/Linux**: Custom window controls, transparent background

### Native App Service (`src/services/nativeAppService.ts`)

**File System Operations**:
```typescript
class NativeAppService extends AppService {
  async importBooks(files: File[]): Promise<void> {
    // Use Tauri FS API to copy files to library directory
  }

  async loadBookFile(book: BookMetadata): Promise<File> {
    // Use Tauri FS API to read book file
  }

  async saveSettings(settings: SystemSettings): Promise<void> {
    // Use Tauri FS API to write settings file
  }

  // ... other native operations
}
```

**Tauri API Usage**:
- **@tauri-apps/plugin-fs**: File read/write operations
- **@tauri-apps/plugin-dialog**: File picker dialogs
- **@tauri-apps/plugin-os**: Platform detection, paths
- **@tauri-apps/plugin-http**: HTTP requests (if needed)

---

## Web Platform Support (Added Dec 2024)

### Web App Service (`src/services/webAppService.ts`)

The web platform uses **IndexedDB** for persistent storage instead of the native filesystem.

**Key Differences from Native:**

| Feature | Web Platform | Native Platform |
|---------|-------------|-----------------|
| **File System** | IndexedDB (browser storage) | Tauri file system plugins |
| **Storage Limit** | Browser quota (~500MB-5GB) | OS file system limits |
| **File Selection** | Not supported (throws error) | Native file dialogs |
| **Directory Selection** | Not supported | Supported via openDialog |
| **Custom Root Directory** | Not supported | Supported (portable/sandboxed) |
| **App Platform Flag** | `appPlatform = 'web'` | `appPlatform = 'tauri'` |
| **Window Controls** | Not rendered | Platform-specific controls |
| **Auto Updater** | Not available | Full updater support |
| **File Fetch** | Standard `fetch()` | Tauri HTTP plugin |

**IndexedDB Structure:**
```typescript
// Database: AppFileSystem
// Store: files
// Key: path (e.g., "Readest/Books/book.epub")
// Value: { path, content: ArrayBuffer | string }

// Virtual directories:
// - Readest/Books
// - Readest/Fonts
// - Readest/Images
// - Readest/Data
```

**File Operations:**
```typescript
class WebAppService extends BaseAppService {
  // Read from IndexedDB
  async readFile(path: string, base: BaseDir): Promise<ArrayBuffer> {
    const db = await openIndexedDB();
    const file = await db.get('files', path);
    return file.content;
  }

  // Write to IndexedDB
  async writeFile(path: string, content: ArrayBuffer | string, base: BaseDir) {
    const db = await openIndexedDB();
    await db.put('files', { path, content });
  }

  // Not supported operations (throw errors):
  async selectFiles() {
    throw new Error('File selection not supported in browser');
  }

  async selectDirectory() {
    throw new Error('Directory selection not supported in browser');
  }
}
```

### PWA Support

**Detection:**
```typescript
export const isPWA = () =>
  window.matchMedia('(display-mode: standalone)').matches;
```

**PWA Features:**
- Installable from browser
- Offline support (via service worker)
- Safe area insets (like native apps)
- Add to home screen (mobile)

**Service Worker** (Next.js PWA plugin):
- Caches static assets
- Provides offline fallback
- Pre-caches book files for offline reading

### Platform-Specific UI

**Components with Platform Detection:**
- `LibraryHeader.tsx`: Shows "Download Readest" button on web
- `WindowButtons.tsx`: Only renders on Tauri
- `UpdaterWindow.tsx`: Only on Tauri
- `BookMenu.tsx`: Different options for web vs native

**Example Platform-Specific Rendering:**
```typescript
import { isTauriAppPlatform } from '@/services/environment';

function LibraryHeader() {
  return (
    <header>
      {isTauriAppPlatform() ? (
        <WindowButtons />
      ) : (
        <a href="/download">Download Readest</a>
      )}
    </header>
  );
}
```

### URL Handling

**Web Platform:**
```typescript
// Uses Blob URLs for local files
getURL(path: string) {
  if (isValidURL(path)) return path;
  return URL.createObjectURL(new Blob([content]));
}
```

**Native Platform:**
```typescript
// Uses Tauri's convertFileSrc for file:// URLs
getURL(path: string) {
  return isValidURL(path) ? path : convertFileSrc(path);
}
```

### Remote File Support

Both platforms support loading books from URLs:
```typescript
async openFile(path: string, base: BaseDir) {
  if (isValidURL(path)) {
    return await new RemoteFile(path, filename).open();
  }
  // Platform-specific file loading (IndexedDB or FS)
}
```

### Deployment

**Web Platform:**
```bash
# Build for web
NEXT_PUBLIC_APP_PLATFORM=web pnpm build

# Deploy to Vercel/Netlify/etc.
pnpm deploy
```

**Native Platform:**
```bash
# Build for Tauri
pnpm tauri build
```

### Menu Integration

**Help Menu** (in `lib.rs`):
```rust
let _ = global_menu.append(
    &SubmenuBuilder::new(app, "Help")
        .text("privacy_policy", "Privacy Policy")
        .separator()
        .text("report_issue", "Report An Issue...")
        .text("readest_help", "Readest Help")
        .build()?,
);

app.on_menu_event(move |app, event| {
    if event.id() == "privacy_policy" {
        let _ = app.shell().open("https://readest.com/privacy-policy", None);
    } else if event.id() == "report_issue" {
        let _ = app.shell().open("mailto:support@bilingify.com", None);
    } else if event.id() == "readest_help" {
        let _ = app.shell().open("https://readest.com/support", None);
    }
});
```

### macOS-Specific Features

**Traffic Light Positioner** (`tauri_traffic_light_positioner_plugin.rs`):
- Positions window control buttons (close, minimize, maximize)
- Uses macOS Cocoa APIs via objc bindings
- Only compiled on macOS (`#[cfg(target_os = "macos")]`)

**Title Bar Overlay**:
- Allows content to extend into title bar area
- Provides native feel with custom UI
- Handled by `TitleBarStyle::Overlay`

### Build Configuration

**Tauri Config** (`tauri.conf.json`):
```json
{
  "build": {
    "beforeDevCommand": "pnpm dev",
    "beforeBuildCommand": "pnpm build",
    "devPath": "http://localhost:3000",
    "distDir": "../out"
  },
  "bundle": {
    "active": true,
    "targets": ["dmg", "app", "nsis", "appimage"],
    "identifier": "com.readest.app"
  }
}
```

**Build Targets**:
- **macOS**: DMG, App Bundle, App Store variant
- **Windows**: NSIS installer
- **Linux**: AppImage

## AI Agent Modification Guidelines

### Adding a New Tauri Plugin

To integrate a new Tauri plugin (e.g., clipboard):

1. **Add dependency** to `Cargo.toml`:
   ```toml
   [dependencies]
   tauri-plugin-clipboard = "2.0"
   ```

2. **Register plugin** in `lib.rs`:
   ```rust
   use tauri_plugin_clipboard;

   let builder = tauri::Builder::default()
       // ... existing plugins
       .plugin(tauri_plugin_clipboard::init());
   ```

3. **Use in TypeScript**:
   ```typescript
   import { writeText, readText } from '@tauri-apps/plugin-clipboard';

   async function copyToClipboard(text: string) {
     await writeText(text);
   }
   ```

### Adding Platform-Specific Code

To add platform-specific functionality:

1. **Rust side** (in `lib.rs` or custom module):
   ```rust
   #[cfg(target_os = "windows")]
   fn setup_windows_specific() {
       // Windows-only code
   }

   #[cfg(target_os = "linux")]
   fn setup_linux_specific() {
       // Linux-only code
   }

   // In setup()
   #[cfg(target_os = "windows")]
   setup_windows_specific();

   #[cfg(target_os = "linux")]
   setup_linux_specific();
   ```

2. **TypeScript side**:
   ```typescript
   import { platform } from '@tauri-apps/plugin-os';

   const currentPlatform = platform();

   if (currentPlatform === 'windows') {
     // Windows-specific UI
   } else if (currentPlatform === 'macos') {
     // macOS-specific UI
   }
   ```

### Creating Custom Tauri Commands

To expose Rust functions to JavaScript:

1. **Define command** in Rust:
   ```rust
   #[tauri::command]
   fn custom_operation(param: String) -> Result<String, String> {
       // Perform operation
       Ok(format!("Processed: {}", param))
   }
   ```

2. **Register command**:
   ```rust
   tauri::Builder::default()
       .invoke_handler(tauri::generate_handler![custom_operation])
       // ...
   ```

3. **Call from TypeScript**:
   ```typescript
   import { invoke } from '@tauri-apps/api/core';

   const result = await invoke<string>('custom_operation', {
     param: 'test'
   });
   ```

### Handling Deep Links / File Associations

To handle "open with" functionality:

1. **Configure in `tauri.conf.json`**:
   ```json
   {
     "bundle": {
       "fileAssociations": [
         {
           "ext": ["epub"],
           "name": "EPUB Book",
           "role": "Viewer"
         }
       ]
     }
   }
   ```

2. **Handle in Rust** (already has skeleton in `lib.rs`):
   ```rust
   // Listen for file open events
   app.listen_global("file-drop", |event| {
       // Handle dropped file
   });
   ```

3. **Process in TypeScript**:
   ```typescript
   import { listen } from '@tauri-apps/api/event';

   listen('file-drop', (event) => {
     const filePath = event.payload as string;
     // Import the book
   });
   ```

### Improving Window Management

To add window control features:

1. **Multiple windows**:
   ```rust
   use tauri::Manager;

   #[tauri::command]
   fn open_new_window(app: tauri::AppHandle) -> Result<(), String> {
       WebviewWindowBuilder::new(
           &app,
           "secondary",
           WebviewUrl::App("/reader".into())
       )
       .title("New Reader")
       .build()
       .map_err(|e| e.to_string())?;
       Ok(())
   }
   ```

2. **Window state management**:
   - Save/restore window size and position
   - Remember maximized state
   - Multi-monitor support

### Adding Native Notifications

To show system notifications:

1. **Add plugin**:
   ```toml
   tauri-plugin-notification = "2.0"
   ```

2. **Send notification**:
   ```typescript
   import { sendNotification } from '@tauri-apps/plugin-notification';

   await sendNotification({
     title: 'Readest',
     body: 'Finished reading chapter 5',
   });
   ```

### Implementing Auto-Updates

To add auto-update functionality:

1. **Add plugin**:
   ```toml
   tauri-plugin-updater = "2.0"
   ```

2. **Check for updates**:
   ```typescript
   import { check } from '@tauri-apps/plugin-updater';

   const update = await check();
   if (update) {
     await update.downloadAndInstall();
     // Restart app
   }
   ```

3. **Configure update endpoint** in `tauri.conf.json`

### Adding System Tray

To add a system tray icon:

1. **Configure in `tauri.conf.json`**:
   ```json
   {
     "tray": {
       "iconPath": "icons/tray-icon.png",
       "menuOnLeftClick": true
     }
   }
   ```

2. **Build tray menu**:
   ```rust
   use tauri::tray::{TrayIconBuilder, Menu};

   let tray_menu = Menu::new(app)?;
   // Add menu items

   TrayIconBuilder::new()
       .menu(&tray_menu)
       .build(app)?;
   ```

## Entry Points for Modifications

| Task | Primary File | Supporting Files |
|------|--------------|------------------|
| Add Tauri plugin | `Cargo.toml`, `lib.rs` | TypeScript service files |
| Custom commands | `lib.rs` | `nativeAppService.ts` |
| Window management | `lib.rs` | Window components |
| Menu items | `lib.rs` | - |
| File associations | `tauri.conf.json` | Event handlers |
| Platform-specific UI | `lib.rs`, React components | `environment.ts` |
| Build config | `tauri.conf.json`, `package.json` | Build scripts |

## Common Issues and Debugging

### Problem: Tauri command not found

- Verify command is registered in `invoke_handler`
- Check function signature matches TypeScript call
- Ensure command name matches (case-sensitive)
- Check Tauri build output for errors

### Problem: File system permission denied

- Check capability definitions in `capabilities/`
- Verify paths are within allowed directories
- On macOS, check sandbox entitlements
- Test with absolute vs. relative paths

### Problem: Window decorations incorrect

- Check platform-specific window builder config
- Verify CSS doesn't conflict with native controls
- Test `decorations(false)` vs `decorations(true)`
- Check traffic light positioner on macOS

### Problem: Build fails on specific platform

- Check platform-specific dependencies
- Verify build prerequisites are installed
- Check `Cargo.toml` for platform-specific features
- Review build script output for errors

## Dependencies

- **Tauri Core**: v2.1.x
- **Tauri Plugins**: fs, dialog, http, os, log, shell
- **Rust Toolchain**: Latest stable
- **Platform SDKs**: Xcode (macOS), Visual Studio (Windows), build-essential (Linux)

## Build Scripts

**Development**:
```bash
pnpm tauri dev          # Hot-reload development
```

**Production**:
```bash
pnpm tauri build                           # Current platform
pnpm build-macos-universal                 # macOS universal binary
pnpm build-win-x64                         # Windows x64
pnpm build-linux-x64                       # Linux x64
```

**Platform-Specific Builds** (from `package.json`):
- `build-macos-universial`: DMG for distribution
- `build-macos-universial-appstore`: App Store build
- `build-win-x64`, `build-win-arm64`: Windows installers
- `build-linux-x64`: AppImage

## Performance Considerations

- **Minimize IPC calls**: Batch operations when possible
- **Async operations**: Don't block UI thread
- **Resource cleanup**: Close file handles, free memory
- **Lazy plugin loading**: Only load needed plugins
- **Binary size**: Remove unused plugins to reduce app size

---

## Version 0.9.64 - 0.9.67 Updates (f5b686ab → 33b2ba16)

### Update Notes for Releases (v0.9.64, #1552)

**Feature**: Display release notes when new application updates are available.

**Overview**: The auto-updater now shows a changelog/release notes dialog before applying updates, helping users understand what's new or fixed in each version.

**Implementation** (`src-tauri/src/updater.rs`):
```rust
use tauri_plugin_updater::{UpdaterExt, Update};

#[tauri::command]
async fn check_for_updates(app: AppHandle) -> Result<Option<Update>, String> {
    let update = app.updater()
        .check()
        .await
        .map_err(|e| e.to_string())?;

    if let Some(update) = update {
        // Extract release notes from update manifest
        Ok(Some(update))
    } else {
        Ok(None)
    }
}

#[tauri::command]
async fn install_update(update: Update) -> Result<(), String> {
    update.download_and_install().await
        .map_err(|e| e.to_string())
}
```

**Frontend** (`src/components/UpdateDialog.tsx`):
```typescript
const UpdateDialog = ({ update }: { update: Update }) => {
  const [installing, setInstalling] = useState(false);

  const handleInstall = async () => {
    setInstalling(true);
    try {
      await invoke('install_update', { update });
      toast.success('Update installed! Restarting...');
    } catch (error) {
      toast.error('Update failed: ' + error);
      setInstalling(false);
    }
  };

  return (
    <Dialog open>
      <h2>Update Available: v{update.version}</h2>

      <div className="release-notes">
        <h3>What's New:</h3>
        <ReactMarkdown>{update.body}</ReactMarkdown>
      </div>

      <div className="actions">
        <button onClick={() => setOpen(false)}>Later</button>
        <button onClick={handleInstall} disabled={installing}>
          {installing ? 'Installing...' : 'Install Now'}
        </button>
      </div>
    </Dialog>
  );
};
```

**Update Manifest**: Release notes are fetched from GitHub releases API or update server.

**User Experience**:
- Automatic update check on app launch
- Release notes displayed in markdown format
- Option to install now or postpone
- Progress indicator during download/install
- Automatic app restart after installation

**Files**:
- `src-tauri/src/updater.rs` - Update checking logic
- `src/components/UpdateDialog.tsx` - Update UI
- `src/services/updaterService.ts` - Update service abstraction

### Window Borders (v0.9.64-0.9.65, #1556, #1599)

**Feature**: Added window borders on Windows 10 and Linux for better visual consistency.

**Windows 10** (#1556):
```rust
// src-tauri/src/lib.rs
#[cfg(target_os = "windows")]
fn configure_window(builder: WindowBuilder) -> WindowBuilder {
    use windows::Win32::UI::WindowsAndMessaging::{WS_BORDER, WS_THICKFRAME};

    builder
        .decorations(true)
        .transparent(false)
        .additional_browser_args("--disable-features=OverlayScrollbar")
}
```

**Linux** (#1570, #1599):
```rust
#[cfg(target_os = "linux")]
fn configure_window(builder: WindowBuilder) -> WindowBuilder {
    builder
        .decorations(true)
        .resizable(true)
}
```

**Issues Addressed**:
- Windows 10: Borderless window appeared frameless
- Linux: Inconsistent window decorations across DEs
- Better integration with system theme

**Files**:
- `src-tauri/src/lib.rs` - Platform-specific window configuration
- `src-tauri/tauri.conf.json` - Window decoration settings

### Multiple Reader Windows on Desktop (v0.9.65, #1596)

**Major Feature**: Support for opening multiple reader windows simultaneously on desktop platforms.

**Overview**: Users can now open different books in separate windows, each with independent reading state, settings, and position.

**Architecture**:

1. **Window Management** (`src-tauri/src/window_manager.rs`):
   ```rust
   use tauri::{Manager, WindowBuilder, WindowUrl};
   use std::collections::HashMap;

   pub struct WindowManager {
       windows: HashMap<String, Window>
   }

   impl WindowManager {
       pub fn create_reader_window(
           &mut self,
           app: &AppHandle,
           book_hash: &str
       ) -> Result<Window, Error> {
           let label = format!("reader-{}", book_hash);

           if let Some(existing) = self.windows.get(&label) {
               existing.set_focus()?;
               return Ok(existing.clone());
           }

           let window = WindowBuilder::new(
               app,
               &label,
               WindowUrl::App(format!("reader?book={}", book_hash).into())
           )
           .title("Readest")
           .inner_size(1200.0, 800.0)
           .min_inner_size(800.0, 600.0)
           .build()?;

           self.windows.insert(label.clone(), window.clone());
           Ok(window)
       }
   }
   ```

2. **Tauri Command** (`src-tauri/src/commands/window.rs`):
   ```rust
   #[tauri::command]
   pub async fn open_book_in_new_window(
       app: AppHandle,
       book_hash: String
   ) -> Result<(), String> {
       let mut window_manager = app.state::<WindowManager>();
       window_manager.create_reader_window(&app, &book_hash)
           .map_err(|e| e.to_string())?;
       Ok(())
   }
   ```

3. **Frontend Integration** (`src/services/windowService.ts`):
   ```typescript
   export const openBookInNewWindow = async (bookHash: string) => {
       if (!isDesktop()) {
           // Fallback: navigate in current window
           router.push(`/reader?book=${bookHash}`);
           return;
       }

       await invoke('open_book_in_new_window', { bookHash });
   };
   ```

**Features**:
- Each window has independent reading state
- Window position and size saved per book
- Close window without closing other windows
- Windows can be on different monitors
- Taskbar shows separate entries for each book

**User Actions**:
- Right-click book → "Open in New Window"
- Keyboard shortcut: `Ctrl/Cmd+Shift+O`
- Drag book to desktop to open new window (macOS)

**State Management**:
- Each window has its own Zustand store instance
- Reading position synced independently
- Settings can be per-window or global

**Limitations**:
- Desktop only (Tauri apps)
- PWA/web version uses single window with navigation
- Mobile apps don't support multiple windows

**Files**:
- `src-tauri/src/window_manager.rs` - Window management
- `src-tauri/src/commands/window.rs` - Window commands
- `src/services/windowService.ts` - Frontend service
- `src/app/library/components/BookCard.tsx` - UI integration

### Performance Improvements

**Multi-part Download with Range Access** (v0.9.67, #1690):

**Feature**: Improved download performance using HTTP range requests for parallel downloads.

**Implementation** (`src/services/downloadService.ts`):
```typescript
const CHUNK_SIZE = 1024 * 1024; // 1 MB chunks
const MAX_CONCURRENT_CHUNKS = 4;

async function downloadWithRangeAccess(
  url: string,
  totalSize: number
): Promise<Blob> {
  const chunks: Blob[] = new Array(Math.ceil(totalSize / CHUNK_SIZE));
  const downloadChunk = async (index: number) => {
    const start = index * CHUNK_SIZE;
    const end = Math.min(start + CHUNK_SIZE - 1, totalSize - 1);

    const response = await fetch(url, {
      headers: { Range: `bytes=${start}-${end}` }
    });

    chunks[index] = await response.blob();
  };

  // Download chunks in parallel
  const chunkCount = Math.ceil(totalSize / CHUNK_SIZE);
  for (let i = 0; i < chunkCount; i += MAX_CONCURRENT_CHUNKS) {
    const batch = Array.from(
      { length: Math.min(MAX_CONCURRENT_CHUNKS, chunkCount - i) },
      (_, j) => downloadChunk(i + j)
    );
    await Promise.all(batch);
  }

  return new Blob(chunks);
}
```

**Benefits**:
- 2-4x faster downloads for large books
- Resumable downloads
- Better network utilization
- Progress tracking per chunk

**Files**:
- `src/services/downloadService.ts`
- `src/utils/http.ts`

---

