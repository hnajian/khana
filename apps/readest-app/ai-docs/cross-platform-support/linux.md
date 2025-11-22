# Linux Platform Support

**Platform Category**: Desktop, Cross-Platform
**Status**: Full Support
**Related Commits**: #682

## Overview

Readest provides comprehensive Linux support through Tauri v2, offering native Linux integration via AppImage packaging, F-Droid availability, and support for various Linux distributions with GTK-based rendering.

## Platform Detection

### AppService Configuration

```typescript
class NativeAppService extends AppService {
  osPlatform: 'linux';
  appPlatform: 'tauri';

  hasWindow: true;
  hasWindowBar: true;
  hasContextMenu: true;
  hasUpdater: true;
  isDesktopApp: true;
  isLinuxApp: true;
  distChannel: 'readest';  // or 'fdroid' for F-Droid builds
}
```

## File System Integration

### Storage Locations

**Linux Paths** (XDG Base Directory Specification):
- **Books**: `~/.local/share/com.readest.app/Books`
- **Settings**: `~/.config/com.readest.app/Settings`
- **Cache**: `~/.cache/com.readest.app`
- **Logs**: `~/.local/state/com.readest.app/logs`

### File Associations

**Desktop Entry** (`com.readest.app.desktop`):
```ini
[Desktop Entry]
Type=Application
Name=Readest
Comment=Modern ebook reader
Exec=readest %U
Icon=com.readest.app
Terminal=false
Categories=Office;Viewer;
MimeType=application/epub+zip;application/pdf;application/x-mobipocket-ebook;application/x-cbz;
```

**Installation**:
```bash
cp com.readest.app.desktop ~/.local/share/applications/
update-desktop-database ~/.local/share/applications/
```

## Font Management

### Free Fonts Filter

**File**: `src/components/settings/FontPanel.tsx`

```typescript
const NON_FREE_FONTS = [
  'Times New Roman',
  'Arial',
  'Verdana',
  'Georgia',
  'Calibri',
  'Cambria'
];

const filterNonFreeFonts = (font: string) => {
  const osplatform = getOSPlatform();
  return !['android', 'linux'].includes(osplatform) ||
         !NON_FREE_FONTS.includes(font);
};

// Only show free fonts on Linux
const linuxFonts = allFonts.filter(filterNonFreeFonts);
```

**Free Font Alternatives**:
| Proprietary | Free Alternative |
|-------------|------------------|
| Times New Roman | Liberation Serif, Noto Serif |
| Arial | Liberation Sans, Noto Sans |
| Verdana | DejaVu Sans |
| Georgia | Gelasio |
| Calibri | Carlito |

### System Font Detection

**File**: `src/utils/bridge.ts`

```typescript
export const getSysFontsList = async (): Promise<string[]> => {
  if (isLinuxPlatform()) {
    return await invoke('plugin:native-bridge|get_sys_fonts_list');
  }
  return [];
};
```

**Implementation** (scans `/usr/share/fonts`, `~/.local/share/fonts`):
```rust
pub fn get_sys_fonts_list() -> Result<Vec<String>, String> {
    let mut fonts = Vec::new();

    // System fonts
    if let Ok(entries) = std::fs::read_dir("/usr/share/fonts") {
        for entry in entries.flatten() {
            if let Some(font_name) = parse_font_file(entry.path()) {
                fonts.push(font_name);
            }
        }
    }

    // User fonts
    if let Ok(home) = std::env::var("HOME") {
        let user_fonts = format!("{}/.local/share/fonts", home);
        if let Ok(entries) = std::fs::read_dir(user_fonts) {
            for entry in entries.flatten() {
                if let Some(font_name) = parse_font_file(entry.path()) {
                    fonts.push(font_name);
                }
            }
        }
    }

    fonts.sort();
    fonts.dedup();
    Ok(fonts)
}
```

## Build and Deployment

### Build Configuration

**File**: `apps/readest-app/src-tauri/tauri.conf.json`

```json
{
  "bundle": {
    "linux": {
      "deb": {
        "depends": ["libwebkit2gtk-4.1-0", "libgtk-3-0"]
      },
      "appimage": {
        "bundleMediaFramework": true,
        "files": {}
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

**Production AppImage**:
```bash
pnpm build-linux-x64
```

**Debian Package**:
```bash
pnpm tauri build --target x86_64-unknown-linux-gnu --bundles deb
```

### F-Droid Integration

**Feature**: F-Droid metadata (#682)

**File**: `fastlane/metadata/android/en-US/full_description.txt`

```
Readest is a modern, feature-rich ebook reader supporting EPUB, PDF, MOBI, CBZ, and FB2 formats.

Features:
• Multiple format support (EPUB, PDF, MOBI, CBZ, FB2, TXT)
• Cloud sync for reading progress and annotations
• Text-to-Speech with 100+ voices
• Customizable themes and fonts
• Annotation and highlighting tools
• Translation and dictionary integration
• RTL language support (Arabic, Hebrew)
• Vertical reading mode for CJK languages
• Cross-platform (Android, iOS, Linux, macOS, Windows, Web)

Privacy-focused:
• Open source
• Self-hostable
• No tracking or ads
• FOSS-only dependencies on F-Droid
```

**Metadata Files**:
```
fastlane/
├── metadata/
│   └── android/
│       ├── en-US/
│       │   ├── title.txt
│       │   ├── short_description.txt
│       │   ├── full_description.txt
│       │   ├── images/
│       │   │   ├── phoneScreenshots/
│       │   │   └── icon.png
│       └── de-DE/
│           └── ...
```

**Related Commits**:
- #682: Add F-Droid metadata

## GTK Theming

### Dark Mode Detection

**File**: `src/utils/bridge.ts`

```typescript
export const getSystemColorScheme = async (): Promise<'light' | 'dark'> => {
  if (isLinuxPlatform()) {
    return await invoke('plugin:native-bridge|get_system_color_scheme');
  }
  return 'light';
};
```

**Implementation**:
```rust
pub fn get_system_color_scheme() -> Result<String, String> {
    // Check GTK theme
    if let Ok(theme) = std::env::var("GTK_THEME") {
        if theme.contains("dark") {
            return Ok("dark".to_string());
        }
    }

    // Check gsettings
    let output = std::process::Command::new("gsettings")
        .args(["get", "org.gnome.desktop.interface", "gtk-theme"])
        .output();

    if let Ok(output) = output {
        let theme = String::from_utf8_lossy(&output.stdout);
        if theme.contains("dark") {
            return Ok("dark".to_string());
        }
    }

    Ok("light".to_string())
}
```

## Distribution-Specific Notes

### Ubuntu/Debian

**Dependencies**:
```bash
sudo apt install libwebkit2gtk-4.1-dev libgtk-3-dev libayatana-appindicator3-dev
```

### Fedora/RHEL

**Dependencies**:
```bash
sudo dnf install webkit2gtk4.1-devel gtk3-devel libappindicator-gtk3-devel
```

### Arch Linux

**Dependencies**:
```bash
sudo pacman -S webkit2gtk gtk3 libappindicator-gtk3
```

### NixOS

**Configuration** (`flake.nix`):
```nix
{
  description = "Readest ebook reader";

  inputs = {
    nixpkgs.url = "github:NixOS/nixpkgs/nixos-unstable";
    rust-overlay.url = "github:oxalica/rust-overlay";
  };

  outputs = { self, nixpkgs, rust-overlay }:
    let
      system = "x86_64-linux";
      pkgs = import nixpkgs { inherit system; };
    in {
      packages.${system}.default = pkgs.rustPlatform.buildRustPackage {
        pname = "readest";
        version = "0.9.31";

        buildInputs = with pkgs; [
          webkitgtk_4_1
          gtk3
          libappindicator-gtk3
        ];
      };
    };
}
```

## Known Issues

### Issue: Tray Icon Not Showing

**Symptom**: System tray icon invisible on some desktop environments

**Workaround**: Install `libappindicator-gtk3`
```bash
# Ubuntu/Debian
sudo apt install libayatana-appindicator3-1

# Fedora
sudo dnf install libappindicator-gtk3
```

### Issue: Wayland Rendering Issues

**Symptom**: Blurry text or glitches on Wayland

**Workaround**: Force X11 backend
```bash
GDK_BACKEND=x11 readest
```

## Related Documentation

- **[Cross-Platform Support Index](./index.md)** - Overview of all platforms
- **[macOS Platform](./macos.md)** - macOS-specific features
- **[Windows Platform](./windows.md)** - Windows-specific features

---

**Last Updated**: Documentation for commit f4908c45 (February 2025)
**Related Commits**: #682
